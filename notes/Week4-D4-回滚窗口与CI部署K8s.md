# Week4 D4：回滚窗口、CI 接入 K8s、探针拦截实测

> 时间：2026-11-02（东八区）
> 环境：k8s-master `192.168.245.133` + k8s-node1 `192.168.245.150`，kubeadm v1.23，docker 运行时
> 应用：`devops/hello`（Deployment 2 副本 + NodePort 30863）
> 流水线：`devops/gitlab-ci-lab`，shell executor Runner，gitlab-ce 19.5.1

---

## 一、今晚做了什么（一句话）

把 W3 的 CI 从「单机 `docker run`」升级为「**操作 K8s Deployment**」，并用一个坏镜像验证了 **探针是唯一能拦住"构建成功但运行失败"的门禁**。

---

## 二、实验 1：回滚窗口实测

### 2.1 机制

`revisionHistoryLimit` 是 Deployment 自己的字段（**不在 `spec.template` 里**，改它不产生新 revision），限制「副本为 0 的历史 RS 保留几个」，超出的被 controller **真删除对象**。

**回滚窗口 = `revisionHistoryLimit ÷ 每天发布次数`（天）**

### 2.2 实测数据（limit 从 10 改为 1）

| 阶段 | RS 总数 | 现存 revision | 说明 |
|---|---|---|---|
| 初始（limit=10） | 3 | 10, 11, 12 | 基线 |
| `patch limit=1` 后 6 秒 | **2** | 11, 12 | 一次删掉 REV 10，保留「最近的旧版」+「当前」 |
| 发 `a3e53917` | 2 | 12, 13 | 新建 REV 13，REV 11 被挤出 |
| 发 `15e4195d` | 2 | 13, 14 | 新建 REV 14，REV 12 被挤出 |

**关键现象**：整个过程服务一直是 `HTTP=200`，**历史清理不影响线上**。

> 总数始终是 2，一次都没涨到 3 —— 这是**契约**，不是巧合。

### 2.3 清理是「按 revision 号」而不是「按创建时间」

改 limit 前，两个历史 RS 的时间戳：

| REV | CREATED(UTC) | |
|---|---|---|
| **10** | 12:14:**24** | ← 编号最小，但创建时间**更晚** |
| **11** | 12:14:**11** | ← 编号更大，但创建时间**更早** |

**被删的是 REV 10**，而它创建得更晚。

→ 如果按「创建时间升序」清理，被删的应该是 REV 11。**实际不是。**
→ 结论：**编号是"发布顺序的记账"，不是"对象年龄"。**（编号重排的后果，见 §3.2）

### 2.4 退不回已删版本

```bash
$ kubectl rollout undo deploy/hello --to-revision=10
error: unable to find specified revision 10 in history
```

**对照排查**：这不是 `Forbidden`（权限）、不是 `connection refused`（网络），而是"**在这份 history 里找不到这个编号**"——那个 RS 对象已经不存在了。

### 2.5 生产含义

| 发布频率 | limit=10 覆盖 | 该怎么做 |
|---|---|---|
| 每天 1-3 次 | 3-10 天 | 够用 |
| 每天 10 次 | **约 1 天** | 调到 30-50 |
| 每天 50 次 | **约 5 小时** | 靠**不可变 tag** 重新部署，不能只靠 `undo` |

**最危险的场景不是"回不去"，是"你不知道自己回不去了"**：周五发了 50 次，周五早上的版本早被挤出窗口；周一发现问题时，`undo` 只能退到周五晚上。

**成本视角**：一个 RS 对象只有几 KB，把 limit 调大几乎零成本；真正占空间的是 Pod 和镜像。**没有理由把 limit 设小。**

---

## 三、实验 2：两条回滚通道

### 3.1 通道对比

| | `rollout undo` | `set image ...:tag` |
|---|---|---|
| 按什么寻址 | **revision 号** | **registry 的 tag** |
| 真相源 | 集群里的 RS 历史 | registry 里的制品 |
| 失效条件 | RS 被 GC，**或编号被重排** | tag 被删/覆盖 |
| 速度 | 秒级 | 秒级（镜像已在节点上） |
| 服务中断 | 无（滚动） | 无（滚动） |
| 跨集群 | ❌ 编号是集群本地的 | ✅ tag 是全局的 |
| 是否产生"新版本" | 复用旧 RS，**编号重排** | **分配新 revision** |

