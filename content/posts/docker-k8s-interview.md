---
title: "Docker 与 Kubernetes 高频面试题全解析"
date: 2026-08-31
tags: ["Docker", "Kubernetes", "K8s", "云原生", "容器", "面试", "DevOps"]
showToc: true
---

> 本文整理了 Docker 与 Kubernetes 在面试中被高频提问的知识点，每个问题独立成节，答案配合适当拓展，帮助系统备战云原生方向的面试。内容综合自博客园、CSDN、小林 coding 等多个技术社区的面试题集，以及《云原生架构与 GitOps 实战》课程知识。

---

## 一、Docker 篇

### 1. 什么是 Docker？它的工作原理是什么？

**答：** Docker 是一个基于 C/S（客户端-服务端）架构的容器引擎。其核心组件包括：

- **Docker Client**：用户命令行接口，发送指令给 Daemon。
- **Docker Daemon**：运行在宿主机上的守护进程，负责管理镜像、容器、网络、存储等。
- **Docker Image**：只读的应用模板，包含运行环境及代码。
- **Docker Container**：镜像的运行实例，拥有自己的进程空间和文件系统。

Docker 底层依赖 Linux 内核的三大特性：

| 特性 | 作用 |
|------|------|
| **Namespace（命名空间）** | 实现进程隔离（PID、Network、Mount、UTS、IPC、User） |
| **Cgroups（控制组）** | 限制和记录进程组所使用的物理资源（CPU、内存等） |
| **UnionFS（联合文件系统）** | 将多个目录叠加挂载到同一个虚拟文件系统，实现镜像分层 |

> **拓展：** Docker 从 1.11 开始使用 `containerd` 作为高层容器运行时，负责容器生命周期管理；`runc` 作为低层运行时，负责创建实际的容器进程。Docker 本身已逐步从 K8s 的默认运行时中移除，取而代之的是 `containerd` 和 `CRI-O`。

---

### 2. Docker 和虚拟机（VM）有什么区别？

**答：**

| 对比维度 | Docker 容器 | 虚拟机 |
|----------|------------|--------|
| **启动速度** | 秒级 | 分钟级 |
| **资源占用** | MB 级别 | GB 级别 |
| **操作系统** | 共享宿主机内核 | 每个 VM 有独立 OS 内核 |
| **隔离性** | 进程级隔离（较弱） | 硬件级隔离（较强） |
| **性能** | 接近原生 | 有 Hypervisor 开销 |
| **密度** | 单机可运行数百个容器 | 单机通常几十个 VM |
| **镜像大小** | 通常 MB 级 | 通常 GB 级 |

**为什么 CentOS 的 Docker 镜像只有几十 MB？** 因为所有容器共享宿主机的 Linux 内核，镜像只需要提供一个最小的 rootfs（基础文件系统），不需要包含完整的操作系统内核。

> **拓展：** 如果对隔离性要求较高（如多租户场景），可以考虑使用 **gVisor**（Google 出品的用户态内核）或 **Kata Containers**（轻量级 VM + 容器），它们在保持容器级启动速度的同时提供接近 VM 的隔离强度。

---

### 3. Docker 镜像分层机制是什么？

**答：** Docker 镜像由多个只读层（Layer）堆叠而成，每一层对应 Dockerfile 中的一条指令。底层使用 **UnionFS**（如 OverlayFS、AUFS）实现分层合并。

```
┌─────────────────────────┐
│   可写层（Container）    │  ← 容器运行时添加
├─────────────────────────┤
│   Layer 3: CMD/ENTRYPOINT│  ← 只读
├─────────────────────────┤
│   Layer 2: COPY 文件     │  ← 只读
├─────────────────────────┤
│   Layer 1: FROM ubuntu   │  ← 只读（Base Image）
└─────────────────────────┘
```

分层的核心优势：
1. **资源共享**：多个镜像可以共享相同的底层（如 ubuntu base），节省磁盘和传输带宽。
2. **构建缓存**：Dockerfile 中某一层未变化时，直接使用缓存，加速构建。
3. **增量传输**：拉取镜像时只需下载差异层。

> **拓展：** 分层也带来一个问题——层数过多会导致镜像膨胀。实践中常使用**多阶段构建（Multi-stage Build）**来减小最终镜像体积，只保留运行时必要的文件。

---

### 4. 什么是 Copy-on-Write（写时复制）？

**答：** Copy-on-Write（CoW）是 Docker 容器对文件系统的操作策略：

- 镜像层是**只读**的。
- 容器启动时，在镜像顶部加载一个**可写层**。
- 当容器需要**修改**某个文件时，Docker 会先将该文件从只读层**复制**到可写层，再在可写层上进行修改。
- 当容器**新建**文件时，直接写入可写层。
- 当容器**删除**文件时，在可写层设置一个 "whiteout" 标记，遮蔽下层的文件。

这意味着：如果多个容器共享同一镜像，读操作直接走底层共享；只有写操作才会产生独立的副本。

> **拓展：** OverlayFS（overlay2 驱动）是 Docker 当前推荐的存储驱动，相比早期的 AUFS 和 DeviceMapper，它在性能和稳定性上更优。可通过 `docker info` 查看当前使用的存储驱动。

---

### 5. Dockerfile 中常用的指令有哪些？

**答：**

