# 监控够用清单（Grafana + Prometheus）

> 面向目标：**会用、能干活**，不追究原理。
> 环境：k8s-node1 `192.168.245.150`
> 适用场景：日常巡检、加监控目标、判断异常

---

## 一、四个动作（做完就能上班用）

### 1. 起服务

```bash
docker run -d --name prometheus --restart always \
  -p 9090:9090 \
  -v /opt/prometheus:/etc/prometheus \
  --memory=400m \
  prom/prometheus:v2.45.0 \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=6h \
  --web.enable-lifecycle

docker run -d --name node-exporter --restart always \
  --network host --pid host \
  -v /proc:/host/proc:ro -v /sys:/host/sys:ro -v /:/rootfs:ro \
  --memory=100m \
  prom/node-exporter:v1.6.1 \
  --path.procfs=/host/proc --path.sysfs=/host/sys --path.rootfs=/rootfs

docker run -d --name grafana --restart always \
  -p 3000:3000 --memory=300m \
  grafana/grafana:10.2.0
```

**要点**：
- `--restart always` 必须有，否则 VM 重启后监控全没了
- **挂目录，不挂单文件**：`-v /opt/prometheus:/etc/prometheus`
- Prometheus 启动参数要写全（`docker run` 的参数会整体替换镜像 CMD）

### 2. 改配置 + 热重载

```bash
vi /opt/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload      # 不用重启容器
```

配置文件形状：

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'node'                            # 见名知意，别写 dead-target
    static_configs:
      - targets: ['192.168.245.150:9100']
```

**坑**：`sed -i` 会换 inode，单文件挂载时容器看不到新内容。挂目录就没这个事。
不放心就 `stat -c %i 文件` 和 `docker exec 容器 stat -c %i 文件` 对比一下。

### 3. 确认目标活着

```bash
curl -s http://localhost:9090/api/v1/query?query=up
curl -s http://localhost:9090/api/v1/query?query=scrape_samples_scraped
```

| 看什么 | 正常值 | 说明 |
|---|---|---|
| `up` | `1` | HTTP 抓到了（2xx） |
| `scrape_samples_scraped` | 几百以上 | **真的有数据** |
| `up=1` 但 samples=0 | ⚠️ | **通了但是空的**，监控形同虚设 |

浏览器也可以看：`http://192.168.245.150:9090/targets`

### 4. Grafana 接数据源 + 导看板

1. 打开 `http://192.168.245.150:3000`，`admin` / `admin` 登录
2. ☰ → `Connections` → `Data sources` → `Add new data source` → 选 `Prometheus`
3. **URL 填 `http://192.168.245.150:9090`**（写 `localhost` 连不上——容器内 localhost 是它自己）
4. 点 `Save & test`，看到绿色 `Successfully queried the Prometheus API`
5. ☰ → `Dashboards` → `New` → `Import` → `Import via grafana.com` 填 **1860** → Load → Import
6. 看板顶部下拉框选：`Datasource=Prometheus`、`Instance=192.168.245.150:9100`

**1860 = Node Exporter Full**，社区最通用的机器监控看板。

---

## 二、看板怎么读

| 面板 | 看什么 | 异常长什么样 |
|---|---|---|
| CPU Busy | 当前 CPU 忙的比例 | 长期 > 80% |
| Sys Load | 系统负载 | 持续高于 CPU 核数 |
| RAM Used | 内存占用 | 接近 100%，或持续上涨不回落 |
| Root FS Used | 根分区磁盘 | > 80% 要处理，满了服务会挂 |
| Uptime | 开机时长 | 突然变小 = **机器重启过**，要问为什么 |

**本次实测**（2026-11-06）：CPU Busy 6.5% / RAM 85.1% / Root FS 69.3% / CPUs 2 / Uptime 53.1 min。

**上班第一件事**：打开看板，扫一眼这几个数，看有没有红的、有没有曲线突然断了。

---

## 三、出问题按这个顺序查

```
1) 看板空白 / No data
   → 先查 Grafana 数据源：Save & test 还是绿的吗
   → 查看板顶部 Instance 下拉选对了没

2) /targets 里某个目标 DOWN
   → 看 lastError 原文：
     connection refused  → 端口没进程监听（服务没起 / 端口写错）
     HTTP 404            → 服务在，但 /metrics 路径不对
     timeout             → 网络/防火墙

3) up=1 但没数据
   → curl 目标/metrics 看是不是 200 但内容为空
   → 查 scrape_samples_scraped 是不是 0

4) 改了配置没生效
   → reload 返回 200 了吗
   → docker exec 容器 grep 配置，看容器里是不是新内容
   → 不一致 = inode 坑（单文件挂载），改成挂目录

5) 容器全没了
   → VM 重启了。检查 docker ps -a，没 --restart 的重新起
```

