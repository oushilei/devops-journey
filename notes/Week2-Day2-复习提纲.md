# Week2 Day2 复习提纲 · Docker-outside-of-Docker 与第一条流水线

> 用法：先做最后的「自测 8 题」，**自己答完再看答案**。读一遍答案 ≠ 会了，
> 能自己讲出来才算会（这是 Week1 暴露的"提取缺口"教训）。
>
> 配套文档：`Week2-Day2-第一条流水线-原理讲解.md`（详细原理版）

---

## 一、一句话总览

昨天做成的是一件事：**让跑在容器里的 Jenkins，能够指挥宿主机的 Docker 去构建镜像，并把这件事写成一条可重复执行的流水线。**

它由三层构成：

| 层 | 做了什么 | 解决什么问题 |
|---|---|---|
| 第 1 层：打通能力 | DooD（挂 socket + 注入 CLI + 加组） | 容器里的 Jenkins 能操作宿主机的 Docker |
| 第 2 层：确认零件 | 核实插件（94 个） | 流水线需要的功能组件都在 |
| 第 3 层：跑通流程 | Jenkinsfile 三阶段 | 把"人手工敲命令"变成"机器自动执行" |

---

## 二、第 1 层：Docker-outside-of-Docker（DooD）

### 为什么需要它

Jenkins 跑在**容器**里，而容器是个**隔离的沙箱**：它有自己的文件系统、进程、网络，**看不到宿主机的 Docker**。
可我们偏偏要在 Jenkins 里执行 `docker build`。矛盾就在这里。

### 为什么"真的装一个 Docker 进去"不行

你可能会想：那我在容器里装个 Docker 不就行了？
- 那叫 **Docker-in-Docker（DinD）**，需要在容器里再跑一个独立的守护进程；
- 需要 `--privileged` 特权模式，**隔离性反而更差**；
- 镜像存在容器内部，**宿主机看不见**，K8s 用不上，还得来回搬运。

### DooD 的三步（缺一不可）

| 步骤 | 命令 | 作用 | 为什么 |
|---|---|---|---|
| ① 挂载套接字 | `-v /var/run/docker.sock:/var/run/docker.sock` | 建立**通信通道** | 容器里的客户端要通过这个 socket 才能跟宿主机的守护进程通话 |
| ② 注入客户端 | `docker cp $(which docker) jenkins:/usr/local/bin/docker` | 提供**遥控器** | 容器是干净 Debian 镜像，**里面根本没有 `docker` 这个命令** |
| ③ 加入用户组 | `--group-add 995` | 给**权限** | socket 权限是 `root:docker 660`，容器里 jenkins 用户不属 docker 组 → 读写会被拒 |

**关键理解：`docker` 命令本身不干活。**
它是 **C/S 架构**的客户端 —— 敲 `docker build` 时，它只是把参数打包成 HTTP 请求，通过 socket 发给宿主机上的 **`dockerd`（守护进程）**，由 dockerd 真正执行构建。

> 验证：`docker exec jenkins docker ps` 列出的全是**宿主机的容器**（jenkins/nginx/k8s_*），
> 而**不是**"容器里的容器"—— 这就是 DooD 生效的直观证据。

### 只做一半会怎样（面试常被追问）

| 只做 | 报错 | 病因 |
|---|---|---|
| 只挂 socket，不拷 CLI | `docker: command not found` | 有通道，没遥控器 |
| 只拷 CLI，不挂 socket | `Cannot connect to the Docker daemon` | 有遥控器，没通道 |
| 两个都做，不加组 | `permission denied ... docker.sock` | 有通道有遥控器，但没钥匙 |

### 为什么用"拷二进制"而不是"apt 装"

- `docker` CLI 是**静态编译的 Go 二进制（39.7MB）**，不依赖特定发行版的库；
- 所以从 CentOS 宿主机拷进 Debian 容器**能直接运行**（昨天已实测验证）；
- 对比 apt 装：要先配 Docker 的 apt 源、下载一堆依赖、国内源还慢。

> **已实测结论**：昨天 `docker cp` 完直接可用，39.7MB 拷进去即可执行。

### 风险（面试加分点）

把 `docker.sock` 交给一个容器 = **给了这个容器宿主机 root 权限**。
拿到 socket 就能起特权容器、挂载宿主机任意目录、读取宿主机文件系统。

**因此生产上的三种做法**：
1. 严格控制 Jenkins 权限，只让它接触必要的资源；
2. 用 **Kaniko / Buildah**：不需要守护进程，直接构建镜像 → **现代 CI 的主流做法**；
3. DinD + 网络隔离（较少用，需要特权）。

---

## 三、第 2 层：插件核实（一个真实的排错案例）

### 昨天的疑问与答案

`ls /var/jenkins_home/plugins | wc -l` 显示 **188**，但我推断**真实插件数是 94**。
验证命令：

```bash
docker exec jenkins bash -c 'ls /var/jenkins_home/plugins | sed "s/.*\.//" | sort | uniq -c'
docker exec jenkins bash -c 'ls /var/jenkins_home/plugins/*.jpi | wc -l'
```

