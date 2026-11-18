# 03 · Kubernetes

> 核心一句话：**K8s 不执行命令，它持续把"实际状态"往"期望状态"上拉。**
> 所以 `kubectl set image` 之后没人"去部署"，是控制器自己发现了差异。

---

## 0. 一分钟版本

> 我用 kubeadm 搭了单主集群（v1.23.17 + docker + flannel），用 Deployment + Service(NodePort) 部署应用，配了 readiness/liveness 探针。做过一组对照实验：同一份"进程能起来但 80 端口不监听"的坏镜像，**不配探针**时 K8s 判定发布成功（`successfully rolled out`），但外部访问全是 000；**配上探针**后新 Pod 卡在 0/1、`rollout` 超时，而旧 Pod 继续服务，外部访问全程 200。这就是"探针把坏版本挡在流量之外"的完整证据。

---

## 一、原理层

### 1.1 声明式与控制循环

**命令式**：`docker run` —— 我说一步，它做一步，做完就完事。
**声明式**：`kubectl apply` —— 我说"我要 2 个副本、镜像是什么"，然后**有人持续保证它成立**。

```
控制循环（reconcile loop）：
  读期望状态（spec）
  读实际状态（status）
  有差异 → 执行动作消除差异
  然后无限重复
```

**这条机制解释了四件事**：

| 现象 | 原因 |
|---|---|
| 删掉一个 Pod，几秒后自己又起来了 | RS controller 发现副本数不足，补一个 |
| 改 Deployment 的镜像，没人执行"部署命令" | controller 发现模板变了，走滚动流程 |
| 手动改 Pod 的镜像会被改回去 | Pod 归 RS 管，RS 的模板没变，会被纠正（或直接重建） |
| 节点宕机后 Pod 会漂移 | controller 发现副本不足，在新节点重建 |

**代价**：控制循环是**异步**的。所以 `kubectl apply` 返回成功 ≠ 已经生效。要等就得 `rollout status`。

### 1.2 Deployment / ReplicaSet / Pod 三层为什么这么设计

| 对象 | 职责 | 为什么需要这一层 |
|---|---|---|
| **Pod** | 最小调度单元，一组共享网络命名空间的容器 | 容器本身不是调度单位（要一起调度的东西需要一个"壳"） |
| **ReplicaSet** | 保证 N 个**完全相同**的 Pod 在跑 | 副本管理是独立职责，和"版本"无关 |
| **Deployment** | 管理**版本**：滚动、回滚、历史 | 一个 Deployment 下有多个 RS，每个 RS 是一个版本 |

**关键关系**：

```
Deployment（管版本）
  ├── RS rev=22（旧版本，副本 0，保留）
  ├── RS rev=23（当前版本，副本 2）
  └── RS rev=24（新版本，副本 1，但 Pod 不 Ready）
```

**为什么不能只有 Deployment 和 Pod**：如果 Deployment 直接管 Pod，滚动更新时就没法"同时保留新旧两批 Pod"——需要 RS 作为**版本的载体**。

### 1.3 三种探针（这是 K8s 模块的核心）

| 探针 | 失败后果 | 什么时候用 |
|---|---|---|
| **livenessProbe** | **重启容器** | 检测"进程卡死但没退出"——重启能恢复的场景 |
| **readinessProbe** | **从 endpoints 摘掉，不接流量**（容器继续跑） | 检测"现在能不能服务"——启动预热、依赖未就绪 |
| **startupProbe** | 失败则重启；**成功前禁用其他两种探针** | 慢启动应用（如 JVM），避免 liveness 过早杀掉 |

**最容易被问倒的对比**：

| | liveness 失败 | readiness 失败 |
|---|---|---|
| 容器 | **被重启** | **继续运行** |
| 流量 | 会中断（因为重启） | 被摘掉，其他副本接 |
| 语义 | "这个进程没救了，重来" | "它暂时不能服务，先别给它流量" |

> **一句话记法**：**liveness 管"活着"，readiness 管"能干活"。**
> 活着但不一定能干活（比如数据库还没连上），所以两个都要有。

**探测方式三种**：

