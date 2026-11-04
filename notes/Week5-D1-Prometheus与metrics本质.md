# Week5 D1：Prometheus 跑通与 metrics 的本质

> 时间：2026-11-04（东八区）
> 环境：k8s-node1 `192.168.245.150`，CentOS 7，docker 26.1.4
> 组件：Prometheus v2.45.0（容器 9090）+ node_exporter v1.6.1（容器 9100，host 网络）

---

## 一、metrics 到底是什么

**一句话：一个纯文本的 HTTP 页面，每行是「指标名 + 标签 + 数值」。**

没有协议、没有认证、没有数据库。`curl` 就能看——**Prometheus 干的事和 `curl` 一模一样**。

真实内容（本机 node_exporter）：

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0.5"} 8.969e-05
# TYPE go_goroutines gauge
go_goroutines 32
```

**只有三类行**：

| 行式 | 作用 | 谁消费 |
|---|---|---|
| `# HELP 名 说明` | 人类可读注释 | 人 / Grafana |
| `# TYPE 名 类型` | 声明类型 | 机器（决定能否用 `rate()`） |
| `名{标签="值"} 数值` | **真正的数据** | 机器 |

---

## 二、时间序列：理解 Prometheus 的唯一门槛

`go_goroutines 32` **不是**时间序列，它只是一个点。

时间序列 = **一组标签 + 一条按时间排列的点**：

```
{__name__="go_goroutines", job="prometheus", instance="localhost:9090"}
  13:05:46 → 32
  13:06:01 → 31
  13:06:16 → 33
```

> **决定"这是哪条序列"的是标签组合，不是指标名。**
> `go_goroutines{job="a"}` 和 `go_goroutines{job="b"}` 是两条完全独立的序列。

所以 PromQL 里到处是 `sum by()` / `rate()`——**你永远在处理"一堆序列"，不是"一个数字"**。

---

## 三、四种指标类型（决定能不能算速率）

| 类型 | 特点 | 例子 | 正确用法 |
|---|---|---|---|
| **Counter** | 只增不减，重启归零 | `http_requests_total` | **必须 `rate()`**，直接看值无意义 |
| **Gauge** | 可增可减，当前值 | `go_goroutines`、内存 | 直接看 / `avg_over_time()` |
| **Histogram** | 按桶分组计数 | 请求延迟 | `histogram_quantile()` 算 P99 |
| **Summary** | 客户端算好的分位数 | `go_gc_duration_seconds{quantile="0.5"}` | 直接读分位 |

**为什么 Counter 必须 `rate()`**：

```
http_requests_total = 1000     ← 无意义（开机至今累计）
rate(http_requests_total[1m])  ← 有意义（每秒请求数）
```

> **监控的本质：你几乎从不关心"累计多少"，只关心"现在多快"。**

---

## 四、Prometheus 的工作流程

```
① 定时器每 15s（scrape_interval）
     ↓
② GET 每个 target 的 /metrics
     ↓
③ 解析文本 → 时间序列（自动加 job / instance 标签）
     ↓
④ 存进本地 TSDB（按 2 小时分块，每个样本 = 时间戳 + 值）
     ↓
⑤ PromQL 查询（Grafana / API / 告警规则都走这里）
```

**目标侧只做一件事**：暴露 `/metrics`。它**不知道 Prometheus 在哪，也不主动发任何东西**。

### 为什么是 pull 而不是 push

| 理由 | 说明 |
|---|---|
| **被监控方零配置** | 不用告诉它监控服务器在哪。容器环境极重要（Pod IP 天天变） |
| **能判断目标死没死** | 拉不到 = `up=0`。push 模型里目标挂了只是"没数据"，**分不清是故障还是空闲** |
| **调试极简** | `curl 目标/metrics` 就能看到 Prometheus 能看到的一切 |

### 代价（必须知道）

| 代价 | 说明 |
|---|---|
| **永远滞后** | 最坏要等一个 `scrape_interval`（15 秒起步）才发现故障。这是设计，不是配置问题 |
| **短命任务拉不到** | CronJob 跑 3 秒就退出 → 要靠 **Pushgateway** 补 |
| **数据不长期** | 本地 TSDB 默认只留 15 天（本次配 `6h`），长期要远程存储 / Thanos |

---

## 五、`up` 的真实含义（今晚踩出来的）

`up` **不是应用吐出来的**，是 **Prometheus 每次抓取后自己写的**："我这次拿到了吗"。

**判定成功的标准是「HTTP 状态码 2xx」，不是"有没有数据"。**

### 四种组合（必须背下来）

| 服务端返回 | `up` | 含义 |
|---|---|---|
| **200 + 有指标** | **1** | ✅ 正常 |
| **200 + 空内容** | **1** | ⚠️ **通了但没指标**——最阴险，监控形同虚设 |
| **404 / 500** | **0** | ❌ 服务在，路径或应用坏了 |
| **connection refused / timeout** | **0** | ❌ 服务不在，或网络不通 |

