# 会话学习笔记：Kubernetes 与 CKAD 应用部署实践

## 1. 本次学习概述

本次学习目标是以 **CKAD + 应用部署实践** 为主线，建立本地 Kubernetes 学习环境，并开始掌握应用部署、稳定访问、扩缩容和故障自愈。

本次使用 Codex 作为学习助手，采用“理解概念 → 执行小步骤 → 查看输出 → 排查错误 → 复盘”的学习方式。

本次已完成：

- 明确 CKAD 应以应用部署和排障为核心，而不是先学习控制平面运维。
- 建立 Windows Terminal、PowerShell、WSL Ubuntu、Docker Desktop 和本地 Kubernetes 环境。
- 解决 Docker 用户访问 Docker Socket 的权限、Docker Hub 网络和 Docker Desktop WSL Integration 问题。
- 启用并确认 Docker Desktop 内置 Kubernetes 集群。
- 观察已有 Nginx 的 Deployment、Pod、Service、Endpoints 和系统组件。
- 通过 Port Forward 从本机访问 Kubernetes 中的 Nginx。
- 理解 Watch、Deployment 自愈和 Service 稳定访问。

当前进度：Docker Desktop Kubernetes 集群已可用；当前上下文为 docker-desktop，节点 desktop-control-plane 为 Ready。下一步是编写并部署自己的 web-demo Deployment YAML。

## 2. 我提出的所有问题

1. 如何系统性学习 Kubernetes？
2. 以 CKAD 和应用部署实践为目标时，如何安排学习？
3. 能否将所需知识点写成文档？
4. 不会 Linux、终端、进程、端口和环境变量时应该怎么办？
5. Kubernetes 应该在哪种环境练习，是否需要 VMware？
6. 能否用阿里云 4 台服务器搭建 Kubernetes 集群？
7. 本地连接一台阿里云服务器作为控制平面、管理另外 3 台是否可行？
8. 如何一步一步搭建本地与云上学习环境？
9. Windows 下该使用 CMD 还是 PowerShell？
10. Windows Terminal 中如何切换磁盘、进入目录和创建目录？
11. 如何安装 WSL Ubuntu，并确认自己已进入 Ubuntu？
12. Apt 更新命令各部分是什么意思，下载很慢时如何处理？
13. Docker 无权限访问 Docker Socket 时如何处理？
14. Docker 拉取 Docker Hub 镜像出现 EOF 时如何处理？
15. Docker Engine JSON 中应在哪里添加镜像加速器？
16. Docker Desktop 的 Ubuntu WSL Integration 异常时如何排查？
17. Docker Desktop 是否内置 Kubernetes，是否还需要 Kind？
18. Kubernetes 中带 Watch 和不带 Watch 的区别是什么？
19. 为什么删除 Pod 后 Kubernetes 会自动创建新 Pod？
20. Service、Endpoints 和 Pod IP 的关系是什么？
21. 如何自己编写第一份 Deployment YAML？

## 3. 核心知识点汇总

## 3.1 Kubernetes 的学习方式

学习 Kubernetes 应形成以下闭环：

```text
理解概念 → 编写 YAML → 部署资源 → 验证状态 → 制造故障 → 排查问题 → 记录复盘
```

CKAD 的重点是 **在 Kubernetes 中交付和维护应用**，而不是优先搭建高可用控制平面。

重点学习对象：

- Pod、Deployment、ReplicaSet。
- Service、Ingress、服务发现。
- ConfigMap、Secret。
- Probe、资源 requests 和 limits。
- Volume、PVC、StatefulSet。
- Job、CronJob。
- ServiceAccount、RBAC、NetworkPolicy。
- 日志、事件、故障排查。
- Kustomize、Helm。

当前不优先学习：etcd 备份、高可用控制平面、集群升级、深度 CNI、服务网格和 Operator 开发。

## 3.2 学习环境分工

```text
Windows Terminal
├─ PowerShell：Docker、Kubectl、Kubernetes 资源管理
└─ WSL Ubuntu：Linux 命令、Shell、容器内部排障

Docker Desktop
└─ 内置 Kubernetes：本地 K8s 练习集群
```

