# Week4 总结：K8s 部署（Deployment / 探针 / 滚动 / 回滚 / CI 接入）

> 周期：2026-10-27 ~ 2026-11-02（东八区）
> 环境：k8s-master `192.168.245.133` + k8s-node1 `192.168.245.150`，kubeadm v1.23.17，docker 26.1.4

---

## 一、目标与达成

| 原定目标 | 达成 |
|---|---|
| 应用部署上集群（Deployment + Service + NodePort） | ✅ |
| 滚动更新与探针（含"假滚动"对照实验） | ✅ **两轮对比，教科书级证据** |
| 回滚（history / undo / --to-revision） | ✅ 含 `--to-revision` 失效与编号重排两个坑 |
| 接进流水线（CI 操作 Deployment） | ✅ 但延迟 3 天（原定 10/30，实际 11/02） |
| 失败自动回滚 | ✅ 双通道设计（undo / 指定 tag） |
| 自测 + 周复盘 | ⚠️ 周复盘本次补上，**自测未做** |

**整体判断**：技术目标全部达成，**节奏目标未达成**（计划 6 天，实际跨 7 个自然日，其中 4 天无提交）。

---

## 二、计划 vs 实际

| 计划日 | 内容 | 实际完成 | 偏差 |
|---|---|---|---|
| D1 10/27 | 部署链路 | ✅ 10/27 | 按时 |
| D2 10/28 | 滚动更新 + 探针 | ✅ 10/27（当天追加） | 提前 |
| D3 10/29 | 回滚 | ✅ 10/28 23:24 ~ 10/29 00:10 | 按时 |
| D4 10/30 | 接进流水线 | ⚠️ **11/02** | **延迟 3 天** |
| D5 10/31 | 失败自动回滚 + README + k8s 入库 | ⚠️ 部分（双通道回滚已做，README 未同步） | 部分 |
| D6 11/01 | 自测 10 题 + 周复盘 | ❌ → 本次补复盘 | 延迟 |

**断更日**：10/30、10/31、11/01、11/03。前三天是 W4 尾段，11/03 是 W4 结束后第一天。

---

## 三、关键实测数据（技术问答可直接引用）

### 3.1 探针：本轮最有价值的数据

| 场景 | Pod 状态 | rollout | curl | 结论 |
|---|---|---|---|---|
| **无探针** + 坏镜像 | 2 个全 `1/1 Running` | **successfully rolled out** | **全 000** | **假成功**：K8s 把"进程起来了"当成"能服务" |
| **有探针** + 坏镜像 | 新 Pod `0/1` → `CrashLoopBackOff`，旧 Pod `1/1` | `timed out` | **全 200** | 探针把坏版本挡在流量之外 |

**这两行对比是整个 W4 的核心**：同一份坏镜像，配不配探针，结果从"全站挂"变成"用户无感"。

### 3.2 回滚窗口

| 操作 | 结果 |
|---|---|
| `revisionHistoryLimit` 10 → 1 | RS 从 3 个变 2 个，一次删掉 3 个历史（REV 2/3/5） |
| 之后每发一版 | 总数恒为 2，最老的被挤出 |
| 清理顺序判据 | **按 revision 号，不按创建时间**（REV 10 创建更晚却被删） |
| `--to-revision=10`（已删） | `error: unable to find specified revision 10 in history` |

**窗口计算**：`limit ÷ 每天发布次数` 天。每天 10 次 → 约 1 天；每天 50 次 → 约 5 小时。

### 3.3 回滚的两条通道

| 通道 | 寻址依据 | 实测 |
|---|---|---|
| `rollout undo` | **revision 编号** | 复用 RS：REV 15→17，**CREATED 不变**，老编号永久失效 |
| `set image ...:tag` | **registry 的 tag** | 能把"被删掉"的版本内容救回来，但分配新 REV |

**关键坑**：RS 对象还在、镜像正在跑，但 `--to-revision=15` 照样报 `unable to find` —— **`undo` 找的是门牌号，不是房子。**

### 3.4 坏镜像（监听 8080）的完整链路

| 环节 | 结果 |
|---|---|
| `test.sh` 三项目检 | ✅ 全过（只看静态内容） |
| `docker build` | ✅ 成功 |
| 容器状态 | ✅ Running，nginx 3 个进程 |
| 探针探 80 | ❌ `connect: connection refused` |
| 新 Pod | `0/1`，`RESTARTS 2`（liveness 反复重启） |
| 旧 Pod / 外部访问 | `1/1`，**HTTP=200** |
| `rollout status` | 90 秒超时，`maxUnavailable:0` 造成死锁 |

