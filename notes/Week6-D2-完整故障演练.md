# Week6 D2：完整故障演练（事故时间线）

> 时间：2026-11-10（东八区）
> 目标：把 W4（探针 / 回滚）与 W5（告警 / 看板）从三次孤立实验串成一次完整事故闭环
> 结果：**拦截成功、用户无感、告警缺口被暴露并补上、环境已恢复**

---

## 一、一句话结论

故意发布一个"构建成功、进程活着、但探针探不通"的坏版本，验证了：

```
出事 → 探针挡住 → 数据被采集 → 新规则告警 → 按 tag 回滚 → 环境恢复
```

期间**外部 HTTP 全程 200**。流水线红、用户无感，这正是 `maxUnavailable: 0` + readinessProbe 的价值。

---

## 二、故障注入

在 `gitlab-ci-lab/Dockerfile` 加回一行（素材 `app/nginx-bad.conf` 一直保留在仓库）：

```dockerfile
COPY app/nginx-bad.conf /etc/nginx/conf.d/default.conf
```

该配置让 **nginx 监听 8080 而不是 80**。

**部署前先验证坏法**（这一步避免了"点了才发现坏法不对"）：

| 检查 | 结果 |
|---|---|
| 容器状态 | `Up`（没崩溃） |
| nginx 监听 | `0.0.0.0:8080` |
| 探 80 | 拒绝/超时 |
| 探 8080 | `200` |
| 宿主映射到 80 | `000` |

因为没碰 `index.html`，`test.sh` 与 `docker build` 都放行，**只有探针能拦住它**。

**commit**：`af006ff` → registry tag `af006ffe`。

---

## 三、事故时间线（实测）

| 时刻 | 事件 | 证据 |
|---|---|---|
| 约 23:10 | push，pipeline #57 自动 build/test/package 全绿 | 新 tag `af006ffe` |
| 23:1x | 点 `deploy-job` ▶ | `kubectl set image` → REV 24 |
| 约 23:14 | **滚动卡住** | 新 Pod `hello-5c9579fc47-k5rcn` **0/1**、`CrashLoopBackOff`、`RESTARTS 2` |
| 同时 | **旧 Pod 撑住流量** | REV 23 `DESIRED 2 / READY 2`，endpoints 只有 `10.244.1.60/61:80` |
| 约 23:16 | `deploy-job` 超时变红 | 日志 `error: timed out waiting for the condition` + 自动打印现场 |
| 全程 | **用户无感** | `HTTP 200`、页面仍是 `版本：w6d1-端到端演练` |
| 约 23:20 | 监控侧取证 | 四个 target 全 `up`、**Firing 数 = 0** |
| 23:25 | 补规则后 `PodNotReady` **firing** | `activeAt 15:25:15Z`，value=1 |
| 23:3x | `rollback-job` 用 `TARGET_TAG=7d75d679` | 镜像回到 `7d75d679`、2/2、坏 Pod 消失 |
| 23:38 | 告警恢复 | Firing 数 0、不 Ready 的 Pod 查询结果为空 |

---

## 四、deploy-job 自动打印的现场（W4 的设计生效）

流水线失败时日志里直接出现，不用 SSH 上机器：

```
!!! 滚动更新未完成（超时）—— 多半是探针拦住了新版本
--- 1) Pod 状态：重点看 READY 列是不是 0/1 ---
hello-5c9579fc47-k5rcn   0/1   CrashLoopBackOff
--- 2) RS 状态 ---
--- 3) 探针失败的具体原因：probe failed / connection refused
--- 4) 线上是否仍在服务：旧 Pod 应该还挂在 endpoints 上 ---
hello   10.244.1.60:80,10.244.1.61:80
```

> 这是 W4-D4 加的「不让流水线变黑盒」设计，本次在真实事故里第一次派上用场。

---

## 五、最重要的发现：能采集 ≠ 能告警

### 5.1 缺口被证实

| 检查 | 结果 |
|---|---|
| 四个 target 的 `up` | **全是 1** |
| `Firing` 告警数 | **0** |
| `kube_pod_status_ready{condition="false"}` | 能看到坏 Pod = 1（**数据有**） |
| `replicas_available` | `2`（旧 Pod 撑着，所以副本数也不报警） |

原有的 `TargetDown`（`up == 0`）**抓不到这次故障**——因为 exporter 全都在正常工作。