### 3.2 实测：复用 RS 时编号会重排

CI 部署 `0e8dc0f4` 后执行 `rollout undo`：

| | 回滚前 | 回滚后 |
|---|---|---|
| RS 名字 | `hello-6cb86498d4` | `hello-6cb86498d4` **没变** |
| REV | **15** | **17** |
| CREATED | `12:47:48` | `12:47:48` **没变** |
| 镜像 | `92f170b6` | `92f170b6` |

**CREATED 没变 → 同一个对象被复用；编号 15 永久作废。**

> 判断"复用还是新建"看 `CREATED`，别看名字（名字 = Pod 模板的 hash，内容相同就同名）。

### 3.3 关键坑：内容还在，但编号没了

RS 对象**存在**、镜像**正在跑**，但：

```bash
$ kubectl rollout undo deploy/hello --to-revision=15
error: unable to find specified revision 15 in history
```

> `undo` 找的是「**门牌号**」，不是「**房子**」。编号一重排，老门牌号永久失效，哪怕房子还站在原处。

**所以 `undo` 有两个独立的失效条件**，报错**一模一样**：

| 原因 | 报错 |
|---|---|
| RS 对象被 limit 删掉 | `unable to find ... in history` |
| RS 对象还在，但编号被重排 | **同一个报错** |

**结论：回滚脚本必须记镜像 tag，不能记 revision 号。**（GitLab deployments API 返回的 `ref` 是 commit sha，正好就是 tag——这是设计好的对应。）

### 3.4 K8s 回滚 vs W3-D4 手写 `TARGET_TAG` 回滚

| 维度 | W3-D4 手写（docker run） | W4 K8s（rollout undo） |
|---|---|---|
| 记录存在哪 | CI 变量 / 人的脑子 | etcd 里的 RS + revision 注解 |
| 回滚动作 | `docker rm -f` + `docker run` | 复制历史模板 → 触发滚动 |
| 服务中断 | **有**（单容器，先杀后起） | **无**（多副本 + 滚动） |
| 速度 | 数秒～数十秒（含拉镜像） | 约 6 秒 |
| 谁能触发 | 人点 GitLab ▶ | **任何人 `kubectl`，无需审批** |
| 审计 | GitLab 有 job 记录 | 只有 etcd 事件，默认没人看 |
| 失败边界 | 镜像不在 registry → 回滚即挂 | **RS 被 GC → 回不去**；镜像不在 → 同样挂 |

**核心结论**：**这两者不是替代关系，是两道不同的闸门，生产上都要保留。**

- `TARGET_TAG` 解决「**回到某个旧制品**」——跨集群、跨时间有效，只要 registry 还在；
- `rollout undo` 解决「**把集群状态退回某个旧模板**」——快，但绑死在"这个集群还留着这段历史"上。

**两者共享同一个地基：registry 里的镜像必须不可变。** 如果 tag 会被覆盖，`undo` 回去照样拉起一个"旧地址、新内容"的镜像——那叫**假回滚**，比回不去更可怕。

**生产上真正的坑：两套账本会对不上**

| 事故场景 | 后果 |
|---|---|
| 有人绕过流水线直接 `kubectl set image` | K8s revision +1，CI 完全不知道 |
| CI 记录 deploy 成功，但 K8s 被探针拦住 | 账本说上线了，实际没生效 |
| 两个人同时 `undo` | 编号飞掉，说不清现在是第几版 |

**解法（成本从低到高）**：RBAC 收权限（人只能 `get`，不能 `patch`）→ 回滚也走流水线 → GitOps（集群状态以 Git 为准）。

> 今晚手敲了 6 次 `kubectl set image`，**在真实生产里每一次都是事故**——绕过审批、绕过记录，两本账必然对不上。

---

## 四、实验 3：坏镜像 + 探针拦截（今晚最重要）

### 4.1 坏镜像怎么造的

不改 `index.html`（否则 `test.sh` 会拦），改成让 nginx **监听 8080 而不是 80**：

```dockerfile
COPY app/nginx-bad.conf /etc/nginx/conf.d/default.conf
```

推送前先在 node1 上本地验证，确认"坏成预期的样子"：

| 检查 | 结果 |
|---|---|
| `test.sh` 三项目检 | ✅ **全过**（文件在、grep 命中、产物在） |
| `docker build` | ✅ 成功 |
| 容器状态 | ✅ `Up`，nginx 有 3 个进程 |
| **80 端口**（探针探的） | ❌ 连不上 |
| 8080 端口（实际监听） | ✅ 有响应 |

