# Week6 D1：端到端演练（代码 → 上线 → 观测）

> 时间：2026-11-09（东八区）
> 目标：把 W3（CI）、W4（K8s）、W5（监控）三段孤立实验**串成一条能跑、能验证的链路**
> 口径：不引入新工具，只做整合与验证

---

## 一、一句话结论

改一行页面文字 → push → 流水线自动构建并产出不可变制品 → 人工点一次发布 →
K8s 声明式地换掉 Pod → 三个 exporter 把变化记成指标 → 用查询验证它真的发生了。

**六段里只有第 1 步和第 4 步需要人**，其余全靠机器。

---

## 二、全链路体检（开局）

| 段 | 检查 | 结果 |
|---|---|---|
| 基础 | 时钟 | ✅ 两机 `21:12 CST`，一致（VM 冷启动后必查） |
| CI | GitLab / registry / Runner | ✅ 302 / 200 / active |
| K8s | 两节点 / hello / NodePort | ✅ Ready / 2/2 / HTTP 200 |
| 监控 | 4 容器 / 4 target / 告警规则 | ✅ 全自启 / 全 up / `TargetDown` health ok |
| 监控 | Grafana | ✅ 302 |

> GitLab 刚开机时短暂返回 502（容器 `health: starting`），稍后自愈为 302。
> 这是启动顺序问题，不是故障。

---

## 三、六段链路与实际操作

### 第 1 段：本地提交与推送

**在哪做**：Windows 本地 Git Bash，**不是** Xshell / node1。

| 位置 | 角色 |
|---|---|
| Windows `Claw/.../gitlab-ci-lab` | 写代码的地方 |
| GitLab `192.168.245.150` | 代码服务 + CI 调度 |
| node1 `/root/builds/...` | Runner 构建副本，改了会被覆盖 |

```bash
cd "/c/Users/18790/WorkBuddy AI/Claw/2026-09-21-16-30-29/gitlab-ci-lab"
git fetch origin
sed -i 's/版本：v2（回滚演示用）/版本：w6d1-端到端演练/' app/index.html
grep -n "版本" app/index.html
git add app/index.html
git commit -m "test(w6d1): 端到端演练——修改页面版本号"
git push origin main
```

**产物**：commit `7d75d67`，push 输出 `b7e27da..7d75d67  main -> main`。

**一个坑**：`git status` 曾显示 `ahead 6`，是**假象**。之前几次推送用带凭据的完整 URL，没走 `origin`，本地 `origin/main` 指针陈旧。

```bash
git ls-remote origin refs/heads/main    # 服务端真实 HEAD
git fetch origin                        # 刷新本地缓存
```

**机制**：push 到 GitLab **会**触发流水线（GitLab 既是 Git 服务器又是 CI 调度器）；
push 到 registry **不会**触发部署（registry 只是被动存储，无事件能力）。

### 第 2 段：CI 自动构建与门禁（pipeline #56）

| stage | job | 实际动作 | 门禁意义 |
|---|---|---|---|
| build | `build-job` | 写 `dist/build-info.txt`（sha + 时间），作为 artifact 传递 | 可追溯产物 |
| test | `test-job` | `bash scripts/test.sh`：查文件、grep 页面字符串、查构建产物 | 单测门禁，不过全停 |
| package | `package-job` | `check-tag.sh` → `docker build` 双 tag → `docker push` ×2 | 制品不可变，不覆盖已存在 tag |

**你敲了什么**：无。三个 job 自动跑，只在浏览器看结果。

**关键配置**：`GIT_DEPTH: 0`（禁用浅克隆）。node1 的 git 1.8.3.1 在非空仓库做增量浅 fetch
必报 `expected shallow list`；今天 #56 跑绿，证明 W4 的修复长期有效。

### 第 3 段：制品入库

`package-job` 内部执行（无需手敲）：

```bash
docker build -t $IMAGE:$CI_COMMIT_SHORT_SHA -t $IMAGE:latest .
docker push $IMAGE:$CI_COMMIT_SHORT_SHA
docker push $IMAGE:latest
```

**为什么用 sha tag**：`latest` 会变，sha 不会。W4 双通道回滚依赖它。

