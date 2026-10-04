# 用一个网页学会 Argo CD

这个目录是一个练习，只有一个 nginx 网页。你要做的事很具体：集群里跑几份这个网页，由 Git 里的文件决定，Argo CD 负责把集群调成和 Git 一样。

下面先讲它是什么，再讲你怎么把它跑起来。命令都由你自己执行。

## 它是什么

Argo CD 是装在 Kubernetes 里的一个持续交付控制器。它的工作只有一件：反复比较「Git 里写的集群样子」和「集群现在的样子」，发现不一样就按 Git 改集群。

这种做法叫 GitOps。期望状态放在 Git 里。谁想改线上，谁就改 Git、提交、合并。集群本身不接受人手直接改作为正式入口。

和它旁边的工具分工是这样的：

- CI（GitHub Actions、Jenkins 这类）负责构建镜像、跑测试、把新的镜像标签写进 Git。
- Argo CD 负责读 Git，把 Deployment、Service 这些资源应用到集群。
- Kubernetes 负责真正把 Pod 跑起来。

所以 Argo CD 不构建镜像，也不替代 `kubectl`。它是「Git 到集群」这一段的自动对账程序。

## 为什么需要它

没有它的时候，发布通常是某个人在自己电脑上执行 `kubectl apply`。集群里实际跑的是什么，取决于谁最后执行了什么命令。Git 里的文件和集群会慢慢分叉，出了问题也说不清是哪一次手工操作造成的。

有了它之后，Git 提交记录就是发布记录。回滚就是把 Git 回到上一个提交，再同步一次。多个人、多套环境时，每个人看的都是同一份声明，而不是各自电脑上的一次 apply。

## 架构：集群里实际有什么

安装包会在 `argocd` 这个命名空间里起一组服务。你日常只要记住四个：

| 组件 | 它干什么 |
| --- | --- |
| application-controller | 核心。每隔一会儿拉一次 Git，和集群对比，按策略决定要不要同步。 |
| repo-server | 克隆仓库，把目录、Kustomize 或 Helm 渲染成最终的 YAML。 |
| server | 提供网页和 API。你在浏览器里看到的就是它。 |
| redis | 缓存仓库内容和集群状态，避免每次都全量重算。 |

旁边还有 `dex-server`，用来接公司的单点登录。这个练习用本地管理员账号，可以先不管它。

你真正要管的对象只有三个：

1. **Repository**：一个 Git 地址，加上它怎么认证。
2. **Application**：一份「从哪个仓库的哪个目录，部署到哪个集群的哪个命名空间」的声明。这是 Argo CD 的基本单位。
3. **AppProject**：权限边界。规定这个项目能用哪些仓库、能部署到哪些集群和命名空间。练习里用自带的 `default` 项目即可。

每个 Application 有两盏灯，要分开看：

- **Sync**：集群里的 YAML 和 Git 渲染结果是否一致。`Synced` 表示一致，`OutOfSync` 表示有人改了 Git 还没同步，或者有人直接改了集群。
- **Health**：这些资源现在是否工作正常。Deployment 副本没起来时会是 `Progressing` 或 `Degraded`，起来了才是 `Healthy`。

两盏灯独立。文件已经一致但 Pod 起不来，是 Synced + Degraded。Pod 好好的但 Git 改过还没同步，是 OutOfSync + Healthy。

## 这个项目里有什么

`app/` 是要被部署的网页：

- `configmap.yaml`：网页内容，标题是 `hello from git`
- `deployment.yaml`：1 个 nginx 副本，把上面的网页挂进去
- `service.yaml`：集群内访问入口

`gitops/application.yaml` 是交给 Argo CD 的那份声明。里面的仓库地址现在是占位符，推到你自己的 GitHub 之后再改。

先故意不写自动同步。第一次由你在网页上点 Sync，你才能看见「Git 变了」和「集群变了」是两步。

## 动手

当前这台机器上：`kubectl`、`kind`、`helm` 已安装，`argocd` 命令行还没有，Docker 引擎刚才没有在运行。kind 需要 Docker。先打开 Docker Desktop，等到引擎就绪再继续。