| 环境 | 主要用途 |
| --- | --- |
| PowerShell | 执行 Docker、Kubectl 和 Kubernetes 管理命令 |
| WSL Ubuntu | 学 Linux、网络、进程、日志和容器内部排障 |
| Docker Desktop | 提供容器运行环境和本地 Kubernetes 集群 |
| VS Code / Notepad | 编写 YAML 文件 |
| 阿里云 ECS | 后续多节点 Kubernetes 和完整项目实战 |

当前阶段不需要 VMware。后续学习 Kubeadm、控制平面运维、高可用或 CKA 时再使用。

## 3.3 PowerShell、WSL 与 Ubuntu

推荐在 Windows Terminal 中使用 **PowerShell**，无需优先学习 CMD。

- PowerShell 适合 Docker、Kubectl、文件操作和自动化脚本。
- WSL 是 Windows Subsystem for Linux，可在 Windows 中直接运行 Linux。
- Ubuntu 用于练习 Linux 命令、网络、进程和容器内部排障。

Ubuntu 提示符：

```text
hjw-243@DESKTOP-B8V27R6:/mnt/d$
```

- hjw-243 是 Linux 用户名。
- DESKTOP-B8V27R6 是电脑名。
- /mnt/d 是 Windows D 盘在 Ubuntu 中的挂载路径。
- $ 表示当前是普通用户。

磁盘映射：

```text
C:\ 对应 /mnt/c
D:\ 对应 /mnt/d
```

## 3.4 Linux 基础

| 目标 | PowerShell | Linux |
| --- | --- | --- |
| 查看当前位置 | Get-Location | pwd |
| 查看目录 | ls | ls -la |
| 进入目录 | cd | cd |
| 查看文件 | Get-Content | cat |
| 创建目录 | mkdir | mkdir |

核心概念：

- **进程**：正在运行的程序；Linux 可用 ps 查看。
- **端口**：网络服务入口，例如 80、443、8080。
- **环境变量**：程序启动时读取的配置。
- **标准输出/错误输出**：容器日志主要来自这里。
- **容器主进程**：主进程退出后，容器通常退出。

常用 Linux 排查命令：

```bash
ps aux
ss -lntp
rg "error" .
curl http://localhost:8080
```

## 3.5 Apt、Docker 与 WSL Integration

Apt 命令：

```bash
sudo apt update && sudo apt upgrade -y
```

- sudo：管理员权限。
- apt：Ubuntu 软件包管理工具。
- update：刷新软件包清单，不升级软件。
- upgrade：升级已安装软件。
- -y：自动确认。
- &&：前一条成功后才执行后一条。

正确写法：

```bash
sudo apt upgrade -y
```

软件下载极慢通常是软件源网络问题，不是电脑性能问题。下载阶段可按 Ctrl + C 停止，再考虑使用国内镜像源。

Docker Desktop 可通过 WSL Integration 向 Ubuntu 提供 Docker 命令。正常时，Docker Version 输出应同时包含 Client 和 Server：

```bash
docker version
```

Docker Socket 权限问题通常是当前用户不在 docker 用户组。应将用户加入该组并重启 WSL 会话，而不是长期使用 sudo docker。

Docker Hub 拉取镜像时出现 EOF，说明连接 Docker Hub 时中断。可检查网络、重启 Docker Desktop，必要时配置阿里云 ACR 专属镜像加速器。配置镜像加速器前应先保证 WSL Integration 稳定。

## 3.6 Docker Desktop 内置 Kubernetes

Docker Desktop 可直接启用 Kubernetes；当前阶段无需立即安装 Kind。

```text
Docker Desktop：容器运行环境和本地开发工具
Kubernetes：Docker Desktop 可选启用的容器编排集群
Kind：基于 Docker 快速创建 Kubernetes 集群的工具
```

当前策略：

