# Week2 Day3 · Jenkinsfile 语法（六个语法点，全部实机验证）

> 本文所有结论均来自 `k8s-node1` 上 Jenkins 的**真实构建日志**，不是书本知识。
> 配套文档：`Week2-Day2-第一条流水线-原理讲解.md`、`Week2-Day2-复习提纲.md`
>
> **验证环境**：Jenkins 2.580.1（Docker 容器 `jenkins`），流水线以 `jenkins` 用户（uid=1000）运行。

---

## 目录

1. `agent` —— 在哪台机器执行
2. `environment` + `parameters` —— 变量与构建输入
3. `when` —— 条件执行
4. `parallel` —— 并行执行
5. `post` —— 构建后处理（9 个分支）
6. `script` —— 嵌入 Groovy 逻辑

---

## 1. `agent` —— 在哪台机器执行

### 作用
指定流水线（或某个阶段）**在哪台机器上执行**。它决定：代码拉到哪里、`sh` 命令在谁的 shell 里跑、工作区目录在哪。

### 三种写法

```groovy
// ① 流水线级：所有阶段共用一个节点
pipeline {
    agent any
    ...
}

// ② 流水线级 none + 阶段级各自指定
pipeline {
    agent none                        // 顶层不指定，每个 stage 必须自己写 agent
    stages {
        stage('A') {
            agent any
            steps { ... }
        }
        stage('B') {
            agent { label 'build-node' }  // 指定标签的节点
            steps { ... }
        }
    }
}

// ③ 其他写法
agent { label 'linux' }               // 按标签选节点
agent { docker { image 'golang' } }   // 在指定镜像的容器里执行
```

### 实机验证（job `day3-agent-demo`）

```
Running on Jenkins in /var/jenkins_home/workspace/day3-agent-demo   ← 每个阶段各出现一次
+ hostname
817a93ff945f                                                        ← 容器短 ID
```

**结论**：`agent any` 落到了**内置节点**，也就是 **Jenkins 容器本身**（`hostname` 返回容器 ID 是铁证）。
用 `agent none` + 阶段级 agent 时，**每个阶段各自申请一次工作区**。

### 面试怎么答
> "默认用 `agent any` 跑在内置节点。规模变大后会给构建机打标签，用 `agent { label 'build' }` 把构建任务分到专门的构建机上，减轻主节点压力。也可以用 `agent none` 让不同阶段落在不同节点。"

---

## 2. `environment` + `parameters` —— 变量与构建输入

### 2.1 两者本质不同

| | `environment` | `parameters` |
|---|---|---|
| **本质** | 给变量**赋值** | **声明**一个输入项（元数据） |
| **生效位置** | 脚本**内部** | 脚本**外部**（构建触发时） |
| **值从哪来** | 写在代码里，**写死** | **由触发方在构建时提供** |
| **每次构建** | 都一样 | 可以不同 |
| **取值写法** | `${env.XXX}` | `${params.XXX}` |
| **UI 表现** | 无 | **自动生成表单/下拉框** |

**一句话**：`environment` 是"我在脚本里定好一个值"；`parameters` 是"我声明需要一个值，由运行时提供"。

**判断标准**：**这个值是谁给的？** 我给的 → `environment`；**构建的人给的** → `parameters`。

### 2.2 两者可以配合

```groovy
parameters {
    string(name: 'BUILD_VERSION', defaultValue: 'v1', description: '版本号')
}

environment {
    IMAGE_NAME = 'devops-demo'
    FULL_TAG   = "devops-demo:${BUILD_VERSION}"     // 引用参数，值随之变化
    REGISTRY   = 'localhost:5000'
}
```

**实际项目常见做法**：`parameters` 做"可变入口"，`environment` 做"加工后的常量"（如拼好的完整镜像地址），拼接逻辑只写一次。

### 2.3 参数类型

| 类型 | UI 表现 | 用途 |
|---|---|---|
| `string` | 单行输入框 | 版本号、分支名 |
| `booleanParam` | 勾选框 | 是否跳过测试 |
| `choice` | 下拉框 | 从固定选项选择 |
| `text` | 多行输入 | 备注、批量数据 |
| `password` | 密码框 | 敏感值 |
| `file` | 文件上传 | 传配置文件 |

**注意拼写**：`choice(name: ..., choices: [...])` —— `choice` 是类型，`choices` 是列表。

### 2.4 立即构建 vs 使用参数构建

| | 立即构建 | 使用参数构建 |
|---|---|---|
| 参数 | 用 `defaultValue` | 弹表单可覆盖 |
| 无参数任务 | 正常 | **不显示此选项** |

