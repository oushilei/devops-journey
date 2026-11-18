# 02 · CI/CD（Jenkins + GitLab CI）

> 核心一句话：**CI 的价值不在"自动跑命令"，在"用门禁把不合格的东西挡在制品库之外"。**
> 门禁失效一次，整套流水线就退化成"一键部署脚本"。

---

## 0. 一分钟版本

> 我搭过两条流水线。一条是 Jenkins，用 **DooD**（挂载宿主机 `docker.sock`，容器内调宿主机 daemon），第一条流水线全绿 34 秒。另一条是自建 GitLab CE + Shell Runner，跑 build → test → package → deploy 四阶段，并做了四道门禁：单测不过即中断、制品 tag 不可变、发布需人工审批、支持一键回滚。并发从 1 调到 2 之后，同 stage 的两个 job 从排队变成并行，耗时 57 秒降到 28 秒。

---

## 一、原理层

### 1.1 CI 系统的三层模型

```
pipeline（一次运行）
  └── stage（串行，前一个全绿才进下一个）
        └── job（同 stage 内并行）
```

**为什么 stage 串行、job 并行**：
- stage 之间是**依赖关系**（没构建出制品，测什么？）
- stage 内是**独立任务**（单元测试和 lint 互不依赖，没理由排队）

**这条模型决定了性能优化的方向**：想让流水线变快，要么减少 stage 串行深度，要么给同一 stage 加槽位。

### 1.2 Runner 是什么（经常被混淆）

| 角色 | 是什么 | 在哪 |
|---|---|---|
| **GitLab 服务端** | 存代码、存 pipeline 定义、调度 job | `192.168.245.150:80` |
| **Runner** | 一个**常驻 agent**，主动去服务端"领活"然后执行 | node1，systemd 服务 |

**关键：Runner 是主动拉取（pull），不是服务端推送。** 所以 Runner 可以躲在 NAT 后面，只要能出网访问 GitLab 就行。

**executor 类型**（这是高频追问点）：

| executor | 怎么跑 job | 优点 | 代价 |
|---|---|---|---|
| **shell** | 直接在 Runner 所在机器上跑脚本 | 最快、最简单、能用宿主所有工具 | **无隔离**——job 能改宿主任何东西；并发要自己管工作目录 |
| **docker** | 每个 job 起一个容器 | 环境干净、可指定镜像 | 需要拉镜像，慢一些 |
| **kubernetes** | 每个 job 起一个 Pod | 弹性、隔离好 | 要 K8s 集群 + RBAC |

**我用的是 shell executor**，原因：node1 上已经有 docker daemon 和 kubectl，shell 模式下 job 直接调用它们，不用在容器里再做嵌套配置。

**代价（必须主动说）**：shell executor 的 job **以 Runner 用户的身份跑在宿主机上**，能读到 `/root/.kube/config`、能 `docker` 任意操作。这是实验环境的便利，**生产上一般不会这么用**。

### 1.3 并发槽位：为什么 57 秒变成 28 秒

**两个参数决定同时能跑几个 job**：

| 参数 | 含义 | 我的值 |
|---|---|---|
| `concurrent` | Runner **全局**最多同时跑几个 job | 1 → **2** |
| `limit` | **单个 Runner** 最多同时跑几个 | 未单独设 |

**实际并发数 = min(待跑 job 数, Σconcurrent, Σlimit)**

**实测证据**：

| 场景 | 结果 |
|---|---|
| `concurrent=1`，同 stage 两个 job | 串行，总耗时 **57 秒** |
| `concurrent=2` | 并行，总耗时 **28 秒** |
| 3 个 job 抢 2 槽位 | 前两个跑，第三个 `Queued 28s / Duration 0s` |

> **第三行是关键证据**：`Queued 28s / Duration 0s` 说明它**一直在等槽位，等到时前面已经跑完了**。这比"看总时间变短"更能证明槽位机制的存在。

### 1.4 artifacts vs cache（最容易被问混的一组）