输出 `94 jpi`，且 `.jpi` 文件数 = **94** → **推断了成立**。

### 为什么是 188 而不是 94

Jenkins 会**同时保留两份**：
- `plugins/<短名>.jpi` —— 压缩包文件本身
- `plugins/<短名>/` —— 解压后的目录

**每个插件占 2 个条目** → 94 × 2 = 188。

> **方法论**：看到异常数字先别下结论，用命令验证。这就是"推断 vs 事实"的差别。

### 昨天更正的一个错误

10/7 我曾经说 `ssh-agents` 插件"缺失"。**这是我的检查名写错了**：
"SSH Build Agents" 的**真实插件短名是 `ssh-slaves`** —— 它一直都装着。
那次"缺失"是**假阴性**（我的命令错，不是环境缺）。

### 为什么插件短名容易搞错

插件在 **UI 上显示的名字**（如 "SSH Build Agents"）和**磁盘上的短名**（如 `ssh-slaves`）**往往不一致**。
查插件要用短名，或者直接在 **插件管理页** 里搜显示名。

### Week2 需要的插件（全部已就位 ✅）

| 用途 | 插件短名 |
|---|---|
| Pipeline 核心 | `workflow-aggregator` `workflow-cps` `workflow-job` `workflow-basic-steps` |
| 声明式语法 | `pipeline-model-definition` |
| 拉代码 | `git` `git-client` |
| 凭据（Day4 用） | `credentials` `credentials-binding` `plain-credentials` `ssh-credentials` |
| GitHub 集成 | `github` `github-api` `github-branch-source` |
| SSH 构建节点 | `ssh-slaves` |
| 中文界面 | `localization-zh-cn` |
| 辅助 | `ws-cleanup` `timestamper` |

---

## 四、第 3 层：第一条流水线

### 手工 vs 流水线

Week1 你手工敲的：

```bash
git clone https://github.com/oushilei/devops-demo && cd devops-demo
docker build -t devops-demo:v1 .
docker images | grep devops-demo
```

**手工的三个死穴**：不可复现、无记录、不可审查。
流水线就是把这三条命令**写进代码**，让机器重复执行。

### Jenkinsfile（昨天用的版本）

```groovy
pipeline {
    agent any

    stages {
        stage('拉代码') {
            steps {
                echo '=== 从 GitHub 拉取代码 ==='
                git branch: 'main', url: 'https://github.com/oushilei/devops-demo.git'
            }
        }

        stage('构建镜像') {
            steps {
                echo '=== 构建 Docker 镜像 ==='
                sh 'docker build -t devops-demo:v1 .'
            }
        }

        stage('验证') {
            steps {
                echo '=== 确认镜像已生成 ==='
                sh 'docker images | grep devops-demo'
            }
        }
    }

    post {
        success {
            echo '流水线全部成功'
        }
    }
}
```

### 阶段 ↔ 手工命令对照

| 阶段 | Jenkinsfile 写法 | 等价手工命令 |
|---|---|---|
| 拉代码 | `git branch:'main', url:'…'` | `git clone …` |
| 构建镜像 | `sh 'docker build -t devops-demo:v1 .'` | `docker build -t devops-demo:v1 .` |
| 验证 | `sh 'docker images \| grep devops-demo'` | `docker images \| grep devops-demo` |

### 三个决策（面试核心）

**决策 1：Pipeline 任务，不是 FreeStyle 任务**
- FreeStyle：网页上点选配置，配置**只存在 Jenkins 数据库里** → 不可版本化、不可评审。
- Pipeline：流程写成**代码** → 可进 Git、可评审、可迁移。
- 招聘 JD 写的是 Pipeline / Jenkinsfile，FreeStyle 已属过时。

**决策 2：Jenkinsfile 文件，不是网页点选** —— 这就是 **Pipeline as Code**
- 流程的每次改动有历史、可追溯、可回滚；
- 换一台 Jenkins，Jenkinsfile 还在仓库里，拷过去就能用；
- 流程变更走 PR，团队能评审。

**决策 3：声明式（`pipeline {}`），不是脚本式**
- 声明式：结构固定（agent / stages / post），可读性高，Jenkins 能做语法校验和阶段视图；
- 脚本式：`node { }` 纯 Groovy，灵活但可读性差；
- 需要复杂逻辑时，用 `script { }` 块嵌 Groovy → 兼顾规范与灵活。

### 关键参数

| 写法 | 含义 |
|---|---|
| `agent any` | 在任意可用节点执行。你只有**内置节点（= Jenkins 容器本身）** → **这就是为什么容器里必须有 docker CLI** |
| `git branch, url` | 用 `git` 插件步骤而非 `sh 'git clone'`：**它会记录 revision**，UI 里能看到"这次构建对应哪个提交"（昨天页面显示的 `Revision: c8dedf5…` 就是它） |
| `sh '… .'` | 末尾的 `.` 是**构建上下文（build context）**：告诉 Docker 在当前目录找 `Dockerfile`，并把当前目录作为构建上下文发送给守护进程 |
| `post { success {} }` | 构建**结束后**执行的钩子，用于通知、告警、清理工作区 |