**只要 Jenkinsfile 里有 `parameters` 段，任务页就会多出「使用参数构建」按钮。**

### 2.5 插值规则（★ 实测结论，纠正常见错误说法）

**常见错误说法**："单引号不插值，双引号才插值。" —— **对 Jenkinsfile 里的 Groovy 字符串，这个说法是错的。**

**实测数据**：

| 写法 | 实测输出 |
|---|---|
| `sh 'echo C-${MYVAR}'` | `HelloFromEnv` ← **单引号也插值** |
| `sh "echo D-${MYVAR}"` | `HelloFromEnv` |
| `sh 'echo A-\${MYVAR}'` | `${MYVAR}` ← 字面量（转义生效） |
| `sh "echo B-\${MYVAR}"` | `HelloFromEnv` ← **转义"失效"** |

**统一模型（记住这个就够了）**：

```
Jenkins 执行 sh 步骤 = 翻译两次
  第 1 次：Groovy 把 ${...} 换成变量的值（environment 变量此时已可用）
  第 2 次：结果作为 shell 命令交给 bash 执行（bash 再处理环境变量、$PATH 等）
```

**关键**：`environment` 里的变量**同时存在于"Groovy 层"和"shell 环境变量层"**，所以转义也可能被其中一层处理掉——这就是上表 A/B 行不同的原因。

**实用规则（够用）**：

| 场景 | 写法 |
|---|---|
| 用 `environment` 变量（99% 情况） | `sh "docker build -t ${IMAGE_NAME}:${BUILD_VERSION} ."` ← 双引号直接用 |
| 想用 **shell 自己的**变量（`$PATH`、自行 `export` 的） | `sh "echo \$PATH"` ← **必须转义**，否则 Groovy 先消费掉 |
| 要输出字面量 `${...}` | `sh 'echo \${X}'`（单引号 + 转义） |

> **注意**：这条属于"知道即可"的细节，不是核心考点。日常用双引号 `${env.XXX}` 即可。

### 2.6 面试怎么答
> "`environment` 放不变的东西——镜像名前缀、仓库地址；`parameters` 放每次构建会变的东西——版本号、目标环境。需要拼接派生的值也在 `environment` 里引用参数，这样拼接逻辑只写一次。改参数不用动脚本，改常量也不影响构建输入。声明了参数 Jenkins 会自动生成表单，手动触发能改，自动触发用默认值。"

---

## 3. `when` —— 条件执行

### 作用
给阶段加条件，满足才执行，否则**跳过（skipped）**—— **跳过不等于失败**。

### 为什么需要
代码有多个分支，不同分支该做不同的事：

| 分支 | 应该做 |
|---|---|
| `main` | 构建镜像 **+ 推送到仓库** |
| `feature-xxx` | 只跑测试/检查，**不推镜像** |

不加 `when` 的话，任何分支一提交就推镜像 → **镜像仓库塞满临时镜像，还可能覆盖正式版本**。

### 常用条件

| 写法 | 含义 |
|---|---|
| `branch 'main'` | 分支是 main |
| `branch pattern: 'release-*'` | 分支名匹配模式 |
| `environment name: 'X', value: 'Y'` | 环境变量匹配 |
| `expression { ... }` | 自定义 Groovy 表达式为真 |
| `not { ... }` | 取反 |
| `anyOf { ... }` / `allOf { ... }` | 满足任一 / 全部 |
| `beforeAgent true` | **先判断条件再申请节点** |

### `beforeAgent true` 详解

`when` 判断与"申请执行节点"是两件独立的事，**默认顺序是先申请节点、再判断**：

```
默认：           ① 申请节点 → ② 判断 when → ③ 不满足则跳过（节点白占）
beforeAgent true：① 判断 when → ② 不满足则跳过，不申请节点 ✅
```

**价值在多节点环境**：条件不满足的任务不再占用构建机；若构建机按量付费还能省钱。**单节点环境无感，但加了无害，推荐总是加上。**

### 实机验证（job `day3-when-demo`）

```
当前模拟分支: dev
这个阶段没有 when，永远会跑
Stage "只在 main 执行（推镜像）" skipped due to when conditional    ← 准确措辞
🚧 非主干分支 → 只跑代码检查，不推镜像
===== 构建结束（不管成功失败都会执行） =====
Finished: SUCCESS
```

**三个验证点**：
1. `skipped due to when conditional` —— when 不满足时 Jenkins 的标准日志
2. **跳过的阶段里没有出现 `Running on Jenkins`** → **`beforeAgent true` 生效，未申请节点**
3. 整体仍是 **SUCCESS** —— **跳过 ≠ 失败**
4. `TARGET_BRANCH=main` 时该阶段**绿色执行**（正反两面均已验证）

