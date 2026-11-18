# 01 · Docker 容器化

> 核心一句话：**容器不是"轻量虚拟机"，是"被隔离和限额的普通进程"。**
> 这句话能不能站住，决定了后面所有原理题和排障题。

---

## 0. 一分钟版本

> 我把一个 Go 写的 HTTP 服务容器化。最开始用 `golang:1.21` 单阶段构建，镜像 884MB——因为它把整个编译环境（Debian、Go 工具链、gcc、源码、构建缓存）都装进了交付物，而这些运行时一个都不需要。改成多阶段构建后，第二阶段换成 `alpine:3.19`（7.7MB），只 `COPY --from` 编译好的二进制（6.4MB），最终 14.1MB，缩小 62.7 倍。按 20 节点集群算，一次全量分发从 17.3GB 降到 282MB，同时攻击面大幅缩小。

---

## 一、原理层

### 1.1 镜像到底是什么

**不是"一个压缩包"，是"一组只读层 + 一份清单"。**

```
镜像 = manifest（清单）+ config（启动配置）+ N 个 layer（文件系统差异）
```

| 概念 | 本质 |
|---|---|
| layer | 一次构建步骤产生的**文件系统 diff**（tar 包），内容寻址（sha256） |
| digest | 整个镜像内容的 sha256，**唯一身份** |
| tag | 指向 digest 的**可变别名**（`latest` 就是最典型的可变别名） |

**为什么要分层**：
- **复用**：10 个镜像都基于 `alpine:3.19`，这一层在磁盘和传输上只有一份
- **增量拉取**：改了应用代码，只需要拉最上面那一层

**代价**（这几个点经常被追问）：

| 现象 | 原因 |
|---|---|
| 镜像里 `rm` 了一个大文件，体积没变小 | 删除只在下层之上加了一个 whiteout 标记，**下层的字节还在**。要真变小，必须在**同一层内**完成"下载+删除" |
| 层数太多启动慢 | 联合挂载的层数越多，文件查找路径越长（overlayfs 上限通常 128 层） |
| 同一份内容 build 两次 digest 不同 | 构建时间戳等元数据进了层 |

### 1.2 容器是什么（这是最容易被问倒的一题）

**容器 = 一个普通进程 + 三样东西：**

| 机制 | 作用 | 具体是什么 |
|---|---|---|
| **namespace** | 隔离"看得见什么" | PID / NET / MNT / UTS / IPC / USER / CGROUP |
| **cgroup** | 限制"能用多少" | CPU、内存、IO、PID 数量 |
| **rootfs** | 决定"文件系统长什么样" | 镜像的只读层 + 一个可写层，用 overlayfs 联合挂载 |

**关键事实**：容器里的进程，在宿主机上**就是普通进程**。

```bash
# 宿主机上能直接看到容器里的进程
ps aux | grep <容器内进程名>
# 容器里的 1 号进程，在宿主机上有个真实的 PID
```

**和虚拟机的根本区别**：

| | 容器 | 虚拟机 |
|---|---|---|
| 内核 | **共享宿主机内核** | 各自独立内核 |
| 启动 | 毫秒~秒级（就是起进程） | 数十秒（要引导内核） |
| 隔离强度 | 弱（内核共享，一个内核漏洞可能穿透） | 强（硬件虚拟化边界） |
| 能跑异构 OS | ❌ Linux 上跑不了 Windows 容器 | ✅ |
| 密度 | 高（无内核开销） | 低（每个 VM 一套内核） |

> **一句话答法**："容器是共享内核的隔离进程，虚拟机是独立内核的完整系统。所以容器快、密度高，但隔离强度不如 VM，也不能跑异构系统。"

### 1.3 写时复制（Copy-on-Write）

镜像所有层**只读**，容器启动时在最上面加一个**可写层**。

| 操作 | 实际发生什么 |
|---|---|
| 读文件 | 从最上层往下找，找到就返回（**下层优先**） |
| 改文件 | 把文件从只读层**复制到可写层**再改（copy-up） |
| 删文件 | 在可写层加 whiteout 标记，**下层字节不动** |