### 1. 起一个本地集群

```powershell
kind create cluster --name argocd-learn
kubectl get nodes
```

看到一个 `Ready` 的节点就可以。这是你自己电脑上的集群，和公司的 EKS 无关。

### 2. 安装 Argo CD

```powershell
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

这一步会下载官方清单并创建上面那组组件。等 `argocd-server` 变成 Available。

要用 `--server-side`。清单里的 `ApplicationSet` 定义超过了 Kubernetes 注解 256KB 的上限，普通 `kubectl apply` 会把它漏掉，`argocd-applicationset-controller` 随后会 CrashLoopBackOff。这个控制器只负责批量生成 Application，漏掉它不影响后面的单个应用练习。已经装过一次的话，把上面这条 `kubectl apply` 再执行一遍即可。

### 3. 打开网页并登录

另开一个终端，保持它运行：

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

浏览器打开 `https://localhost:8080`。证书是自签的，继续访问即可。

用户名是 `admin`。初始密码：

```powershell
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

登录后左边是应用列表，右边是同步状态。现在列表是空的，因为还没有 Application。

### 4. 先用官方示例看一次同步

Argo CD 跑在集群里面，它拉不到你 Windows 磁盘上的这个文件夹。在你把自己的仓库推上去之前，先用官方公开示例，确认控制器、网页、同步这条链路是通的。

在网页里点 **NEW APP**，这样填：

- Application Name: `guestbook`
- Project: `default`
- Sync Policy: Manual
- Repository URL: `https://github.com/argoproj/argocd-example-apps.git`
- Revision: `HEAD`
- Path: `guestbook`
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `guestbook`

创建后状态是 OutOfSync。点 **SYNC**，再点 **SYNCHRONIZE**。变成 Synced 和 Healthy 之后，看一下资源树：Deployment、Service、Pod 都挂在这个 Application 下面。这一棵树就是 Argo CD 的管理界面。

### 5. 换成你自己的网页

在 GitHub 新建一个空仓库，例如 `argocd-learn`。在这个目录里：

```powershell
git init
git add app gitops README.md
git commit -m "add hello page for Argo CD practice"
git branch -M main
git remote add origin https://github.com/你的用户名/argocd-learn.git
git push -u origin main
```

把 `gitops/application.yaml` 里的 `repoURL` 改成这个地址，提交并推送。然后：

```powershell
kubectl apply -f gitops/application.yaml
```

回到 Argo CD 网页，会出现 `hello`。它同样是 OutOfSync。点 Sync。`CreateNamespace=true` 会顺手建出 `hello` 命名空间。

确认网页内容：

```powershell
kubectl -n hello port-forward svc/hello 8081:80
```

浏览器打开 `http://localhost:8081`，应看到 `hello from git`。

### 6. 改 Git，看 OutOfSync，再同步

把 `app/deployment.yaml` 的 `replicas: 1` 改成 `2`，把 `app/configmap.yaml` 里的 `replicas: 1` 改成 `replicas: 2`。提交并推送。

网页上点这个应用的 **REFRESH**。状态变成 OutOfSync。点 **DIFF**，能看到副本数和网页文字的差异。再点 Sync。

```powershell
kubectl -n hello get pods
```

应有 2 个 Pod。刷新 `http://localhost:8081`，文字也变成 2。

这一步就是 Argo CD 的全部主循环：Git 是源，Refresh 发现差异，Sync 把差异写进集群。

### 7. 打开自动同步

在网页里编辑 `hello`，把 Sync Policy 改成 Automatic，并勾上 **PRUNE**。PRUNE 的含义是：Git 里删掉的资源，集群里也删掉。

再把副本改回 1，提交推送。这次不要点 Sync。等十几秒到一分钟，应用会自己回到 Synced，Pod 回到 1 个。

自动同步加上自愈（Self Heal，在同一页里）之后，有人用 `kubectl scale` 直接改集群，控制器也会把副本数改回 Git 里的值。可以试一次：

```powershell
kubectl -n hello scale deploy/hello --replicas=5
```