```yaml
readinessProbe:
  httpGet:                    # HTTP 探测：状态码 200-399 算成功
    path: /
    port: 80
  initialDelaySeconds: 3      # 容器启动后等多久开始探
  periodSeconds: 5            # 多久探一次
  timeoutSeconds: 2           # 单次超时
  failureThreshold: 3         # 连续失败几次判定失败
  successThreshold: 1         # 连续成功几次判定成功
```

另外两种：`tcpSocket`（端口能连上）、`exec`（容器内执行命令，退出码 0）。

**配置陷阱（高频追问）**：

| 陷阱 | 后果 |
|---|---|
| `initialDelaySeconds` 太短 + 应用启动慢 | 容器还没起完就被判失败 → 反复重启（CrashLoopBackOff） |
| `timeoutSeconds` 太短 | 高负载时偶发超时 → 无谓重启 |
| 只配 liveness 不配 readiness | 滚动更新时新 Pod 一 `Running` 就进 endpoints，**此时应用可能还没准备好** → 用户看到 502 |
| liveness 探测依赖外部服务 | 数据库抖一下 → 全部 Pod 一起重启 → **雪崩** |

> **最后一条是真实生产事故的常见根因**：liveness 只该检测"自己还能不能活"，不该检测"依赖是否健康"。

### 1.4 Service / endpoints / kube-proxy 三层链路

这是"Service 不通"排障的理论基础。

```
Service（ClusterIP / NodePort）
   ↓ selector 匹配 Pod
Endpoints（只包含 Ready 的 Pod IP:Port）      ← 关键！
   ↓ endpoints controller 维护
kube-proxy（每个节点上）
   ↓ 把 Service 的访问规则翻译成
iptables / ipvs 规则
   ↓
真正转发到 Pod
```

**每一层的职责和失效表现**：

| 层 | 谁维护 | 失效表现 |
|---|---|---|
| Service | 你创建的 | selector 写错 → endpoints 永远空 |
| **Endpoints** | endpoints controller（看 Pod 的 Ready 条件） | Pod 不 Ready → **被摘掉**，endpoints 为空 |
| kube-proxy 规则 | kube-proxy（watch Service/Endpoints 变化） | 规则没收敛 → 短暂 000（这就是"假失败"的来源） |
| Pod 网络 | flannel/CNI | 跨节点不通 → NodePort 000 但 Pod 全 1/1 |

**这条链路最重要的一句话**：

> **Service 不直接指向 Pod，指向 endpoints。而 endpoints 只包含 Ready 的 Pod。**
> 所以"探针不过" = "不进 endpoints" = "不接流量" = "用户无感"。

这正是 D2 故障演练的机制基础。

### 1.5 滚动更新的数学

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 最多超出期望副本数几个
    maxUnavailable: 0    # 最多允许几个副本不可用
```

`replicas: 2` + `maxSurge: 1` + `maxUnavailable: 0` 的实际行为：

| 步骤 | 总 Pod 数 | 说明 |
|---|---|---|
| 初始 | 2（旧） | — |
| 起 1 个新 | **3**（2 旧 + 1 新） | 允许超 1 个 |
| 新 Pod Ready | 3 | 进 endpoints |
| 删 1 个旧 | 2（1 旧 + 1 新） | 不允许不可用，所以必须先 Ready 才能删 |
| 起第 2 个新 | 3 | — |
| 新 Pod Ready → 删最后 1 个旧 | 2（全新） | 完成 |

**新 Pod 永不 Ready 时会怎样**（这是 D2 的实测）：

```
起 1 个新（3 个）→ 它一直不 Ready
  → maxUnavailable: 0 不许删旧 Pod（删了就只剩 1 个可用，违反约束）
  → 永远卡在 3 个：2 旧 + 1 坏
  → rollout status 一直等到 90 秒超时
