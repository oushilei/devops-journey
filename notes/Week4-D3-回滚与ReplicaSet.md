# Week4 D3：回滚机制与 ReplicaSet（含我的六个问题）

> 时间：2026-10-28 23:24 - 10-29 00:10（东八区）
> 环境：k8s-master `192.168.245.133` + k8s-node1 `192.168.245.150`（kubeadm v1.23.17，docker 运行时）
> 应用：`devops/hello`（Deployment 2 副本 + NodePort 30863）

## 一、开场先修环境：VM 重启后的"假故障"

**现象**：`kubectl get nodes` 报 `The connection to the server 192.168.245.133:6443 was refused`。

**诊断**：`uptime` 显示 **up 2 min** —— master 虚拟机刚重启过，不是集群坏了。

**恢复顺序**（值得记住）：

```
VM 启动
 → kubelet 拉起静态 Pod（etcd / apiserver / scheduler / controller-manager）
 → apiserver 可用后，flannel / kube-proxy 才能被调度
 → flannel 写入 /run/flannel/subnet.env → CNI 可用
 → 业务 Pod 才能重建
```

全程 1-3 分钟。这期间 `connection refused` 是正常的。

**教训**：遇到 `connection refused`，**先看 `uptime` 和 `kubectl get nodes`**，别急着重建集群。

---

## 二、今天的核心知识：回滚

### 1. 回滚 = 切模板，不是重建镜像

`kubectl rollout undo` 只是把某个历史 RS 的 **Pod 模板**复制回 Deployment 的 `spec.template`，然后触发一次滚动。**不重新构建、不碰镜像**，所以是毫秒级。

### 2. 三种回滚命令

| 命令 | 作用 |
|---|---|
| `kubectl rollout undo deploy/hello` | 退回**上一个**版本 |
| `kubectl rollout undo deploy/hello --to-revision=6` | 退回**指定**版本 |
| `kubectl rollout history deploy/hello` | 看有哪些版本 |
| `kubectl rollout history deploy/hello --revision=N` | 看某版本改了什么 |

### 3. 回滚的风险（重要）

**回滚的是整个 Pod 模板，不是只回滚镜像。**

如果 `--to-revision=3` 指向的是"无探针"的版本，回滚会把探针一起去掉——**护栏被回滚掉了**。

所以回滚前必须看目标版本的完整配置：
```bash
kubectl rollout history deploy/hello --revision=3
```

### 4. 回滚的前提条件

| 前提 | 说明 |
|---|---|
| RS 还在 | 被 `revisionHistoryLimit` 清理掉就回不去 |
| 镜像还在 registry | RS 里只有镜像**地址字符串**，镜像被删/覆盖照样拉不到 |

**这就是 W3 学的"制品不可变"和这里"部署历史"必须配套的原因。**

---

## 三、ReplicaSet 深度（今天真正的知识缺口）

### 1. 三层关系

```
Deployment  hello（版本管理者）
   ├── RS-hello-8fc95585f   模板 A   副本 0   ← 历史存档
   ├── RS-hello-8b5477789   模板 B   副本 0   ← 历史存档
   ├── RS-hello-769b64d44d  模板 C   副本 0   ← 历史存档
   └── RS-hello-6f84f74d5d  模板 D   副本 2   ← 当前生效
          ├── Pod x7k2p
          └── Pod m9q4t
```

| 对象 | 一句话职责 | 类比 |
|---|---|---|
| Deployment | 管版本：哪个模板生效、怎么滚、留几版历史 | 项目经理 |
| ReplicaSet | 管数量：这个版本的 Pod 该有几个 | 班组长 |
| Pod | 跑容器 | 工人 |

### 2. RS 的核心机制（无限循环调数量）

```
每秒：用 selector 数一下"属于我"的 Pod 有几个
  if 实际 < 期望 → 创建 Pod
  if 实际 > 期望 → 删除 Pod
```

**只做"数量的自动维护"，不看健康。**

### 3. 谁负责什么（最容易混的三件事）

