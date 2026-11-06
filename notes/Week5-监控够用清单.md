# 监控够用清单（Grafana + Prometheus）

> 面向目标：**会用、能干活**，不追究原理。
> 环境：k8s-node1 `192.168.245.150`
> 适用：6-8k 初级运维岗日常——打开看板、看状态、加目标、判断异常

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

## 二、看板怎么读（面试能说的部分）

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

## 五、一句话认知（够用即可）

> Prometheus 每 15 秒去各个目标 `GET /metrics`，把数字存起来；
> Grafana 不存数据，只是把 Prometheus 里的数字画成图；
> `up` 是 Prometheus 自己写的"我这次抓到没"，不是应用给的。

**不用记**：TSDB 分块、pull vs push 设计哲学、四种指标类型推导、histogram_quantile、标签基数优化。碰到再查。

---

## 六、待办

- [ ] 把 `job_name: 'dead-target'` 改成 `job_name: 'node'`（实验残留，命名要见名知意）
- [ ] 补一条简单告警规则并真的触发一次
- [ ] Shell 每日 1 题（对 6-8k 岗比 PromQL 重要）