- 使用 Docker Desktop 内置 Kubernetes 练习 Pod、Deployment、Service、日志和排障。
- 后续需要高频重建或本地多节点模拟时再使用 Kind。
- 本地练习成熟后，用 4 台阿里云 ECS 搭建 1 控制平面 + 3 Worker 的 K3s 集群。

## 3.7 Pod、Deployment、ReplicaSet 与自愈

```text
Deployment
→ ReplicaSet
→ 多个 Pod
```

- Pod 是最小部署单位，但名称和 IP 不稳定。
- Deployment 声明目标状态，例如运行 3 个或 4 个副本。
- ReplicaSet 负责让实际 Pod 数量接近期望副本数。
- 删除 Pod 不等于删除 Deployment；Deployment 会补充名字不同的新 Pod。

控制器逻辑：

```text
期望状态：4 个 Pod
实际状态：3 个 Pod
结果：ReplicaSet 创建 1 个新 Pod
```

| 操作 | 结果 |
| --- | --- |
| 删除 Pod | Deployment 自动补充新 Pod |
| 将副本数设为 0 | 不再维持运行中的 Pod |
| 删除 Deployment | 它管理的 Pod 被删除且不再重建 |

## 3.8 Watch、Service 与 EndpointSlice

普通查询只显示当前状态：

```powershell
kubectl get pods
```

Watch 持续输出资源变化：

```powershell
kubectl get pods -w
```

- -w 是 --watch 的缩写。
- Ctrl + C 只停止本地观察，不会停止集群中的 Pod。

Service 为一组 Pod 提供稳定访问入口：

```text
客户端
→ Service（稳定名称和 ClusterIP）
→ Label Selector
→ 多个会变化的 Pod IP
```

Service Selector 必须匹配 Pod Labels：

```yaml
selector:
  app: nginx
```

```yaml
labels:
  app: nginx
```

当前 Nginx Service 名称是 nginx，Selector 是 app=nginx，后端包含 3 个 Nginx Pod。新版 Kubernetes 中 Endpoints 已逐步弃用，应优先使用 EndpointSlice。

## 3.9 Deployment YAML

Kubernetes YAML 的核心结构：

```text
apiVersion：资源 API 版本
kind：资源类型
metadata：名称、标签等元信息
spec：资源的期望状态
selector：Deployment 管理哪些 Pod
template：Pod 模板
containers：容器定义
```

重要规则：Deployment 的 Selector 必须匹配 Pod Template 的 Labels。YAML 只能用空格缩进，不能使用 Tab。

## 3.10 云上 4 台 ECS 规划

```text
本地电脑
└─ Kubectl / SSH
   └─ ECS-1：K3s Server / 控制平面
      ├─ ECS-2：Worker-1
      ├─ ECS-3：Worker-2
      └─ ECS-4：Worker-3
```

- 4 台 ECS 最好位于同一地域和同一 VPC。
- 节点间应通过私网通信。
- 本地电脑是远程管理端，不承担控制平面。
- 初期使用 K3s，后续再深入 Kubeadm。
- 安全组仅开放必要端口，不把 Kubernetes 管理端口完全暴露到公网。

## 4. 示例代码 / 实操命令

## 4.1 PowerShell 目录操作

```powershell
D:
cd D:\k8s-labs
Get-Location
mkdir D:\k8s-labs
```

## 4.2 WSL 与 Ubuntu

```powershell
wsl --status
wsl -l -v
wsl --install -d Ubuntu
wsl --set-default Ubuntu
wsl
```

```bash
sudo apt update
sudo apt install -y curl ca-certificates dnsutils iproute2 procps ripgrep
which curl rg ps ss
```

## 4.3 Docker 与 Ubuntu 集成

```bash
docker version
groups
sudo usermod -aG docker $USER
exit
```

```powershell
wsl --shutdown
```

```bash
docker run --rm hello-world
curl -I --connect-timeout 10 https://registry-1.docker.io/v2/
```

## 4.4 Kubernetes 查看与排查

```powershell
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A
kubectl get deployment,rs,pods,service
kubectl describe deployment nginx
kubectl logs deployment/nginx
kubectl get events --sort-by=.metadata.creationTimestamp
```