```bash
curl -s http://192.168.245.150:5000/v2/devops/hello/tags/list
# 结果里出现 7d75d679
```

### 第 4 段：人工审批发布（唯一需要人点的一步）

`deploy-job` 配了 `when: manual`，前三个 job 绿了它仍是灰的。点 ▶ 后执行：

```bash
kubectl set image deploy/$DEPLOYMENT $CONTAINER=$IMAGE:$CI_COMMIT_SHORT_SHA
kubectl rollout status deploy/$DEPLOYMENT --timeout=90s
kubectl get rs -o custom-columns='NAME:...,REV:...,DESIRED:...,IMAGE:...'
CODE=$(curl -s -o /dev/null -w "%{http_code}" "$NODE_PORT_URL/" || true)   # 重试 6 次
```

**用到的身份**：W4 配的受限 ServiceAccount `ci-deployer`（只能改 deployments）。

**两个隐藏设计**（W4 踩坑换来的）：
- `|| true`：Shell executor 开着 `set -e`，curl 非 0 会让脚本立即中止、连 echo 都不打印；
- **重试**：`rollout status` 成功 ≠ 流量可达，iptables 收敛有秒级延迟，单次冒烟会假失败。

### 第 5 段：K8s 滚动更新

`set image` 只改 Deployment 的 spec，之后全是控制器自动调谐：

```
Deployment controller → 新建 RS → RS controller 建 2 个 Pod
→ kubelet 从 registry 拉镜像 → readinessProbe 通过 → 进 endpoints
→ 缩掉旧 RS 副本
```

**没有命令，全是观察**。这正是声明式与"手工 docker run"的根本区别。

**实测证据**（master 窗口）：

```bash
curl -s http://192.168.245.133:30863/ | grep -o '版本：[^<]*'
kubectl get deploy hello
kubectl get rs -o custom-columns='NAME:.metadata.name,REV:.metadata.annotations.deployment\.kubernetes\.io/revision,DESIRED:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image'
```

```
版本：w6d1-端到端演练
hello   2/2   2   2   13d
23   hello-8c74fb44     2    ...:7d75d679
22   hello-5cffd4cf4b   0    ...:ddbc0750
21   hello-865c67c987   0    ...:6dbaa112
```

| 时间戳（UTC） | 事件 |
|---|---|
| `13:27:48Z` | 新 RS `hello-8c74fb44` 创建 |
| `13:27:49Z` / `13:27:59Z` | 两个新 Pod 启动，镜像 `7d75d679`、`READY true` |

**顺带澄清一个疑问**：当前 `revisionHistoryLimit=10`，但只看到 3 个 RS。
因为 10 是**上限**不是配额，且 W4 把 limit 改成 1 时更早的 RS 已被**真删除**，
改回 10 不会复活。当前 RS 也不占额度。REV 已到 23 但对象只剩 3 个，
说明「编号是只增不减的记账，对象会被 GC」。

### 第 6 段：观测与验证

| 数据源 | 来自 | 记录的证据 |
|---|---|---|
| `node_exporter` | 宿主机 `/proc` | 机器 CPU / 内存 / 磁盘 |
| `cAdvisor` | kubelet（apiserver proxy） | **新 Pod 的内存序列从 21:28 开始有值** |
| `kube-state-metrics` | apiserver | `replicas_available=2`、Pod 状态、重启次数 |

Prometheus 每 15 秒拉一次存进 TSDB；Grafana 只画图。

**一个诚实发现**：`sum(kube_pod_status_phase{Running})` 在 21:20–21:42 全程为 `2`，
**没有曲线凸起**——两个 Pod 在 ~10 秒内换完，而 kube-state-metrics 15 秒抓一次，
采样点刚好错过。这是实验环境的粒度问题，不是配置错。

**所以真正的发布痕迹不是"曲线波动"，而是「旧序列消失、新序列从某时刻开始有值」**。

在 node1 窗口执行（`localhost:9090` 只在 node1 有效；master 上要写 `192.168.245.150:9090`）：