```

**结论：`maxUnavailable: 0` 用"多占资源"换"零中断"，代价是坏版本会让发布**永久卡住**，必须人工介入（回滚）。**

### 1.6 revision 编号机制（最容易讲错的一块）

**编号规则**：
- Deployment 的 **spec 变更**（pod template 变了）→ 新建或复用 RS → revision **+1**
- revision 号**只增不减**，**永不回收**

**`rollout undo` 的实际行为**（实测）：

| 项 | 结果 |
|---|---|
| 操作 | `kubectl rollout undo deploy/hello` |
| revision | **15 → 17**（不是 16） |
| RS 的 CREATED 时间 | **不变**（复用了同一个 RS 对象） |
| 老编号 15 | **永久失效** |

**为什么 15 会失效**：因为那个 RS 对象的 `deployment.kubernetes.io/revision` 注解被改成了 17。一个 RS 只保留**最新**的编号。

所以：

```bash
kubectl rollout undo deploy/hello --to-revision=15
# error: unable to find the specified revision
```

> **这句要能说出来**：**`undo` 找的是门牌号，不是房子。房子（RS 对象）还在、镜像还在跑，但门牌号已经换成 17 了。**

**`revisionHistoryLimit` 的真实行为**（实测）：

| 项 | 结果 |
|---|---|
| 从 10 改成 1 | RS 从 3 个变 2 个，**一次删掉 REV 2/3/5** |
| 清理依据 | **按 revision 号**，不是按创建时间（REV 10 创建更晚却被删） |
| 改回 10 会恢复吗 | **不会**。删掉的对象不会复活 |
| 当前 RS 占额度吗 | **不占**。limit 限制的是"副本为 0 的历史 RS" |

**回滚窗口的计算**：

```
可回滚天数 ≈ revisionHistoryLimit ÷ 每天发布次数
```

每天发 10 次、limit=10 → 只覆盖 **1 天**。每天发 50 次 → 约 **5 小时**。

> **这条是"看似配了 10 个版本很安全"的破除**：窗口不是永久属性，**是被发布次数持续消耗的**。

### 1.7 两条回滚通道的本质区别

| 通道 | 依据 | 优点 | 失效条件 |
|---|---|---|---|
| A：`rollout undo` | **revision 编号** | 快，不用记 tag | 目标 revision 已被清理（`limit` 删掉）或编号已重排 |
| B：`set image ...:TARGET_TAG` | **registry 里的 tag** | 不受 RS 历史限制 | registry 里那个 tag 没了 |

**为什么要有通道 B**：`undo` 依赖"RS 对象还在"，而 RS 会被 `limit` 清理。通道 B 只依赖**制品还在**——这就是为什么"tag 不可变"是回滚的地基（见 02 模块）。

### 1.8 RBAC（CI 为什么需要受限身份）

| 概念 | 是什么 |
|---|---|
| **ServiceAccount** | Pod/程序用的身份（不是人） |
| **Role / ClusterRole** | 一组权限规则（`资源 + 动词`） |
| **RoleBinding / ClusterRoleBinding** | 把规则绑到身份上 |

**我做的事**：给 CI 建了一个 `ci-deployer` ServiceAccount，只授予 `deployments` 的 `get/list/watch/patch/update`。

**为什么**（这题必问）：

> CI 的凭据是**最容易泄露的**——它在流水线日志、变量、Runner 机器上。如果给它 cluster-admin，
> 一个被攻破的 Runner 就能删掉整个集群。**最小权限是 CI 接入 K8s 的第一原则。**

**验证方法**：

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:ci-deployer
kubectl auth can-i delete pods --as=system:serviceaccount:default:ci-deployer   # 应该是 no
```

### 1.9 flannel 与 Pod 网络

**flannel 的 vxlan 模式**：
- 每个节点有一个 `flannel.1` 虚拟网卡（vxlan 隧道端点）
- Pod 网段（如 `10.244.1.0/24`）通过 vxlan 封装在节点间传输
- 同节点 Pod 通信走本地网桥，跨节点走 vxlan 隧道

**关键坑（实测）**：**NetworkManager 会接管 `flannel.1` 并把 vxlan 设备置 DOWN**，表现为：

| 现象 | 实际原因 |
|---|---|
| `kubectl get pods` 全 `1/1 Running` | Pod 本身没问题 |
| NodePort 访问 **000** | 跨节点 Pod 网络断了 |
| `ip link show flannel.1` 显示 DOWN | NM 干的 |

**为什么重启 flannel Pod 修不好**：flannel 启动时如果发现 `flannel.1` **已存在就复用**，不会重新 `LinkSetUp`。必须 `ip link del flannel.1` 后重启 flannel 才会重建。

**根治**：让 NetworkManager 忽略这个设备

```
# /etc/NetworkManager/conf.d/99-k8s-unmanaged.conf
[keyfile]
unmanaged-devices=interface-name:flannel.*;interface-name:cni0;interface-name:docker0
```