### 面试怎么答
> "我们用 `when` 做分支守门：只有 main 才执行推镜像，其他分支只跑测试，避免开发分支的临时镜像污染仓库。另外加了 `beforeAgent true`，条件不满足时不去分配执行节点，避免无意义的资源占用——在多节点或按量付费环境很有价值。"

---

## 4. `parallel` —— 并行执行

### 作用
多个阶段**同时跑**，而不是排队。

### 为什么需要
代码提交后要做几件**互相独立**的事：单元测试、代码扫描、安全扫描。串行就是浪费时间：

```
串行：测试(5s) → 扫描(5s) → 安全(5s)  = 15 秒
并行：测试(5s) ┐
      扫描(5s) ├ 同时进行            = 5 秒
      安全(5s) ┘
```

**这是缩短流水线总时长的第一手段。** 面试问"流水线太慢怎么优化"，标准答案就是**并行化 + 缓存**。

### 语法

```groovy
stage('并行执行') {
    parallel {
        stage('任务A') { steps { ... } }
        stage('任务B') { steps { ... } }
        stage('任务C') { steps { ... } }
    }
}
```

**两个要点**：
- `parallel { }` 里装的是 **stage**，不是直接写 `steps`
- 并行的子阶段**默认共用外层 agent**；要各用不同节点需在子 stage 单独写 `agent`

### 实机验证（job `day3-parallel-demo`）

| 阶段 | 耗时 |
|---|---|
| 串行执行（3 个 sleep 5） | **16 秒** |
| 并行执行（3 个 sleep 5） | **5 秒** |
| ├ 任务A | 5 秒 |
| ├ 任务B | 5 秒 |
| └ 任务C | 5 秒 |
| Post Actions | 16 ms |
| 总计 | 22 秒 |

**16 秒 → 5 秒，缩短约 3 倍**，与理论完全一致。
**阶段视图上**：串行是一条直线，并行是**三个并列的分支**。

### 面试怎么答
> "独立的检查任务——单元测试、代码扫描、安全扫描——放在 `parallel` 块里同时执行，总时长取决于最慢的那个分支而不是累加。我把原来串行的 16 秒压到了 5 秒。配合依赖缓存和 Docker 层缓存，整体构建时间能明显下降。"

---

## 5. `post` —— 构建后处理

### 作用
流水线**结束后**执行的动作，按条件触发。

### 为什么需要
构建结束后总要"善后"：

| 场景 | 动作 |
|---|---|
| 构建**失败** | 通知负责人（钉钉/企微/邮件） |
| 构建**成功** | 通知、打 tag、归档产物 |
| **不管成败** | 清理工作区、记录耗时、删临时文件 |

**`post` 是"CI 能无人值守"的关键**——失败了系统主动找人，而不是人去找失败。

### ★ 完整 9 个分支（不是 6 个）

| 条件 | 触发时机 |
|---|---|
| `always` | 不管结果如何，都执行 |
| `changed` | 结果与上次**不同**（粗粒度） |
| `fixed` | **上次失败 → 这次成功**（修好了） |
| `regression` | **上次成功 → 这次失败**（变坏了） |
| `aborted` | 构建被中止（手动取消/超时） |
| `failure` | 失败 |
| `success` | 成功 |
| `unstable` | 成功但有警告 |
| `cleanup` | 在其他所有 post 条件之后执行，适合彻底清理 |

### `changed` / `fixed` / `regression` 的区别

| 上次 → 本次 | `changed` | `fixed` | `regression` |
|---|---|---|---|
| 失败 → 成功 | ✅ | ✅ | ❌ |
| 成功 → 失败 | ✅ | ❌ | ✅ |
| 失败 → 失败 | ❌ | ❌ | ❌ |
| 成功 → 成功 | ❌ | ❌ | ❌ |
| 不稳定 → 成功 | ✅ | ✅ | ❌ |

**记忆**：
- `changed` = "**变了**"（不关心往哪变）
- `fixed` = "**修好了**"（明确往好变）
- `regression` = "**变坏了**"（明确往坏变）

**用法**：想"结果一变就通知"用 `changed`；想专门在"修好了"发喜报用 `fixed`；
想专门在"变坏了"告警用 `regression`（比 `changed` 精准，不会因"失败变不稳定"这种好转而误报）。

### ★ 执行顺序

```
always
  → changed
    → fixed / regression
      → aborted / failure / success / unstable
        → cleanup
```