**这解释了两个常见现象**：
1. 容器删了重启，改动全没了（可写层随容器销毁）
2. 容器里改了大文件很慢（要先 copy-up）

### 1.4 多阶段构建为什么能小 62 倍

**根因一句话：把"生产过程"和"最终产物"分开。**

| 阶段 | 内容 | 体积 |
|---|---|---|
| 单阶段（优化前） | `golang:1.21` = Debian + Go 工具链 + gcc + 源码 + 构建缓存 + 产物 | **884MB** |
| 阶段 1（builder） | 同上，但**不进入最终镜像** | — |
| 阶段 2（runtime） | `alpine:3.19`（7.7MB）+ 二进制（6.4MB） | **14.1MB** |

```dockerfile
# 阶段 1：编译（这个阶段的所有东西都不会进最终镜像）
FROM golang:1.21 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/devops-demo .

# 阶段 2：运行
FROM alpine:3.19
COPY --from=builder /out/devops-demo /usr/local/bin/devops-demo
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/devops-demo"]
```

**几个必须能解释的点**：

| 问题 | 答法 |
|---|---|
| 两条 `FROM` 是"魔法"吗？ | 不是。**Docker 会为每个 FROM 建一个独立的文件系统**，阶段之间不共享。`COPY --from=builder` 是唯一的通道，只搬你指定的文件 |
| 为什么用 `CGO_ENABLED=0`？ | 静态编译，不依赖 glibc。alpine 用的是 musl libc，动态链接的二进制在 alpine 上会报 `not found`（这个报错极具迷惑性，它不是说文件不存在，是说解释器不存在） |
| 为什么选 alpine 不选 scratch？ | `scratch` 是空镜像（0 字节），最小，但**没有 shell、没有 CA 证书、没有时区数据**。要调试或访问 HTTPS 就得自己塞。alpine 是折中 |
| 62 倍这个数字怎么算？ | `884 / 14.1 ≈ 62.7`。注意是**倍数**不是百分比 |

**业务意义（被问"小有什么用"时）**：

```
20 节点集群，一次全量分发：
  优化前 884MB × 20 = 17.3GB
  优化后 14.1MB × 20 = 282MB
```

三个收益：**拉取快**（发布变快、扩容变快）、**存储省**、**攻击面小**（编译工具链和系统库是漏洞扫描重点，砍掉就不存在这些 CVE）。

### 1.5 网络模型

Docker 默认创建三种网络，加一种常用自定义：

| 类型 | 机制 | 什么时候用 |
|---|---|---|
| **bridge**（默认） | 宿主机建 `docker0` 网桥，每个容器一对 veth pair 接上去，出网走 NAT | 单机、容器间用 IP 互访 |
| **host** | 容器**不建独立网络命名空间**，直接用宿主机网络栈 | 极致性能，但端口会冲突 |
| **none** | 只有 lo，没有网卡 | 完全隔离的批处理任务 |
| **自定义 bridge** | 自己建的网桥，**内嵌 DNS** | **多容器互访（推荐）** |

**默认 bridge 的一个关键限制**（这是我做过对照实验的点）：

> 默认 bridge 网络下，容器之间**只能用 IP 互访，不能用容器名**。
> 因为默认 bridge 没有内嵌 DNS 服务。

自定义网络下，Docker 内置 DNS 监听 `127.0.0.11`，容器名 → IP 自动解析。

```bash
# 对照实验
docker network create mynet
docker run -d --name app --network mynet ...
docker run -d --name proxy --network mynet ...
docker exec proxy ping -c1 app      # 通（DNS 解析成功）

docker run -d --name app2 ...                        # 默认 bridge
docker exec proxy ping -c1 app2     # 不通（名字解析不了）
```

**端口映射的底层**：`-p 8080:80` 是在宿主机 iptables 的 `nat` 表里加 DNAT 规则，把到 8080 的包改写成容器 IP:80。所以：

```bash
# 端口映射没生效时，第一件事是看 iptables 规则在不在
iptables -t nat -L DOCKER -n | grep 8080
```