---

## 二、项目层

### 2.1 探针对照实验（这是整个 K8s 模块最硬的数据）

**实验设计**：同一份坏镜像（nginx 监听 8080 而不是 80，进程能正常跑），分别部署到"无探针"和"有探针"的 Deployment。

| 配置 | Pod 状态 | rollout 结果 | 外部 HTTP | 判定 |
|---|---|---|---|---|
| **无探针** | 2 个全 `1/1 Running` | `successfully rolled out` | **全 000** | **假成功** |
| **有探针** | 新 Pod `0/1` → `CrashLoopBackOff`，旧 Pod `1/1` | `timed out` | **全 200** | 拦截成功 |

**这两行是整个 W4 的核心结论**：

> **K8s 默认认为"容器进程起来了"就等于"能服务"。这个假设在真实应用里经常不成立**
> （端口没监听、依赖没连上、健康检查接口没实现）。
> **探针是把这个假设变成事实的唯一手段。**

### 2.2 端到端故障演练（W6-D2，完整链路版）

| 时间点 | 事件 | 证据 |
|---|---|---|
| 注入坏版本 | Dockerfile 加 `COPY nginx-bad.conf`，nginx 监听 8080 | 部署前实测：容器 Up、8080 通、80 拒绝 |
| 流水线 | build/test/package 全绿 | 单测和构建都放行（**故意不碰 index.html**） |
| deploy-job | **failed**，`timed out waiting for the condition` | 90 秒超时 |
| K8s | 新 Pod `0/1`、`RESTARTS 2`、REV 24、READY 空 | 探针拦住 |
| 旧版本 | REV 23、`DESIRED 2`、`READY 2`，endpoints 只有旧 Pod IP | 旧 Pod 撑流量 |
| **用户** | **HTTP 200 全程，页面仍是上一版** | **核心结论** |
| 监控 | 4 个 target 全 up、Firing=0 | **告警缺口暴露** |
| 补规则 | `PodNotReady` → `firing`，`activeAt` 23:25 | 补上后能抓到 |
| 回滚 | `TARGET_TAG=7d75d679`（通道 B） | 镜像回到好版本，2/2，坏 Pod 消失 |

### 2.3 其他实测

| 项 | 数据 |
|---|---|
| 集群 | kubeadm **v1.23.17**，docker **26.1.4**，flannel vxlan |
| 节点 | master `192.168.245.133` + node1 `192.168.245.150` |
| Service | NodePort **30863** → 容器 80 |
| 副本 | 2（`maxSurge: 1` / `maxUnavailable: 0`） |
| CI 身份 | ServiceAccount `ci-deployer`，只能操作 deployments |
| 当前 RS 数 | 3 个（REV 21/22/23），limit=10 但编号已到 24 |

### 2.4 踩过的坑

| # | 坑 | 根因 | 修法 |
|---|---|---|---|
| 1 | flannel 被 NetworkManager 接管 | NM 把 vxlan 设备置 DOWN | NM 忽略配置 + `ip link del flannel.1` 重建 |
| 2 | `rollout status` 成功但冒烟 000 | endpoints/iptables 收敛有秒级延迟 | 冒烟重试 6 次 |
| 3 | `maxUnavailable: 0` 导致死锁 | 新 Pod 不 Ready 时不许删旧 Pod | 设计如此，靠回滚解 |
| 4 | `undo --to-revision=15` 报找不到 | revision 编号被重排（15→17） | 用通道 B 按 tag 回退 |
| 5 | `limit` 改成 1 后历史全删 | controller 真删 RS 对象 | 改回不复活，注意窗口 |

---

## 三、排障层（现场题）

### 3.1 「Pod 一直 Pending」

```bash
kubectl describe pod <pod> | tail -20        # 看 Events，这是关键
```

| Events 里的关键词 | 原因 | 处置 |
|---|---|---|
| `Insufficient cpu/memory` | 节点资源不足 | 加节点 / 降 requests |
| `node(s) had taint` | 节点有污点 | 加 toleration 或换节点 |
| `node(s) didn't match node affinity` | 亲和性不满足 | 改亲和规则 |
| `persistentvolumeclaim not found` | PVC 没绑定 | 查 StorageClass / PV |
| `no nodes available` | 节点全 NotReady | 先修节点 |

