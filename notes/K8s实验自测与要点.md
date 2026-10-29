# K8s 主节点故障实验 —— 自测与技术要点

> **用法**：先做「第一部分 自测题」，**不要翻答案**，凭记忆写下来（写不出来就空着）。
> 全部做完后再看「第二部分 答案」。**写不出来的题，才是你真正需要复习的地方。**
>
> 基于 2026-10-02 真实实验数据编写。

---

## 第一部分：自测题（15 题，凭记忆作答）

### A 组 · 你亲眼看到的（事实记忆）

1. 你的集群是 kubeadm 装的哪个版本？master 和 node1 的 IP 分别是多少？容器运行时是什么、版本多少？
2. flannel 的 DaemonSet 叫什么名字？在哪个命名空间？（提示：不在 kube-system）
3. 你的集群里，flannel 创建的网桥叫什么名字？
4. master 上 `flannel.1` 的 IP 是多少？为什么是这个看起来"不合法"的值？
5. `kubectl get events -A --field-selector reason=NodeNotReady` 的结果是什么？

### B 组 · 原理理解

6. 主节点关机后，node1 上已经在运行的 Pod 为什么还能工作？
7. 同一时刻为什么不能创建新的 Pod？
8. kubectl 的本质是什么？主节点挂掉后它为什么失效？
9. kubelet 有哪两套心跳机制？各自的 API 路径和频率？
10. 断电期间 node1 为什么没有被标记 NotReady？给出两个原因。
11. `Connection refused` 和 `Connection timed out` 分别意味着什么？你看到的是哪个？

### C 组 · 动手排查

12. 怎么查一个 Pod 的容器为什么重启了？写出两条命令。
13. 容器名 `k8s_nginx_test-app-846b7f7456-x9tfs_default_997cd629-..._1` 里的 `_1` 表示什么？
14. 容器重启后，Pod 的 IP 会变吗？为什么？
15. 单主集群最危险的地方是什么？（提示：答案不是"etcd 是单点"）

---
---

## 第二部分：答案

### 📌 先记住这条因果链（B 组全部答案都挂在它上面）

**一个根事实**：控制面（apiserver / etcd / controller-manager / scheduler）**全在 master**；
数据面（kubelet / Docker / kube-proxy / flannel）**每个节点都有一套**。

由此可以推导出 B 组全部答案：

| 现象 | 推导过程 | 题号 |
|---|---|---|
| 容器还在跑 | kubelet 在节点上，独立于 apiserver | B6 |
| 建不了新 Pod | 创建需要 apiserver（入口）+ scheduler（选节点），两者都在 master | B7 |
| kubectl 失效 | kubectl 只是 apiserver 的客户端 | B8 |
| 每 10 秒报 lease 失败 | kubelet 要持续汇报心跳，连不上就重试 | B9 |
| 报 timeout 不是 refused | 整机断电 → ARP 无响应 → 包被丢弃 | B11 |
| 没人标 NotReady | 负责标记的 node controller 也在 master 上 | B10 |
| 单主最危险的是静默故障 | 控制面挂了，健康检查机制同时失效 | B15 |

**记忆方法**：不要背 7 个结论，只背那**一个根事实**。
考场上从根事实往外推，7 个结论都能现场推出来。

### 📌 C 组同理：记动作，不记答案

| 题号 | 你要回答的问题 | 命令 | 看什么 |
|---|---|---|---|
| C12 | 容器为什么重启了？ | `kubectl describe pod <name> \| tail -25` | Events 里的 `(x2 over 19m)` |
| C13 | 这是第几次重启？ | `docker ps \| grep <关键词>` | 容器名末尾的 `_1` |
| C14 | 容器重启后 IP 会变吗？ | — | 不变，沙箱被复用 → **Pod ≠ 容器** |

**C 组靠手熟，不靠背**。同一组命令连做三天，肌肉记忆自然形成。

### ⚠️ A 组里有两题其实是原理题，不要跳过

- **第 3 题**（网桥为什么叫 `cbr0`）——考的是 CNI 插件如何把配置委托给 bridge 插件
- **第 4 题**（`flannel.1` 为什么是 `10.244.0.0/32`）——考的是 flannel VXLAN 后端的 VTEP 地址机制

其余 A 组题目是环境专属数字，评审方不关心，优先级放低。

---

### A 组

**1.** kubeadm **v1.23.6**；master `192.168.245.133`，node1 `192.168.245.150`；
容器运行时 **Docker 26.1.4**（不是 containerd）。

**2.** `kube-flannel-ds`，在 **`kube-flannel`** 命名空间。
新版 flannel 清单把它装在独立命名空间，不在传统的 `kube-system`。