| | artifacts | cache |
|---|---|---|
| 存在哪 | **GitLab 服务端**（对象存储/本地） | **Runner 本地**（或配 S3） |
| 作用 | **在 stage 之间传产物**，保证正确性 | **只加速**，不保证有 |
| 能不能下载 | 能，Web 界面可下 | 不能直接下 |
| 过期 | 可配 `expire_in` | 可配，但可能被清理 |
| 命中率 | 100%（同一 pipeline 内必有） | 可能 MISS |
| 典型用途 | `dist/` 构建产物、测试报告 | 依赖包目录（`node_modules`、`.m2`） |

**一句话区分**：**artifacts 是"必须送到下一站的东西"，cache 是"上次留下的、这次可能用得上"。**

**我的实测**：

| 现象 | 日志证据 |
|---|---|
| artifacts 传递 | `Downloading artifacts for build-job` —— 明确经 GitLab 中转 |
| cache 首次 | `CACHE MISS`，5 秒 |
| cache 二次 | `CACHE HIT`（显示上次写入时间戳），1 秒 |
| cache 存储位置 | `stored only locally` —— 存在 Runner 本机，不是服务端 |

**一个坑**：受保护分支的 cache key 会自动加 `-protected` 后缀。所以同一个分支从"非保护"改成"保护"后，cache **必然 MISS** 一次——这不是 bug，是 GitLab 的隔离机制。

### 1.5 tag 与 digest（回滚的地基）

| 概念 | 本质 | 可变性 |
|---|---|---|
| **tag** | 指向 digest 的**别名** | **可变**——可以被重新 push 覆盖 |
| **digest** | 内容 sha256 | **不可变**——内容变则 digest 必变 |

**为什么 CI 里用 commit sha 做 tag**：

```
sha tag = 代码版本的直接映射
  → 看到 7d75d679 就知道对应哪个 commit
  → 回滚时能精确指定"退到哪个代码版本"
```

**但 sha tag 本身也是"可变的"**（技术上可以被覆盖）。所以我在 `package` 阶段加了门禁：**tag 已存在就跳过推送**，用流程保证它实际不可变。

**实测**：双 tag（sha + latest）指向**同一 digest**；第二次 push 时所有层 `Layer already exists`——因为内容完全相同。

### 1.6 四道门禁（这是整个 CI 模块的核心）

| # | 门禁 | 实现方式 | 挡什么 |
|---|---|---|---|
| 1 | **质量门禁** | `test` 阶段跑 `scripts/test.sh`，非 0 退出即中断 | 单测不过的代码不进制品库 |
| 2 | **制品不可变** | `check-tag.sh` 查 tag 是否存在，存在则跳过 push | 同一个版本号被不同内容覆盖 |
| 3 | **发布审批** | `deploy` job 配 `when: manual` | 未经人确认就上线 |
| 4 | **可回滚** | `rollback` job + `TARGET_TAG` 变量 | 出事无法快速退回 |

**门禁 2 的深层意义**（经常被追问"为什么要这样"）：

> 如果 tag 可以被覆盖，那"回滚到 `v1.2.3`"就是一句空话——你不知道那个 tag 现在指向的是哪个内容。
> **制品不可变是回滚能成立的前提**，不是可有可无的规范。

### 1.7 `rules` / `when` / 变量优先级

```yaml
deploy-job:
  stage: deploy
  when: manual            # 必须人点
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - when: never         # 其他分支不出现这个 job
```

**变量优先级（从高到低）**：pipeline 变量 > job 变量 > 全局 `variables` > `.gitlab-ci.yml` 默认值。

**`when: manual` 的两个形态**（差别很大）：

| 写法 | 效果 |
|---|---|
| `when: manual`（默认 `allow_failure: false`） | job 不点就**卡住整个 pipeline**，不算完成 |
| `when: manual` + `allow_failure: true` | 可选 job，不点也不影响 pipeline 变绿 |

我用的是前者——**发布必须明确点，不能"忘了点就当成功"**。