**注意区分**：`Pending` 是**调度失败**，容器根本没起来，所以 `kubectl logs` 没东西可看。

### 3.2 「ImagePullBackOff / ErrImagePull」

```bash
kubectl describe pod <pod> | grep -A5 "Failed to pull\|Failed to pull image"
```

| 报错 | 原因 |
|---|---|
| `manifest unknown` | **tag 不存在**（不是网络问题） |
| `unauthorized` / `authentication required` | 私有 registry 没配 `imagePullSecrets` |
| `no such host` | DNS 解析不了 registry 地址 |
| `x509: certificate signed by unknown authority` | 私有 registry 用自签证书 |
| `TLS handshake timeout` | 网络不通 / MTU 问题 |

**验证顺序**：

```bash
# 1) tag 真的存在吗（在能访问 registry 的机器上）
curl -s http://192.168.245.150:5000/v2/devops/hello/tags/list

# 2) 节点上能拉吗（绕过 K8s 直接试）
docker pull 192.168.245.150:5000/devops/hello:<tag>

# 3) imagePullSecret 配了吗
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'
```

### 3.3 「CrashLoopBackOff」

```bash
kubectl logs <pod> --previous        # 看上一次崩溃的日志（关键！）
kubectl describe pod <pod> | grep -A5 "Last State"
```

**`--previous` 是重点**：容器重启后当前实例可能是刚起的，日志是空的，必须看 `--previous`。

| 现象 | 原因 |
|---|---|
| 启动就退出，日志有 panic/exception | 应用错误 |
| 退出码 137 | OOM（内存超限）或被杀 |
| 退出码 1，日志无内容 | 启动命令写错 / 配置缺失 |
| 反复重启但日志正常 | liveness 探针太激进（`initialDelaySeconds` 太短） |

### 3.4 「Pod Running 但 0/1（不 Ready）」

**这是 D2 演练的现场，也是最容易被误判的一类**。

```bash
kubectl get pods -l app=hello              # READY 列 0/1
kubectl describe pod <pod> | grep -A8 "Readiness probe failed"
kubectl get endpoints hello                # 看这个 Pod IP 在不在里面
```

| 检查 | 预期（故障时） |
|---|---|
| `READY` | `0/1` |
| `RESTARTS` | 可能为 0（readiness 失败**不重启**） |
| `describe` 里 | `Readiness probe failed: ... connection refused` |
| `endpoints` | **不包含这个 Pod IP** |
| 外部访问 | **200**（旧 Pod 还在） |

**判定逻辑**：

```
Pod Running + 0/1 + endpoints 里没有它 + 外部正常
  → 探针拦住了坏版本，服务由旧 Pod 撑着
  → 这不是"服务故障"，是"发布失败"
  → 处置：回滚，不是重启 Pod
```

> **最容易犯的错**：看到 0/1 就去 `kubectl delete pod`。删了会被 RS 重建，还是 0/1。**根因在镜像/配置，不在 Pod。**

### 3.5 「Service 访问不通」

按四层查（从下往上，不要跳）：

```bash
# 第 1 层：Pod 自己好吗
kubectl get pods -l app=hello -o wide

# 第 2 层：endpoints 里有 IP 吗（最关键的一步）
kubectl get endpoints hello
#   空 → selector 不匹配 或 Pod 都不 Ready

# 第 3 层：Service 的 selector 和 Pod 的 label 对得上吗
kubectl get svc hello -o jsonpath='{.spec.selector}'
kubectl get pods -l app=hello --show-labels

# 第 4 层：kube-proxy 规则生成了吗
#   在节点上：
iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>
#   或 ipvsadm -Ln | grep <ClusterIP>
```

**判定表**：

| endpoints | Pod 状态 | 结论 |
|---|---|---|
| 空 | Pod `1/1 Running` | **selector 写错了**（label 不匹配） |
| 空 | Pod `0/1` | 探针没过，Pod 被摘掉 |
| 有 IP | 正常 | 问题在 kube-proxy 规则或网络层 |
| 有 IP | 正常，但只有部分节点不通 | 该节点的 kube-proxy / flannel 异常 |