## 4.5 Nginx 扩缩容、自愈与访问

```powershell
kubectl scale deployment/nginx --replicas=4
kubectl get pods -w
kubectl delete pod <Pod名称>
kubectl scale deployment/nginx --replicas=3
```

```powershell
kubectl get service
kubectl describe service nginx
kubectl get endpoints nginx
kubectl get endpointslice -l kubernetes.io/service-name=nginx
kubectl port-forward service/nginx 8080:80
```

```powershell
curl.exe http://localhost:8080
```

## 4.6 第一个自定义 Deployment YAML

```powershell
cd D:\k8s-labs
notepad web-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  labels:
    app: web-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27.5-alpine
          ports:
            - containerPort: 80
```

```powershell
kubectl apply --dry-run=client -f .\web-deployment.yaml
kubectl apply -f .\web-deployment.yaml
kubectl get deployment,pods -l app=web-demo -w
kubectl describe deployment web-demo
```

## 5. 操作流程与步骤清单

1. 使用 Windows Terminal，并以 PowerShell 作为 Windows 侧主终端。
2. 安装 WSL2 和 Ubuntu，创建 Linux 用户名和密码。
3. 在 Ubuntu 中安装 Curl、Ripgrep、进程工具和网络工具。
4. 安装并启动 Docker Desktop，启用 Ubuntu 的 WSL Integration。
5. 将 Ubuntu 用户加入 docker 用户组，关闭 WSL 后重新进入。
6. 使用 Hello World 验证 Docker Client、Server 与镜像拉取。
7. 在 Docker Desktop 中启用 Kubernetes。
8. 用 Kubectl 确认上下文、节点和系统 Pod 正常。
9. 查看 Nginx Deployment、Pod、Service、Endpoints 和 EndpointSlice。
10. 使用 Port Forward 从本机访问 Nginx Service。
11. 扩容 Nginx Deployment，观察新 Pod 创建。
12. 删除一个 Pod，观察 Deployment 自动恢复副本数。
13. 将副本数恢复到原先数量。
14. 在 D 盘练习目录创建 web-demo Deployment YAML。
15. 使用 Dry Run 校验 YAML，再正式部署。
16. 为 web-demo 编写 Service YAML，并通过 Service 访问。
17. 学习配置、密钥、探针、资源限制、存储、Ingress 和排障。
18. 本地练习稳定后，使用 4 台阿里云 ECS 搭建 K3s 多节点集群。

## 6. 坑点、易错点、重要提醒

- 推荐 PowerShell；PowerShell 中要使用真正的 Curl 时使用 curl.exe。
- 目录不存在时必须先创建，不能直接进入。
- Linux 输入密码时不显示字符属于正常现象。
- Apt 正确参数是 -y，不是 - y；upgrade 不能拼错。
- 软件源下载慢时优先检查网络或镜像源。
- Docker Socket 权限应通过用户组解决，不建议长期使用 sudo docker。
- Docker Hub 网络不通时，Kind 同样无法拉取创建集群所需镜像。
- Docker Desktop WSL Integration 异常时，先重启 Docker Desktop 和 WSL，不要直接恢复出厂设置。
- Docker Engine 配置必须是合法 JSON，逗号错误会导致配置无法应用。
- Kubectl 参数区分大小写，-A 与 -a 含义不同。
- Pod 不稳定，应用不应依赖 Pod IP；Service 才是稳定访问入口。
- Service Selector 必须匹配 Pod Labels，否则没有可用后端。
- 删除 Pod 不等于删除 Deployment，因此会被自动补回。
- Ctrl + C 停止 Watch 或 Port Forward，只停止本地命令。
- Port Forward 会占用当前终端窗口。
- Endpoints 正逐步弃用，应优先了解 EndpointSlice。
- YAML 只能使用空格缩进，不能使用 Tab。
- Deployment Selector 与 Pod Template Labels 必须一致。

## 7. 总结与后续学习计划

本次已完成 Kubernetes 学习的基础环境建设，并理解了应用部署中最关键的关系：