| 指令 | 说明 |
|------|------|
| `FROM` | 指定基础镜像，每个 Dockerfile 必须以 FROM 开头 |
| `RUN` | 在构建时执行命令（shell 格式或 exec 格式），每条 RUN 产生一个新层 |
| `COPY` | 将宿主机文件复制到镜像中 |
| `ADD` | 类似 COPY，但支持自动解压 tar 包和支持远程 URL |
| `ENV` | 设置环境变量 |
| `EXPOSE` | 声明容器监听的端口（仅作文档说明，不实际映射） |
| `WORKDIR` | 设置工作目录 |
| `CMD` | 容器启动时执行的默认命令，可被 `docker run` 参数覆盖 |
| `ENTRYPOINT` | 容器启动时执行的入口命令，不会被覆盖（除非 `--entrypoint`） |
| `VOLUME` | 声明匿名挂载点 |
| `ARG` | 构建时变量（仅在构建阶段可用） |
| `LABEL` | 给镜像添加元数据标签 |

**CMD 与 ENTRYPOINT 的区别：**
- `CMD`：可以被 `docker run` 后面的参数完全覆盖。
- `ENTRYPOINT`：不会被覆盖，`docker run` 的参数会追加到 ENTRYPOINT 之后。
- 常见组合：`ENTRYPOINT` 定义固定命令，`CMD` 提供默认参数。

> **拓展：** `RUN` 指令有多种写法，多行 RUN 可以用 `&&` 合并为一条来减少镜像层数：
> ```dockerfile
> RUN apt-get update && apt-get install -y \
>     curl \
>     vim \
>     && rm -rf /var/lib/apt/lists/*
> ```

---

### 6. Docker 有哪些网络模式？

**答：**

| 网络模式 | 说明 |
|----------|------|
| **bridge**（默认） | 容器连接到 docker0 虚拟网桥，拥有独立 IP，通过 NAT 与外部通信 |
| **host** | 容器直接使用宿主机的网络命名空间，不分配独立 IP |
| **none** | 容器有独立的 network namespace，但不进行任何网络配置 |
| **container** | 新容器与指定容器共享同一个 network namespace（共享 IP 和端口空间） |
| **overlay** | 跨主机的容器网络，用于 Swarm 或多主机通信 |

```
┌──────────────┐       ┌──────────────┐
│  Container A │       │  Container B │
│  172.17.0.2  │       │  172.17.0.3  │
└──────┬───────┘       └──────┬───────┘
       │  veth pair          │  veth pair
       ▼                     ▼
  ┌─────────────────────────────┐
  │       docker0 (bridge)      │
  │         172.17.0.1          │
  └─────────────┬───────────────┘
                │ NAT
                ▼
          ┌──────────┐
          │  宿主机   │
          └──────────┘
```

> **拓展：** 在 K8s 环境中，容器网络模型（CNI）更加复杂。常见的 CNI 插件包括 Calico、Flannel、Cilium 等，它们实现了 Pod 之间的扁平化网络通信，每个 Pod 拥有独立 IP，Pod 间可以直接通信。

---

### 7. Docker 数据持久化有哪些方式？

**答：**

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| **Volume（数据卷）** | 由 Docker 管理，存储在 `/var/lib/docker/volumes/` 下 | 推荐方式，跨容器共享、备份迁移方便 |
| **Bind Mount（绑定挂载）** | 直接映射宿主机指定路径到容器 | 开发环境，代码热加载 |
| **tmpfs** | 存储在宿主机内存中，不写入磁盘 | 敏感信息的临时存储 |

```bash
# Volume
docker run -v my_volume:/data nginx

# Bind Mount
docker run -v /host/path:/container/path nginx

# tmpfs
docker run --tmpfs /tmp nginx
```

> **拓展：** Volume 是生产环境推荐的方式，因为：(1) 由 Docker 统一管理，不依赖宿主机目录结构；(2) 可以通过 `docker volume` 命令方便地创建、列出、删除；(3) 支持 Volume Driver 实现远程存储（如 NFS、云盘）。

---

### 8. 如何进入一个正在运行的容器？

**答：** 有两种主要方式：

```bash
# 方式一：docker exec（推荐）
docker exec -it <container_id> /bin/bash

# 方式二：docker attach
docker attach <container_id>
```

**区别：**
- `docker exec`：在容器内**新开一个进程**，退出时不会影响容器运行。
- `docker attach`：连接到容器主进程的**标准输入/输出**，退出（Ctrl+C）时可能导致容器停止。

> **拓展：** 如果容器内没有 `/bin/bash`，可以尝试 `/bin/sh`，或者使用 `nsenter` 从宿主机进入容器的命名空间：
> ```bash
> nsenter --target <PID> --mount --uts --ipc --net --pid
> ```

---

### 9. Docker Compose 是什么？它解决了什么问题？

**答：** Docker Compose 是一个用于定义和运行**多容器应用**的工具。通过一个 `docker-compose.yml` 文件描述所有服务、网络和卷，一条命令即可启动全部服务。

```yaml
version: "3.8"
services:
  web:
    image: nginx
    ports:
      - "80:80"
    depends_on:
      - api
  api:
    build: ./backend
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
```

核心价值：
1. **一键编排**：`docker-compose up -d` 启动所有服务。
2. **服务依赖**：`depends_on` 控制启动顺序。
3. **共享网络**：所有服务自动加入同一网络，通过服务名互相访问。
4. **环境隔离**：不同项目使用不同的项目名，互不干扰。