---

## 四、常见空面板（不用修）

1860 看板是按老版 node_exporter 写的，v1.6.1 改过一些指标名，所以会有：

- `Pressure` → `No data`
- `SWAP Used` → `N/A`

**这是正常的**，CPU / 内存 / 磁盘 / 网络这些核心面板有数据就够用。生产上没人会为了两个空面板去改看板。

---

## 五、Grafana 在真实环境中的作用

> 说明：这一节是**行业普遍做法的归纳，不是从本机测出来的**，属于认知性内容。进了公司可以对照验证。

### 5.1 它是什么、不是什么

**一句话**：Grafana 是**可视化层**——自己不采集、不存储任何监控数据，只在打开页面时去数据源查一次，把结果画成图。

```
被监控目标 → Prometheus（抓取并存储）→ Grafana（画图）→ 运维（扫一眼判断）
                    ↑_______________________|
                      打开页面时才去查（PromQL）
```

**由此推出两条实用结论**：

| 现象 | 原因 |
|---|---|
| 数据源挂了 → 看板全白 | 但历史数据**还在 Prometheus 里**，没丢 |
| Grafana 挂了 / 换一台 | 接上同一个 Prometheus，看板立刻回来 |

### 5.2 一个看板由什么组成（对着本机界面）

四层：**数据源**（去哪查）→ **顶层条件**（变量 + 时间范围，决定"哪台机器 / 哪段时间"）→ **面板**（一条查询 + 一种画法）→ **多个面板拼成看板**。

本机 1860 看板顶部就有：`Datasource` / `Job` / `Instance` 三个下拉 + `Last 24 hours` + `1m` 刷新。导入 `1860` 等于一次拿到别人写好的 10 条查询。

**Grafana 还能接** Loki（日志）、Elasticsearch、MySQL、云监控，所以公司里常常一个 Grafana 看所有东西。

### 5.3 五类用途（按使用频率）

| # | 场景 | 谁用 | 具体动作 |
|---|---|---|---|
| 1 | **日常巡检** | 值班运维 | 上班打开总览，扫一眼有没有红的、曲线有没有断 |
| 2 | **故障定位（最高频）** | 运维 + 开发 | 告警响了 → 打开看板 → **时间范围拉到出事前 10 分钟** → 看哪条线先拐弯 → 缩小到哪台机器 |
| 3 | **发版观察** | 运维 + 开发 | 上线后盯错误率/延迟曲线，判断这次发布行不行（对应 W4 的滚动更新） |
| 4 | **容量趋势** | 运维/主管 | 磁盘还剩多久满、内存是否在缓慢上涨、QPS 比上周高多少 |
| 5 | **值班大屏/汇报** | 主管/客户 | 墙上电视常开，或截图进周报 |

**第 2 条是它真正的价值**：出事时人不可能去读几百行 JSON，看板让你一眼看出"几点几分哪条线先异常"。

### 5.4 组织里谁干什么

| 角色 | 干什么 |
|---|---|
| 平台 / SRE 团队 | 搭 Grafana、配数据源、**沉淀看板模板**、配告警规则 |
| 多数运维 / 开发 | **只是打开看**，顶多改时间范围和变量下拉 |
| **使用者** | 会看、会选对变量、能说清"哪块红了" |

> 多数公司看板是**别人做好的**，不需要自己写 PromQL。本机做的「导入 1860 + 顶部选实例」就是最真实的日常动作。

### 5.5 它不管什么（别搞混）

| 不管 | 归谁 |
|---|---|
| 采集、存储数据 | Prometheus / Loki / ES |
| **判断该不该告警** | Prometheus 告警规则 + Alertmanager |
| 自动修复 | 人 / 自动化脚本 |

Grafana 自己也能配告警，但生产上大多仍走 Prometheus + Alertmanager。**看板是给人看的，告警是给机器判的。**

### 5.6 代价与坑（比功能重要）