### 4.2 流水线表现

| 阶段 | 结果 | 为什么 |
|---|---|---|
| test-job | ✅ 绿 | `test.sh` 只看**静态内容**，容器行为它看不见 |
| package-job | ✅ 绿 | `docker build` 成功 |
| deploy-job | ❌ **红** | `error: timed out waiting for the condition`（90 秒超时） |

### 4.3 集群现场

```
NAME                     READY   STATUS    RESTARTS      AGE     IP            NODE
hello-7b47d54b7b-ls5vx   1/1     Running   0             7m37s   10.244.1.44   k8s-node1
hello-7b47d54b7b-sccbr   1/1     Running   0             7m40s   10.244.1.43   k8s-node1
hello-865c67c987-tf2rz   0/1     Running   2 (20s ago)   90s     10.244.1.45   k8s-node1
```

| 项 | 数据 |
|---|---|
| Pod 总数 | **3**（2 旧 + 1 新） |
| 新 Pod READY | **0/1** |
| 新 Pod RESTARTS | **2** |
| **外部访问** | **`HTTP=200`** ← **全部意义所在** |
| endpoints | `10.244.1.43:80, 10.244.1.44:80`（**只有旧 Pod**） |

RS 状态：

```
NAME               DESIRED   CURRENT   READY
hello-7b47d54b7b   2         2         2        ← 旧版本，继续服务
hello-865c67c987   1         1         0        ← 想跑 1 个、跑了 1 个、就绪 0 个
```

> `DESIRED=1 / CURRENT=1 / READY=0` 这三个数字的差异，就是死锁的可视化。

### 4.4 探针报错原文（要能背）

```
Readiness probe failed: Get "http://10.244.1.45:80/": dial tcp 10.244.1.45:80: connect: connection refused
```

**三种探针失败的区分**：

| 报错 | 含义 | 典型原因 |
|---|---|---|
| **connection refused** | 有主机，**但端口没人监听** | 应用没起、**监听错端口**、进程已崩 |
| **i/o timeout** | 包发出去没回应 | 网络不通、防火墙丢包 |
| **404 / 500** | 通了，**但应用返回错** | 路由/业务逻辑问题 |

### 4.5 两个探针的后果完全不同

```
Warning  Unhealthy  Readiness probe failed: ... connection refused
Warning  Unhealthy  Liveness probe failed:  ... connection refused
Normal   Killing    Container hello failed liveness probe, will be restarted
```

| 探针 | 失败后做什么 | 现象 |
|---|---|---|
| **readiness** | 从 Service 后端**摘掉**，**不重启** | Pod 永远 `0/1`，不接流量 |
| **liveness** | **重启容器** | `RESTARTS` 持续增长 |

**所以这个坏 Pod 的命运是：永远不 Ready + 被反复重启。**

**生产建议**（这条比结论本身重要）：
- `liveness` 探**进程本身**（`/healthz`，只回答"我还活着"）；
- `readiness` 探**依赖是否可用**（`/ready`，包含 DB 连接）。
- 一句话：**liveness 要迟钝，readiness 要敏感。**

**为什么**：如果 liveness 探了依赖外部的东西，依赖一挂，**所有 Pod 全部重启一遍**——而重启根本救不了依赖问题，只会让情况更糟（雪崩）。

### 4.6 `maxUnavailable: 0` 造成的死锁

```
新 Pod 不 Ready
   → maxUnavailable=0 说「一个旧 Pod 都不许删」
   → 旧 Pod 不能删、新 Pod 又不行
   → 永远卡住 → 90 秒超时
```

这是**好事**：如果 `maxUnavailable` 是 1，K8s 就会删掉一个旧 Pod，容量瞬间掉一半。**严格模式用"卡住"换"零影响"。**

### 4.7 这个实验最重要的认知

> **`deploy-job` 失败 ≠ 线上出事。**
> 反而可能说明 **探针 + 滚动策略正常工作了**。
> 真正危险的是 `deploy-job` **绿了但线上是坏的** —— 那就是 W4-D2 学过的「假成功」。

---

## 五、今晚踩的三个坑（都是真实故障）

### 坑 1：`GIT_DEPTH=20` + 老 git 1.8.3.1 → 第二次构建起必挂

