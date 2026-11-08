# Week5 总结：监控（Prometheus + Grafana + 告警）

> 周期：2026-11-04 ~ 2026-11-08
> 环境：k8s-node1 `192.168.245.150`，kubeadm v1.23.17，docker 26.1.4
> 目标口径：**会用、能干活**，不追究底层原理

---

## 一、目标与达成

| 原定（第一版） | 第二版（按实际目标压缩） | 结果 |
|---|---|---|
| 系统学完监控栈 6 单元 | 4 件活：起 Grafana / 加目标 / 配告警 / 排错速查表 | ✅ 全部完成 |
| D2 PromQL 四个函数 | ❌ 砍掉 | 看板是现成的，不写查询 |
| D3 K8s 指标 | ✅ **用户要求加回，两个组件都接齐** | 超额完成 |
| D5 Alertmanager 全链路 | 🟡 压缩为 Prometheus 规则 + 真触发 | ✅ 闭环 |

**整体判断**：技术目标全部达成，且**中途根据用户反馈修正了两次方向**。

---

## 二、五天实际进度

| 日期 | 做了什么 |
|---|---|
| 11/04 | W4 复盘；Prometheus v2.45.0 + node_exporter 跑通，产出 D1 笔记（4 个坑） |
| 11/06 | **用户明确目标修正**（会用、能干活，不求深度理解）→ 计划压缩为任务驱动；Grafana 10.2.0 起、接数据源、导入 1860；产出「监控够用清单」 |
| 11/07 | 用户要求把 K8s 指标加回计划 → cAdvisor + kube-state-metrics 全接齐、导入 7249；发现并修正「镜像拉不到」的错误结论；补充「看板会骗人」实例 |
| 11/08 | Shell 每日 1 题（首次完成）；告警规则 `TargetDown` 上线并**真的触发一次**（Inactive → Pending → Firing → 恢复） |

**断更**：11/05 无提交。其余 4 天均有产出。

---

## 三、最终环境状态

| 组件 | 版本 / 端口 | 状态 |
|---|---|---|
| Prometheus | v2.45.0 / 9090 | ✅ `--restart always`，`retention=6h`，`--web.enable-lifecycle` |
| node_exporter | v1.6.1 / 9100 | ✅ host 网络 + `--pid host` |
| **kube-state-metrics** | v2.9.2 / **18080** | ✅ 1102 条 `kube_*` 指标 |
| Grafana | 10.2.0 / 3000 | ✅ 看板 1860（机器）+ 7249（K8s） |
| **告警规则** | `basic.yml` / `TargetDown` | ✅ `up == 0` + `for: 1m` |

**四个 scrape target 全 up**：`prometheus` / `node` / `k8s-cadvisor` / `kube-state-metrics`。

---

## 四、关键实测数据（可直接引用）

| 检查 | 结果 |
|---|---|
| `scrape_samples_scraped` | node 943 / prometheus 419（随负载浮动） |
| cAdvisor `container_*` 指标 | 1761 条 |
| kube-state-metrics `kube_*` 指标 | 1102 条 |
| `kube_deployment_status_replicas_available{deployment="hello"}` | **2** |
| `kube_pod_container_status_restarts_total{namespace="default"}` | **3**（W4 坏镜像实验残留） |
| `container_memory_working_set_bytes{pod=~"hello-.*"}` | 约 2.8 MB / Pod |
| 看板 1860 读数 | CPU Busy 6.5% / RAM 85.1% / Root FS 69.3% / CPUs 2 |
| 告警触发 | `Firing(1)`，`instance=192.168.245.150:9100`，`job=node`，`value=0` |
| 告警恢复 | `docker start` 后回到 `Inactive(1)` |

---

## 五、踩坑清单（本 W5 共 11 条，全部真实踩过）