1. **看板会骗人**：时间范围选错、变量选错实例、面板 `No data` 被当成正常。本机就有两个空面板，若那是关键指标就会漏看。
2. **显示有延迟**：Prometheus 15s 抓一次 + 界面 `1m` 刷新 → **看到的是至少 15 秒前的世界**，不是实时。
3. **看板腐化**：指标改名后一堆空面板，没人维护，最后大家都不看了。
4. **权限要收**：生产上不该人人都能 Edit——改错一条查询，所有看的人都看到错数据。
5. **看板要入库**：正规做法是把 Dashboard 的 JSON 放进 Git（`Dashboard settings → JSON Model` 可导出），误删能恢复。

### 5.7 一句话说明这个工具

> "Grafana 是可视化层，自己不采集也不存数据，通过数据源去查 Prometheus、Loki 这些，把结果画成面板拼成看板。主要用于巡检和故障定位：告警触发后把时间范围拉到出事前后，看哪条曲线先异常，缩小到具体机器或服务。看板通常是平台团队沉淀的模板，日常使用会配数据源、导入社区看板、改时间范围和变量。要注意看板显示有延迟，且时间范围或变量选错会看到假象。"

**核心两点：「它不存数据」+「故障定位时拉时间范围」。**

---

## 六、一句话认知（够用即可）

> Prometheus 每 15 秒去各个目标 `GET /metrics`，把数字存起来；
> Grafana 不存数据，只是把 Prometheus 里的数字画成图；
> `up` 是 Prometheus 自己写的"我这次抓到没"，不是应用给的。

三者分工记一句：**Prometheus 负责「记下来」，Grafana 负责「让人看懂」，Alertmanager 负责「出事叫人」。**

**不用记**：TSDB 分块、pull vs push 设计哲学、四种指标类型推导、histogram_quantile、标签基数优化。碰到再查。

---

## 七、K8s 指标（补回 W5 计划，已完成）

> 原第一版计划的 D3 曾被砍掉，2026-11-07 按用户要求重新加回。
> **两个数据源都接齐了**：cAdvisor（容器资源）+ kube-state-metrics（对象状态）。

### 7.1 为什么需要它

`node_exporter` 只看**机器**（CPU 6.5%、内存 85%），完全看不到 Pod。
有了 K8s 指标才能回答：哪个 Pod 在吃资源、哪个 Pod 一直重启、副本有没有到位。

### 7.2 三个数据源的分工

| 组件 | 数据来自 | 提供什么 | 本机状态 |
|---|---|---|---|
| `node_exporter` | 宿主机 `/proc` | 机器级 CPU / 内存 / 磁盘 | ✅ 9100，943 样本 |
| **cAdvisor** | kubelet（内置） | 容器级 CPU / 内存 / 网络 / 磁盘 | ✅ 1761 条 `container_*` |
| **kube-state-metrics** | apiserver | Deployment 副本数、Pod 状态、**重启次数** | ✅ 18080，1102 条 `kube_*` |

> cAdvisor 回答「用了多少资源」，kube-state-metrics 回答「对象现在什么状态」。
> **W4 做的探针 / `RESTARTS 2` / `CrashLoopBackOff` / 副本 `0/1`，只有后者能变成指标。**

### 7.3 kube-state-metrics 部署

镜像在国内源的正确地址（**不是** `docker.m.daocloud.io` 拼 registry.k8s.io 路径）：

```bash
docker pull m.daocloud.io/registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2
docker tag <上面那个> registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2
```

> 曾用 8 个地址都失败，原因是一直用 `docker.m.daocloud.io` 前缀拼 K8s 官方仓库路径。
> 正确映射域名是 **`m.daocloud.io/registry.k8s.io/...`** 或 `k8s.m.daocloud.io/...`。

正式启动命令（8080 被 W2 的 Jenkins 占用，所以映射到 18080/18081）：

```bash
docker run -d --name kube-state-metrics --restart always \
  -p 18080:8080 -p 18081:8081 \
  -v /opt/prometheus/ksm-kubeconfig:/etc/ksm/kubeconfig:ro \
  --memory=150m \
  registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2 \
  --kubeconfig=/etc/ksm/kubeconfig \
  --port=8080 --telemetry-port=8081 \
  --resources=pods,deployments,replicasets,services,endpoints,nodes,namespaces,...
```

**`--resources` 必须限制**：不写的话它默认 watch 所有对象类型，会刷一堆 `Forbidden` 日志。
限制到有权限的类型后，实测日志无 Forbidden。

