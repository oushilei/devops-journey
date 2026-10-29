# Jenkins 与 GitLab CI 对比（W3 产出）

## 一、结论（约 100 字）

两者都能做 CI/CD，**都支持流水线即代码**，不存在"谁不会 PaC"的问题。真正的差异不在功能，在**架构形态**：Jenkins 是一套独立的 CI 系统，GitLab CI 是代码平台的内生能力。选型看团队形态，不看谁更"先进"。

如果团队足够大、流水线逻辑复杂，我会选 Jenkins；反之（团队小、希望低维护成本）我会选 GitLab CI。

## 二、相同点：两者都是流水线即代码（约 150 字）

Jenkins 2.0（2016）起，Jenkinsfile 就进仓库了——和代码同版本、可 review、可回滚。我在 Week 2 实际验证过：Jenkinsfile 六语法（agent / environment / when / parallel / post / script）全部跑通，当时的选择理由就是"FreeStyle 的配置只存在 Jenkins 数据库里，不可版本化、不可评审"。

GitLab CI 的 `.gitlab-ci.yml` 同理，也在仓库里。**所以"流水线即代码"不是 GitLab 的卖点，是两者的共同前提。**

## 三、三个真实差异（约 400 字）

### 1. 表达力：编程 vs 配置

- Jenkinsfile 是 **Groovy 程序**，图灵完备：能写循环、函数、条件分支、共享库
- `.gitlab-ci.yml` 是**声明式配置**：有限关键字（stages / rules / needs / parallel / extends），复杂逻辑只能塞进 `script` 调 shell

这不是代差，是哲学差异：Jenkins 给你编程能力（代价是容易写出没人看得懂的天书），GitLab 约束你写配置（代价是复杂编排会憋得难受）。

### 2. 系统边界：两个系统 vs 一个平台

- **Jenkins**：CI 服务器和代码仓库是**两个系统**，靠桥连接——要么轮询 SCM（我配过 `H/5 * * * *`，最坏 5 分钟延迟），要么在 git 服务器配 webhook
- **GitLab CI**：git 服务器和 CI 是**同一个系统**，push 是内部事件，不需要桥

这条差异带来的工程代价很具体：Jenkins 里为了让 CI 够到 Docker，我做过 DooD 三件套（挂 `docker.sock` + `docker cp` 注入 CLI + `--group-add 995`）；GitLab 的 shell executor 直接跑在宿主机上，`docker` 命令天然可用。

### 3. 服务端残留配置：两边都有

- Jenkins：credentials、工具版本、节点定义在服务端
- GitLab：CI/CD Variables 在项目设置里

**"全在仓库里"的 CI 不存在**——secrets 永远在仓库外，这是安全约束，不是缺陷。

## 四、各自短板（约 200 字）

**Jenkins**：要自己养服务器和插件（我数过，真实插件 94 个），升级要等插件跟上；单点风险靠架构解决。

**GitLab CI**：

- `post { always/success/failure }` 没有原生对应，`after_script` 在新 shell 里跑、拿不到 script 的变量
- 没有逻辑能力，复杂规则堆 `rules` 可读性会崩
- 变量是注入式环境变量，整个 job 可见，**泄露面比 Jenkins 的块级 withCredentials 大**（靠 masked + protected 补偿）
- 没有 Replay（Jenkins 能改 Jenkinsfile 直接重跑不提交）

这四条里我最在意表达力——它决定了一条流水线能长到多复杂还不失控。

## 五、选型与实测数据（约 150 字）

**选 Jenkins 的场景**：编排复杂、需要 Groovy 级别的逻辑、团队有平台工程师维护 CI 基础设施。

**选 GitLab CI 的场景**：团队小、要低总拥有成本、代码本来就在 GitLab、希望 CI 和 MR/仓库深度集成。

实测数据（自建 GitLab CE + shell executor）：

- 并发：`concurrent=1` 时同 stage 两个 job 排队，流水线 57 秒；改成 2 后并行，28 秒
- 四个门禁落地：单测门禁（不过不打包）、制品不可变（sha tag 不覆盖）、manual 审批、一键回滚
- artifacts 经 GitLab 中转（日志 `Downloading artifacts`），cache 存 Runner 本地（日志 `stored only locally`）

这次实践扩充了我的技能库，也让我在流水线产品选型上有了自己的判断依据——不再只看功能列表，而是看并发模型、制品管理、回滚路径这些真正影响交付的东西。

## 六、技术问答版答案（背这段就够）

> "两者都支持流水线即代码，我用 Jenkins + Jenkinsfile 落地过。差异在架构和表达力：Jenkins 是独立 CI 系统，靠 webhook 或轮询与仓库桥接，Jenkinsfile 用 Groovy 表达力强但需要代码规范兜底；GitLab CI 内生于代码平台，声明式 YAML 和 MR 深度集成，触发走 git 事件零延迟。选型看团队形态：编排复杂、有平台团队维护选 Jenkins；小团队要低总拥有成本选 GitLab CI。"