```
fatal: git fetch-pack: expected shallow list
fatal: The remote end hung up unexpectedly
```

**根因**：`--depth` 是**增量浅克隆**，需要客户端与服务端协商"我已经有哪些 commit"。git 1.8.3.1（2013 年）实现不完整。

**判定方法（控制变量）**：

| 变量 | 结果 |
|---|---|
| 空目录 + `fetch --depth=20` | ✅ 成功 |
| **同一目录再** `fetch --depth=20` | ❌ `expected shallow list` |
| 同目录 `fetch`（**去掉 depth**） | ✅ 成功 |

只有"目录是否已有对象"这一个变量不同 → 原因就在这里。

**为什么第一次构建能成功**：目录为空时不需要协商，所以不触发 bug。**这不是偶发，是必现。**

**修复**：`GIT_DEPTH: 0`（注意：**不是"深度为 0"，是禁用浅克隆的开关值**，语义容易记反）。

**代价**：仓库大时首次全量克隆慢。但对象已在本地时只取增量，**第二次依然快**。

### 坑 2：`set -e` + `CODE=$(curl ...)` → 日志"凭空消失"

**现象**：job 红了，但**连 `部署后 HTTP=` 这行都没打印**。

**根因**：GitLab Shell executor 默认 `set -e`。而

```bash
CODE=$(curl ...)    # curl 返回非 0（超时 exit 28 / 拒连 exit 7）时，整个脚本立即中止
```

后面的 `echo` 和 `if` **一行都不执行**。所以"看起来什么都没发生就失败了"。

**修复**：`CODE=$(curl ... || true)`

### 坑 3：`rollout status` 成功 ≠ 流量可达 → 假失败

```
回滚后第 1 次: HTTP=000
回滚后第 2 次: HTTP=200     ← 重试救回来了
```

**根因**：`rollout status` 只代表 **Pod Ready**。从 Ready 到流量真正可达还有一段：

```
Pod Ready → endpoints 更新 → 各节点 kube-proxy 更新 iptables 规则     ← 秒级延迟
```

这一两秒内打过去，可能连到刚被删掉的旧 Pod IP → 超时。

**为什么必须修**：假失败会让团队养成"重跑一下"的习惯，**门禁就废了**。

> **门禁的价值不在于它会不会红，而在于它红的时候，人还愿不愿意信。**

**修复**：重试 6 次、间隔 2 秒。

---

## 六、CI 接入 K8s 的关键设计

### 6.1 权限收敛：不给 admin

```bash
# ServiceAccount ci-deployer + Role：只给 deployments 的 get/list/watch/patch/update
# replicasets / pods / services / endpoints 只给 get/list/watch
```

**验证越权会被拒**：

```bash
$ kubectl get pods -n kube-system
Error from server (Forbidden): User "system:serviceaccount:default:ci-deployer"
  cannot list resource "pods" in API group "" in the namespace "kube-system"

$ kubectl delete deploy hello
Error from server (Forbidden): deployments.apps "hello" is forbidden:
  ... cannot delete resource "deployments" in API group "apps" in the namespace "default"
```

**为什么**：Runner 上放 `admin.conf` = 任何能改 yml 的人都能 `delete namespace`。

### 6.2 双通道回滚

```yaml
if [ -n "$TARGET_TAG" ]; then
    kubectl set image ...:$TARGET_TAG    # 通道 B：不受 limit 限制
else
    kubectl rollout undo                 # 通道 A：快，但受 limit + 编号重排限制
fi
```

### 6.3 冒烟必须真的判断状态码

```bash
# 错：只看 curl 是否"执行成功"（5xx 也会被当成成功）
if ! curl -s -o /dev/null -w "%{http_code}" $URL; then ...

# 对：重试 + 判断 200 + 保护 curl
CODE=$(curl ... || true)
if [ "$CODE" != "200" ]; then ...
```

### 6.4 部署失败要自己打印现场

`rollout status` 超时时自动输出：Pod 状态 / RS 状态 / 探针报错 / **endpoints（证明线上还在服务）**。

**为什么**：不这样，GitLab 只会给你一句 `timed out waiting for the condition`，排查得靠人 SSH 上机器——**流水线就成了黑盒**。

---

## 七、从代码到生产的完整链路（今晚的定位）