> **拓展：** Docker Compose 适用于开发和测试环境。在生产环境中，通常使用 Kubernetes 进行容器编排。Docker Swarm 虽然也是编排方案，但生态和社区活跃度远不如 K8s。

---

### 10. Docker 容器退出后，数据会丢失吗？

**答：** 取决于数据存储的位置：

- **容器可写层的数据**：容器被删除后，可写层一起被删除，数据**丢失**。
- **挂载了 Volume 或 Bind Mount 的数据**：数据存储在宿主机上，容器删除后数据**保留**。

因此，需要持久化的数据（如数据库文件、日志、用户上传的文件）一定要通过 Volume 或 Bind Mount 挂载到宿主机。

> **拓展：** 可以通过 `docker cp` 命令在容器和宿主机之间复制文件，但这只是临时手段，生产环境应使用 Volume。

---

## 二、Kubernetes 篇

### 11. 什么是 Kubernetes？它解决了什么问题？

**答：** Kubernetes（简称 K8s）是一个开源的容器编排平台，最初由 Google 设计，现在由 CNCF 维护。它的核心目标是**自动化部署、扩缩和管理容器化应用**。

K8s 解决的核心问题：
1. **部署效率**：通过声明式配置，一条命令完成复杂应用的部署。
2. **弹性伸缩**：根据负载自动扩缩容（HPA、VPA、Cluster Autoscaler）。
3. **自愈能力**：容器崩溃自动重启，节点故障自动迁移 Pod。
4. **服务发现与负载均衡**：自动为 Pod 分配 IP 和 DNS，分发流量。
5. **滚动更新与回滚**：零停机更新，出问题秒级回滚。
6. **配置与密钥管理**：通过 ConfigMap 和 Secret 管理配置，无需重新构建镜像。

> **拓展：** K8s 的名字来源于 K 和 s 之间有 8 个字母（ubernete）。它源于 Google 内部系统 **Borg** 的十几年运维经验，Borg 的论文是理解 K8s 设计理念的重要参考。

---

### 12. Kubernetes 的架构是怎样的？有哪些核心组件？

**答：** K8s 集群由 **Master（控制平面）** 和 **Node（工作节点）** 两部分组成：

```
┌─────────────────── Master ───────────────────┐
│                                              │
│  ┌──────────────┐  ┌──────────────────────┐  │
│  │ kube-apiserver│  │ etcd                 │  │
│  └──────┬───────┘  └──────────────────────┘  │
│         │                                    │
│  ┌──────┴───────────┐  ┌─────────────────┐  │
│  │ kube-scheduler   │  │ kube-controller │  │
│  │                  │  │    -manager      │  │
│  └──────────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────┘
          │
          ▼
┌─────────────── Node ─────────────────────────┐
│  ┌──────────┐  ┌───────────┐  ┌───────────┐ │
│  │ kubelet  │  │ kube-proxy│  │ Container │ │
│  │          │  │           │  │ Runtime   │ │
│  └──────────┘  └───────────┘  └───────────┘ │
│  ┌────────────────────────────────────────┐  │
│  │  Pod  │  Pod  │  Pod  │  Pod          │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**Master 组件：**

| 组件 | 职责 |
|------|------|
| **kube-apiserver** | 集群的统一入口，提供 RESTful API，所有操作都通过它 |
| **etcd** | 分布式 KV 存储，保存集群的所有状态数据 |
| **kube-scheduler** | 负责 Pod 调度，选择合适的 Node 运行 Pod |
| **kube-controller-manager** | 运行各种控制器（Node Controller、ReplicaSet Controller 等），维护集群期望状态 |

**Node 组件：**

| 组件 | 职责 |
|------|------|
| **kubelet** | 管理 Pod 生命周期，监听 API Server 的指令，确保容器按规格运行 |
| **kube-proxy** | 维护节点上的网络规则，实现 Service 的负载均衡 |
| **Container Runtime** | 实际运行容器的软件（containerd、CRI-O 等） |

> **拓展：** etcd 是 K8s 的"大脑"，它使用 Raft 协议保证数据一致性。生产环境中 etcd 通常部署为奇数个节点（3 或 5）的集群，并且建议与 Master 组件分开部署以减少资源竞争。

---

### 13. 什么是 Pod？为什么 K8s 不直接管理容器？

**答：** Pod 是 K8s 中可以创建和管理的**最小调度单元**。一个 Pod 包含一个或多个容器，这些容器共享：
- **网络命名空间**（同一个 IP，端口空间共享）
- **存储卷**（可以挂载相同的 Volume）
- **生命周期**（一起调度、一起销毁）

**为什么不直接管理容器？** 因为有些应用天然需要紧密耦合的多个进程协作。例如：
- 主应用容器 + Sidecar 日志收集容器
- 主应用容器 + Init 容器（初始化数据）
- Web 服务器 + 配置热加载容器

Pod 将这些紧密关联的容器组合为一个调度单元，保证它们在同一节点上运行并共享资源。

> **拓展：** Pod 内部有一个特殊的 **Pause 容器**（infrastructure container），它的职责是"占位"——为 Pod 创建并维持网络命名空间。业务容器通过加入 Pause 容器的网络命名空间来实现共享 IP。当 Pod 中所有业务容器都退出后，Pause 容器仍然维持着 IP 地址，使得 Pod 的 IP 在重启期间保持不变。

---

### 14. Pod 的生命周期和状态有哪些？

**答：** Pod 的状态（Phase）包括：

| 状态 | 含义 |
|------|------|
| **Pending** | Pod 已被接受，但容器尚未全部运行（可能在拉取镜像或调度中） |
| **Running** | Pod 已绑定到节点，至少有一个容器在运行 |
| **Succeeded** | 所有容器正常退出（退出码 0），不再重启 |
| **Failed** | 至少有一个容器异常退出 |
| **Unknown** | 无法获取 Pod 状态（通常是与 Node 通信失败） |

**Pod 内的容器有三种探针：**

| 探针 | 作用 |
|------|------|
| **livenessProbe（存活探针）** | 检测容器是否存活，失败则重启容器 |
| **readinessProbe（就绪探针）** | 检测容器是否准备好接收流量，失败则从 Service 端点移除 |
| **startupProbe（启动探针）** | 检测容器是否已启动，保护慢启动应用不被存活探针误杀 |

**Pod 重启策略（restartPolicy）：**
- `Always`（默认）：容器退出后总是重启
- `OnFailure`：仅在非 0 退出时重启
- `Never`：从不重启

> **拓展：** 一个常见的面试陷阱是混淆 Pod 的 Phase 和容器状态。Pod 处于 `Running` 并不代表所有容器都健康——需要结合 readinessProbe 来判断是否真正可用。这也是为什么生产环境中必须配置就绪探针，否则流量可能被路由到尚未就绪的 Pod。

---

### 15. Pod 的创建流程是怎样的？

**答：** 一个 Pod 从创建到运行的完整流程：

```
用户/kubectl 提交创建请求
        │
        ▼
  kube-apiserver 接收请求，进行认证和鉴权
        │
        ▼
  写入 etcd（状态为 Pending）
        │
        ▼
  kube-scheduler 监听到新的未调度 Pod
        │
        ▼
  根据调度策略选择最优 Node
        │
        ▼
  将绑定信息写回 apiserver → etcd
        │
        ▼
  目标 Node 上的 kubelet 监听到 Pod 被分配给自己
        │
        ▼
  kubelet 通过 Container Runtime 拉取镜像并创建容器
        │
        ▼
  kubelet 向 apiserver 上报 Pod 状态（Running）