| 事件 | 谁负责 |
|---|---|
| 容器进程崩了 | **kubelet**（重启容器，Pod 不变） |
| Pod 被删 / 节点失联 | **RS**（补一个新 Pod） |
| 容器起来了但不健康 | **探针**（readiness 摘流量 / liveness 触发重启） |

### 4. 顺序：RS 先于 Pod

```
apply Deployment → Deployment 创建 RS（RS 里存模板）→ RS 创建 Pod
```

**不是"Pod 被记录成 RS"**，是 **RS 生出 Pod**。Pod 一出生，父 RS 就已经存在。

### 5. 版本变更时到底发生什么

```
改 template → 新建 RS-B → RS-B 创建新 Pod → 新 Pod 就绪 → 旧 RS-A 缩容到 0
                                                    ↓
                                        【旧 Pod 被删除，旧 RS 对象保留】
```

**保留的是 RS（模板），不是 Pod。** 历史版本 = 一堆副本数为 0 的 RS。

### 6. 只有 Deployment 用 RS

| 工作负载 | 用 RS？ | 历史机制 |
|---|---|---|
| Deployment | ✅ | ReplicaSet + revision 注解 |
| StatefulSet | ❌ 直接管 Pod | ControllerRevision |
| DaemonSet | ❌ 直接管 Pod | ControllerRevision |
| Job / CronJob | ❌ | 无 |

**为什么 StatefulSet 不用 RS**：RS 假设"Pod 可随意替换、彼此等价"；StatefulSet 的 Pod 有固定身份（`web-0` 名字、固定 DNS、独立 PVC），不能随便换，所以它直接管 Pod，更新按**逆序**逐个替换。

---

## 四、我的六个问题与答案（今天最有价值的部分）

### Q1：为什么我的 RS 不完整，缺了一些？

**不是缺，是编号被重排了。** 4 个 RS 一个没少，但编号是 `2,3,5,6`（缺 1 和 4）。

原因：`rollout undo` 如果发现"目标模板与某个历史 RS 完全相同"，会**复用那个 RS**（不新建对象），但给它分配新编号，老编号作废。

### Q2：RS 里面不应该是 yml 文件吗，为什么呈现形式和我想的不一样？

**RS 不是文件，是 API 对象**（etcd 里的一条结构化记录）。YAML 只是它的**文本序列化格式**。

同一对象的四种视图：

| 命令 | 视图 |
|---|---|
| `kubectl get rs` | 表格（挑几个字段做列） |
| `kubectl describe rs` | 人类可读摘要 |
| `kubectl rollout history --revision=N` | Pod 模板摘要（裁剪过） |
| `kubectl get rs -o yaml` | **完整 YAML** |

> **K8s 从来没存过你的 YAML 文件**，它存的是解析后的字段。

### Q3：我明明是复用了 RS，为什么会多出一个 7？

**"编号 +1"和"新建 RS"是两件独立的事**：

| 情况 | 新建 RS 对象？ | 编号 |
|---|---|---|
| 模板从未出现过 | ✅ 新建 | max+1 |
| 模板与历史 RS 完全相同 | ❌ 复用 | **也会 max+1**（老编号作废） |

判断依据是 `pod-template-hash`（对**整张 Pod 模板**算 hash：镜像+探针+环境变量+资源全算进去）。`a38117e5 + 探针` 这个组合从没出现过 → 必须新建 RS → 编号 7。

### Q4：Pod 满足什么条件时 K8s 会把模板记录成 RS？生产一台 node 上几十个 Pod，RS 只存 10 个够吗？不同 Pod 的不同 RS 怎么区分？

**（1）触发条件不是"Pod 满足什么"，而是 `spec.template` 变了。**

| 改什么 | 产生新 RS？ |
|---|---|
| image / 探针 / 环境变量 / resources / template.labels | ✅ |
| **replicas / strategy / revisionHistoryLimit** | ❌（不在 template 里） |
| Pod 挂了、被删、节点重启 | ❌（同一个 RS 补 Pod） |

**（2）`revisionHistoryLimit: 10` 是"每个 Deployment 各自 10 个"，不是集群级。**