### 1.6 存储三件套

| 类型 | 位置 | 生命周期 | 典型用途 |
|---|---|---|---|
| **volume** | `/var/lib/docker/volumes/`（Docker 管理） | 容器删了还在 | 数据库数据、需要备份的 |
| **bind mount** | 宿主机任意路径 | 跟宿主机文件一致 | 配置文件、开发时挂代码 |
| **tmpfs** | 内存 | 容器停就没 | 临时缓存、敏感数据 |

**为什么数据库用 volume 不用 bind**：
- volume 由 Docker 管理，`docker volume ls/inspect/prune` 有统一操作面
- bind mount 把宿主机目录结构耦合进容器，换机器就要重建目录
- bind mount 的权限问题（SELinux/uid 映射）在 CentOS 上尤其多

### 1.7 代价与局限（主动说，显得懂边界）

| 局限 | 具体表现 |
|---|---|
| 隔离弱 | 共享内核，容器逃逸风险高于 VM；不能作为强安全边界 |
| 不能异构 | Linux 宿主机跑不了 Windows 容器 |
| 单机编排 | 原生 Docker 只管一台机器；跨机要 Swarm/K8s |
| 日志默认不轮转 | `json-file` driver 默认无上限，**能把磁盘写满**（真实高频事故） |

---

## 二、项目层

### 2.1 我做的三件事

| # | 做的事 | 数据/结果 |
|---|---|---|
| 1 | Go 服务容器化 + 多阶段构建 | 884MB → **14.1MB**（62.7 倍） |
| 2 | `docker-compose` 编排 app + Nginx 反向代理 | 一条命令起两个容器，Nginx 转发到 app |
| 3 | 默认 bridge vs 自定义网络对照 | 自定义网络下容器名可解析，默认 bridge 不行 |

### 2.2 关键取舍（这些比"我会 Docker"值钱）

| 决策 | 选择 | 理由 | 代价 |
|---|---|---|---|
| 基础镜像 | alpine 而非 scratch | 需要 shell 调试 + CA 证书 | 比 scratch 大 7.7MB |
| 编译方式 | `CGO_ENABLED=0` 静态编译 | alpine 是 musl，动态链接会挂 | 某些依赖 cgo 的库用不了 |
| 镜像 tag | 后面 CI 里用 **commit sha** | `latest` 会变，sha 不会 | 需要清理策略，否则 tag 越积越多 |
| 数据持久化 | volume | 可统一管理、可备份 | 不如 bind 直观 |

### 2.3 踩过的坑

**坑 1：`nginx:alpine` 在 CentOS 7 上起不来**

| 项 | 内容 |
|---|---|
| 现象 | 容器启动即退出，日志信息不明确 |
| 排查路径 | 先怀疑配置（compose、volume 挂载）→ 用隔离实验排除（单独 `docker run` 最简参数）→ 排除配置因素 → 锁定镜像本身 |
| 结论 | 指向**镜像依赖较新的内核特性**，CentOS 7 是 kernel 3.10 |
| 处置 | 换成兼容的镜像 tag 后正常 |
| **诚实边界** | **没有深挖到具体 syscall 层面**，这是【未验证】部分，被追问要说清 |

**坑 2：容器间用容器名访问不通**

| 项 | 内容 |
|---|---|
| 现象 | Nginx 配 `proxy_pass http://app:8080` 报 `host not found in upstream` |
| 原因 | 用了**默认 bridge**，没有内嵌 DNS |
| 修法 | 建自定义网络，两个容器都 `--network` 进去 |
| 验证 | `docker exec proxy getent hosts app` 能返回 IP |

**坑 3：镜像 tag 越积越多**

| 项 | 内容 |
|---|---|
| 现象 | registry / 本地镜像列表膨胀 |
| 原因 | 每次构建打新 sha tag，`latest` 被覆盖但旧 sha 一直留着 |
| 处置 | 这是**有意为之**——sha tag 不可变是回滚的地基（见 02 模块），清理要靠保留策略而不是覆盖 |