### 1.8 两个和 git 有关的坑（都在真实环境踩过）

**坑 A：浅克隆 + 老 git = 必挂**

| 项 | 内容 |
|---|---|
| 现象 | 第二次构建起必报 `expected shallow list` |
| 原因 | Runner 默认 `GIT_DEPTH=20`（浅克隆）；node1 上 git 是 **1.8.3.1**，在**已有对象的仓库**上做增量浅 fetch 会失败 |
| 修法 | `GIT_DEPTH: 0` |
| **注意** | `0` **不是"深度 0"**，是**禁用浅克隆的开关值**。这是个容易误解的细节 |

**坑 B：`set -e` 让脚本静默中止**

| 项 | 内容 |
|---|---|
| 现象 | 冒烟失败时，连状态码都没打印，job 日志一片空白 |
| 原因 | shell executor 默认 `set -e`；`CODE=$(curl ...)` 里 curl 超时返回非 0 → **整个脚本立即中止** |
| 修法 | `CODE=$(curl ... \|\| true)` |
| 教训 | 在 `set -e` 环境下，**任何"允许失败"的命令都必须显式兜底** |

### 1.9 Jenkins 的 DooD vs DinD（W2 的知识，必问）

| | DooD（Docker-outside-of-Docker） | DinD（Docker-in-Docker） |
|---|---|---|
| 做法 | 把宿主机 `/var/run/docker.sock` 挂进容器 | 容器内跑一个**独立的 dockerd** |
| 构建的镜像在哪 | 在**宿主机** daemon 里 | 在容器内的 daemon 里（容器销毁就没了） |
| 速度 | 快（复用宿主层缓存） | 慢（每次冷启动、无缓存） |
| 隔离/安全 | **弱**——等于给了容器 root 权限操作宿主 | 强，需要 `--privileged` |
| 我的选择 | **DooD** | — |

> **为什么选 DooD**：实验环境要的是"构建出来的镜像能直接被宿主机和 K8s 用"，DooD 天然满足。代价是安全性弱——**这一点必须主动说**，否则会被认为不懂风险。

---

## 二、项目层

### 2.1 两条流水线

| | Jenkins（W2） | GitLab CI（W3~W6） |
|---|---|---|
| 触发 | 参数化 + poll SCM | push 自动触发 |
| 定义 | `Jenkinsfile`（Groovy） | `.gitlab-ci.yml`（YAML） |
| 构建环境 | DooD，容器内调宿主 docker | Shell executor，直接用宿主工具 |
| 首次全绿耗时 | **34 秒** | — |
| 门禁 | 无（学习阶段只跑通） | **四道** |
| 现状 | 保留在 node1，占 8080 端口 | **主用**，W6 端到端链路走它 |

**为什么后来换成 GitLab CI**（这题常被问"为什么不用 Jenkins"）：

| 维度 | Jenkins | GitLab CI |
|---|---|---|
| 与代码托管集成 | 要装插件、配 webhook | **同一个系统，push 即触发** |
| 配置位置 | Jenkinsfile 在仓库，但任务配置在 Jenkins 里（**两处**） | 全在 `.gitlab-ci.yml`（**一处**，可 review） |
| 维护成本 | 插件版本地狱 | 一体化，升级简单 |
| 生态成熟度 | 老牌，插件极多 | 够用，但复杂场景不如 Jenkins |

**结论**：小团队、代码已在 GitLab 上 → GitLab CI 更省事；需要复杂编排/大量遗留插件 → Jenkins 更灵活。

### 2.2 四道门禁的实际实现

```yaml
# 门禁 1：质量
test-job:
  stage: test
  script:
    - bash scripts/test.sh        # 非 0 退出即中断

# 门禁 2：制品不可变
package-job:
  stage: package
  script:
    - bash scripts/check-tag.sh $CI_COMMIT_SHORT_SHA $REGISTRY
    - 'if [ -f .skip_push ]; then echo "跳过构建与推送"; else docker build ... && docker push ...; fi'

# 门禁 3：发布审批
deploy-job:
  stage: deploy
  when: manual

# 门禁 4：回滚
rollback-job:
  stage: deploy
  when: manual
  variables:
    TARGET_TAG: ""                # 空 = rollout undo；填了 = 按 tag 回退
```