```text
Deployment
→ ReplicaSet
→ Pod

Service
→ Label Selector
→ 多个 Pod
```

后续按以下顺序学习：

1. 完成 web-demo Deployment YAML 的部署和验证。
2. 手写 web-demo Service YAML，理解 Port、TargetPort 与 Selector。
3. 学习 ConfigMap 和 Secret，实现配置外置。
4. 学习 Startup、Liveness、Readiness Probe。
5. 学习 Requests 与 Limits。
6. 学习 PVC、Job、CronJob 和 Ingress。
7. 练习 Pending、ImagePullBackOff、CrashLoopBackOff、Service 不通等排障场景。
8. 学习 Kustomize 和 Helm。
9. 使用阿里云 4 台 ECS 完成多节点 K3s 集群和完整项目部署。
10. 开始 CKAD 限时实操训练，持续积累 YAML 模板和故障复盘库。
+
---

# 会话学习笔记：后续实践追加复盘

## 1. 本次学习概述

本追加部分记录了本次会话后半段的 Kubernetes 实操学习。学习目标从“本地环境可用”推进到“可部署、可配置、可排障、可扩缩容的应用”。

本阶段学习并实践了：

- 手写 Deployment、Service、ConfigMap、Secret、PVC、Job、CronJob、Namespace、RBAC、Ingress、HPA 等 YAML。
- 使用 VS Code 替代记事本编辑 Kubernetes 配置。
- 排查 ImagePullBackOff，并通过 Events 定位 Docker Desktop 镜像代理返回 500 的问题。
- 使用 ConfigMap、Secret、Probe、资源限制和 PVC 完善应用。
- 理解 Init Container、Sidecar、ResourceQuota、LimitRange、Kustomize 和 Helm 的定位。
- 规划从本地 Docker Desktop Kubernetes 迁移到阿里云 4 节点 K3s 集群的前置检查。

## 2. 我提出的所有问题

1. 自定义 Deployment 创建成功后，为什么同时 Watch Deployment 和 Pod 会报错？
2. Pod 处于 ImagePullBackOff 时，应该如何定位具体原因？
3. PowerShell 中 JsonPath 命令为什么会因反斜杠报错？
4. ConfigMap 如何注入 Deployment，修改 ConfigMap 后为什么旧值仍然存在？
5. Secret 与 ConfigMap 的区别是什么，如何注入 Pod？
6. Startup、Liveness、Readiness Probe 分别负责什么？
7. Resources 中的 Request、Limit 分别有什么作用？
8. PVC 如何挂载到容器，并验证 Pod 重建后数据是否仍存在？
9. Job 与 CronJob 分别适合哪些任务？
10. Namespace 如何隔离资源，Service DNS 如何跨 Namespace 访问？
11. ServiceAccount、Role、RoleBinding 如何实现最小权限？
12. Init Container、Sidecar、emptyDir 分别适合什么场景？
13. Kustomize 与 Helm 的区别是什么？
14. Ingress Controller 为什么是 Ingress 生效的前提？
15. HPA 为什么依赖 Metrics Server？
16. 本地阶段结束后，阿里云 4 台 ECS 应如何做集群盘点？

## 3. 核心知识点汇总

## 3.1 Deployment 创建与 ImagePullBackOff 排障

Deployment 创建成功不代表容器一定能启动。完整过程如下：

```
理解 YAML
→ Deployment 创建
→ ReplicaSet 创建 Pod
→ Scheduler 分配 Node
→ Kubelet 拉取镜像
→ 容器启动
→ Probe 通过
→ Pod Ready
```

本次出现的状态：

- **ErrImagePull**：某次镜像拉取失败。
- **ImagePullBackOff**：Kubernetes 以逐渐延长的间隔继续重试拉取镜像。
- **Pending**：Pod 尚未完全就绪，可能卡在调度、镜像、卷挂载或初始化。
- **Running**：容器正在运行，不一定代表可接收 Service 流量。
- **Ready**：容器通过 Readiness Probe，可接收 Service 流量。