刷新 Argo CD 页面，会看到它把副本拉回去。这就是「集群可以被改，但 Git 说了算」。

### 8. 回滚

打开 `hello` 的 **HISTORY AND ROLLBACK**。每次成功的 Sync 都有一条记录。选上一条，点 Rollback。集群回到那一次同步的内容。

回滚改的是集群里这一次的结果。Git 上最新提交还在。下一次自动同步仍会走向 Git 的最新提交。想让回滚留下来，需要把 Git 也回到对应的提交。

## 管它的时候你在管什么

网页、命令行、`kubectl` 是同一套 API 的三个入口。

- 网页适合看树、看 Diff、点同步、看回滚历史。
- `argocd` 命令行适合脚本。这个练习可以先不装。
- `kubectl apply -f gitops/application.yaml` 是把 Application 本身也放进 Git。以后 Application 多了，常见做法是再做一个「应用的应用」：一个根 Application 指向 `gitops/` 目录，由它创建其他 Application。这个练习只有一个应用，先不用。

日常排查看这四样就够：

1. Sync 是不是 OutOfSync，Diff 里差在哪
2. Health 是不是 Degraded，Pod 事件写了什么
3. repo-server 能不能拉到仓库（私有仓库缺凭据时，应用会停在 Unknown）
4. 这次同步的 History 里，是哪一次提交被应用到了集群

## 这个练习故意没碰的功能

知道名字即可，不必现在做：

- **ApplicationSet**：用一份模板批量生成很多 Application，例如每个集群一份、每个租户一份。
- **AppProject**：把仓库和目标命名空间划开，避免一个应用被部署到不该去的地方。
- **多集群**：一个 Argo CD 管多套 Kubernetes，destination 里写不同的集群地址。
- **Helm / Kustomize**：`source` 里换成 chart 或 kustomize 路径，repo-server 负责渲染。
- **Sync Wave 和 Hook**：控制先建数据库、再跑迁移、最后切流量。
- **通知**：同步成功或失败时发到 Slack 这类渠道。
- **SSO**：用公司账号登录，而不是 admin 密码。

## 做完可以回答的三句话

1. Argo CD 是 Kubernetes 上的对账控制器，Git 里的声明是期望状态。
2. 你管理的基本单位是 Application；要同时看 Sync 和 Health。
3. 发布的正式动作是改 Git。手动 Sync 让你看清差异，自动 Sync 让控制器自己把集群拉齐。

## 做完之后怎么清掉

这个练习占的资源几乎都在一个 kind 集群里。kind 的节点是 Docker 容器，容器里面才是 Kubernetes。删掉这个集群，Argo CD、guestbook、hello 网页、它们的 Pod 和命名空间会一起消失。本目录里的 YAML 还在，GitHub 上的仓库也还在，那两份是文件，不是集群里的东西。

先把开着的端口转发停掉。跑着 `port-forward` 的那个终端按 Ctrl+C。

然后删集群：

```powershell
kind delete cluster --name argocd-learn
```

这条命令会停掉并删除节点容器，同时从 kubeconfig 里拿掉 `kind-argocd-learn` 这个上下文。确认一下：

```powershell
kind get clusters
docker ps --filter "label=io.x-k8s.kind.cluster=argocd-learn"
kubectl config get-contexts
```

`kind get clusters` 里不再出现 `argocd-learn`，`docker ps` 没有这个集群的容器，就清完了。

节点镜像还在 Docker 里，大约占 1GB。集群删了它不会自动删，因为镜像可以被下一个集群继续用。想把磁盘也腾出来再执行：

```powershell
docker rmi kindest/node:v1.34.0 docker.m.daocloud.io/kindest/node:v1.34.0
```

不要用 `docker system prune`。这台机器上还有别的镜像，prune 会把它们一起删掉。

Docker Desktop 不用关，也不用卸。`daemon.json` 里的镜像地址和 Docker Desktop 的内存上限是本机配置，留着不影响别的项目。GitHub 上如果建过 `argocd-learn` 仓库，要删就到网页上删仓库，kind 碰不到它。