---

## 三、排障层（现场题）

### 3.1 「容器起不来」——第一件事看退出码

```bash
docker ps -a                              # 看 STATUS 列
docker inspect <容器> --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'
docker logs --tail 50 <容器>
```

**退出码速查（必须背）**：

| Code | 含义 | 常见原因 |
|---|---|---|
| 0 | 正常退出 | 前台进程跑完了（容器需要一个常驻前台进程） |
| 1 | 应用错误 | 代码异常、配置缺失、端口被占 |
| 125 | **docker run 本身失败** | 参数写错（不是容器内的问题） |
| 126 | 命令不可执行 | 文件没有 x 权限 |
| 127 | 命令找不到 | ENTRYPOINT 路径错 / 动态链接器缺失 |
| 137 | **128+9 = SIGKILL** | 最常见是 **OOM**，也可能是 `docker kill` |
| 139 | 128+11 = SIGSEGV | 程序段错误 |
| 143 | 128+15 = SIGTERM | `docker stop` 正常停止 |

**判断是否 OOM**：

```bash
docker inspect <容器> --format '{{.State.OOMKilled}}'   # true 就是内存超限被杀
docker stats --no-stream                                # 看实时内存
```

> **137 和 OOM 不是必然等号**：137 只说明被 SIGKILL，要看 `OOMKilled` 字段确认。

### 3.2 「镜像拉不到」

```bash
docker pull <镜像>    # 先看完整报错，不要猜
```

| 报错关键词 | 含义 | 处置 |
|---|---|---|
| `manifest unknown` | **tag 不存在**（不是网络问题） | 查 `tags/list` 确认 tag |
| `unauthorized` / `denied` | 认证失败 | `docker login`；私有 registry 要配凭据 |
| `no such host` | DNS 解析失败 | 查 `/etc/resolv.conf`、`daemon.json` 的 dns 配置 |
| `TLS handshake timeout` / `x509` | 证书问题 | 私有 registry 用 HTTP 时要在 `insecure-registries` 里登记 |
| `net/http: TLS handshake timeout` 但 `curl` 能通 | 代理/MTU 问题 | 查 `HTTP_PROXY` 环境变量、`--mtu` |

> **国内环境真实高频**：拉不动往往是**镜像地址拼错**而不是网络。我踩过一次——把 `registry.k8s.io/xxx` 硬拼到 `docker.m.daocloud.io/` 后面，正确写法是 `m.daocloud.io/registry.k8s.io/xxx`。**8 个地址全失败，根因是路径规则，不是网络。**

### 3.3 「容器间不通」——按层排查

```bash
# 1) 两个容器在同一个网络里吗
docker inspect <容器> --format '{{json .NetworkSettings.Networks}}' | python -m json.tool

# 2) 名字能解析吗
docker exec <源容器> getent hosts <目标容器名>

# 3) IP 能通吗（排除 DNS 因素）
docker exec <源容器> ping -c2 <目标IP>

# 4) 端口在听吗
docker exec <目标容器> netstat -tlnp        # 或 ss -tlnp

# 5) 宿主机上端口映射生效了吗
iptables -t nat -L DOCKER -n | grep <端口>
```

**判定表**：

| 现象 | 结论 |
|---|---|
| 名字解析失败、IP 通 | DNS 问题 → 用自定义网络 |
| IP 也不通 | 不在同一网络 / 防火墙 / veth 异常 |
| IP 通、端口不通 | 服务没监听（或只监听 127.0.0.1） |
| 容器内通、宿主不通 | 端口没映射 / 映射到错误端口 |
| 宿主通、外部不通 | 宿主机防火墙（firewalld/iptables） |

> **一个高频细节**：服务监听 `127.0.0.1:8080` 时，从容器外**永远访问不到**，因为容器的 127.0.0.1 是容器自己的。要监听 `0.0.0.0`。这是"容器内 curl 通、外面不通"的头号原因。

### 3.4 「磁盘被写满」