```

**调度器选择 Node 的过程分两步：**
1. **预选（Predicates/Filtering）**：过滤掉不满足条件的节点（资源不足、标签不匹配、污点排斥等）。
2. **优选（Priorities/Scoring）**：对剩余节点打分，选择得分最高的。

> **拓展：** 从 K8s 1.26 开始，调度框架支持 **Scheduling Gates**，允许外部控制器延迟 Pod 的调度，这在某些自定义调度场景中非常有用。

---

### 16. Kubernetes 有哪些工作负载资源？

**答：**

| 资源 | 说明 | 典型场景 |
|------|------|----------|
| **Deployment** | 管理 ReplicaSet，支持滚动更新和回滚 | 无状态应用（Web 服务、API） |
| **StatefulSet** | 每个 Pod 有固定名称和存储，保证部署顺序 | 有状态应用（数据库、消息队列） |
| **DaemonSet** | 确保每个 Node 运行一个 Pod 副本 | 日志收集、监控 Agent |
| **ReplicaSet** | 确保指定数量的 Pod 副本运行 | 通常不直接使用，由 Deployment 管理 |
| **Job** | 运行一次性任务，完成后退出 | 批处理、数据迁移 |
| **CronJob** | 按 Cron 表达式定时创建 Job | 定时备份、定时报告 |

**Deployment vs StatefulSet：**

| 对比 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 名称 | 随机生成 | 有序编号（0, 1, 2...） |
| 存储 | 所有 Pod 共享 Volume | 每个 Pod 有独立的 PVC |
| 启停顺序 | 无序 | 有序（创建 0→1→2，删除 2→1→0） |
| 网络标识 | 不固定 | 固定（通过 Headless Service） |

> **拓展：** 在实际生产中，大多数应用应该设计为无状态的，通过 Deployment 管理。只有像 MySQL、Kafka、Elasticsearch 这类需要持久化标识和存储的应用才使用 StatefulSet。更好的做法是使用 Operator 模式（如 MySQL Operator、Strimzi Kafka Operator）来管理有状态应用。

---

### 17. Kubernetes Service 有哪些类型？

**答：** Service 为一组 Pod 提供稳定的访问入口（固定 IP + DNS 名称），即使 Pod 重建 IP 变化也不影响。

| 类型 | 说明 | 访问范围 |
|------|------|----------|
| **ClusterIP**（默认） | 分配集群内部 IP | 仅集群内可访问 |
| **NodePort** | 在每个节点上暴露一个固定端口（30000-32767） | 集群外可通过 `<NodeIP>:<NodePort>` 访问 |
| **LoadBalancer** | 在 NodePort 基础上调用云厂商 API 创建外部负载均衡器 | 通过云 LB 的外部 IP 访问 |
| **ExternalName** | 将 Service 映射到一个外部 DNS 名称（CNAME 记录） | 用于访问集群外部服务 |

```
                    ┌────────────────────────┐
 集群外用户 ──────→ │   LoadBalancer (云LB)   │
                    └──────────┬─────────────┘
                               │
                    ┌──────────▼─────────────┐
 集群内 Pod  ─────→│   ClusterIP Service     │
                    │   10.96.0.100:80       │
                    └──────────┬─────────────┘
                               │ 负载均衡
                    ┌──────┬───┴───┬──────┐
                    ▼      ▼       ▼      ▼
                  Pod-1  Pod-2  Pod-3  Pod-4
