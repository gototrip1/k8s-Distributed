### 现在要使用的是windows电脑中的虚拟机里面的centos7这个版本，里面已经安装了docker，来部署一个部署k3s，因为我的k3s也没有部署，第二，然后使用k3s来部署其中的分布式redis

导航：**Windows (宿主机) -> VMware/VirtualBox (虚拟机软件) -> CentOS 7 (操作系统) -> Docker (容器引擎) -> K3s (K8s 发行版) -> Redis (应用)**。

2，使用官方的 **Redis Cluster** 模式。这种模式通过数据分片（Sharding）将数据分布在多个节点上，实现真正的分布式存储。

3.我们将使用 `StatefulSet` 来部署 6 个 Redis 实例（3 主 3 从），并配合 `Headless Service` 来保证集群内部的网络通信。

### 部署架构规划

- **环境**：本地 k3s 集群
- **副本数**：6 个 Pod（3 个 Master，3 个 Replica）
- **存储**：使用 k3s 默认的 `local-path` 存储类进行数据持久化
- **网络**：Headless Service 用于集群内部通信

### 第一阶段：CentOS 7 基础环境准备

*请在你的 CentOS 7 虚拟机终端中执行以下操作*

虽然你已经安装了 Docker，但 K3s 对系统有一些特定要求（特别是关闭 Swap 和配置 cgroup），这一步至关重要，否则 K3s 无法启动。

1. **关闭防火墙和 SELinux**（本地测试建议关闭，避免网络策略干扰）：

   bash

   

   ```
   1sudo systemctl stop firewalld && sudo systemctl disable firewalld
   2sudo setenforce 0
   3sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   ```

问题：

![{94BB6077-8B9B-46A3-971C-045CA16BF3DE}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{94BB6077-8B9B-46A3-971C-045CA16BF3DE}.png)

解决方法：这个命令不影响后面的思路，我这个问题主要原因可能是安装的是精简版的centos7版本，因为这个资料是当时自学黑马程序员的时候进行学习的，

在 CentOS 7 中，SELinux 的配置文件通常位于 `/etc/selinux/config`。



2. **关闭 Swap 分区**（K8s/K3s 强制要求）：

bash



```
1sudo swapoff -a
2# 防止重启后自动开启，注释掉 /etc/fstab 中的 swap 行
3sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```



3. **配置 Docker cgroup 驱动**（关键步骤）：
   K3s 默认使用 `systemd`，如果 Docker 配置不一致会导致 Kubelet 启动失败。
   检查 `/etc/docker/daemon.json`，确保包含以下配置（如果没有该文件则创建）：

bash



```
1sudo mkdir -p /etc/docker   # 如果有docker ,就不需要这一行命令了
2sudo vi /etc/docker/daemon.json
```

**文件内容：**

json

这里面需要进行添加这两行的命令

```
1{
2  "exec-opts": ["native.cgroupdriver=systemd"],
3  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
4}
```

*修改后重启 Docker：* `sudo systemctl restart docker`



###  第二阶段：部署 K3s (单节点模式)

我们将使用 Docker 作为 K3s 的容器运行时。

1. **一键安装 K3s**：
   为了确保国内下载速度，我们使用国内镜像源，并指定使用 Docker。

   bash

   

   ```
   1curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" sh -
   ```

   *解释：`--docker` 参数告诉 K3s 不要使用自带的 containerd，而是去调用你已经安装好的 Docker。*

![{BB71420E-EB8F-43AE-B450-06C74ABEB01E}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{BB71420E-EB8F-43AE-B450-06C74ABEB01E}.png)

![{E1882E1A-A45C-48BA-9D5D-626FC5164041}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{E1882E1A-A45C-48BA-9D5D-626FC5164041}.png)

Using v1.34.6+k3s1 as release

![{230ED1BF-2FE7-4E19-B218-0142687D8B1B}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{230ED1BF-2FE7-4E19-B218-0142687D8B1B}.png)

问题：

![{7C3F791F-2AD7-4D62-AC1F-1CB0FF04FFB5}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{7C3F791F-2AD7-4D62-AC1F-1CB0FF04FFB5}.png)

