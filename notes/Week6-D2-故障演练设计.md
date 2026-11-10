# Week6 D2 预备：完整故障演练设计

> 时间：2026-11-10
> 状态：**设计已确认，动手待下次开局**
> 目标：把 W4（探针/回滚）与 W5（告警/看板）从三次独立实验串成一次完整事故闭环

---

## 一、要达成什么

昨天 D1 走通「好版本」链路：代码 → 上线 → 看板有反应。

D2 走反方向，验证前几周装的防护**在真实发布流程里真的生效**：

```
出事 → 被挡住 → 告警响 → 人看板确认 → 回滚 → 恢复
```

**为什么必须串起来做**：W4 做过坏镜像 + 探针拦截，W5 做过告警触发，
但它们是各自单独的实验，中间没有因果关系。工作里故障是一条时间线，不是三个孤岛。

---

## 二、故障注入方式（关键设计）

素材已确认还在：`gitlab-ci-lab/app/nginx-bad.conf`，内容是让 **nginx 监听 8080 而不是 80**。
演练时在 `Dockerfile` 加回一行 `COPY` 即可：

```dockerfile
COPY app/nginx-bad.conf /etc/nginx/conf.d/default.conf
```

**为什么这个坏法最有效**：

| 检查 | 结果 | 意图 |
|---|---|---|
| `test.sh` 三项目检 | ✅ 过 | **故意不碰 index.html**，让单测门禁放行 |
| `docker build` | ✅ 成功 | 让制品门禁放行 |
| 容器内 nginx 进程 | ✅ Running | 不是崩溃型故障 |
| **readinessProbe 探 80** | ❌ `connection refused` | **只有探针能拦住** |

模拟的是生产上最阴险的一类：**构建成功、进程活着、但服务不可用**。
若选「让容器直接崩溃」，`CrashLoopBackOff` 太显眼，反而验证不了探针的价值。

---

## 三、安全边界（演练前必须确认）

```
新 Pod 永远不 Ready
  → maxUnavailable: 0 不许删旧 Pod
  → 旧 Pod 继续挂在 endpoints 上
  → 外部 HTTP 全程 200
  → 只有 deploy-job 因 rollout 超时(90s) 变红
```

最坏结果：**流水线红、页面正常**。用 `TARGET_TAG` 退回上一个好 tag
（`7d75d679`）即可恢复，全程可逆。

---

## 四、执行步骤与验收

| # | 动作 | 要看的证据 |
|---|---|---|
| 1 | 加回坏配置 → 提交 → push → 点 deploy | 新 tag、deploy-job 超时变红 |
| 2 | 探针拦截面 | 新 Pod `0/1`、`RESTARTS` 增长、`describe` 里 `connection refused` |
| 3 | 监控侧 | targets 仍 up、`Pods Running` 异常、看板曲线、告警状态 |
| 4 | 回滚 | `TARGET_TAG` 指定好 tag → `HTTP 200` 恢复 |
| 5 | 写 D2 笔记 | 一份带时间戳的事故时间线 |

---

## 五、预计会暴露的缺口（提前说，这不是失败）

现有告警规则只有一条：

```yaml
- alert: TargetDown
  expr: up == 0
  for: 1m
```

而这次故障里 **四个 target 全是 up**（探针失败不会让 exporter 消失），
所以 `TargetDown` **不会响**。

**这正是演练的价值**：会发现「能采集 ≠ 能告警」，
针对「Pod 一直不 Ready」需要另配规则，例如基于
`kube_pod_status_ready` 或 `kube_deployment_status_replicas_available`。

演练后应补一条规则并验证它能触发——把 W5 的告警从"只覆盖采集层"补到"覆盖发布层"。

---

## 六、W4 / W5 的对应复用

| 本次环节 | 复用自 |
|---|---|
| 坏镜像配置 | W4-D4 `nginx-bad.conf`（当时已保留作实验素材） |
| 探针拦截、旧 Pod 撑流量 | W4-D2 假滚动对照实验 |
| 双通道回滚 `TARGET_TAG` | W4-D4 rollback-job |
| 告警规则、看板 | W5 够用清单第八节 / 看板 7249 |
| 端到端发布流程 | W6-D1 六段链路 |

---

## 七、状态

- [x] 演练设计讲清（目标 / 注入方式 / 安全边界 / 步骤 / 预计缺口）
- [x] 素材确认存在、Dockerfile 当前是干净版本
- [ ] 动手演练
- [ ] 事故时间线笔记
- [ ] 补一条「Pod 不 Ready」告警规则并触发

**下次开局**：先出 Shell 第 3 题 → 环境体检 → 只做第 1 步（加回坏配置并提交）。