```bash
df -h /var/lib/docker
docker system df                    # 分类统计：镜像/容器/卷/构建缓存
docker system df -v                 # 细到单个对象
du -sh /var/lib/docker/* | sort -h  # 谁在占
```

**三个常见来源**：

| 来源 | 特征 | 处置 |
|---|---|---|
| 容器日志（`json-file`） | `/var/lib/docker/containers/*/*-json.log` 巨大 | 配 `log-opts: max-size=100m, max-file=3` |
| 悬空镜像（`<none>`） | `docker images -f dangling=true` 一堆 | `docker image prune` |
| 构建缓存 | `docker system df` 里 Build Cache 很大 | `docker builder prune` |

**注意**：`docker system prune -a` 会删掉**所有未被使用的镜像**，生产上等于把回滚能力删了。**别在生产上随手敲。**

### 3.5 「容器内存超限 / 被 OOM」

```bash
docker run -m 128m ...                  # 限制内存
docker inspect <容器> --format '{{.HostConfig.Memory}}'
docker stats --no-stream                # 看 MEM USAGE / LIMIT
docker inspect <容器> --format '{{.State.OOMKilled}}'
```

**关键认知**：容器内存限制是 **cgroup 限制**，超了内核 OOM killer 直接杀**容器内的进程**，不是"容器变慢"。表现就是**进程突然消失、退出码 137**。

**和 K8s 的对应关系**：K8s 的 `resources.limits.memory` 底层就是设这个 cgroup 值，超了同样是 OOMKilled（见 03 模块）。

---

## 四、追问清单

> 格式：**Q → 标准答法 → 如果继续追问，答什么**

**Q1：容器是轻量级虚拟机吗？**
→ 不是。容器是**共享宿主机内核的隔离进程**，靠 namespace 隔离视图、cgroup 限制资源。VM 有独立内核，走硬件虚拟化。
→ 再追"那隔离性够吗"：作为**部署单元**够用，但**不能当安全边界**。多租户场景要配合 gVisor/Kata 或干脆用 VM。

**Q2：为什么 `rm` 了文件镜像还是那么大？**
→ 因为镜像是**分层只读**的，`rm` 只在上层加了 whiteout 标记，下层字节还在。要在**同一层内**完成下载和删除才会真变小。
→ 再追"怎么修"：把下载和删除写在**同一条 RUN** 里，用 `&&` 连起来。

**Q3：多阶段构建的第二阶段怎么拿到第一阶段的文件？**
→ 只能通过 `COPY --from=<stage名或序号>`。两个阶段是**独立的文件系统**，除此之外没有共享通道。
→ 再追"能 COPY 目录吗"：能，`COPY --from=builder /out/ /usr/local/bin/`。

**Q4：`CGO_ENABLED=0` 是干什么的？**
→ 关掉 cgo，做**静态编译**。因为 alpine 用 musl libc 而不是 glibc，动态链接的 Go 二进制在 alpine 上会报 `not found`——这个报错说的是**动态链接器不存在**，不是文件不存在。
→ 再追"那为什么不用 glibc 的镜像"：体积大十几倍，失去 alpine 的意义。

**Q5：默认 bridge 和自定义 bridge 有什么区别？**
→ 最大区别是**内嵌 DNS**：自定义网络里 Docker 起一个 DNS（容器内看到的是 `127.0.0.11`），支持容器名/别名解析；默认 bridge 没有，只能靠 IP。
→ 再追"IP 会变吗"：会。容器重建 IP 可能变，所以**用名字不用 IP** 才是正解。

**Q6：容器数据怎么持久化？三种方式区别？**
→ volume（Docker 管理，`/var/lib/docker/volumes`，容器删了还在）、bind mount（挂宿主机路径，耦合宿主目录结构）、tmpfs（内存，容器停就没）。
→ 再追"数据库用哪个"：volume。可统一备份、不受宿主机目录结构影响、权限问题少。

**Q7：镜像 tag 和 digest 什么关系？**
→ **tag 是可变别名，digest 是内容指纹**。同一个 tag 可以被重新 push 指向不同 digest（`latest` 就是典型）。生产上要精确锁定就用 digest。
→ 再追"那为什么 CI 里用 sha tag 而不用 digest"：sha tag 可读、可追溯、且我的流程保证它**不可变**（tag 已存在就不覆盖），效果等价但人可读。