分析原因：

**不用担心，这完全不影响后续运行。**

你遇到的问题非常简单，只是 Linux 的环境变量路径问题，加上安装脚本还在后台运行导致的。

以下是详细分析和解决方法：

### 1. 为什么提示“找不到命令”？

**原因分析：**

- 权限问题

  ：你使用的是 

  ```
  sudo kubectl get nodes
  ```

  。

  - K3s 安装脚本默认将 `kubectl` 软链接放在了 `/usr/local/bin/kubectl`。
  - 在很多 CentOS 系统中，普通用户的 `PATH`（环境变量）包含 `/usr/local/bin`，但 `root` 用户（使用 sudo 时）的安全路径可能不包含这个目录，或者你的系统还没有刷新这个路径。

- **安装未完成**：截图显示你在看到 `Starting k3s` 后按了 `Ctrl+C`。虽然服务已经在后台启动了，但安装脚本可能还有一些收尾工作（比如设置权限、刷新 shell 环境）没做完。

### 2. 版本 v1.34.6 会有影响吗？

**完全没有影响，可以放心使用。**

- **解释**：`v1.34.6+k3s1` 是 K3s 的版本号。K3s 的版本号通常会跟随上游 Kubernetes 的版本。
- **现状**：这表明你安装的是非常新的 Kubernetes 版本（或者是一个较新的测试版/最新版），这对部署 Redis 只有好处（性能更好、功能更多），**绝对不会阻碍**你部署分布式 Redis。

### 3. 解决方案：如何解决“找不到命令”并验证状态？

请按照以下步骤操作，即可恢复正常：

#### 方案A：使用全路径验证（最快方法）

既然环境变量没生效，我们就直接告诉系统 `kubectl` 在哪里。K3s 已经把它放在了 `/usr/local/bin/` 下。

请直接运行（**不要加 sudo**，或者使用全路径）：

bash



```
1/usr/local/bin/kubectl get nodes
```

![{70D865F8-0293-4141-8DBF-49170F9080EC}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{70D865F8-0293-4141-8DBF-49170F9080EC}.png)

- **方法 B（刷新环境）：** 让系统重新读取环境变量。

  bash

  

  ```
  1source /etc/profile
  ```

  然后再试：

  bash

  

  ```
  1kubectl get nodes
  ```

### 第三阶段：开始部署 Redis (Root 用户版)

确认 `kubectl get nodes` 能看到节点状态为 `Ready` 后，直接开始部署。

#### 第一步：创建配置文件

在 root 的家目录（`/root`）下直接创建文件。

**1. 创建 ConfigMap**

bash



```
vi redis-configmap.yaml
```

#### 1. 配置管理 (`redis-configmap.yaml`)

我们需要开启集群模式 (`cluster-enabled yes`) 并设置密码。

yaml



```bash
1apiVersion: v1
2kind: ConfigMap
3metadata:
4  name: redis-cluster-config
5data:
6  redis.conf: |
7    cluster-enabled yes
8    cluster-config-file nodes.conf
9    cluster-node-timeout 5000
10    appendonly yes
11    protected-mode no
12    port 6379
13    # 生产环境请务必修改密码
14    requirepass "password"
15    masterauth "password"
```

#### 2.核心服务定义 (`redis-cluster.yaml`)

这是最关键的部分。我们使用 `StatefulSet` 确保 Pod 有稳定的网络标识（如 `redis-cluster-0`, `redis-cluster-1`），并配置了 `postStart` 生命周期钩子来自动初始化集群。



```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--cluster-announce-ip"
          - "$(POD_IP)"
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis
        - name: redis-data
          mountPath: /data
        # 自动创建集群的命令，仅在 redis-cluster-0 节点执行一次
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                if [ "$(hostname)" == "redis-cluster-0" ]; then
                  echo "Waiting for all pods to be ready..."
                  sleep 10
                  redis-cli -a password --cluster create \
                    redis-cluster-0.redis-cluster:6379 \
                    redis-cluster-1.redis-cluster:6379 \
                    redis-cluster-2.redis-cluster:6379 \
                    redis-cluster-3.redis-cluster:6379 \
                    redis-cluster-4.redis-cluster:6379 \
                    redis-cluster-5.redis-cluster:6379 \
                    --cluster-replicas 1 --cluster-yes
                fi
      volumes:
      - name: redis-config
        configMap:
          name: redis-cluster-config
          items:
          - key: redis.conf
            path: redis.conf
  # 使用 k3s 默认的 local-path-provisioner 进行持久化
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: local-path
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  clusterIP: None
  selector:
    app: redis-cluster
```