**我踩过的那个特例**：`kubectl get pods` 全 `1/1`、endpoints 也有 IP，但 NodePort 访问 000。根因是 **flannel 的 vxlan 设备被 NetworkManager 置 DOWN**——Pod 网络跨节点断了。判据：`ip link show flannel.1` 显示 DOWN。

### 3.6 「节点 NotReady」

```bash
kubectl describe node <node> | grep -A10 "Conditions"
systemctl status kubelet
journalctl -u kubelet -n 100
```

| 常见原因 | 判据 |
|---|---|
| kubelet 挂了 | `systemctl status kubelet` 不是 active |
| 磁盘压力（DiskPressure） | `df -h` 看根分区 |
| 内存压力（MemoryPressure） | `free -h` |
| 网络插件异常 | `kubectl get pods -n kube-flannel` |
| 时间不同步 | `date` 与 apiserver 差太多会导致证书校验失败 |

> **时间同步这条特别值得说**：虚拟机挂起/恢复后时钟会漂移，TLS 证书校验失败，控制面直接崩。我在环境里装了 chrony 同步阿里云 NTP，就是为了防这个。

---

## 四、追问清单

**Q1：K8s 是命令式还是声明式？**
→ 声明式。你给的是期望状态，控制器持续对比实际状态并纠正。
→ 再追"那 `kubectl run` 呢"：那是命令式创建，但创建出来的对象仍然被控制器接管。**创建方式**和**运行机制**是两件事。

**Q2：Deployment 和 ReplicaSet 什么关系？为什么不能只有一层？**
→ Deployment 管**版本**（滚动/回滚/历史），RS 管**副本数**。一个 Deployment 下有多个 RS，每个 RS 代表一个版本。
→ 再追"为什么不能 Deployment 直接管 Pod"：滚动时需要**同时存在新旧两批 Pod**，需要 RS 作为"版本的容器"来区分它们。

**Q3：readiness 和 liveness 的区别？**
→ readiness 失败**摘流量但不重启**；liveness 失败**重启容器**。
→ 再追"只配一个行不行"：只配 liveness → 新 Pod 一起来就进 endpoints，可能还没准备好，用户看到 502。只配 readiness → 进程卡死时永远不会被重启。**两个都要。**

**Q4：`maxUnavailable: 0` 有什么代价？**
→ 零中断，但发布期间会**多占资源**（先起新的），且**坏版本会让发布永久卡住**——因为不许删旧 Pod，新 Pod 又永远不 Ready。
→ 再追"那怎么解"：靠回滚（两条通道）。这是"用发布卡住换服务不中断"的自觉取舍。

**Q5：为什么 `undo --to-revision=15` 会报找不到？**
→ 因为 undo 之后 revision 编号被重排了（15→17），那个 RS 对象的注解已经变成 17。**编号是门牌号，会被换掉。**
→ 再追"那怎么退到那个版本"：用通道 B——按 registry 里的 tag 重新 `set image`，不依赖 revision。

**Q6：`revisionHistoryLimit: 10` 是不是能保留 10 个版本？**
→ 不完全。它限制的是**副本为 0 的历史 RS** 最多 10 个，**当前 RS 不占额度**。而且它只保证"最多"，不保证"至少有"。
→ 再追"回滚窗口多长"：`limit ÷ 每天发布次数`。每天发 10 次就只有 1 天。

**Q7：Service 是怎么找到 Pod 的？**
→ Service 用 **selector** 匹配 Pod 的 label，endpoints controller 把匹配到且 **Ready** 的 Pod IP 写进 Endpoints 对象，kube-proxy 再把它翻译成 iptables/ipvs 规则。
→ 再追"Pod 不 Ready 会怎样"：被从 endpoints 摘掉，不接流量。**这就是探针保护流量的机制。**

**Q8：为什么 CI 要用受限的 ServiceAccount？**
→ CI 凭据最容易泄露（日志、变量、Runner 机器）。给 cluster-admin 等于把集群交出去。最小权限是底线。
→ 再追"怎么验证权限够不够"：`kubectl auth can-i --list --as=system:serviceaccount:...`。

**Q9：Pod 一直 Pending，你怎么查？**
→ `kubectl describe pod` 看 Events。Pending 是**调度失败**，容器没起来，`logs` 是空的。常见：资源不足、污点、亲和性、PVC 未绑定。
→ 再追"和 CrashLoopBackOff 区别"：Pending 是**没被调度**，CrashLoop 是**调度了但反复退出**。