```

> **拓展：** Headless Service（`clusterIP: None`）是一种特殊的 Service，它不分配 ClusterIP，而是直接返回后端 Pod 的 IP 列表。常用于 StatefulSet，让客户端直接选择特定的 Pod。

---

### 18. 什么是 Ingress？它和 Service 有什么区别？

**答：** Ingress 是 K8s 中管理**外部 HTTP/HTTPS 流量进入集群**的 API 对象。它定义了路由规则，将不同域名或路径的流量转发到不同的后端 Service。

```
                    用户请求
                       │
                       ▼
              ┌────────────────┐
              │   Ingress      │
              │   Controller   │  ← Nginx / Traefik / HAProxy
              │  (反向代理)     │
              └───────┬────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Service A│ │ Service B│ │ Service C│
   │ /api     │ │ /web     │ │ /admin   │
   └──────────┘ └──────────┘ └──────────┘
```

**Ingress vs Service：**
- **Service（NodePort/LoadBalancer）**：四层（TCP/UDP）负载均衡，每个 Service 需要独立端口或独立 LB。
- **Ingress**：七层（HTTP/HTTPS）路由，一个入口可以根据域名/路径分发到多个 Service，更节省资源。

> **拓展：** Ingress 本身只是一组路由规则，需要配合 **Ingress Controller** 才能工作。常见的 Ingress Controller 有 Nginx Ingress Controller、Traefik、HAProxy Ingress 等。近年来 **Gateway API** 正在成为 Ingress 的下一代替代方案，它支持更丰富的路由模型和更广泛的协议。

---

### 19. Deployment 的更新策略有哪些？

**答：**

| 策略 | 说明 |
|------|------|
| **RollingUpdate（滚动更新，默认）** | 逐步替换旧 Pod 为新 Pod，期间服务不中断 |
| **Recreate（重建）** | 先删除所有旧 Pod，再创建新 Pod，期间服务中断 |

**RollingUpdate 的两个关键参数：**
- `maxSurge`：更新过程中最多超出期望副本数的数量（可以是数字或百分比）。
- `maxUnavailable`：更新过程中最多不可用的 Pod 数量。

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多多出 1 个 Pod
      maxUnavailable: 0    # 更新期间不允许有 Pod 不可用
```

**回滚操作：**
```bash
kubectl rollout history deployment/my-app    # 查看历史版本
kubectl rollout undo deployment/my-app       # 回滚到上一版本
kubectl rollout undo deployment/my-app --to-revision=2  # 回滚到指定版本
```

> **拓展：** 滚动更新虽然能实现零停机，但它不是"蓝绿发布"或"金丝雀发布"。在更高级的发布策略中，可以使用 **Argo Rollouts** 或 **Flagger** 来实现金丝雀发布（逐步切流量）和自动化渐进式交付（根据监控指标自动决定推进还是回滚）。

---

### 20. 什么是 Namespace？它有什么作用？

**答：** Namespace 是 K8s 中的**逻辑隔离机制**，用于在同一集群中划分资源空间。

常见用途：
- **环境隔离**：dev、staging、production 分别使用不同的 Namespace。
- **团队隔离**：不同团队在各自的 Namespace 中工作，互不干扰。
- **资源配额**：可以对 Namespace 设置 ResourceQuota，限制 CPU、内存、Pod 数量等。
- **RBAC 作用域**：权限控制可以限定在某个 Namespace 内。

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production
```

> **拓展：** 默认情况下，K8s 集群有以下内置 Namespace：
> - `default`：未指定 Namespace 时的默认空间。
> - `kube-system`：K8s 系统组件运行的空间。
> - `kube-public`：集群公开信息（如 `cluster-info` ConfigMap）。
> - `kube-node-lease`：Node 心跳（Lease 资源）。
>
> 需要注意的是，Namespace **不能**隔离跨 Namespace 的网络——如果需要网络隔离，要配合 **NetworkPolicy** 使用。

---

### 21. 什么是 RBAC？它有哪些核心概念？

**答：** RBAC（Role-Based Access Control，基于角色的访问控制）是 K8s 的权限管理机制，由三个核心概念组成：

| 概念 | 说明 |
|------|------|
| **Role / ClusterRole** | 定义一组权限（对哪些资源可以执行哪些操作） |
| **RoleBinding / ClusterRoleBinding** | 将 Role 绑定到用户/组/ServiceAccount |
| **ServiceAccount** | Pod 的身份标识，用于 API 认证 |

```yaml
# Role：在 default 命名空间中，允许对 Pod 执行 get/list/watch
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding：将 pod-reader 角色绑定到用户 jane
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Role vs ClusterRole：**
- `Role`：作用域限定在某个 Namespace。
- `ClusterRole`：集群级别，对所有 Namespace 生效。