---

## 四、踩坑清单（本 W4 新增 7 条，全部真实踩过）

1. **flannel 被 NetworkManager 接管** → vxlan 设备置 DOWN、跨节点 Pod 网络全断，表现为 NodePort `000` 但 `kubectl get pods` 全 `1/1`。修：`conf.d/99-k8s-unmanaged.conf` 让 NM 忽略 + `ip link del flannel.1` 重建。
2. **flannel 重启 Pod 修不好已坏设备**：vxlan 已存在则复用、不再 `LinkSetUp`，必须删设备。
3. **`GIT_DEPTH=20` + git 1.8.3.1** → `expected shallow list`，**第二次构建起必挂**（空目录成功、已有对象必失败）。修：`GIT_DEPTH: 0`（是禁用浅克隆的开关，不是"深度 0"）。
4. **`set -e` + `CODE=$(curl ...)`** → curl 非 0 让脚本立即中止，**日志连 echo 都不打印**。修：`|| true`。
5. **`rollout status` 成功 ≠ 流量可达** → endpoints/iptables 收敛有秒级延迟，实测第 1 次 `000`、第 2 次 `200` → 假失败。修：重试 6 次。
6. **`kubectl expose --type=nodeport`** → 值大小写敏感，必须 `NodePort`。
7. **`kubectl describe pod -l hello`** → `-l` 必须 `key=value`，裸 label 是"存在性检查"。

---

## 五、能力盘点（现在会什么）

**能讲清机制 + 有实测数据支撑**：

1. Deployment / ReplicaSet / Pod 三层职责边界，以及"RS 先于 Pod 存在"的顺序
2. readiness vs liveness 的后果差异，以及"liveness 要迟钝、readiness 要敏感"的生产建议
3. 滚动策略 `maxSurge` / `maxUnavailable`，以及 `maxUnavailable:0` 的死锁条件
4. 回滚双通道 + 各自的失效条件 + 为什么脚本要记 tag 不记 revision
5. `revisionHistoryLimit` 与回滚窗口的换算
6. CI 操作 K8s 的权限收敛（ServiceAccount + Role，越权实测被拒）
7. 四个 CI/K8s 层面的真实故障模式（NM 抢管、浅克隆、set -e、收敛延迟）

---

## 六、薄弱点（W5/W6 要补）

| # | 缺口 | 严重度 | 说明 |
|---|---|---|---|
| 1 | **ConfigMap / Secret** | 高 | 完全没用过。生产里配置与镜像必须分离，改配置不能重新构建 |
| 2 | **resources limits 未验证** | 高 | yaml 里写了 `200m/128Mi`，但没测过超限会怎样（OOMKilled / CPU throttling） |
| 3 | **存储 PV / PVC** | 高 | 完全没碰（Prometheus 正好需要持久化，W5 会撞上） |
| 4 | **Ingress** | 中 | 只用了 NodePort（30000+ 随机端口，生产不用） |
| 5 | StatefulSet / DaemonSet / Job | 中 | 只提过概念，没有实操 |
| 6 | 命名空间隔离、taint/toleration | 中 | master 的 taint 只是被动遇到 |
| 7 | NetworkPolicy | 低 | 没碰 |
| 8 | 集群运维（升级 / etcd 备份 / 证书过期） | 中 | 面试常问"证书多久过期、怎么续" |
| 9 | **无自测** | 高 | W4 没做自测题，薄弱点无法被自己发现 |

---

## 七、三条最大收获

1. **"Pod Ready" 不等于 "服务可用"。** 前者是进程状态，后者要探针 + 流量验证才算。
2. **回滚能力是一个会持续缩小的窗口，不是永久属性。** 每发一版缩一格，而人往往以为它一直在。
3. **门禁的价值不在于它会不会红，而在于它红的时候人还愿不愿意信。** 假失败（收敛延迟）比真失败更容易毁掉一条流水线。

---

## 八、W5 方向

**监控与告警（Prometheus + Grafana + Alertmanager）**

选它的三个理由：
1. W3/W4 建的是**发布能力**，全是被动的——出事才动。监控是唯一能在**用户之前**发现问题的东西。
2. 监控能反过来验证 W4：探针失败、滚动卡住、Pod 重启，现在只能靠人肉 `kubectl get pods` 看。
3. 面试里 K8s 与监控几乎成对问（"怎么发现故障""告警怎么配""pull 模型为什么比 push 好"）。

**顺带补齐**：PV/PVC（Prometheus 需要持久化）、ConfigMap（配置与镜像分离）。