**三条规律**：`always` 最先；`cleanup` 最后；中间的是细化/互斥关系。

### 实机验证（job `day3-post-demo`）

`RESULT_MODE=成功` 时的实测输出：

```
【always】不管成败都会执行 —— 这里放清理/记录
【changed】本次结果与上次不同
【success】构建成功
【cleanup】最后执行
```

**顺序完全符合规律**：`always → changed → success → cleanup`。

**★ 一个实测发现**：**该任务第 1 次构建时 `changed` 也触发了** ——
说明**"没有上次记录"时 Jenkins 视为已发生变化**。

### 失败传播（★ 实测发现）

另一个 job 里某个阶段报错后：

```
Stage "script 用法2：捕获命令输出" skipped due to earlier failure(s)
Stage "script 用法3：控制流" skipped due to earlier failure(s)
```

**结论**：一个阶段失败后，**后续阶段默认全部连锁跳过**，整条流水线标记 FAILURE。
这是**默认的 fail-fast 行为**。若希望某阶段失败不影响后续，需用 `catchError` 或 `post` 处理。

**注意区分**：
- `when` 跳过 → 条件不满足，**正常**
- `skipped due to earlier failure(s)` → **前面出错了**

### 面试怎么答
> "`post` 放构建后的善后逻辑。我们用 `failure` 分支做失败告警——构建挂了自动通知对应的人，不用人盯着 Jenkins，这是流水线能无人值守的关键。`always` 做清理和记录，`cleanup` 最后兜底。另外 `post` 有 9 个条件，`changed` 是粗粒度的'结果变了'，`fixed` 和 `regression` 更精准，分别表示'修好了'和'变坏了'——专门给回归发告警就用 `regression`，避免误报。"

---

## 6. `script` —— 嵌入 Groovy 逻辑

### 作用
在声明式流水线里开一个"逃生舱"，**允许写任意 Groovy 代码**（if/else、循环、变量赋值、字符串处理）。

### 为什么需要
声明式的哲学是"结构固定、可读性好"，代价是**表达能力受限**。真实项目会遇到声明式写不出来的需求：

| 需求 | 声明式能写吗 |
|---|---|
| 根据计算结果决定镜像 tag | ❌ |
| 循环处理一批服务 | ❌ |
| 从命令输出提取值再使用 | ❌ |

**`script` 就是给这些场景开的口子**——不必为了几行逻辑就把整条流水线改成脚本式。

### 用法 1：变量作用域（★★ 最容易踩的坑）

```groovy
stage('变量作用域') {
    steps {
        script {
            def localVar = '我只活在 script 块里'      // Groovy 局部变量
            env.SHARED_VAR = '我可以跨阶段使用'        // 写进 env
            echo "块内可见: ${localVar}"
        }
        echo "块外读 env.SHARED_VAR: ${env.SHARED_VAR}"   // ✅ 能读到
        // echo "块外读 localVar: ${localVar}"            // ❌ 会报错！
    }
}
```

**实测报错**：

```
groovy.lang.MissingPropertyException: No such property: localVar for class: groovy.lang.Binding
	at WorkflowScript.run(WorkflowScript:17)
Finished: FAILURE
```

**★ 关键区别（实测确定）**：

| 访问方式 | 结果 |
|---|---|
| 块外 `${localVar}` | 💥 **报错 `MissingPropertyException`，流水线失败**——变量**根本不存在** |
| 块外 `${env.LOCAL_VAR}`（env 里没定义） | 输出**空字符串**，不报错——`env` 是 map，取不存在的 key 返回 null |

**一个是"名字都不存在"，一个是"名字在、只是没值"。** 这是本质区别。

**为什么**：`script { }` 是一个独立的 Groovy 闭包，`def` 变量的作用域就是闭包内部，**出了大括号就消失**。

**跨阶段传值的唯一实用方案**：写进 `env`（全局键值存储，且会被注入成真正的环境变量，`sh` 里也能用）。

### 用法 2：捕获命令输出（★ 真实项目核心手法）

```groovy
script {
    def whoami = sh(script: 'whoami', returnStdout: true).trim()
    def kernel = sh(script: 'uname -r', returnStdout: true).trim()
    echo "当前用户: ${whoami}"
    echo "内核版本: ${kernel}"
}
```

**逐段拆解**：

| 片段 | 含义 |
|---|---|
| `def whoami =` | 声明 Groovy 变量 |
| `sh(` | 调用 sh 步骤 |
| `script: 'whoami'` | 执行的命令 |
| `, returnStdout: true` | **关键开关：把命令输出作为返回值返回** |
| `.trim()` | **去掉首尾空白（主要是末尾换行符）** |