本次 Events 显示 Docker Desktop 本地镜像代理返回 500。根因属于镜像拉取链路，不是 Deployment 的 YAML 结构。应根据 Events 判断原因，而不是盲目修改镜像名称。

## 3.2 ConfigMap 与 Secret

### ConfigMap

ConfigMap 用于保存非敏感配置：

- 运行环境。
- 日志级别。
- 功能开关。
- 普通配置文件内容。

以环境变量注入时，值在容器启动时读取。后续修改 ConfigMap 不会自动更新旧容器中的环境变量；应通过滚动重启等方式创建新 Pod。

### Secret

Secret 用于保存敏感配置：

- 数据库密码。
- API Key、Token。
- TLS 证书。
- 私有镜像仓库凭据。

Secret 默认是 Base64 编码，并不等于默认加密。生产环境还需要 RBAC、静态加密和专业密钥管理方案。

## 3.3 Probe 健康检查

```
startupProbe：应用是否完成启动
livenessProbe：应用是否仍然存活；失败会重启容器
readinessProbe：应用是否可以接收流量；失败时从 Service 后端摘除
```

- Readiness 失败通常不会重启容器。
- Liveness 失败会让 Kubelet 重启容器。
- Startup Probe 成功前，Liveness 和 Readiness 的检查会被延后。
- 错误的 Probe 路径或端口会导致发布卡住、Pod 不 Ready 或反复重启。

## 3.4 Resources、ResourceQuota 与 LimitRange

### Requests 与 Limits

```
requests：调度器为 Pod 预留的最小资源
limits：容器可使用的最大资源
```

- CPU 可用毫核表示，例如 100m 等于 0.1 核。
- 内存可用 Mi 表示，例如 128Mi。
- CPU 超过 Limit 时通常被限速。
- 内存超过 Limit 时可能出现 OOMKilled。
- Request 太大且节点资源不足时，Pod 会变成 Pending。

### ResourceQuota 与 LimitRange

- ResourceQuota 限制一个 Namespace 的总体 Pod 数量、Requests 和 Limits。
- LimitRange 约束单个容器的最小、最大和默认资源。
- LimitRange 只影响新创建的 Pod；已有 Pod 需重建后才能获得默认资源。

## 3.5 存储、Job、CronJob 与多容器 Pod

### PVC 与 emptyDir

```
PVC：持久化数据，独立于 Pod 生命周期
emptyDir：Pod 内临时共享目录，Pod 删除后数据消失
```

本地单节点集群中，多副本共享一个 ReadWriteOnce PVC 可以工作；多节点生产环境不能默认照搬这种存储设计。

### Job 与 CronJob

- Job 适合一次性迁移、备份、报表生成等任务。
- CronJob 按计划创建 Job，适合定时清理、备份和同步。
- Job 成功后 Pod 状态为 Completed，不会像 Deployment 一样持续运行。
- CronJob 应设置并发策略与历史记录数量，避免任务堆积。

### Init Container 与 Sidecar

- Init Container 必须先成功退出，主容器才会启动；适合初始化目录、等待依赖和前置迁移。
- Sidecar 与主容器位于同一 Pod，可共享网络与 Volume；适合日志采集、代理、证书刷新和辅助任务。

## 3.6 Namespace、Service DNS 与 RBAC

### Namespace 与 Service DNS

Namespace 用于逻辑隔离。不同 Namespace 中可创建同名资源。

```
同 Namespace：<service 名称>
跨 Namespace：<service 名称>.<namespace 名称>
完整名称：<service 名称>.<namespace 名称>.svc.cluster.local
```

应用应使用 Service DNS，而不是依赖会变化的 Pod IP。

### RBAC

```
ServiceAccount：Pod 使用的身份
Role：该身份在 Namespace 内允许的操作
RoleBinding：将 Role 授予 ServiceAccount
```

最小权限原则要求只授予应用实际所需权限，例如只允许读取 Pod，不允许删除 Pod。

## 3.7 Kustomize、Helm、Ingress 与 HPA