### 2.3 踩过的坑

| # | 坑 | 根因 | 修法 |
|---|---|---|---|
| 1 | 第二次构建必挂 `expected shallow list` | `GIT_DEPTH=20` + git 1.8.3.1 | `GIT_DEPTH: 0` |
| 2 | 冒烟失败时日志空白 | `set -e` + curl 非 0 | `curl ... \|\| true` |
| 3 | 受保护分支 cache 必 MISS | cache key 自动加 `-protected` | 知道即可，不是 bug |
| 4 | `rollout status` 成功但冒烟 000 | endpoints/iptables 收敛有秒级延迟 | 冒烟重试 6 次 |
| 5 | 并发调到 2 后总时长没降 | 两个 job 在同一 stage 才会并行 | 确认 job 的 stage 划分 |

**第 4 条值得展开**（这是"假失败"的经典案例）：

```
Pod Ready ≠ 流量立刻可达
  → kube-proxy 要更新 iptables/ipvs 规则
  → 有 1~3 秒窗口期，Service 还没有后端
  → 单次冒烟会偶发 000
  → 团队会养成"重跑一下就好"的习惯 → 门禁彻底失效
```

**所以冒烟必须重试**。这不是"降低标准"，是**消除测量误差**。

---

## 三、排障层（现场题）

### 3.1 「job 一直 pending，不开始跑」

```bash
# 1) Runner 在线吗
#    GitLab → Settings → CI/CD → Runners，看绿点
systemctl status gitlab-runner

# 2) Runner 的 tag 和 job 的 tag 匹配吗
cat /etc/gitlab-runner/config.toml | grep -A3 tags

# 3) 槽位满了吗
#    Runner 日志里会写 "no free build slots"
journalctl -u gitlab-runner -n 50 | grep -i "slot\|concurrent"

# 4) 分支被保护、Runner 没权限吗
#    Settings → CI/CD → Runners → 看 "Protected" 勾选
```

**判定表**：

| 现象 | 原因 |
|---|---|
| 灰点 / `no runners available` | Runner 没注册、离线、或 tag 不匹配 |
| `This job is stuck` + 有绿点 | 槽位满（`concurrent` 太小）或 Runner 被限制到其他项目 |
| 只有特定分支卡住 | 该 job 需要 protected runner，但分支没被保护 |

### 3.2 「权限被拒 / 403」

| 场景 | 原因 | 处置 |
|---|---|---|
| 推 registry 报 `unauthorized` | 没 `docker login` 或凭据过期 | CI 里用 `CI_REGISTRY_USER`/`CI_REGISTRY_PASSWORD`，或挂 `~/.docker/config.json` |
| `kubectl` 报 `Forbidden` | ServiceAccount 权限不足 | `kubectl auth can-i --list` 确认权限范围 |
| 拉私有仓库代码 403 | 用了 `CI_JOB_TOKEN` 但项目未授权 | 在目标项目里把源项目加入 job token 白名单 |
| `permission denied` 执行脚本 | 文件没 x 权限 | `chmod +x`，或显式 `bash scripts/xxx.sh` |

> **`Forbidden` 和 `Unauthorized` 要分清**：401 是"你是谁不知道"，403 是"知道你是谁但你不能干这个"。前者查凭据，后者查 RBAC。

### 3.3 「artifacts 找不到 / 404」

```bash
# 确认上游 job 真的产出了
# job 日志里搜 "Uploading artifacts"
# 检查 .gitlab-ci.yml 的 paths 是否包含该文件
```

