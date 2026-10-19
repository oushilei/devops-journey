# Week2 Day2 · 第一条流水线：每一步的作用与为什么

> 本文目的：不是"照着敲能跑通"，而是**知道每一步为什么这么做**，面试被追问时能答得上。
> 配套：Week1-面试话术.md（Docker 部分）、技术转型计划-1003.md（整体路线）

---

## 一、这件事本身为什么存在

Week1 你是**手工**执行的：

```bash
git clone https://github.com/oushilei/devops-demo && cd devops-demo
docker build -t devops-demo:v1 .
docker images | grep devops-demo
```

手工方式的三个问题：

| 问题 | 具体表现 |
|---|---|
| **不可复现** | 换个人、换台机器、隔一周，环境不同结果可能不同 |
| **无记录** | 哪次构建对应哪个 commit？没人知道 |
| **不可审查** | 流程只存在于某个人的脑子里，他不在就没人会 |

流水线要解决的正是这三件事：**把"人的操作"变成"机器的、可重复的、有记录的流程"。**

所以今晚的终点不是"跑通一次"，而是"**以后代码一更新，同一套流程自动跑**"。
（Day5 会加 webhook 触发，那时这个价值才完全体现。）

---

## 二、动手前的三个决策（先懂再动手）

### 决策 1：为什么建「流水线(Pipeline)」任务，而不是「自由风格(FreeStyle)」任务

| 类型 | 怎么用 | 特点 |
|---|---|---|
| FreeStyle | 在网页上用勾选框、文本框配构建步骤 | 老做法；配置只存在 Jenkins 数据库里，不可版本化 |
| **Pipeline** | 整个流程写成代码（Jenkinsfile） | 流程即代码：可进 Git、可评审、可复现 |

**为什么选 Pipeline**
- 流程能进 Git，和代码一起版本管理；
- 多阶段、并行、条件判断等复杂流程能表达；
- 招聘 JD 里写的是 Pipeline / Jenkinsfile，FreeStyle 已属过时技能。

**面试怎么答**
> "我们统一用 Pipeline as Code。流程跟代码同仓库，改流程走代码评审，避免出现'只有某个人会点那个界面'的情况。"

### 决策 2：为什么把流程写成 Jenkinsfile 代码，而不是在网页上点选

这就是 **Pipeline as Code**（流程即代码），三个收益：

- **可版本化**：流程的每次改动都有历史，能追溯、能回滚；
- **可迁移**：换一台 Jenkins，Jenkinsfile 还在仓库里，拷过去就能用；
- **可审查**：流程变更走 PR，团队所有人都能看到、能评审。

**面试怎么答**
> "Jenkinsfile 放在项目仓库根目录，和代码同源同版本。这样'哪个版本的代码对应哪套构建流程'是可追溯的，不是散落在 Jenkins 配置里的。"

### 决策 3：为什么用「声明式(Declarative)」`pipeline {}`，而不是「脚本式」

| 语法 | 样子 | 特点 |
|---|---|---|
| **声明式** | `pipeline { agent / stages / post }` | 结构固定，像填表格；Jenkins 能做语法校验、能出阶段视图 |
| 脚本式 | `node { }` 纯 Groovy | 灵活，但可读性差、结构自由 |

**为什么选声明式**
- 结构固定 → 可读性高，团队协作友好；
- 需要复杂逻辑（循环、条件、变量）时，用 `script { }` 块嵌入 Groovy 即可 → 兼顾规范与灵活。

**面试怎么答**
> "默认用声明式，保证可读性和规范；遇到复杂逻辑用 script 块包 Groovy，不用为了灵活牺牲可读性。"

---

## 三、Jenkinsfile 逐行说明

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