- **Kustomize**：维护自己应用的通用配置与环境差异；适合 base 与 overlays。
- **Helm**：安装复杂第三方组件；适合 Ingress NGINX、Metrics Server、Prometheus 等。
- **Ingress**：只定义路由规则，必须部署 Ingress Controller 才会生效。
- **HPA**：依据资源指标扩缩容 Deployment；使用 CPU 利用率时，容器必须声明 Requests，集群必须有 Metrics Server。

## 3.8 阿里云 4 节点 K3s 规划

```
本地电脑
└─ Kubectl / SSH
   └─ cp-1：K3s Server / 控制平面
      ├─ worker-1：K3s Agent
      ├─ worker-2：K3s Agent
      └─ worker-3：K3s Agent
```

前置要求：

- 4 台 ECS 位于同一地域和同一 VPC。
- 节点之间通过私网 IP 通信。
- 本地电脑仅远程管理控制平面。
- 先盘点主机信息和安全组，再执行安装。
- 不共享密码、SSH 私钥、K3s Token 或完整 kubeconfig。

## 4. 示例代码 / 实操命令

### Deployment、Events 与镜像排障

```powershell
kubectl get pods -l app=web-demo -w
kubectl describe pod <Pod名称>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe deployment nginx | Select-String -Pattern 'Image:'
```

### ConfigMap、Secret、Probe 与 Resources

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-demo-config
data:
  APP_ENV: dev
  LOG_LEVEL: info
  WELCOME_MESSAGE: Hello from ConfigMap
---
apiVersion: v1
kind: Secret
metadata:
  name: web-demo-secret
type: Opaque
stringData:
  DEMO_API_KEY: demo-key-for-learning-only
```

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 15
  periodSeconds: 2
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 2
  periodSeconds: 5
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
```

```powershell
kubectl apply -f .web-configmap.yaml
kubectl apply -f .web-secret.yaml
kubectl rollout restart deployment/web-demo
kubectl rollout status deployment/web-demo
kubectl exec deployment/web-demo -- sh -c 'echo $APP_ENV; echo $LOG_LEVEL; echo $DEMO_API_KEY'
```

### PVC、Job 与 CronJob

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-demo-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```powershell
kubectl apply -f .web-demo-pvc.yaml
kubectl get pvc web-demo-data -w
kubectl exec deployment/web-demo -- sh -c 'date > /data/pvc-proof.txt; cat /data/pvc-proof.txt'
kubectl delete pod <Pod名称>
kubectl exec deployment/web-demo -- cat /data/pvc-proof.txt
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: web-demo-report
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: report
          image: nginx:1.27
          command:
            - sh
            - -c
            - |
              echo "web-demo report job started"
              date
              echo "web-demo report job completed"
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: web-demo-heartbeat
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: heartbeat
              image: nginx:1.27
              command:
                - sh
                - -c
                - |
                  echo "heartbeat started"
                  date
                  echo "heartbeat completed"
```

```powershell
kubectl apply -f .web-demo-job.yaml
kubectl logs job/web-demo-report
kubectl apply -f .web-demo-cronjob.yaml
kubectl create job web-demo-heartbeat-manual --from=cronjob/web-demo-heartbeat
kubectl logs job/web-demo-heartbeat-manual
kubectl delete job web-demo-heartbeat-manual
kubectl delete cronjob web-demo-heartbeat
```

### Namespace、RBAC、Kustomize、Helm 与 HPA

```powershell
kubectl apply -f .
amespace.yaml
kubectl config set-context --current --namespace=ckad-lab
kubectl config set-context --current --namespace=default
kubectl exec deployment/web-demo -n default -- sh -c 'getent hosts web-demo.ckad-lab'
```

```powershell
kubectl apply -n ckad-lab -f .web-demo-rbac.yaml
kubectl auth can-i list pods --as=system:serviceaccount:ckad-lab:web-demo-reader -n ckad-lab
kubectl auth can-i delete pods --as=system:serviceaccount:ckad-lab:web-demo-reader -n ckad-lab
kubectl kustomize .kustomizeoverlaysckad-lab
kubectl apply -k .kustomizeoverlaysckad-lab
```

```powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.service.type=NodePort
kubectl top nodes
kubectl top pods -n ckad-lab
kubectl get hpa web-demo -n ckad-lab -w
```

## 5. 操作流程与步骤清单

1. 使用 VS Code 创建和维护 YAML，不再使用记事本。
2. 使用 Dry Run 校验 Deployment、ConfigMap、Secret、Service、PVC、Job 等资源。
3. Pod 异常时使用 Describe、Events 和 Logs 定位根因。
4. 创建 ConfigMap 和 Secret，并通过环境变量注入容器。
5. 修改 ConfigMap 后通过滚动重启使环境变量生效。
6. 配置 Probe、Requests 和 Limits，确认 Pod 最终 Ready。
7. 创建 PVC、挂载目录、写入文件并删除 Pod 验证持久化。
8. 创建 Job、CronJob，查看任务状态与日志，并清理定时任务。
9. 创建 ckad-lab Namespace 并部署独立资源。
10. 创建 ServiceAccount、Role、RoleBinding，并验证权限。
11. 配置 Init Container、emptyDir 和 Sidecar。
12. 使用 Kustomize 分离通用资源和 ckad-lab 环境差异。
13. 使用 Helm 安装 Ingress Controller 和 Metrics Server。
14. 创建 Ingress、Quota、LimitRange 与 HPA。
15. 完成本地验证后，盘点阿里云 4 台 ECS，准备 K3s 多节点集群。

## 6. 坑点、易错点、重要提醒

- Deployment 创建成功不代表 Pod 已启动成功，必须看状态、Events 和日志。
- Watch 不能同时持续观察多个资源类型，应分别 Watch。
- PowerShell 对 JsonPath 的引号和反斜杠容易产生转义问题；初学排障优先用 Describe。
- ImagePullBackOff 应通过 Events 定位根因。
- 不要把真实密码、Token 或私钥写进 YAML 或提交到 Git。
- Secret 默认 Base64 编码，不是默认加密。
- ConfigMap 以环境变量注入后，更新不会自动进入旧容器。
- Readiness 失败不等于 Liveness 失败；错误 Liveness 可能造成重启循环。
- PVC 是持久化存储，emptyDir 是 Pod 生命周期内的临时存储。
- CronJob 实验结束后应删除，避免持续创建 Job。
- Namespace 是逻辑隔离，不等于权限隔离和网络隔离。
- Service DNS 是稳定入口，Pod IP 不应写入业务配置。
- LimitRange 只影响新建 Pod。
- HPA 依赖 Metrics Server；无流量时不扩容属于正常现象。
- 压力测试必须限时、限范围，实验后恢复副本数或删除 HPA。
- Ingress 没有 Ingress Controller 时不会生效。
- Docker Desktop 镜像代理未恢复时，安装 Controller 或 Metrics Server 可能再次出现 ImagePullBackOff。
- 云上部署前先盘点节点与安全组，绝不共享密码、私钥、Token 或完整 kubeconfig。

## 7. 总结与后续学习计划

本阶段已从基础环境搭建进入应用交付实操，覆盖工作负载、配置、安全、存储、网络入口、资源治理和自动扩缩容。

最重要的收获：

```
应用不是“Pod 跑起来”就结束，
而是需要：
Deployment + Service + 配置 + Secret + Probe + 资源限制 + 存储 + 权限 + 发布策略 + 排障能力
```

后续学习方向：

1. 完成并验证 Kustomize 的 base 与 overlays 结构。
2. 验证 Ingress NGINX Controller、Metrics Server 和 HPA 是否正常运行。
3. 建立至少 10 个故障场景的排障清单。
4. 使用阿里云 4 台 ECS 搭建 K3s 多节点集群。
5. 将 web-demo 扩展为前端、后端、数据库组成的完整应用。
6. 为云上应用添加私有镜像仓库、TLS、日志、监控和 GitOps。
7. 开始 CKAD 限时题训练，沉淀 YAML 模板、命令速查表和错题复盘库。