#### 第二步：执行部署

直接使用 `kubectl` 命令（如果上面方法 A 有效，建议一直用 `/usr/local/bin/kubectl`，或者等一会再用简写）：

bash



```bash
kubectl apply -f redis-configmap.yaml
kubectl apply -f redis-cluster.yaml
```

问题：



![{6A1C366F-7D96-4684-8B7E-C2ED18602A5B}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{6A1C366F-7D96-4684-8B7E-C2ED18602A5B}.png)

原因：我第一行缺少了一个a进行复制的时候

解决方案：

加上那个a

![{7F46E855-C143-4362-AB9B-1B4A45FAFAE9}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{7F46E855-C143-4362-AB9B-1B4A45FAFAE9}.png)

#### 第三步：验证状态

bash



```
kubectl get pods -w
```

看到 6 个 `redis-cluster-x` 都变成 `Running` 状态，就代表部署成功了。

问题：

![{AC76FD89-3642-4AF7-93DE-6F834A0DA8CF}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{AC76FD89-3642-4AF7-93DE-6F834A0DA8CF}.png)

最后一步进行验证的状态，一直输出的cluster-0这几个

分析原因：

这是部署分布式 Redis 时最常见的问题。`CrashLoopBackOff` 和 `PostStartHookError` 意味着 **Pod 里的 Redis 进程启动后立刻退出了**，K8s 试图重启它，但它又立刻退出，陷入了死循环。

这通常是因为 **Redis 配置文件有误** 或者 **集群初始化脚本有逻辑错误**。

解决方案：

1.查看报错详情

```bash
kubectl logs redis-cluster-0
```

![{C5E9E511-868D-40AB-BCA3-C5704F2D96BE}](C:\Users\zzmin\AppData\Local\Packages\MicrosoftWindows.Client.Core_cw5n1h2txyewy\TempState\ScreenClip\{C5E9E511-868D-40AB-BCA3-C5704F2D96BE}.png)

最后的修正版的：

redis-clustter.yaml的文件的内容如下

```bash
# --- 1. Headless Service ---
# 作用：为 StatefulSet 提供稳定的 DNS 域名，用于节点间互相发现
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-headless
  labels:
    app: redis-cluster
spec:
  clusterIP: None
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: gossip
    port: 16379
    targetPort: 16379
  selector:
    app: redis-cluster
---
# --- 2. NodePort Service ---
# 作用：对外暴露服务，允许外部客户端连接
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  labels:
    app: redis-cluster
spec:
  type: NodePort
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
    nodePort: 30379
  selector:
    app: redis-cluster
---
# --- 3. StatefulSet ---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: "redis-cluster-headless" # 必须指向 Headless Service
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      # 解决权限问题
      securityContext:
        runAsUser: 0
        fsGroup: 0
      containers:
      - name: redis
        image: redis:7.2.13
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
          - name: redis-config
            mountPath: /etc/redis
          - name: redis-data
            mountPath: /data
        # 探针配置
        livenessProbe:
          exec:
            command:
              - sh
              - -c
              - "redis-cli -h $(hostname -i) -p 6379 ping"
          initialDelaySeconds: 30
          timeoutSeconds: 10
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
              - sh
              - -c
              - "redis-cli -h $(hostname -i) -p 6379 ping"
          initialDelaySeconds: 15
          timeoutSeconds: 10
          periodSeconds: 5
      volumes:
        - name: redis-config
          configMap:
            name: redis-cluster-config
            defaultMode: 0755
  # 自动创建 PVC，每个 Pod 独有
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```



核心配置两个文件：

#### `redis-configmap.yaml`

#### `redis-cluster.yaml`