> **拓展：** 生产环境的安全最佳实践：
> 1. 禁用匿名访问和基础认证。
> 2. 启用 RBAC，遵循**最小权限原则**。
> 3. 对 ServiceAccount 自动挂载的 Token 进行审查。
> 4. 使用 NetworkPolicy 限制 Pod 间网络通信。
> 5. 启用审计日志（Audit Logging）记录所有 API 访问。

---

### 22. K8s 如何实现服务发现和负载均衡？

**答：** K8s 通过以下机制实现服务发现和负载均衡：

**服务发现：**
1. **环境变量**：Pod 启动时，K8s 自动注入已存在 Service 的环境变量（如 `MY_SERVICE_SERVICE_HOST`）。缺点是必须在 Service 之后创建 Pod。
2. **DNS（推荐）**：CoreDNS 为每个 Service 自动创建 DNS 记录。Pod 可以通过 `<service-name>.<namespace>.svc.cluster.local` 访问其他 Service。

**负载均衡：**
1. **kube-proxy**：运行在每个节点上，监听 Service 变化，维护 iptables/IPVS 规则，将流量分发到后端 Pod。
2. **模式选择：**
   - **iptables 模式**（默认）：使用 Linux netfilter 规则，适合中小规模。
   - **IPVS 模式**：使用 Linux Virtual Server，性能更好，适合大规模集群。

> **拓展：** kube-proxy 的 iptables 模式在 Service 数量很多时性能会下降（规则线性匹配），而 IPVS 模式使用哈希表，性能更稳定。在大规模集群中建议切换到 IPVS 模式：
> ```bash
> # 加载 IPVS 内核模块
> modprobe ip_vs
> modprobe ip_vs_rr
> modprobe ip_vs_wrr
> modprobe ip_vs_sh
> ```

---

### 23. 什么是 ConfigMap 和 Secret？它们有什么区别？

**答：** ConfigMap 和 Secret 都是用来将配置数据与镜像解耦的 K8s 资源。

| 对比 | ConfigMap | Secret |
|------|-----------|--------|
| **用途** | 存储非敏感配置数据 | 存储敏感数据（密码、Token、证书） |
| **编码** | 明文存储 | Base64 编码（注意：不是加密！） |
| **大小限制** | 1 MB | 1 MB |
| **使用方式** | 环境变量、Volume 挂载、API 读取 | 环境变量、Volume 挂载、API 读取 |

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=   # base64 编码的 "password123"
```

> **拓展：** Base64 编码**不等于加密**，任何有权限读取 Secret 的人都能解码。生产环境中应：
> 1. 启用 etcd 静态加密（Encryption at Rest）。
> 2. 使用外部密钥管理系统（如 HashiCorp Vault、AWS Secrets Manager）配合 External Secrets Operator。
> 3. 通过 RBAC 严格限制 Secret 的访问权限。

---

### 24. K8s 的调度策略有哪些？如何控制 Pod 调度到哪个节点？

**答：** K8s 提供多种机制控制 Pod 的调度：

| 机制 | 说明 |
|------|------|
| **nodeSelector** | 最简单的节点选择，通过标签匹配 |
| **Node Affinity（节点亲和性）** | 更灵活的节点选择，支持 `In`、`NotIn`、`Exists` 等表达式 |
| **Pod Affinity / Anti-Affinity** | 根据已运行 Pod 的标签来决定调度，实现"共置"或"分散" |
| **Taints & Tolerations（污点与容忍）** | 在 Node 上设置"排斥"，只有"容忍"该污点的 Pod 才能调度上去 |
| **PriorityClass** | 设置 Pod 优先级，高优先级 Pod 可以抢占低优先级 Pod 的资源 |

```yaml
# nodeSelector 示例
spec:
  nodeSelector:
    disktype: ssd

# Taint 示例（在 Node 上设置）
# kubectl taint nodes node1 key=value:NoSchedule
# 只有 Pod 有对应 Toleration 才能调度上去
```

> **拓展：** 实际生产中常用的调度策略组合：
> - 用 **Node Affinity** 将应用调度到特定机型（如 GPU 节点）。
> - 用 **Pod Anti-Affinity** 确保同一应用的多个副本分布在不同节点上，提高可用性。
> - 用 **Taint** 隔离特殊节点（如 Master 节点默认有 `node-role.kubernetes.io/master:NoSchedule` 污点）。

---

### 25. 什么是 HPA？K8s 如何实现自动弹性伸缩？

**答：** HPA（Horizontal Pod Autoscaler，水平 Pod 自动伸缩器）根据监控指标自动调整 Deployment/StatefulSet 的副本数。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # CPU 使用率超过 70% 时扩容
```

**K8s 的三种弹性伸缩机制：**

| 机制 | 维度 | 说明 |
|------|------|------|
| **HPA** | 水平（Pod 数量） | 根据 CPU/内存/自定义指标增减 Pod |
| **VPA** | 垂直（资源配额） | 自动调整 Pod 的 CPU/内存 request |
| **Cluster Autoscaler** | 集群（Node 数量） | 根据 Pending Pod 自动扩缩节点 |

> **拓展：** HPA 默认支持 CPU 和内存指标。如果要使用自定义指标（如 QPS、请求延迟），需要部署 **Prometheus Adapter** 或 **KEDA**（Kubernetes Event-Driven Autoscaling），后者还支持基于外部事件源（如 Kafka 队列长度）触发伸缩。

---