**RBAC**：复用 `prometheus-k8s` 这个 ServiceAccount，把 ClusterRole 扩成
`pods/deployments/replicasets/services/endpoints/nodes/...` 的 `get/list/watch`。
kubeconfig 写到 `/opt/prometheus/ksm-kubeconfig`，内置 CA + token。

### 7.4 cAdvisor 采集路径

cAdvisor 的 `/metrics/cadvisor` 在 kubelet 10250 端口，**需要认证**（实测 401）。
所以走 **apiserver proxy**，用只读 SA 的 token：

```
Prometheus → https://192.168.245.133:6443
             /api/v1/nodes/k8s-node1/proxy/metrics/cadvisor
             Authorization: Bearer <token>
```

| 项 | 值 |
|---|---|
| token 文件 | `/opt/prometheus/k8s.token` → 容器内 `/etc/prometheus/k8s.token` |
| 权限 | 属主 `65534:65534` + `600`（容器以 nobody 运行，**不能设成 600 root**） |
| 验证 | 带 token 请求返回 200 |

**权限坑**：容器以 `nobody(65534)` 跑，token 若设成 `-rw------- root root`，容器内读不到。
`chmod 644` 虽能读但宿主机任何人可见 token，正确做法是 `chown 65534:65534` + `600`。

### 7.5 完整 scrape 配置

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.245.150:9100']
  - job_name: 'k8s-cadvisor'
    scheme: https
    metrics_path: /api/v1/nodes/k8s-node1/proxy/metrics/cadvisor
    bearer_token_file: /etc/prometheus/k8s.token
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets: ['192.168.245.133:6443']
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['192.168.245.150:18080']
```

**注意**：`metrics_path` 写死 `k8s-node1`，只抓一个节点。生产用 `kubernetes_sd_configs`
自动发现所有节点——那属于平台团队的活，当前不学。

**改配置前先校验**（避免 reload 才报错）：

```bash
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

### 7.6 实测验收（2026-11-07）

| 检查 | 结果 |
|---|---|
| `promtool check config` | `SUCCESS` |
| `reload` | `HTTP=200` |
| 四个 target | `k8s-cadvisor` / `kube-state-metrics` / `node` / `prometheus` 全 **up** |
| `kube_deployment_status_replicas_available{deployment="hello"}` | **`2`** |
| `kube_pod_container_status_restarts_total{namespace="default"}` | **`3`**（W4 坏镜像实验留下的） |
| `curl 18080/metrics \| grep -c '^kube_'` | **1102** |

### 7.7 Grafana 看板

| ID | 名称 | 说明 |
|---|---|---|
| **1860** | Node Exporter Full | 机器监控，已导入 |
| **7249** | Kubernetes Cluster | **K8s 总览，已导入** |

**为什么选 7249**：对比过 6417（25 个指标本机只有 11 个）、13332（31 有 18）、8588（17 有 8），
7249 依赖的 `node_*` + `container_*` + `kube_*` **本机三类全齐**，是唯一不会大面积空白的现成看板。

**看板怎么读**（值班时只看这几块）：

| 面板 | 看什么 |
|---|---|
| Pod Restarts | 时间窗内**新增**重启次数，应为 0 |
| Pods Running Count | Failed / Pending 应为 0 |
| Min Space / Min Mem | 磁盘、内存余量，逼近红线要处理 |
| CPU Idle Avg / Memory | 看**曲线拐点**，不看绝对值 |

**`N/A` / `No data` 不用修**：需要业务埋点或老指标改名，核心面板有数就够。
判断某块面板算的是累计值还是增量：点标题 → **Edit** → 看 `expr`。

> 例：`Pod Restarts = 0` 是**正确的**——它算的是"过去 30 分钟新增重启"，
> 而 W4 那 3 次是两天前的，所以累计值 3、面板显示 0，两者不矛盾。

### 7.8 看板会骗人：标签改名导致图表算错（实测）

**这比 `No data` 危险**：空面板一眼能看出没数据；**一条看起来正常的错曲线会被当成真的**。

#### 怎么看到 expr

| 路径 | 说明 |
|---|---|
| 面板右上角 ⋮ → **Inspect** → **Panel JSON** | 最快的只读方式，看完关掉即可 |
| 面板标题下拉 → **Edit** → 下方 Query 区 **Code** | 能看查询结构，看完别点 Save |
| 看板右上角齿轮 → **JSON Model** | 一次看全部面板的 expr（也是生产上存进 Git 的那个 JSON） |

#### 实例：7249 的 `Top 5 Memory Intense Pods`

编辑页里的原始 expr：