**3.** **`cbr0`**。flannel 的 conflist 里 `"name":"cbr0"`，委托给 bridge 插件后
创建的网桥就叫这个名字。**不是 `cni0`**。

**4.** `10.244.0.0/32`。flannel 的 VXLAN 后端把节点 Pod 子网的**网络地址**
以 /32 赋给 `flannel.1`，作为 VXLAN 隧道的源地址（VTEP）。
它不是给 Pod 用的地址，所以出现"网络地址"完全合理。

**5.** `No resources found` —— **零事件**。断电期间 node1 从未被标记 NotReady。

### B 组

**6.** 因为 **kubelet 是节点上的独立进程**，容器的实际运行不依赖 apiserver。
Pod 的容器由所在节点的 kubelet + 容器运行时直接管理。

**7.** 因为**创建 Pod 需要 apiserver 和 scheduler**：apiserver 是所有操作的唯一入口，
scheduler 负责选节点。两者都跑在主节点上，一起挂了。

**8.** kubectl 本质是 **apiserver 的客户端**（遥控器）。
它不参与集群运行，只是把请求转发给 apiserver。apiserver 挂了它就废了。

**9.**

| 机制 | API 路径 | 频率 |
|---|---|---|
| **Node Lease**（新，1.14+） | `/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/<node>` | 每 **10 秒** |
| **Node Status**（老） | `/api/v1/nodes/<node>` | 每 **5 分钟**或状态变化时 |

拆成两套是为了降低心跳开销：Lease 只写一个极小对象，比上报完整 Node Status 便宜得多。

**10.** 两个原因叠加：

1. 负责标记 NotReady 的 **node controller 运行在 kube-controller-manager 上，
   而它就在被关掉的主节点里**——监控者和被监控对象一起挂了，没人能执行这个动作。
2. 恢复瞬间，**kubelet 每 10 秒重试续约，比 controller-manager 启动并进入
   node monitor 循环更快**，等 node controller 第一次检查时租约已经新鲜，检查直接通过。

**11.**

- `Connection refused` → 目标主机**在线**，但端口没有进程监听（例如只停了 apiserver）
- `Connection timed out` → 目标主机**整机不可达**（关机后 ARP 无响应，数据包被丢弃）

我们实测看到的是 **`curl: (28) Connection timed out after 5001 milliseconds`**
→ 证明是**真的关掉了整台主节点**，而不是只停了某个服务。

### C 组

**12.**

```bash
# 命令一：看 Events，注意 (x2 over 19m) 这种计数
kubectl describe pod <pod-name> | tail -25

# 命令二：在 Pod 所在节点上，看容器名后缀的重启计数
sudo docker ps | grep <关键词>
```

**13.** **kubelet 维护的重启计数器**。容器命名规则：

```
k8s_<容器名>_<Pod名>_<命名空间>_<Pod UID>_<重启次数>
```

末尾数字直接对应 `kubectl get pods` 的 `RESTARTS` 字段。

**14.** **不会变**。容器重启时 Pod 的**网络沙箱（sandbox）被复用**——
IP 属于沙箱，不属于容器。只有整个 Pod 被删除重建时 IP 才会变。

这就是 **"Pod ≠ 容器"** 最直接的证据：Pod 是外壳（含网络命名空间、IP、卷），
容器是里面的东西。

**15.** **控制面一挂，健康检查机制同时失效——你可能连"发生了故障"都不知道。**

node controller 住在被关掉的主节点上，没人能把失联节点标记为 NotReady。
生产环境必须做控制面高可用（≥3 master），否则故障是**静默**的。

---
---

## 第三部分：三天复习计划

| 时间 | 做什么 |
|---|---|
| **今晚** | 做完自测题，把答错的题圈出来 |
| **明天（10/3）** | 只做答错的那几题，仍然不看答案 |
| **10/5** | 全部 15 题重做一遍 |
| **10/8** | 合上所有资料，对着空气讲一遍第四部分的 15 分钟版本 |

**为什么要间隔复习**：记忆在"快要忘掉的时候"重新提取一次，留存率最高。
一次性看十遍，不如隔三天各看一遍。

---
---

## 第四部分：技术要点成稿（用你自己的数据）

### 4.1 集群搭建（约 30 秒）

> "我用 kubeadm 搭了一个一主一从的 Kubernetes 集群，版本 1.23.6，跑在 CentOS 7 上。
> 容器运行时用的是 Docker，CNI 是 flannel，Pod 网段 10.244.0.0/16。
> 控制面组件 etcd、kube-apiserver、controller-manager、scheduler
> 都是以静态 Pod 的形式跑在主节点上的。"

### 4.2 故障现象（核心，约 2 分钟）