| # | 阶段 | 工具 | 责任人 | 门禁 |
|---|---|---|---|---|
| ① | 代码提交 | Git + MR | 开发 | Code Review ≥1 人 |
| ② | 构建测试 | CI + Runner | CI 系统（运维维护） | 单测/扫描不过即停 |
| ③ | 镜像打包 | Dockerfile | CI 系统 | 分层缓存、基础镜像固定 |
| ④ | 制品存储 | registry | CI 专用账号 | **tag 已存在则拒绝覆盖** |
| ⑤ | 部署编排 | K8s | CI / Release Manager | **探针 + 滚动策略** |
| ⑥ | 流量接入 | Ingress / LB | 运维 | 灰度 |
| ⑦ | 观测应急 | 日志/指标/告警 | SRE | MTTD |

**今晚做的是 ②→⑤ 的打通，并用 ④ 的不可变 tag 支撑回滚。**

---

## 八、我的问题与答案

> 这一节记录的是**我当晚提出的疑问**。标题保留了原话的口吻，但把模糊处改写成了可回答的技术问题——因为"听不懂"和"问错了"往往是一回事。
> 凡是括号里标了「原话」的，是我当时的原始表述。

### Q1：`revisionHistoryLimit` 触发清理时，被删的是哪一个 RS？判据是 revision 号还是创建时间？

（原话：「为什么'少的那一行是 REV 10 或 11 中的一个'而不是确定某一个」）

**这一问问到了要点，而且直接指出了当时的表述缺陷。**

答案是**确定的，不存在"或"**：被删的是 **REV 10**。

判据来自两个字段的排序差异：

| REV | CREATED(UTC) | |
|---|---|---|
| **10** | 12:14:**24** | 编号最小，但创建时间**更晚** |
| **11** | 12:14:**11** | 编号更大，但创建时间**更早** |

- 按「revision 号升序，删最小」→ 删 REV 10 ✅ **实际发生**
- 按「创建时间升序，删最老」→ 删 REV 11 ❌ 未发生

→ **K8s 的清理按 revision 号排序，不按创建时间。**（详见 §2.3）

**但更该记住的是分层**：

| 层面 | 性质 | 能不能依赖 |
|---|---|---|
| **保留几个**（数量） | ✅ API 契约 | 可以，用于规划回滚窗口 |
| **保留哪几个**（顺序） | ⚠️ 实现细节 | **不能**，跨版本可能变 |

**所以生产上只按"数量"规划窗口，绝不写逻辑去赌"留下的是哪一个"。** 需要精确回到某版时，换通道用 registry 的 tag，不要靠 `undo`。

> 附带一条自我修正：我当时说"或"是因为**没有验证过**，又不想说错。用"或"来兜底，本质是把推断伪装成事实。正确做法是给验证方式，或明说"我没验证"。

---

### Q2：RS 有 `revisionHistoryLimit` 限制，registry 里的镜像制品有对应的清理限制吗？

**没有。而且这是刻意设计的结果。**

实测本机 registry 的 `config.yml` 只有四段（log / storage / http / health），**没有任何 retention / gc 配置**；也没有 crontab 或 systemd timer 在清理。

**为什么两边设计相反**：

| | `revisionHistoryLimit`（K8s） | registry tag |
|---|---|---|
| 谁清理 | **K8s 自动**，默认留 10 | **没人自动清** |
| 为什么 | RS 是内存中的 API 对象，无限堆积会压垮 etcd 和 controller | 制品是磁盘文件，**删了就是永久失去** |
| 风险方向 | 留少了 → 回不去（可恢复） | 删多了 → **永远回不去**（不可恢复） |

**关键**：`undo` 依赖的正是这些 tag。K8s 侧已经自动缩窗了，如果 registry 也自动清，**两个窗口同时收缩，就彻底没有退路**。所以业界共识是：registry 清理必须**显式、有人负责、可审计**。

**清理的正确顺序（不能反）**：

```bash
# 1) 删 tag（只是让 tag 指向消失，blob 还在）
curl -X DELETE http://<registry>/v2/<repo>/manifests/<digest>
# 2) 回收 blob（这一步才真正释放磁盘）
docker exec registry registry garbage-collect /etc/docker/registry/config.yml
```

只删 tag 不 GC → 磁盘一点不省；只 GC 不删 tag → 什么都不会删。

`registry:2` 原生**没有** retention policy；要"保留最近 N 个 / 超过 30 天删除"这种策略，得换 **Harbor**（registry + 策略引擎 + 权限 + UI）。