一台 node 上 50 个 Pod 可能来自 20 个 Deployment → 20 × 10 = 200 个 RS，互不影响。

够不够看发布频率：每天 1-3 次 → 覆盖 3-10 天，够用；每天 50 次 → 只够半天，要调大。
成本很低（一个 RS 对象只有几 KB），真正占空间的是 Pod 和镜像。

**（3）三层区分机制**：

| 机制 | 作用 |
|---|---|
| 名字前缀 `<deploy名>-<hash>` | 人眼区分 |
| `ownerReferences` | K8s 管理（级联删除依据） |
| labels（`app` + `pod-template-hash`） | 查询/selector 精确匹配 |

### Q5：一个 Pod 刚被创建时不会记录成 RS，只有即将变动时才把原 Pod 记录成 RS 对吗？只有 Deployment 有 RS 吗？

**（1）顺序反了。** 不是"Pod 变成 RS"，是 **RS 先存在，再创建 Pod**。

**（2）对，只有 Deployment 用 RS**；StatefulSet/DaemonSet 用 ControllerRevision。

### Q6：RS、Deployment、Pod 到底是什么关系？

见上文"三层关系"和"RS 核心机制"。

---

## 五、实测数据（东八区）

### 回滚验证

| 步骤 | 命令 | 结果 |
|---|---|---|
| 发旧版本 | `set image ...:a38117e5` | `grep -c "版本：v2"` → **0** |
| 精确回滚 | `undo --to-revision=6` | `grep -c "版本：v2"` → **1** |
| 看历史 | `rollout history` | `2,3,5,6` → `2,3,5,7,8`（6 消失） |

### RS 全量（区分新建/复用）

| RS | revision | 镜像 | 副本 | CREATED（东八区） | 结论 |
|---|---|---|---|---|---|
| hello-6f84f74d5d | **8** | latest | 2 | **10-27 19:44** | 复用（老对象、新编号） |
| hello-769b64d44d | **7** | a38117e5 | 0 | **10-28 23:38** | 新建（新对象） |
| hello-8b5477789 | 2 | broken | 0 | 10-27 19:16 | 历史 |
| hello-8fc95585f | 3 | latest | 0 | 10-27 19:01 | 历史 |
| hello-98448bb44 | 5 | broken | 0 | 10-27 19:46 | 历史 |

**判断口诀**：CREATED 是刚才 → 新建；CREATED 很久以前但 REV 变了 → 复用。

---

## 六、认知修正与坑

1. **UTC 陷阱**：K8s API 返回的时间带 `Z` 后缀（UTC），引用时必须 +8。`2026-10-27T11:44:00Z` = **10-27 19:44 东八区**。
   换算命令：`date -d "2026-10-27T11:44:00Z" +"%F %T %Z"`
2. **`kubectl describe pod -l hello` 查不到**：`-l` 必须 `key=value`；裸 label 是"存在性检查"。
3. **"1/1 Running" ≠ 服务正常**：昨天 D2 已实测，无探针时 K8s 会把"进程起来了"当成"能接流量"。
4. **VM 重启后的 refused 是假故障**：先看 `uptime`。

---

## 七、技术要点

> "Deployment、ReplicaSet、Pod 是三层：Deployment 管版本（哪个模板生效、怎么滚、保留几版），RS 管副本数（selector 数 Pod，少了补多了删），Pod 跑容器。每次 template 变化产生新 RS，模板 hash 相同的会复用但编号重排，所以 revision 号会断档。回滚用 `--to-revision` 精确跳转，但要注意回滚的是整个模板——目标版本没探针的话，护栏会被一起回滚。而且 RS 只存部署配置，镜像得靠 registry 的不可变 tag。StatefulSet 不用 RS，因为它的 Pod 有固定身份，历史用 ControllerRevision 记录。"

---

## 八、遗留（明天补）

- `revisionHistoryLimit` 实验：设成 1 再发两版，看历史怎么被清
- K8s 回滚 vs W3-D4 手写 `TARGET_TAG` 回滚的对比结论
- **Shell 每日 1 题连续多天未做**（硬纪律③，运维岗硬门槛）