1. **`docker run` 参数整体替换镜像 CMD**：只写 1 个参数 → `--config.file` 丢失 → 找 `/prometheus/prometheus.yml` 退出码 2。用 `docker inspect <img> --format '{{json .Config.Cmd}}'` 预查。
2. **容器内 `localhost` ≠ 宿主机**：Prometheus 用 bridge 网络，`127.0.0.1:19999` 连不上；而 `localhost:9090` 一直 UP 是因为目标是它自己，反而掩盖问题。
3. **`sed -i` 换 inode → 单文件 bind mount 失效（最阴险）**：`reload` 返回 200 但读旧 inode。修法：**挂目录不挂单文件**。
4. **`--web.enable-lifecycle` 默认关闭**（安全默认值），不加则 reload 返回 403。
5. **`up` 判定标准是 HTTP 2xx，不是"有数据"**：404 判失败；`up=1` 但 `scrape_samples_scraped=0` 是"通了但空"，最危险。
6. **token 文件权限**：Prometheus 容器以 `nobody(65534)` 跑，`600 root` 读不到；`644` 能读但宿主机任何人可见 token → 正确做法 `chown 65534:65534` + `600`。
7. **8080 被 W2 的 Jenkins 占用** → kube-state-metrics 映射到 18080/18081。
8. **镜像源地址拼错**：`docker.m.daocloud.io/registry.k8s.io/...` 全部失败；正确是 **`m.daocloud.io/registry.k8s.io/...`** 或 `k8s.m.daocloud.io/...`。
9. **`--resources` 必须限制**：不写会默认 watch 所有对象类型，刷一屏 `Forbidden`。
10. **老看板标签改名导致静默算错**：7249 用 `by (pod_name)`，新版 cAdvisor 标签叫 `pod` → 所有容器被聚合成一条，显示 4.59 GiB 而非单个 Pod。比 `No data` 更危险。
11. **VM 重启后容器没自启** → 全部加 `--restart always`。

---

## 六、能力盘点（现在会什么）

1. 起一套监控栈并配开机自启（Prometheus + node_exporter + kube-state-metrics + Grafana）
2. 改 `prometheus.yml`、`promtool check config` 校验、`/-/reload` 热重载
3. 判断 `/targets` 的 UP/DOWN，按 5 步顺序排查（看板空白 → lastError → `up=1` 但空 → 配置没生效 → 容器全没）
4. 导入社区看板（1860 / 7249），选变量、拉时间范围、读关键面板
5. **点开 Edit 看 expr**，判断数字是累计值还是增量、是否算错
6. 写一条告警规则并触发它，理解 `for` 为什么能防抖动
7. 用只读 ServiceAccount + token 抓 apiserver proxy 的 cAdvisor 指标

**还是没碰**：Alertmanager 通知渠道、PromQL 复杂查询、histogram_quantile、长期存储（Thanos）、日志（Loki）。碰到再查。

---

## 七、三条最大收获

1. **监控的终点不是"能看图"，是"出事有信号"。** 今天 `Firing` 那一刻，整条链才真正闭环。
2. **`for` 是告警里最实用的参数**——网络抖一下不该半夜把人叫起来。
3. **看板上的数字要能追溯回 expr。** 空面板不骗人，静默算错的曲线才骗人。

---

## 八、教学方法的两次修正（本周最重要）

| 时间 | 问题 | 修正 |
|---|---|---|
| 11/06 | 先抛原理、概念并列堆叠 → 用户"迷迷糊糊、学完没印象" | 改为**任务驱动**：每个知识点落到能敲的命令，原理只在排错需要时给一句 |
| 11/07 | 把"我没解决的镜像障碍"包装成"对你目标不必要，砍掉" | **不诚实，已纠正**：查到正确镜像地址并把 K8s 指标全接齐 |

**教训**：不要把一个技术障碍说成方法论取舍；用户能识破，而且会因此少学该学的东西。

---

## 九、遗留与下一步

- [ ] Alertmanager 通知渠道（钉钉/邮件）——W6 或碰到再补
- [ ] Shell 每日 1 题：本周**首次完成 1 题**（日志统计 Top IP），需要变成固定动作
- [ ] W6：全链路整合（CI/CD + 监控串起来）

---

## 十、本周产出文件

| 文件 | 内容 |
|---|---|
| `notes/Week5-D1-Prometheus与metrics本质.md` | 概念与 4 个坑（作为查手册保留） |
| `notes/Week5-监控够用清单.md` | **主产出**：四个动作 / 看板怎么读 / 排错顺序 / Grafana 定位 / K8s 指标完整实测 / 告警规则 |
| `weekly/week05-总结.md` | 本篇 |