**这里有个真账要算**：registry 里还有 9 个 tag（69.4 MB），但 K8s 侧 limit=1 时只有 1 个版本可退。**"真回滚能力"取决于两个窗口的较小值，不是 tag 数量。**

---

### Q3：镜像 push 到 registry 之后，是什么机制让 CI 去发布它的？

**没有任何"通知"机制。 push 不触发任何东西。**

registry 是**被动存储**，它没有事件能力，不会告诉任何人"我这儿多了个镜像"。

真正让两条线对齐的是**同一个变量**：

```yaml
package-job:  docker push $IMAGE:$CI_COMMIT_SHORT_SHA          # 推 :0e8dc0f4
deploy-job:   kubectl set image ...=$IMAGE:$CI_COMMIT_SHORT_SHA # 用同一个 :0e8dc0f4
```

deploy-job **不需要问 registry"新镜像叫什么"**——它和 package-job 在同一个 pipeline 里，`$CI_COMMIT_SHORT_SHA` 是同一个值，**名字是算出来的，不是查出来的**。

> 这也是为什么必须用 sha tag 而不是 `latest`：用 `latest` 的话，deploy **无法知道**这次推的是哪个制品，只能靠"tag 被覆盖了"碰运气。

**"触发"这件事发生在更早一步**：`git push` 触发 CI，因为 **GitLab 既是 Git 服务器又是 CI 调度器**，push 事件是它自己产生的。

> 对比 Jenkins：它不是 Git 服务器，所以必须配 **webhook** 让 GitLab 主动"打电话"，否则只能**轮询**（定时问"有新提交吗"）。这是 GitLab CI 天生比 Jenkins 少一层的原因。

**之后 Pod 怎么真的跑起来**：`set image` 只改 etcd 里的 spec → Deployment Controller 建新 RS → RS 建 Pod → 节点上的 **kubelet** 按 `imagePullPolicy` 去 registry 拉 → 最终锁定的是 **digest**，不是 tag。

---

### Q4：从开发写代码到上生产环境，中间经历哪些阶段、各用什么、它们怎么工作？

见 §7 的七段链路表。**每段的产出物才是重点**：

| # | 阶段 | 产出 | 这一层在解决上一层的什么失败模式 |
|---|---|---|---|
| ① | 代码提交 | commit（唯一 sha） | 改动无记录、无法回退 |
| ② | 构建测试 | 测试报告 | 代码能跑但逻辑错 |
| ③ | 镜像打包 | 镜像（tag=sha） | 开发机跑通、服务器跑不通 |
| ④ | 制品存储 | registry 里的不可变 tag | 部署时找不到当时那个版本 |
| ⑤ | 部署编排 | Pod / RS / revision | 进程挂了没人管 |
| ⑥ | 流量接入 | Ingress / Service | Pod 正常但用户连不上 |
| ⑦ | 观测应急 | 日志/指标/告警 | 都正常但没人知道它在坏 |

**主线**：这条链上每一层都在解决上一层的失败模式。所以真正的关键不是"自动化"，而是**每段产出都是不可变、可追溯、可回滚的**。

---

### Q5：正规生产环境里，这条链各阶段分别由谁负责？谁推镜像、谁发版？

见下表的四个角色。**核心是互不越权**：

| 阶段 | 责任人 | 谁批准 |
|---|---|---|
| 写代码 / MR | 开发工程师 | 技术负责人（Review ≥1 人） |
| 构建 / 打 tag | **CI 系统**（无人值守） | 无人，全自动 |
| 推镜像 | **CI 专用账号**（robot account） | 无人，但**人没有 push 权限** |
| 部署测试环境 | CI 自动 | 测试验收 |
| **发布生产** | **Release Manager / 值班运维** | 变更单 + 发布窗口 |

**两个反直觉的点**：

1. **开发不点发布按钮，运维不改业务代码。** 这是组织制衡，不是礼节——谁写代码谁批准上线，等于没有审批。
2. **人不能推镜像，只有 CI 能推。** 生产 registry 只给 CI 一个 robot account，开发者账号只有 pull 权限。这样"镜像里跑的是什么"永远能追溯到某次构建，**不存在"某人偷偷手动推了个镜像"**。

**为什么"必须由 CI 构建镜像"不是"为了方便"，而是"可追溯 + 不可绕过"**——手动推的镜像没有 commit 对应关系，出问题查不到源头。