### 结果

- Job：`devops-demo-pipeline`，构建 `#1` **成功，耗时 34 秒**；
- 三个阶段全绿；
- 镜像 `devops-demo:v1` **出现在宿主机**（`docker images | grep devops-demo`）→ 闭环成立。

---

## 五、自测 8 题（先自答，再看答案）

**① DooD 是什么？为什么 Jenkins 容器里必须做这一步？**

<details><summary>答案</summary>

Docker-outside-of-Docker：让**容器内的客户端**通过 socket 操作**宿主机的 Docker 守护进程**。
因为 Jenkins 跑在容器里，容器是隔离沙箱，看不到宿主机的 Docker，而流水线里要执行 `docker build`。
</details>

**② 为什么既要挂 `docker.sock`，又要拷 `docker` 二进制？只做一半会怎样？**

<details><summary>答案</summary>

socket 是**通信通道**，CLI 是**遥控器**，两者缺一不可。
只挂 socket → `command not found`（有路没车）；只拷 CLI → `Cannot connect to the Docker daemon`（有车没路）。
</details>

**③ `docker` 命令本身干活吗？谁在真正构建镜像？**

<details><summary>答案</summary>

不干活。`docker` 是 **C/S 架构的客户端**，只把参数打包成 HTTP 请求发给守护进程。
真正构建的是宿主机上的 **`dockerd`**。
</details>

**④ `--group-add 995` 的 995 是什么？不加会报什么错？**

<details><summary>答案</summary>

宿主机 **docker 组的 GID**。因为 `docker.sock` 权限是 `root:docker 660`，容器内 jenkins 用户需属于该组才能读写。
不加 → `permission denied while trying to connect to the Docker daemon socket`。
（查法：`getent group docker`）
</details>

**⑤ `ls plugins | wc -l` = 188，所以装了 188 个插件。对还是错？**

<details><summary>答案</summary>

**错**。是 **94 个**。因为每个插件占 2 个条目：`.jpi` 压缩包文件 + 同名解压目录。
验证：`ls /var/jenkins_home/plugins/*.jpi | wc -l` → 94。
</details>

**⑥ 流水线三个阶段是什么？各对应哪条手工命令？**

<details><summary>答案</summary>

拉代码（`git clone`）→ 构建镜像（`docker build`）→ 验证（`docker images | grep`）。
</details>

**⑦ 为什么用 `git` 步骤而不是 `sh 'git clone ...'`？**

<details><summary>答案</summary>

`git` 插件步骤会**记录 revision（commit 哈希）和分支信息**，Jenkins UI 里能直接看到"这次构建对应哪个提交"，
还支持凭据管理、变更集展示。`sh 'git clone'` 则什么记录都没有。
</details>

**⑧ `docker build -t devops-demo:v1 .` 里的 `.` 是什么意思？**

<details><summary>答案</summary>

**构建上下文（build context）**：告诉 Docker 在当前目录找 `Dockerfile`，并把当前目录内容作为上下文发送给守护进程。
（如果写成别的路径，就是指定那个目录为上下文。）
</details>

---

## 六、面试速答（可直接背）

**Q：你们 CI 怎么做的？**
> Jenkins + Pipeline as Code。Jenkinsfile 跟代码放同一个仓库，流程变更走 PR 评审。典型三段：拉代码、构建镜像、验证。

**Q：为什么用 Pipeline 不用 FreeStyle？**
> FreeStyle 配置只存在 Jenkins 里，不可版本化、不可复现、不可评审。Pipeline 把流程变成代码，可进 Git、可迁移、可评审。

**Q：Jenkins 怎么构建 Docker 镜像？**
> Jenkins 本身跑在容器里，我们挂载宿主机的 docker.sock 并注入 docker CLI，实现 DooD。容器里的 docker 命令直接操作宿主机守护进程，镜像落在宿主机上，K8s 和镜像仓库都能直接用。

**Q：这套方案什么风险？**
> 把 docker.sock 给容器等于给了宿主机 root 权限 —— 能起特权容器、挂载宿主机任意目录。所以生产上要么严格限制 Jenkins 权限，要么用 Kaniko / Buildah 这类无守护进程的构建工具。

**Q：为什么用声明式而不是脚本式？**
> 声明式结构固定、可读性好、支持语法校验和阶段视图。复杂逻辑用 `script` 块嵌 Groovy，兼顾规范与灵活。

---

## 七、昨天踩过的坑（避免重犯）

| 坑 | 教训 |
|---|---|
| 我把 `ssh-agents` 当插件名去查，误报"缺失" | **UI 显示名 ≠ 插件短名**，查证要用真名或插件管理页 |
| `188` 个条目被误读成 188 个插件 | 异常数字先验证，不凭直觉下结论 |
| 我"先给操作、等你问了才解释" | **已改为：新知识点先讲 ①作用 ②为什么 ③面试怎么答** |