| 写法 | 起什么作用 | 为什么这么写 |
|---|---|---|
| `pipeline { }` | 声明式流水线的**外壳** | 固定结构 = Jenkins 能解析、能做语法校验、能画阶段视图 |
| `agent any` | 指定**在哪台机器/节点**上执行 | `any` = 任意可用节点。你现在只有**内置节点（= Jenkins 容器本身）**，所以流程就跑在容器里 —— **这正是为什么容器内必须有 docker CLI** |
| `stages { }` | 阶段的容器 | 把流程拆成可视化阶段，UI 上显示成色块，**失败一眼看出卡在哪一步** |
| `stage('拉代码')` | 一个**阶段** | 名字是给人看的；面试问"你流水线有几个阶段"就靠它答 |
| `steps { }` | 阶段内的**具体动作** | 声明式语法里，动作只能写在 steps 里 |
| `git branch:'main', url:'…'` | 从 GitHub **拉代码到工作区** | 用 `git` 插件步骤而不是 `sh 'git clone'`：它会记录 commit 和变更，UI 里能看到"这次构建对应哪个提交" |
| `sh 'docker build -t devops-demo:v1 .'` | **构建镜像** | 这条命令会发给**宿主机的 dockerd** 执行（DooD）；末尾的 `.` 是**构建上下文**=当前工作区 |
| `post { success { } }` | 构建**结束后的钩子** | 常用来发通知、清理；面试可答"成功通知、失败告警、清理工作区都放 post" |

---

## 四、UI 操作逐条说明

| 操作 | 起什么作用 | 为什么 |
|---|---|---|
| 新建任务 → 选「流水线」 | 创建一个 Pipeline 类型的 Job | 决定这个 Job 用**代码**描述流程 |
| 名称 `devops-demo-pipeline` | Job 的唯一标识 | 它会成为工作区目录名：`/var/jenkins_home/workspace/devops-demo-pipeline` |
| 定义 = `Pipeline script` | 脚本**直接存在 Jenkins 里** | 第一次先放 Jenkins 内，跑通后再改成"从 SCM 读取 Jenkinsfile"（Day3 做） |
| 保存 → 立即构建 | 触发一次执行 | 手动触发是最基础的触发方式；Day5 会加 webhook 自动触发 |
| 看「控制台输出」 | 看每一步的**实时日志** | 排障的唯一入口，报错必看这里 |

---

## 五、成功判据

- 构建历史出现**蓝色的 `#1`**（不是红球）；
- 三个阶段**全部绿色**；
- 控制台最后出现 `devops-demo   v1   <image id>`；
- 在**宿主机**上执行 `docker images | grep devops-demo` 也能看到 —— 因为镜像是建在宿主机上的，**这就是 DooD（Docker-outside-of-Docker）的效果**。

---

## 六、面试问答速记（可直接背）

**Q：你们 CI 怎么做的？**
> 用 Jenkins 的 Pipeline as Code。Jenkinsfile 跟代码放同一个仓库，流程变更走 PR 评审。典型的流水线分三段：拉代码、构建镜像、验证。

**Q：为什么用 Pipeline 不用 FreeStyle？**
> FreeStyle 的配置只存在 Jenkins 里，不可版本化、不可评审、没法复现。Pipeline 把流程变成代码，可进 Git、可迁移、可评审。

**Q：Jenkins 怎么构建 Docker 镜像的？**
> Jenkins 本身跑在容器里，我们把宿主机的 docker.sock 挂载进容器，再把 docker CLI 拷进去，容器里的 docker 命令就能直接操作宿主机的 Docker 守护进程，也就是 DooD 方案。好处是镜像直接建在宿主机上，K8s 和镜像仓库都能直接用。

**Q：这套方案有什么风险？**
> 把 docker.sock 交给容器，等于给了这个容器宿主机 root 权限，拿到 socket 就能起特权容器、挂载宿主机任意目录。所以生产上要么严格控制 Jenkins 权限，要么用 Kaniko / Buildah 这类不需要守护进程的构建工具。

**Q：为什么用声明式而不是脚本式？**
> 声明式结构固定、可读性好、Jenkins 能做语法校验和阶段视图。需要复杂逻辑时用 script 块嵌 Groovy，兼顾规范和灵活。