> "我做了一次主节点故障演练。先在主节点上创建一个 nginx 的 Deployment，
> 确认它被调度到了工作节点 k8s-node1 上，拿到 10.244.1.12 这个 IP。
> 然后把主节点整机断电。
>
> 断电之后我在 node1 上观察到四个现象：
> 第一，那个 nginx 容器**还在正常运行**，容器 ID 都没变；
> 第二，curl 主节点的 6443 端口，报的是 **Connection timed out** 而不是 refused，
> 说明是整机不可达，不是单个服务停了；
> 第三，kubelet 进程仍然是 active；
> 第四，kubelet 日志里每 **10 秒**报一次 lease 更新失败。
>
> 这四条合起来说明一件事：**控制面和数据面是分离的**。
> 业务容器的运行只依赖节点上的 kubelet 和容器运行时，不依赖 apiserver。"

### 4.3 一个反直觉的发现（加分项，约 1 分钟）

> "还有一个我觉得挺有意思的点。按教科书说法，节点失联 40 秒后应该被标记 NotReady。
> 但我查了事件，**整个断电过程中一条 NodeNotReady 事件都没有**。
>
> 原因是负责标记 NotReady 的 node controller 跑在 kube-controller-manager 上，
> 而它就在被我关掉的那台主节点里——**监控者和管理对象一起挂了**。
>
> 这让我意识到，单主集群最危险的地方不是 etcd 单点，
> 而是**故障可能是静默的**：控制面挂了，健康检查机制同时失效，
> 你可能连'发生了故障'都不知道。这也是生产环境必须做控制面高可用的根本原因。"

### 4.4 恢复与容器自愈（约 1 分钟）

> "把主节点开机之后，集群在几分钟内自动恢复了，两个节点都是 Ready。
> 因为所有集群状态都持久化在 etcd 里。
>
> 我还额外做了一个测试：在 node1 上手动 kill 掉那个 nginx 容器。
> kubelet 立刻把它重新拉起来了，**容器 ID 变了、容器名后缀从 _0 变成 _1，
> 但 Pod 的 IP 没变**。这说明容器重启时 Pod 的网络沙箱是被复用的——
> Pod 是外壳，容器是里面的东西。
>
> 所以 kubelet 是有本地自愈能力的，但它感知不到期望状态的变化，
> 不能扩缩容、不能调度新 Pod。"

### 4.5 生产思考（收尾，约 30 秒）

> "这次演练让我意识到几件事：单主集群的 etcd 是单点，必须做定时快照备份；
> 控制面至少要 3 个 master 才能容忍单点故障；
> 另外还要有**独立于集群之外**的监控手段——
> 因为集群自身的健康检查在控制面故障时是不可靠的。"

---
---

## 附：今晚学到的 5 个可迁移排查技能

这五条不限于你这套集群，换任何 K8s 环境都能用。

| # | 技能 | 命令 / 要点 |
|---|---|---|
| 1 | 排查集群级组件先用 `-A` | `kubectl get pods -A \| grep xxx`，不要假设组件都在 `kube-system` |
| 2 | 查容器重启原因看 Events | `kubectl describe pod <name> \| tail -25`，`(x2 over 19m)` 就是线索 |
| 3 | 看节点级问题用 journalctl | `journalctl -u kubelet -n 50 --no-pager` |
| 4 | 用报错类型判断故障层次 | `refused` = 服务停了；`timeout` = 整机不可达 |
| 5 | 会读容器名 | `k8s_<容器>_<Pod>_<ns>_<UID>_<重启次数>` |

---

## 一句总结

今晚你没有拿到"技能"，你拿到的是**素材**：
一条别人背不出来的实测结论、五个具体到能唬住人的细节、五个可迁移的排查方法。

素材本身不值钱，**能讲出来才值钱**。这份文档的作用，就是帮你把素材变成能讲出来的话。

---
---

## 附录 B：etcd 速查

### 是什么

Kubernetes 的**数据库**。整个集群的状态（所有 API 对象）都存在里面。
**apiserver 本身是无状态的**——它只做鉴权、校验和转发，真正"记住事情"的是 etcd。

### 访问路径

```
kubectl / kubelet / controller-manager / scheduler / kube-proxy
                            ↓  全部通过
                      kube-apiserver
                            ↓  唯一能读写 etcd 的组件
                          etcd
```

其他组件**都不能直接访问 etcd**。这是 K8s 的硬性架构约束。

### 在本集群中的形态

| 项目 | 值 |
|---|---|
| 形态 | 静态 Pod，名字 `etcd-k8s-master` |
| 命名空间 | `kube-system` |
| 数据目录 | `/var/lib/etcd/` |
| 清单文件 | `/etc/kubernetes/manifests/etcd.yaml` |
| 端口 | 2379（客户端）/ 2380（etcd 节点之间） |