**为什么必须 `returnStdout: true`**：`sh` 默认**只执行、不返回内容**。我们的目的不是"跑一下看看"，而是**拿到结果做后续处理**。

**对比**：`returnStatus: true` 返回的是**退出码**（成功 0 / 失败非 0），不是输出内容。**不要混用。**

**为什么必须 `.trim()`**：命令输出自带尾部换行符。不 trim 的话：

```groovy
env.TAG = "devops-demo:${u}"     // → "devops-demo:root\n"
sh "docker build -t ${env.TAG} ." // 命令被换行打断，可能报错
```

**★ 真实项目里的标准写法（面试加分点）**：

```groovy
script {
    def GIT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    env.IMAGE_TAG = "${env.IMAGE_NAME}:${GIT_COMMIT}"
}
```

**为什么这么做**：如果 tag 用 `v1` 这种固定值，**你根本不知道线上跑的是哪次提交的代码**。
用 commit 哈希做 tag，**从镜像名就能倒推回源码版本**——这是可追溯性的基础。

### 用法 3：控制流（if / for）

```groovy
script {
    def jobs = ['service-a', 'service-b', 'service-c']
    for (job in jobs) {
        echo "依次处理: ${job}"
    }

    if (env.IMAGE_TAG.contains('root')) {
        echo 'root 用户构建'
    } else {
        echo '非 root 用户构建'
    }
}
```

### 实机验证（job `day3-script-demo`）

```
块内可见: 我只活在 script 块里
块外读 env.SHARED_VAR: 我可以跨阶段使用
当前用户: jenkins
内核版本: 3.10.0-1160.71.1.el7.x86_64
动态拼出的镜像 tag: devops-demo-jenkins
下个阶段也能用: devops-demo-jenkins
依次处理: service-a
依次处理: service-b
依次处理: service-c
非 root 用户构建
Finished: SUCCESS
```

**★ 意外收获**：`whoami` 返回 **`jenkins`**，不是 root。

**这说明了什么**：**流水线以 `jenkins` 普通用户（uid=1000）身份运行**。

**与 Day2 的 DooD 配置形成完整因果链**：

> 因为跑流水线的是 **jenkins 用户**（不是 root），而 `docker.sock` 权限是 `root:docker 660`，
> 所以它**默认无权读写** → **这就是 `--group-add 995` 必须存在的根本原因**。

### 面试怎么答
> "声明式流水线结构清晰但表达能力有限，需要复杂逻辑时用 `script` 块嵌 Groovy。最常用的是拿命令输出当变量——`returnStdout: true` 取出 `git rev-parse --short HEAD` 的结果，拼成镜像 tag，保证每个镜像都能追溯到对应代码版本。要注意 `def` 定义的局部变量出不了 `script` 块，访问会直接抛异常，跨阶段传值必须写进 `env`。"

---

## 附：六个语法点的验证证据一览

| # | 语法 | 验证方式 | 关键证据 |
|---|---|---|---|
| 1 | `agent` | `hostname` | `817a93ff945f`（容器 ID），每阶段各申请一次工作区 |
| 2 | `environment`/`params` | 打印变量 | `[Pipeline] withEnv` 出现；参数 v2 生效；拼接出完整镜像地址 |
| 3 | `when` | main vs dev 两次构建 | `skipped due to when conditional`；跳过的阶段无 `Running on Jenkins` |
| 4 | `parallel` | 对比耗时 | 串行 16s / 并行 5s；阶段视图呈分支形 |
| 5 | `post` | 成功/失败/重复构建 | `always → changed → success → cleanup` 顺序实证；首次构建触发 changed |
| 6 | `script` | 作用域 + 取输出 + 控制流 | 块外访问局部变量报 `MissingPropertyException`；`whoami` 返回 jenkins |

---

## 附：三天内我讲错的三个点（已更正，不重犯）

| 讲错的 | 正确的 |
|---|---|
| "单引号不插值" | `environment` 变量在单双引号里**都会**被 Groovy 插值 |
| "`localVar` 块外预期为空" | 实际是**直接报错、流水线失败**（不是空值） |
| 执行顺序里写了 `fixed/regression` 但分支表只列 6 个 | `post` 共 **9 个**条件，`fixed`/`regression`/`aborted` 被我漏讲 |

**共同教训**：讲枚举类知识点要**先列全集再讲子集**；讲"预期输出"要说清**具体会发生什么**，不用"预期为空"这类模糊措辞。