**其他生产细节**：
- `latest` 基本被禁（会变 → **假回滚**），有的公司用准入控制器直接拦截 `image: xxx:latest`；
- **数据库变更与代码发布是两条独立的线**：代码可随便回滚，DB 迁移不能（数据已写进去）。所以要求迁移脚本**向前兼容**（先加字段、后删字段，分两次发）；
- 生产不是一次性全量，是**灰度**（5% → 25% → 50% → 100%）；
- 发布有**窗口**（工作日白天、避开周五晚和节假日、大促封网）；
- **回滚预案必须发布前就绪**（用哪条命令、回到哪个 tag、谁拍板），不是出事再想。

---

### Q6：`kubectl get rs` 的输出里看不到 revision，怎么显示？

`kubectl get rs` 的默认列是固定的（NAME / DESIRED / CURRENT / READY / AGE）。**REV 不在默认列**，它存在注解 `deployment.kubernetes.io/revision` 里，必须显式要求：

```bash
kubectl get rs -o custom-columns='NAME:.metadata.name,REV:.metadata.annotations.deployment\.kubernetes\.io/revision,DESIRED:.spec.replicas'
```

**逐段拆解**：
- `-o custom-columns=` → 不要默认表格，自己定义列
- `NAME:.metadata.name` → 列名 `NAME`，值取 `.metadata.name`
- `REV:.metadata.annotations.deployment\.kubernetes\.io/revision` → 注解名里的**点必须用 `\` 转义**

> **坑**：custom-columns 的路径写错**不会报错，会静默丢列**。第一次写漏了转义，结果 `REV` 整个列消失，命令看起来"成功"了。

---

### Q7：为什么这次 `rollout undo` 成功了，而之前 `--to-revision=10` 会失败？

**因为 `undo`（不带参数）是"退到上一个"，而"上一个"一定在历史里；`--to-revision=N` 是"跳到指定编号"，那个编号可能已经不存在。**

| 命令 | 语义 | 失效条件 |
|---|---|---|
| `rollout undo` | 退到**上一个** revision | 历史为空（几乎不会） |
| `rollout undo --to-revision=10` | 按**编号**跳转 | 该 RS 被 GC，**或编号被重排** |

实测：limit=1 时保留了 1 个历史 RS，所以 `undo` 一定能成功一次；但退回后**编号会重排**（15 → 17），老编号 15 立刻失效，再想 `--to-revision=15` 就报 `unable to find`。

→ 详见 §3.2、§3.3。**这就是"能退一步"和"能退到指定版本"的区别，也是必须做双通道回滚的原因。**

---

## 九、技术要点（面试用）

> "回滚能力 = 副本为 0 的 RS 还剩几个，这个窗口由 `revisionHistoryLimit` 决定，而且会随发布次数持续缩小——limit=1 时每发一版就缩一格。清理是按 revision 号排的，不是按创建时间。
>
> 两条回滚通道失效条件不同：`undo` 依赖 RS 历史，`set image` 依赖 registry 的 tag。而且 `undo` 找的是编号，编号一重排，哪怕对象还在也报 `unable to find`。所以生产上回滚脚本记的是镜像 tag，不是 revision 号。
>
> 把 CI 接进 K8s 时，用受限的 ServiceAccount 而不是 admin，deploy 走 `set image` + `rollout status` + 真判断 200 的冒烟。
>
> 最后，探针是唯一看真实行为的门禁：能挡住'构建成功但端口不监听'这类故障——实测坏镜像新 Pod 永远 0/1、liveness 反复重启、旧 Pod 保持 200、endpoints 只挂旧 Pod，流水线红了但用户无感。"

---

## 十、遗留

- [x] `revisionHistoryLimit` 实验（§2）
- [x] K8s 回滚 vs 手写 `TARGET_TAG` 对比（§3.4）
- [x] CI 接入 K8s（§4、§6）
- [x] revert 坏镜像（Dockerfile 已还原）
- [x] `revisionHistoryLimit` 恢复为 10
- [ ] **W4 周复盘**（`weekly/week04-总结.md`）
- [ ] Shell 每日 1 题（连续多天未做，运维岗硬门槛）

---

## 十一、下次开局待办

1. 确认环境：`kubectl get nodes`（VM 重启后 1-3 分钟内 `connection refused` 是正常的，先看 `uptime`）
2. 写 W4 周复盘
3. 决定 W5 方向（监控 / Prometheus，或 Helm / GitOps）