**Q8：容器日志把磁盘写满了怎么办？**
→ 先定位：`docker system df -v` + `du -sh /var/lib/docker/containers/*`。根因是 `json-file` driver **默认不轮转**。
→ 修法：`/etc/docker/daemon.json` 配 `log-opts: {"max-size":"100m","max-file":"3"}`，重启 docker 生效（已存在的容器要重建才生效）。

**Q9：`docker stop` 和 `docker kill` 区别？**
→ `stop` 先发 **SIGTERM**（默认等 10 秒），没退出再 SIGKILL；`kill` 直接 SIGKILL。所以 `stop` 的退出码常是 143，`kill` 是 137。
→ 再追"为什么应用要处理 SIGTERM"：优雅退出——停止接新请求、处理完在途请求、关闭连接。不处理就会丢请求。

**Q10：为什么容器必须有一个前台进程？**
→ 容器的生命周期 = **PID 1 的生命周期**。PID 1 退出容器就结束。所以 `nginx` 要 `daemon off`、`systemd` 类镜像要用 `--init` 之类的方式。
→ 再追"`docker run` 加 `-d` 不是后台吗"：那是**docker 客户端**后台，容器里 PID 1 仍是前台运行的。

**Q11：怎么减小镜像体积？说出至少 5 种手段。**
→ ① 多阶段构建 ② 换更小的基础镜像（alpine/scratch/distroless）③ 合并 RUN 减少层数 ④ `.dockerignore` 排除无用文件 ⑤ 清理包管理器缓存（`rm -rf /var/lib/apt/lists/*`，且要在**同一层**）⑥ 静态编译去掉运行时依赖。

**Q12：你的 884MB 里到底装了什么？**
→ Debian 基础系统 + Go 工具链 + gcc（cgo 需要）+ 源码 + 构建缓存 + 最终二进制。**运行时真正需要的只有那个二进制**。
→ 再追"那为什么不直接 `FROM scratch`"：没有 shell 无法调试、没有 CA 证书无法访问 HTTPS、没有时区数据。alpine 是折中。

---

## 五、闭卷自测（8 题）

> 合上文档，口头答。答不出的标红，第二天只复习红题。

| # | 题 | 及格线 |
|---|---|---|
| 1 | 容器和虚拟机的根本区别是什么？ | 说到"共享内核" vs "独立内核" |
| 2 | 镜像分层带来什么好处、什么代价？ | 好处说复用+增量；代价说 `rm` 不减体积 |
| 3 | 多阶段构建为什么能变小？ | 说到"生产过程不进最终镜像"+`COPY --from` |
| 4 | 默认 bridge 为什么不能用容器名互访？ | 说到"没有内嵌 DNS" |
| 5 | 退出码 137 说明什么？怎么确认是不是 OOM？ | 137=SIGKILL；查 `OOMKilled` 字段 |
| 6 | 容器内 `curl` 通、宿主访问不到，最可能是什么原因？ | 服务监听 `127.0.0.1` 而非 `0.0.0.0` |
| 7 | `CGO_ENABLED=0` 为什么在 alpine 上是必须的？ | 说到 musl vs glibc、动态链接器缺失 |
| 8 | 884MB → 14.1MB，具体砍掉了什么？ | 能列出：Debian、Go 工具链、gcc、源码、构建缓存 |

**已知薄弱点（W1 闭卷自测 3.5/8 未及格）**：第 2、5、7 题当时答得最差。这三题要重点过。

---

## 六、边界

| 这块我没碰过 | 说明 |
|---|---|
| Swarm / 跨主机编排 | 只用过单机 compose 和 K8s |
| 镜像签名（cosign/Notary） | 没做过供应链安全 |
| BuildKit 高级特性（cache mount、多平台构建） | 用的是默认 builder |
| 容器逃逸攻防 | 只理解原理，没有实操 |