### 26. 什么是 PV 和 PVC？存储是如何工作的？

**答：** K8s 通过 PV（PersistentVolume）和 PVC（PersistentVolumeClaim）实现持久化存储。

| 概念 | 说明 | 角色 |
|------|------|------|
| **PV** | 集群级别的存储资源，由管理员创建或由 StorageClass 动态创建 | 提供者 |
| **PVC** | 用户对存储的请求（大小、访问模式），类似"申请单" | 消费者 |
| **StorageClass** | 定义存储类型（如 SSD、HDD），实现动态供给 | 模板 |

**访问模式：**
- `ReadWriteOnce (RWO)`：单节点读写
- `ReadOnlyMany (ROX)`：多节点只读
- `ReadWriteMany (RWX)`：多节点读写

**回收策略：**
- `Retain`：保留数据，手动清理
- `Delete`：删除 PV 时同时删除后端存储

```
用户创建 PVC（需要 5Gi 存储）
        │
        ▼
K8s 匹配或动态创建 PV
        │
        ▼
PV 与 PVC 绑定
        │
        ▼
Pod 通过 Volume 挂载 PVC
        │
        ▼
容器读写数据
```

> **拓展：** 在生产环境中，通常使用 **StorageClass** 实现动态存储供给——用户只需创建 PVC，K8s 自动向云厂商申请云盘（如 AWS EBS、腾讯云 CBS）。不同 StorageClass 可以对应不同性能等级（如标准盘、SSD 盘），用户按需选择。

---

### 27. K8s 中如何排查 Pod 异常？

**答：** 常见的排查思路和命令：

```bash
# 1. 查看 Pod 状态
kubectl get pods -n <namespace> -o wide

# 2. 查看 Pod 详细事件（调度失败、镜像拉取失败等）
kubectl describe pod <pod-name> -n <namespace>

# 3. 查看容器日志
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous   # 查看上一次崩溃的日志

# 4. 进入容器排查
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# 5. 查看节点状态
kubectl get nodes
kubectl describe node <node-name>
```

**常见问题排查表：**

| 现象 | 可能原因 |
|------|----------|
| `ImagePullBackOff` | 镜像名错误、仓库认证失败、网络不通 |
| `CrashLoopBackOff` | 应用启动失败、健康检查配置不当、OOMKilled |
| `Pending` | 没有可用节点、资源不足、nodeSelector 不匹配 |
| `OOMKilled` | 容器内存超出 limit |
| `Evicted` | 节点资源压力过大，Pod 被驱逐 |

> **拓展：** 排查问题时，`kubectl get events --sort-by='.lastTimestamp'` 可以查看集群最近的事件，快速定位问题。对于复杂的网络问题，可以使用 `kubectl debug` 命令创建一个调试 Pod，加入目标 Pod 的网络命名空间进行抓包分析。

---

### 28. K8s 如何保证高可用？

**答：** K8s 的高可用体现在多个层面：

**控制平面高可用：**
- **etcd 集群**：至少 3 节点，容忍 1 节点故障。
- **API Server**：多实例部署，前面挂负载均衡器。
- **Controller Manager / Scheduler**：多实例 + Leader 选举，只有一个在工作，其余热备。

**数据平面高可用：**
- **Pod 多副本**：通过 Deployment 设置 `replicas: N`，配合 Anti-Affinity 分散到不同节点。
- **PodDisruptionBudget (PDB)**：限制同时不可用的 Pod 数量，保证滚动更新/节点维护时服务可用。
- **健康检查**：livenessProbe 自动重启故障容器，readinessProbe 将故障 Pod 从流量中移除。
- **节点故障自动迁移**：Node 失联后，其上的 Pod 被标记为 Unknown，Controller 在其他节点重新创建。

```yaml
# PDB 示例：保证至少有 2 个 Pod 可用
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

> **拓展：** 除了 K8s 自身的高可用机制，生产环境还需要关注：
> 1. **多可用区部署**：将节点分布在不同可用区（AZ），容忍单 AZ 故障。
> 2. **备份与恢复**：定期备份 etcd 数据，使用 Velero 等工具备份集群资源。
> 3. **灾备演练**：定期进行混沌工程（Chaos Engineering）测试，如使用 LitmusChaos 验证系统韧性。

---

### 29. Docker 容器和 K8s Pod 中的网络模型有什么区别？

**答：**

| 对比 | Docker 默认网络 | K8s 网络模型 |
|------|----------------|-------------|
| **IP 分配** | 每个容器从 docker0 网桥获取 IP | 每个 Pod 拥有独立 IP |
| **通信方式** | 同一主机通过网桥，跨主机需端口映射 | 所有 Pod 扁平网络，可直接通过 IP 通信 |
| **服务发现** | 依赖 Docker DNS 或手动配置 | CoreDNS 自动为 Service 创建 DNS |
| **网络插件** | bridge / overlay / macvlan | CNI 插件（Calico、Flannel、Cilium 等） |
| **网络策略** | 有限支持 | NetworkPolicy 精细控制 Pod 间通信 |

K8s 网络模型的四个基本要求：
1. 所有 Pod 之间可以不加 NAT 直接通信。
2. 所有 Node 可以与所有 Pod 之间不加 NAT 直接通信。
3. Pod 看到自己的 IP 与其他 Pod 看到的一致（不需要 NAT 转换）。
4. 每个 Pod 拥有独立 IP。

> **拓展：** Cilium 是近年来备受关注的 CNI 方案，它基于 **eBPF** 技术实现网络和数据包过滤，性能优于传统的 iptables/IPVS 方案，同时原生支持 L7 策略、可观测性和服务网格功能。

---

### 30. 什么是 Operator 模式？

**答：** Operator 是一种扩展 K8s 的方式，通过自定义资源（CRD）和自定义控制器来实现对复杂应用的自动化运维。

**核心思想：** 将人类运维专家的知识编码到软件中，让 K8s 自动完成复杂的运维操作。

```
┌────────────────────────────────────┐
│           Operator                 │
│                                    │
│  ┌──────────┐    ┌──────────────┐ │
│  │   CRD    │    │  Controller  │ │
│  │ (自定义   │    │  (控制循环    │ │
│  │  资源)    │    │   调谐逻辑)  │ │
│  └──────────┘    └──────────────┘ │
└────────────────────────────────────┘
         │                 │
         ▼                 ▼
   用户声明期望状态    控制器自动执行
   (如: 创建MySQL     (备份、扩容、
    集群)             故障恢复等)