| 原因 | 判断 |
|---|---|
| 上游 job 失败 | 失败 job 默认**不传 artifacts**（除非 `when: always`） |
| 路径写错 | `paths` 是相对 `CI_PROJECT_DIR` 的 |
| 已过期 | `expire_in` 到期被清理 |
| 下游 job 没声明依赖 | 默认会拉**前面所有 stage** 的 artifacts；用 `dependencies:` 限定 |

### 3.4 「cache 不命中」

| 原因 | 说明 |
|---|---|
| `key` 里带了 `$CI_COMMIT_SHA` | 每个 commit 都是新 key，永远 MISS |
| 受保护分支 | key 自动加 `-protected` |
| Runner 不是同一台 | cache 默认存 Runner 本地，换 Runner 就没有 |
| `cache:key:files` 的文件变了 | 依赖清单变了，本来就该重建缓存 |

### 3.5 「构建成功但部署失败」

按这个顺序查（**从制品往外查**，不要一上来就看 K8s）：

```bash
# 1) 制品真的在 registry 里吗
curl -s http://192.168.245.150:5000/v2/devops/hello/tags/list

# 2) 部署时用的 tag 和制品对得上吗
#    看 deploy job 日志里的 kubectl set image 那一行

# 3) K8s 侧看到的是什么
kubectl get deploy hello -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get pods -l app=hello
kubectl describe pod <pod> | tail -30

# 4) 拉镜像失败？（ImagePullBackOff）
kubectl describe pod <pod> | grep -A5 "Failed to pull"
```

**判定**：

| Pod 状态 | 含义 |
|---|---|
| `ImagePullBackOff` | 镜像拉不到（tag 不存在 / registry 认证 / 网络） |
| `CrashLoopBackOff` | 容器起来了但反复退出（应用问题） |
| `Running 0/1` | 进程活着但**探针没过**（见 03 模块） |
| `Pending` | 调度不上（资源不足 / 污点 / 亲和性） |

---

## 四、追问清单

**Q1：CI 和 CD 的区别？**
→ CI 是**持续集成**——代码合并后自动构建+测试，保证主干始终可用。CD 是**持续交付/部署**：交付 = 制品随时可发但发布由人决定；部署 = 自动发到生产。
→ 再追"你的是哪种"：我是**持续交付**——`deploy-job` 是 `when: manual`，人点才发。

**Q2：为什么 stage 之间串行、stage 内并行？**
→ stage 之间是**依赖关系**（没构建出制品就没法测），stage 内是**独立任务**。
→ 再追"怎么让流水线更快"：① 缩短串行链（合并无依赖的 stage）② 加槽位（`concurrent`）③ 用 cache 省重复下载 ④ 只跑受影响的 job（`rules:changes`）。

**Q3：artifacts 和 cache 有什么区别？**
→ artifacts 经 GitLab 中转、保证存在、可下载、用于 stage 间传产物；cache 存 Runner 本地、只加速、可能 MISS。
→ 再追"cache 能当 artifacts 用吗"：不能。cache 不保证存在，把它当传递通道会出现"本地能跑、换 Runner 就挂"。

**Q4：为什么 tag 不可变这么重要？**
→ 因为回滚要靠 tag 定位制品。tag 可被覆盖的话，"退到 v1.2.3"就是不确定的。
→ 再追"怎么保证不可变"：① 流程门禁（已存在就跳过）② registry 的 immutable tag 策略 ③ 直接用 digest 部署。

**Q5：Runner 的 `concurrent` 和 `limit` 有什么区别？**
→ `concurrent` 是**全局**上限，`limit` 是**单个 Runner** 上限。实际并发 = min(待跑 job 数, Σconcurrent, Σlimit)。
→ 再追"调大就更快吗"：不一定。受 CPU/内存/IO 限制，调太大反而互相拖慢。我的机器上 2 是合理的。

**Q6：shell executor 有什么风险？**
→ job **以 Runner 用户身份直接跑在宿主机上**，没有隔离。恶意或写错的 job 能删宿主文件、读所有凭据。
→ 再追"生产怎么办"：用 docker/k8s executor，每个 job 一个干净环境；Runner 用专用低权限用户；敏感凭据用 masked variable + protected branch。