### 常用命令（本集群为 Docker 运行时）

```bash
# 健康检查
sudo docker exec $(sudo docker ps -qf name=etcd) sh -c 'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health'

# 成员列表
sudo docker exec $(sudo docker ps -qf name=etcd) sh -c 'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list'

# 数一数 etcd 里存了多少个 key（= 集群的全部状态对象）
sudo docker exec $(sudo docker ps -qf name=etcd) sh -c 'ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry --prefix --keys-only' | grep -v '^$' | wc -l

# 数据目录大小
sudo du -sh /var/lib/etcd/

# 备份（删主节点前必做）
sudo docker exec $(sudo docker ps -qf name=etcd) sh -c '
  ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save /var/lib/etcd/snapshot.db'
```

### 为什么单主危险：Raft 需要多数派

etcd 用 **Raft 一致性协议**，写入必须获得**多数派（quorum）**同意：

| 节点数 | 多数派 | 可容忍故障 |
|---|---|---|
| 1 | 1 | **0 个** |
| 3 | 2 | 1 个 |
| 5 | 3 | 2 个 |

生产环境必须用**奇数**个（3 或 5）。2 个或 4 个没有额外收益——
2 个节点的多数派也是 2，挂一个就不可写。

### 为什么 etcd 快照是唯一完整的备份

备份 YAML 只能恢复"你手动写过的对象"，恢复不了：

- Service 自动分配的 ClusterIP
- controller 生成的 ReplicaSet / Pod 名字与 UID
- Secret / ConfigMap 的当前值
- 所有对象的 resourceVersion 与创建时间

**etcd 快照 = 整个集群在某一刻的完整状态**，这是 K8s 灾难恢复的唯一完整方案。

### 与 2026-10-02 实验的关联

```
主节点关机  →  etcd 一起消失  →  集群"失忆"
                                但容器照跑（运行不依赖 etcd）
主节点开机  →  etcd 从 /var/lib/etcd/ 读回状态  →  集群自愈
```

**"集群能自愈"的根本原因不是 kubelet 聪明，而是 etcd 把状态持久化了。**

这也补上了 B15 题的最后一块拼图：单主集群最危险的地方 =
「etcd 是单点」+「健康检查机制同时失效」。

---

## 2026-10-19 补充：NAT 迁移后集群重建（离线恢复全链路）

### 事故经过（一句话版）

VMware 桥接网络依赖物理网卡混杂模式，换 WiFi 后失效 → 两 VM 失联 → 挂起恢复时
**节点无 NTP、时钟漂移** → apiserver 证书 TLS 校验失败 → 控制面周期性崩溃。
修好网络后集群已病危，只能重建：kubeadm reset → 换个网段（192.168.245.x）init → 加回 node1。

### 四个可复用经验

1. **桥接 vs NAT 的适用边界**：bridged 让 VM 像"局域网里的一台真机"，但 WiFi
   网卡不支持混杂模式，所以桥接天然不适用于无线环境；NAT 借宿主上网，与物理
   链路解耦，迁移环境永远可用。**技术问答可答"虚拟网络模式选择与物理介质的关系"**。

2. **挂起/恢复虚拟机必须配 NTP**：VM 挂起期间时钟停走，恢复后无 NTP 同步就会
   漂移；TLS 证书校验卡的是"当前时间在 NotBefore/NotAfter 之间"，漂移几分钟
   照样能让控制面全崩。**chronyd + ntp.aliyun.com 是虚拟机标配**。

3. **镜像名不匹配 ≠ 拉不到，可以 docker tag 本地复用**：node1 缓存的是
   `registry.aliyuncs.com/google_containers/kube-proxy:v1.23.17`，而 addon 要的是
   `registry.k8s.io/kube-proxy:v1.23.17`——`docker tag` 一打，零流量复用。
   **技术问答可答"镜像仓库镜像/加速器的实际原理"**。

4. **kubeadm join 没有 --image-repository flag**（v1.23 实测报 unknown flag）；
   换镜像仓库要靠 init 时指定或 ClusterConfiguration，join 阶段只能靠
   `--ignore-preflight-errors=ImagePull` + 本地 tag。**细节准确性是技术问答加分项**。

### kubeadm init 国内离线参数（本次实战验证）

```bash
# 镜像名对齐本地缓存，跳过 registry.k8s.io 直连；
# pod-network-cidr 必须与 flannel net-conf 的 Network 一致；
# apiserver-advertise-address 写新网段地址
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.23.17 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.245.133 \
  --ignore-preflight-errors=ImagePull
```

flannel manifest（v0.28.9，与镜像缓存对齐）归档在本仓库 `k8s/kube-flannel.yml`。