```

常见的 Operator：
- **Prometheus Operator**：自动化 Prometheus 监控栈的部署和管理。
- **MySQL Operator / Percona Operator**：自动化数据库集群的部署、备份、主从切换。
- **Strimzi**：自动化 Kafka 集群管理。
- **Cert-Manager**：自动化 TLS 证书的申请和续期。

> **拓展：** 开发 Operator 的主流框架有 **Operator SDK**（支持 Go、Ansible、Helm）和 **Kubebuilder**。Operator 的核心是**控制循环（Reconciliation Loop）**——不断比较"期望状态"和"实际状态"，并执行操作使两者一致。这种声明式的理念与 K8s 本身的设计一脉相承。

---

## 三、综合与进阶篇

### 31. Docker 和 Kubernetes 的关系是什么？

**答：** Docker 和 K8s 是**互补关系**，不是竞争关系：

- **Docker** 负责**构建和运行容器**——打包应用及其依赖为标准化镜像。
- **Kubernetes** 负责**编排和管理容器**——在大规模集群中调度、扩缩、自愈容器化应用。

简单说：Docker 是"打包和运行"，K8s 是"管理和调度"。

> **拓展：** K8s 从 1.24 版本开始移除对 Docker 作为容器运行时（dockershim）的支持，但这**不意味着 Docker 不能与 K8s 一起使用**。Docker 仍然可以用来构建镜像（`docker build`），推送到镜像仓库后，K8s 通过 containerd 或 CRI-O 拉取和运行这些镜像。Docker 构建的镜像与 K8s 完全兼容。

---

### 32. 容器化应用如何保证安全性？

**答：** 容器安全需要从多个层面考虑：

| 层面 | 措施 |
|------|------|
| **镜像安全** | 使用可信基础镜像、扫描漏洞（Trivy、Clair）、最小化镜像层 |
| **运行时安全** | 不以 root 运行容器、设置 `readOnlyRootFilesystem`、禁用特权模式 |
| **网络安全** | 使用 NetworkPolicy 限制 Pod 间通信 |
| **权限控制** | RBAC 最小权限、Pod Security Standards/Admission |
| **密钥管理** | 使用 Secret + 外部密钥管理系统，不硬编码到镜像 |
| **资源限制** | 设置 CPU/Memory 的 request 和 limit，防止资源滥用 |

```yaml
# 安全上下文示例
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

> **拓展：** K8s 从 1.25 版本引入了 **Pod Security Admission**（替代了废弃的 PodSecurityPolicy），通过 Namespace 级别的标签来控制安全策略：
> - `privileged`：无限制。
> - `baseline`：最小限制，防止特权升级。
> - `restricted`：严格限制，遵循最佳安全实践。

---

### 33. 如何将单体应用迁移到 K8s？

**答：** 推荐渐进式迁移策略：

1. **容器化**：先将单体应用打包为 Docker 镜像，编写 Dockerfile 和 K8s 部署文件。
2. **部署到 K8s**：使用 Deployment + Service 部署，验证基本功能。
3. **逐步拆分**：将高频变更或需要独立扩展的模块拆分为微服务。
4. **流量管理**：使用 Ingress 或 Service Mesh 进行流量切换，逐步将流量从旧系统迁移到 K8s。
5. **数据迁移**：有状态组件（数据库）可以使用 Operator 管理，或逐步迁移到云数据库。

> **拓展：** 不建议一次性将单体应用完全拆分为微服务。更好的方式是先" lift and shift"（整体迁移），在 K8s 上稳定运行后，根据实际业务需要逐步拆分。微服务拆分应该以**业务边界**（领域驱动设计）为依据，而不是技术层次。

---

## 参考来源

- [博客园 - K8s 和 Docker 面试题](https://www.cnblogs.com/qylogs/p/20231139.html)
- [小林 coding - Docker 面试题](https://www.xiaolincoding.com/interview/docker.html)
- [CSDN - 40 道常见 K8S 面试题总结](https://blog.csdn.net/2401_83974087/article/details/137699268)
- [百度开发者 - Docker 和 Kubernetes 面试题及答案](https://developer.baidu.com/article/details/2810756)
- 《云原生架构与 GitOps 实战》极客时间课程