### 5.2 补上的规则

```yaml
- alert: PodNotReady
  expr: kube_pod_status_ready{condition="false",namespace="default"} == 1
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "{{ $labels.pod }} 持续 1 分钟未就绪"
```

`promtool check rules` → `SUCCESS: 2 rules found`，reload `200`。
坏 Pod 已持续不就绪超过 1 分钟，所以规则加载后直接 `firing`。

### 5.3 为什么旧 Pod 撑流量，仍能发现新 Pod 不 Ready

**Prometheus 不"决定看谁"**，它每 15 秒拉一次 `kube-state-metrics:18080/metrics`，
那个页面是 kube-state-metrics **把 apiserver 里所有 Pod 对象全量翻译**出来的：

```
hello-8c74fb44-8lzfz     condition="true"  1   ← 旧 Pod
hello-8c74fb44-fsxjk     condition="true"  1   ← 旧 Pod
hello-5c9579fc47-k5rcn   condition="false" 1   ← 坏 Pod，照样在清单里
```

| | 控制面（指标来自这里） | 数据面（流量走这里） |
|---|---|---|
| 内容 | etcd 里**所有** Pod 对象 | endpoints 里**Ready 的** Pod |
| 坏 Pod 在不在 | ✅ 在 | ❌ 不在 |

**采集是全量的，判断在查询/规则里。** 每个 Pod 固定输出 `true/false/unknown` 三行（值 0 或 1），
所以规则必须写 `condition="false" == 1`，只写 `== 0` 会把另外两种状态也算进去。

---

## 六、回滚与恢复

`rollback-job` 填 `TARGET_TAG=7d75d679`（W4 双通道里的**通道 B**：按 registry tag 回滚，
不受 RS 历史限制）。

| 检查 | 结果 |
|---|---|
| 当前镜像 | `192.168.245.150:5000/devops/hello:7d75d679` |
| Deployment | `2/2  2  2` |
| Pod | 两个旧 Pod `1/1 Running`，坏 Pod 已消失 |
| 页面 | `HTTP 200`、`版本：w6d1-端到端演练` |
| 告警 | `PodNotReady` 从 firing → 无 Firing |
| 四个 target | 全 `1` |
| 规则文件 | 保留 2 条规则（`TargetDown` + `PodNotReady`） |

**环境收尾（我代做，如实说明）**：本地 `gitlab-ci-lab` 用 `git revert` 生成 `591edec`
撤销坏配置，并推送到内网 GitLab，服务端的 Dockerfile 也恢复干净版本。
素材 `app/nginx-bad.conf` 仍保留，供以后复现。

---

## 七、三层证据对照（可复用）

| 层 | 本次看到什么 |
|---|---|
| 发布层 | 流水线红、REV 24 未完成、旧 RS 仍承重 |
| 应用层 | `HTTP 200` 全程不变、用户无感 |
| 监控层 | target 全 up 但 `PodNotReady` firing（补规则后） |

**这三层结论不一致才是正常的**：流水线红 ≠ 服务挂；服务正常 ≠ 没有隐患。
值班时要同时看，不能只看一个。

---

## 八、本次踩到 / 澄清的坑

1. **`grep -o '"labels":{[^}]*}'` 匹配不上**：`{` 在基础正则里是区间起始，导致验证命令"无输出"，
   误以为规则没生效。改用 `tr ',' '\n' | grep -E 'alertname|"state"|"pod"'`。
2. **`for: 1m` 从规则加载时重新计时**：reload 后只 `sleep 20` 查，状态还是 `pending`，
   不是 `firing`。要等满观察期。
3. **GitLab 页面顶部 "An error occurred while making the request"**：UI 层的偶发请求错误，
   job 状态本身正常（截图里 rollback-job 已绿），不影响演练。
4. **回滚后必须 revert 本地 Dockerfile**：否则下次任何 push 都会再产出坏镜像。

---

## 九、D2 完成状态

- [x] 注入故障并走完整发布流程
- [x] 验证探针拦截 + 用户无感
- [x] 暴露「能采集 ≠ 能告警」缺口
- [x] 补 `PodNotReady` 规则并实测 firing
- [x] 按 `TARGET_TAG` 回滚、环境恢复
- [x] 撤销坏配置、服务端同步干净版本
- [x] 事故时间线（本篇）

**W6 D2 完成。**