**Q7：DooD 和 DinD 的区别？**
→ DooD 挂宿主 `docker.sock`，构建的镜像落在宿主 daemon；DinD 在容器内跑独立 dockerd，需要 `--privileged`。
→ 再追"安全吗"：DooD **等于给容器 root 权限操作宿主**，安全性弱。实验环境可接受，生产要慎用。

**Q8：`GIT_DEPTH: 0` 是什么意思？**
→ **禁用浅克隆**（拉全量历史），不是"深度为 0"。
→ 再追"为什么要设"：Runner 默认 `GIT_DEPTH=20` 浅克隆，node1 上 git 1.8.3.1 在已有对象的仓库上做增量浅 fetch 会报 `expected shallow list`。代价是每次拉全量历史，仓库大时变慢。

**Q9：`when: manual` 的 job 不点会怎样？**
→ 默认 `allow_failure: false`，**pipeline 会一直停在"blocked"状态**，不会变绿。
→ 再追"那怎么让可选 job 不影响"：加 `allow_failure: true`。

**Q10：怎么保证部署的镜像就是测试过的那个？**
→ 关键是**同一个制品贯穿全流程**：test 阶段测的是代码，package 阶段用**同一个 commit** 构建镜像，deploy 用**同一个 sha tag**。中间不重新构建。
→ 再追"如果重新构建会怎样"：即使代码相同，构建环境差异也可能产出不同二进制（Go 的 build id、时间戳）。**"一次构建，多处部署"是基本原则**。

**Q11：流水线红了但服务正常，是 bug 吗？**
→ 不是。它反映的是**"这次发布没成功"**，不是"服务挂了"。探针拦住坏版本、旧 Pod 继续服务，所以服务正常但发布失败。
→ 再追"那要不要让流水线变绿"：不要。绿了就意味着"发布成功"，会掩盖问题。**红是正确信号，要修的是发布，不是信号。**

**Q12：你的门禁有没有被绕过的情况？**
→ 有。`when: manual` 的门禁依赖人**不点**，如果人点了坏版本照样发。真正的防护是**探针**（03 模块），它在 K8s 侧拦，不依赖人的判断。
→ 再追"那门禁意义何在"：门禁挡"不该进制品库的东西"，探针挡"不该接流量的东西"。**两层各管一段，不能互相替代。**

---

## 五、闭卷自测（8 题）

| # | 题 | 及格线 |
|---|---|---|
| 1 | artifacts 和 cache 的三条区别 | 存储位置、保证性、用途 |
| 2 | 实际并发数的公式 | min(job 数, Σconcurrent, Σlimit) |
| 3 | 四道门禁分别挡什么 | 质量、覆盖、未授权发布、无法回退 |
| 4 | 为什么 sha tag 是回滚的地基 | 说到"tag 可变则回滚目标不确定" |
| 5 | DooD 和 DinD 的区别与风险 | 挂 sock vs 独立 daemon；DooD 安全性弱 |
| 6 | `expected shallow list` 的根因和修法 | 浅克隆 + 老 git；`GIT_DEPTH: 0` |
| 7 | 流水线红了但服务正常，怎么解释 | 发布失败 ≠ 服务故障 |
| 8 | 冒烟为什么要重试 | Ready ≠ 可达，iptables 收敛有延迟 |

---

## 六、边界

| 没碰过 | 说明 |
|---|---|
| ArgoCD / Flux（GitOps） | 只做过 push 式部署，没做过声明式同步 |
| 多环境（dev/staging/prod）发布策略 | 只有一个 lab 环境 |
| 蓝绿 / 金丝雀发布 | 只做过 K8s 默认滚动 |
| Jenkins 共享库、Pipeline 进阶 | 只写过单文件 Jenkinsfile |
| 流水线安全（secrets 扫描、SBOM） | 没做过供应链安全 |