```bash
START=$(date -d '2026-11-09 21:20:00' +%s)
END=$(date -d '2026-11-09 21:42:00' +%s)
curl -sG \
  --data-urlencode 'query=container_memory_working_set_bytes{pod=~"hello-8c74fb44-.*",container="hello"}' \
  --data-urlencode "start=$START" --data-urlencode "end=$END" \
  --data-urlencode 'step=120s' \
  http://localhost:9090/api/v1/query_range \
| tr '}' '\n' \
| grep -o '"pod":"[^"]*"\|\[1794[0-9]*,"[0-9]*"\]' \
| awk '/^"pod"/{gsub(/"/,"");sub(/pod:/,"");pod=$0;n=0;next} {if(n>0)next; split($0,a,",");t=substr(a[1],2);v=a[2];gsub(/[^0-9]/,"",v);cmd="date -d @"t" +%H:%M:%S";cmd|getline hm;close(cmd);printf "%-28s 首点 %s  %6.1f KiB\n",pod,hm,v/1024;n++}'
```

**格式化说明**：`curl` 原始输出是一整坨 JSON，必须格式化才能读。
`tr` 拆条 → `grep -o` 只留 Pod 名和 `[时间戳,值]` → `awk` 换算时间与 KiB、
每个 Pod 只打首点。用 `container="hello"` 而非 `!="POD"` 可避免重复序列。

**输出**：

```
hello-8c74fb44-fsxjk   首点 21:28:00   1780 KiB
hello-8c74fb44-8lzfz   首点 21:30:00   2324 KiB
```

**对照 K8s 侧**：Pod 启动是 21:27:49 / 21:27:59，监控首点 21:28 / 21:30，
差约 1~2 分钟 = `scrape_interval 15s` + cAdvisor 数据就绪延迟。
**两套独立数据对得上，链路闭合**——也直观印证了「监控永远滞后」。

---

## 四、三层验证清单（可复用）

| 层 | 看什么 | 今天的结果 |
|---|---|---|
| 1. 发布本身成功 | 页面文字 / `2/2` / 新 RS 承重 | ✅ |
| 2. 发布留下痕迹 | 新 Pod 指标序列的首点时间 | ✅ 21:28 / 21:30 |
| 3. 监控没被发布搞坏 | 四个 target 全 up、告警无 Firing | ✅ |

**第 3 层最容易被忽略**：发布动作本身可能把 exporter 挤掉、把探针搞挂，
发完不看监控等于盲发。

---

## 五、六段用到的东西（一览表）

| 段 | 工具 / 组件 |
|---|---|
| 1 | Git、GitLab CE |
| 2 | `.gitlab-ci.yml`、Shell Runner、`scripts/test.sh`、`scripts/check-tag.sh`、`GIT_DEPTH=0` |
| 3 | Docker、registry:5000、`nginx:1.24-alpine`、sha tag |
| 4 | `deploy-job`（manual）、`kubectl set image`、受限 SA `ci-deployer`、冒烟重试 |
| 5 | Deployment / RS / Pod / Service(NodePort 30863) / readinessProbe / `maxSurge:1,maxUnavailable:0` |
| 6 | node_exporter / cAdvisor / kube-state-metrics → Prometheus → Grafana / `query_range` / 告警规则 |

---

## 六、本次踩到 / 澄清的坑

1. **`localhost:9090` 只在 node1 有效**：在 master 窗口查会静默无输出（`curl -s` 吞掉错误）。
   跨机要写 `192.168.245.150:9090`。
2. **原始 JSON 人眼读不了**：API 查询必须格式化，否则"什么都看不出来"。
3. **`git status` 的 `ahead N` 只是本地缓存比较**：用 `git ls-remote` 确认服务端真实 HEAD。
4. **`limit=10` 不是配额**：历史上被删的 RS 不会复活。
5. **发布不一定能在图上看成凸起**：小副本 + 无流量时，改用「序列起点」作证据。

---

## 七、D1 完成状态

- [x] 全链路体检
- [x] 改代码 + 提交 + push
- [x] 自动构建出制品
- [x] 人工审批发布
- [x] 三层验证
- [x] 端到端记录（本篇）

**W6 D1 完成。**