**所以生产上除了 `up`，必须同时看 `scrape_samples_scraped`**：

```promql
scrape_samples_scraped{job="node"}   # 应该几百；如果是 0，就是"通了但是空的"
```

---

## 六、实测数据（今晚的真实过程）

| 阶段 | 现象 | 学到什么 |
|---|---|---|
| 初始 | `up{dead-target}=0`，`connection refused` | 目标没监听 |
| 起 faketarget（19999） | `up` 仍 0，报 `dial tcp [::1]:19999: connection refused` | **容器内 `localhost` ≠ 宿主机** |
| 改指向 `192.168.245.150:19999` | `up=0`，报 `HTTP 404` | **`up` 判的是 2xx**，404 也算失败 |
| 换 node_exporter（9100） | `up=1`，`scrape_samples_scraped=914` | 通了且有数据 |

**最终状态**：

```
up{job="dead-target", instance="192.168.245.150:9100"} = 1
scrape_samples_scraped{job="dead-target"} = 914
up{job="prometheus", instance="localhost:9090"} = 1
scrape_samples_scraped{job="prometheus"} = 443
```

> `914` 与 `grep -c "^node_"` 得到的 `868` 不同：`scrape_samples_scraped` 统计**全部样本**（`node_*` + `go_*` + `process_*` + `promhttp_*`），不只 `node_` 前缀。

---

## 七、今晚踩的四个坑

### 坑 1：`docker run 镜像 参数` 会整体替换 `CMD`，不是追加

**现象**：

```
Error loading config (--config.file=prometheus.yml)
file=/prometheus/prometheus.yml
err="open prometheus.yml: no such file or directory"
```

**根因**：prom/prometheus 镜像默认 `CMD` 有 4 个参数：

```
--config.file=/etc/prometheus/prometheus.yml
--storage.tsdb.path=/prometheus
--web.console.libraries=...
--web.console.templates=...
```

只写了一个参数 → **整个 CMD 被替换** → `--config.file` 丢失 → 退回内置默认相对路径 `prometheus.yml` → 在 `WorkingDir=/prometheus` 下找 → 不存在 → 退出码 2。

**查默认参数的命令**：

```bash
docker inspect <镜像> --format '{{json .Config.Cmd}}{{"\n"}}{{.Config.WorkingDir}}'
```

**修法**：把需要的参数**全部写全**。

### 坑 2：容器内 `localhost` 不是宿主机

**现象**：宿主机 `curl 127.0.0.1:19999` 通，Prometheus 容器内不通。

**根因**：Prometheus 用 bridge 网络（只 `-p 9090:9090`），它的 `127.0.0.1` 是**容器自己**。宿主机上的 19999 在它的 `127.0.0.1` 上不存在。

> 而 `localhost:9090` 那个 job 一直 UP，因为那个目标是 **Prometheus 自己**——容器内的 9090 确实有它。这反而掩盖了问题。

**修法**：写**宿主机的真实 IP**（`192.168.245.150`），或让目标容器用 `--network host`。

### 坑 3：`sed -i` 换 inode，导致单文件 bind mount 失效（今晚最值钱）

**现象**：宿主机配置已改，`curl -X POST /-/reload` 返回 **200**，但目标还是旧的。

**证据**：

| | inode | 内容 |
|---|---|---|
| 宿主机 | 37223198 | `...:9100` ✅ |
| 容器内 | 37073634 | `...:19999` ❌ |

**根因**：

```
sed -i 实际做的是：新建文件 → 写入 → rename 覆盖 → 删除旧 inode
Docker 单文件 bind mount 绑的是 inode，不是路径
→ 容器仍持有旧 inode，宿主机怎么改都看不到
```

**为什么最阴险**：`reload` 返回 200，日志不报错，**一切看起来正常**。

**快速判断**：

```bash
stat -c %i /宿主机/文件
docker exec 容器 stat -c %i /容器内/文件
# 不一样 → 就是这个坑
```

**修法（根治）**：**挂目录，不挂单文件**。

```bash
-v /opt/prometheus:/etc/prometheus     # ✅ 目录挂载，sed -i 也能生效
-v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml   # ❌ 单文件，怕 sed -i
```

**为什么 K8s 的 ConfigMap 没这问题**：kubelet 用 `..data` 符号链接整体切换，容器始终解析到最新。**这是 K8s 比裸 Docker 更适合配置管理的具体原因之一。**

**补充**：inode 已经错位后，`cat >` 也救不回来（宿主机 inode 已变，容器死抱旧的）。只能重建容器。

### 坑 4：`--web.enable-lifecycle` 默认关闭

reload 端点默认不可用（返回 403）。**这是安全默认值**——否则任何能访问 9090 的人都能让 Prometheus 重载配置。