**Q10：Pod Running 但 0/1，你会怎么做？**
→ 先看 `describe` 里的 readiness probe 报错和 `endpoints`。如果 endpoints 里没有它、外部访问正常，说明**探针拦住了坏版本**，服务由旧 Pod 撑着 → 这是**发布失败**，处置是**回滚**，不是重启 Pod。
→ 再追"为什么不能删 Pod"：删了 RS 会重建，还是 0/1。根因在镜像或配置。

**Q11：flannel 的 vxlan 是怎么工作的？**
→ 每个节点有 `flannel.1` 设备作为隧道端点，Pod 网段通过 vxlan 封装在节点间传输。同节点走本地网桥，跨节点走隧道。
→ 再追"如果它挂了会怎样"：Pod 全 Running 但跨节点不通，NodePort 访问 000。**这个现象特别容易误判成 Service 问题。**

**Q12：滚动更新时新旧 Pod 会同时接流量吗？**
→ 会。新 Pod Ready 后进 endpoints，此时旧 Pod 还在，两者同时接流量。这正是"零中断"的实现方式。
→ 再追"那会不会有新版本 bug 只影响部分用户"：会。这是滚动的固有特性——**所以需要监控发布时段的错误率**，而不是只看"发布成功"。

**Q13：`resources.requests` 和 `limits` 的区别？**
→ `requests` 是**调度依据**（节点要留够这么多才调度上去），`limits` 是**运行时上限**（超了 CPU 被限流、内存被 OOMKill）。
→ 再追"内存超 limits 会怎样"：容器被 OOMKilled，退出码 137，Pod 重启。**这块我没实测过**（见边界）。

**Q14：你集群是单 master，生产能用吗？**
→ 不能。单 master 是单点，etcd 挂了整个集群失去控制面。生产要 3 节点 etcd + 多 master。
→ 再追"那你的实验验证了什么"：验证了**工作机制**（调度、探针、滚动、回滚），没验证**高可用**。这两件事要分开说。

---

## 五、闭卷自测（10 题）

| # | 题 | 及格线 |
|---|---|---|
| 1 | 控制循环是什么？为什么 `apply` 成功不等于生效 | 说到"异步 + 持续对比" |
| 2 | Deployment/RS/Pod 三层各自职责 | 说到"RS 是版本的载体" |
| 3 | readiness 和 liveness 失败后果的差别 | 摘流量 vs 重启 |
| 4 | `maxUnavailable: 0` 为什么会死锁 | 不许删旧 Pod + 新 Pod 永不 Ready |
| 5 | Service → endpoints → kube-proxy 三层链路 | 能说出 endpoints 只含 Ready Pod |
| 6 | 为什么 `undo --to-revision` 会找不到 | 编号被重排，门牌号换了 |
| 7 | 两条回滚通道的依据和失效条件 | revision vs registry tag |
| 8 | Pod Running 0/1 的处置逻辑 | 判断是发布失败，处置是回滚 |
| 9 | flannel 挂了的现象和判据 | Pod 全 Running 但 NodePort 000，`ip link` 看 DOWN |
| 10 | 单 master 的局限 | 控制面单点，etcd 挂了失去控制 |

---

## 六、边界（必须主动划）

| 没碰过 | 说明 |
|---|---|
| **resources limits 实测** | yaml 里写过 `200m/128Mi`，但**没有实际制造过 OOMKilled 观察现象**。这是 W7-D1 取消后留下的洞 |
| **ConfigMap / Secret 实操** | 概念清楚（两种注入方式、更新差异、subPath 坑、base64 不是加密），但**没有在自己环境里做过** |
| Ingress | 只用过 NodePort（30000+ 端口，生产不用） |
| StatefulSet / Operator | 没做过有状态应用 |
| HPA 自动扩缩 | 没配过 |
| 多 master / etcd 高可用 | 单 master |
| Helm | 没用过，都是手写 yaml |
| 网络插件原理 | 会用 flannel，没读过 CNI 规范 |

> **上面两条加粗的是"能讲原理但没实测"的**。被问到时正确说法是：
> "这块我理解机制，但没有在自己环境里跑过，所以不敢说会。"