```
topk(5, sum(container_memory_usage_bytes{image!=""}) by (pod_name))
```

逐段：`container_memory_usage_bytes` = 容器当前内存 → `{image!=""}` 排掉汇总行
→ `sum(...) by (pod_name)` 按 Pod 分组求和 → `topk(5, ...)` 取最大的 5 个。

**问题**：本机 cAdvisor 的真实标签是 **`pod`**，不是 `pod_name`（`container_name` 同理改成了 `container`）。
老看板用的是旧标签名，查询**不报错、照样出图**，只是含义全变了。

实测分组结果：

| 分组 | 序列数 | 含义 |
|---|---|---|
| `by (pod_name)` | **1** | 标签不存在 → 所有容器被塞进同一组求和 |
| `by (pod)` | **5** | 正确，能看到各个 Pod |

原查询实际返回：

```json
{"metric":{}, "value": [..., "4923617280"]}
```

`metric` 为空、`4,923,617,280` 字节 ≈ **4.59 GiB** —— 这不是"最吃内存的 5 个 Pod"，
而是**所有容器内存的合计**。截图里只有一条叫 `Value` 的曲线，就是这么来的。

**由此解释了另一个矛盾**：1860 的 `Memory` 面板是 `937 MiB`（单容器），
7249 这块是 `4.59 GiB`（全部求和），两者不是指标不同，是**分组标签失效**。

#### 修法（当前不修，留作参考）

```
topk(5, sum(container_memory_usage_bytes{image!="", container!="POD"}) by (pod))
```

`container!="POD"` 顺手排掉 pause 容器。改完应出现多条曲线，图例带 `pod="hello-..."`。

#### 结论（比修图更重要）

> **看板上的数字要能追溯回 expr。** 遇到数值看着不对，第一反应是点开 Edit 看它到底算的什么。
> 老看板 + 新版组件 = 标签改名，`No data` 只是最温和的一种后果；
> **静默算错**才是真正会误导判断的。

---

## 八、告警规则（已完成）

### 8.1 规则文件

`/opt/prometheus/rules/basic.yml`：

```yaml
groups:
  - name: basic
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} 抓不到了"
```

| 字段 | 作用 |
|---|---|
| `expr: up == 0` | 条件：某个目标抓不到了 |
| `for: 1m` | **持续 1 分钟才确认**，防止网络抖动误报 |
| `labels.severity` | 级别标签，以后按级别路由 |
| `annotations.summary` | 人看的说明，`{{ $labels.job }}` 自动填具体 job |

### 8.2 挂进 Prometheus

在 `prometheus.yml` **顶层**追加（与 `global` / `scrape_configs` 同级）：

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml
```

路径写**容器内**的 `/etc/prometheus/...`，不是宿主机 `/opt/prometheus/...`。

```bash
docker exec prometheus promtool check rules /etc/prometheus/rules/basic.yml
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

浏览器看：`http://192.168.245.150:9090/alerts`

### 8.3 真的触发一次（已完成）

```bash
docker stop node-exporter     # --restart always 不会把手工 stop 的容器拉起来
sleep 30                      # 状态 = Pending（观察期，不报）
sleep 40                      # 跨过 for: 1m → Firing
docker start node-exporter    # 恢复 → Inactive
```

**实测三段状态**：

| 时刻 | 状态 | 含义 |
|---|---|---|
| 刚停 | Inactive → Pending | 条件满足，进入观察期 |
| 30 秒 | **Pending** | `for: 1m` 未走完，不报 |
| 70 秒 | **Firing(1)** | `instance=192.168.245.150:9100`，`job=node`，`value=0` |
| 恢复后 | **Inactive(1)** | 告警清零，四个 target 全 up |

> **今天不装 Alertmanager**：只做 Prometheus 侧规则 + 界面看状态。
> 通知渠道（钉钉/邮件/短信）属于后续内容，不影响"告警规则"本身。

---

## 九、待办

- [x] 把 `job_name: 'dead-target'` 改成 `job_name: 'node'`
- [x] 接上 K8s 指标：cAdvisor + kube-state-metrics（见第七节）
- [x] 导入 Grafana 看板 1860（机器）+ 7249（K8s）
- [x] 补一条简单告警规则并真的触发一次（见第八节，已实测 Firing）
- [x] W5 周复盘 `week05-总结.md`，README 进度表标 ✅
- [ ] Shell 每日 1 题（基础能力，优先级高于复杂查询）