**修法**：启动参数加 `--web.enable-lifecycle`。

---

## 八、被监控端怎么暴露 metrics（五条路）

| 路径 | 适用 | 代价 |
|---|---|---|
| **① 跑现成 exporter** | 机器/MySQL/Redis/Nginx/K8s | 零代码改动，但多一个进程要维护 |
| **② 应用自己埋点** | 有自己的服务 | **唯一能看业务指标**；要改代码+重发；指标名难改 |
| **③ 翻译层** | 应用吐自家格式（如 nginx `stub_status`） | 多一跳，多一个失败点 |
| **④ Blackbox 外部探测** | 改不了的目标 / 想知道"外部能不能访问" | 只能看外部视角，看不到内部 |
| **⑤ Pushgateway** | 短命任务（CronJob） | **旧值会一直留着，`up=1` 会骗人**，必须配 `push_time_seconds` 判断 |

**本机实际情况**：`hello` 应用（nginx）`GET /metrics` 和 `/stub_status` 都是 **404**——**这就是真实世界的常态，绝大部分应用天生不暴露指标。**

---

## 九、指标设计（比"怎么暴露"更重要）

### 四大黄金指标（Google SRE）

| 指标 | 含义 | 类型 |
|---|---|---|
| **Latency** | 延迟，**要区分成功/失败的延迟** | Histogram |
| **Traffic** | QPS | Counter |
| **Errors** | 错误率 | Counter |
| **Saturation** | 饱和度（CPU、连接池、队列） | Gauge |

### 命名规范

```
<namespace>_<subsystem>_<name>_<unit>

http_server_requests_total      ✅
requestCount                    ❌ 没单位、没类型提示
```

- **Counter 必须以 `_total` 结尾**（约定，很多工具依赖）；
- **单位用基本单位**：秒不是毫秒，字节不是 KB；
- 不要手工加 `_count`（那是 Summary 自动生成的）。

### 最致命的坑：标签基数爆炸

**绝不能当标签的**：

```python
Counter('requests_total', ['user_id', 'request_id', 'url_full'])   # 灾难
```

```
序列数 = 各标签取值范围之积
user_id(10万) × request_id(无限) × status(5) = 无法计算
```

**每个序列都占 TSDB 内存 → 标签一爆炸，Prometheus OOM 崩掉。**

**正确做法**：
- 高基数信息**进日志，不进标签**；
- URL 用**路由模板**（`/user/:id`），不用真实路径；
- 状态码可保留具体值（5~10 个可以，几百个不行）。

> **面试常问**："你们监控为什么崩了？" 标准答案是**"标签基数爆炸"**，不是"数据太多"。

---

## 十、技术要点（面试用）

> "metrics 是一个纯文本 HTTP 接口，每行是'名字+标签+数值'。Prometheus 是爬虫+时序数据库+查询引擎三合一，按 scrape_interval 主动 GET 目标的 /metrics，解析成时间序列存进本地 TSDB，再用 PromQL 查询。
>
> 选 pull 不选 push 的核心原因：它能用 `up` 这个指标把'目标是否存活'变成可查询的信号。push 模型里目标挂了只是'没数据'，分不清故障和空闲。代价是短命任务拉不到，要 Pushgateway 补。
>
> 一个关键细节：`up` 判定成功的标准是 HTTP 2xx，不是有没有数据。所以生产上必须同时看 `scrape_samples_scraped`，否则'通了但是空的'会让监控形同虚设。
>
> 实施上，被监控端有五条路：跑 exporter（零改造）、应用埋点（唯一能看业务指标）、翻译层、blackbox 外部探测、Pushgateway（短命任务）。指标设计上最致命的是标签基数爆炸，user_id 这种高基数字段绝不能进标签，否则 TSDB 会被撑爆。"

---

## 十一、环境快照

| 组件 | 配置 |
|---|---|
| Prometheus | `prom/prometheus:v2.45.0`，`-p 9090:9090`，`--memory=400m`，`retention=6h`，`--web.enable-lifecycle` |
| 配置挂载 | **`-v /opt/prometheus:/etc/prometheus`（整目录）** |
| node_exporter | `prom/node-exporter:v1.6.1`，`--network host`，`--pid host`，挂 `/proc` `/sys` `/rootfs` |
| 抓取目标 | `localhost:9090`（自身，443 样本）、`192.168.245.150:9100`（914 样本） |
| 镜像加速 | `docker.m.daocloud.io/prom/...` 拉取后 `docker tag` 回标准名 |

---

## 十二、下一单元

**D2：PromQL**——现在手上有 1357 个真实样本，可以开始查：

- `rate()` 为什么必须用于 Counter
- `sum by()` / `avg by()` 聚合
- 算 CPU 使用率、内存使用率
- `histogram_quantile()` 算 P99
