# Week3 开题：GitLab CI（10/22 - 10/26）

> 计划依据：《技术转型计划-1003.md》第 72 行（W3 = GitLab CI：`.gitlab-ci.yml` 跑通 + 与 Jenkins 对比话术）
> 技术路线：gitlab.com 免费账号 + 本地 Runner（**不**本地装 GitLab CE，省 3.5G）
> 开题日期：2026-10-21 22:30（晚于计划窗口 2 天，因 10/19 VM 迁移修复占满 + 断更 2 天，整体顺延）

## 一、W3 目标（可验证）

1. gitlab.com 上一条 `.gitlab-ci.yml` 流水线完整跑通（build → test → 产出 artifact）
2. 本地 Runner 注册并执行 job（先 shell executor 起步，避开 VM 内 docker-in-docker 坑）
3. 产出一篇 Jenkins vs GitLab CI 对比文（≥1000 字，含面试话术）并入库
4. 每日 GitHub ≥1 提交 + Shell 每日 1 题，不再断更

## 二、五天分解

| 天 | 日期 | 内容 | 验收标准 |
|---|---|---|---|
| D1 | 10/22 四 | 概念五段式吃透；gitlab.com 账号/项目/Runner 注册；hello pipeline 首绿 | Runner online + pipeline passed |
| D2 | 10/23 五 | stages / rules / artifacts / cache 逐项最小实验 | 四要素各一个能说清的实验 |
| D3 | 10/24 六 | 用 devops-demo 重写流水线（build→test→docker build→push） | 完整流水线绿 |
| D4 | 10/25 日 | Jenkins vs GitLab CI 对比 + 面试话术成文 | 对比文 commit |
| D5 | 10/26 一 | 周复盘 + 自测题 | 周记 commit |

## 三、关键概念（D1 开讲，五段式）

> ⚠️ 本节 2026-10-21 夜已修正，原文错误表述见文末「修正记录」，勿照原文复习。

- **先立边界**：Jenkins + Jenkinsfile **也是**流水线即代码（Week 2 已实操验证）。PaC 不是 GitLab CI 的差异点，把它当差异点是常见教程谬误
- 真差异只有三条：① 表达力——Groovy 程序（图灵完备）vs 声明式 YAML（有限关键字，哲学差异非代差）；② 系统边界——Jenkins 的 CI 与仓库是两个系统，靠 poll SCM / webhook 桥接（Week 2 配过 H/5），GitLab 的 CI 内生于仓库平台，push 是内部事件；③ 服务端残留配置——Jenkins 的 credentials/工具/节点、GitLab 的 CI/CD Variables 都在仓库外，完全"全在仓库"的 CI 不存在
- 怎么工作：`.gitlab-ci.yml` → Runner 认领 job → executor 执行 → 结果回传 GitLab
- 代价/局限：Runner 机器要常开（关机 = CI 停摆）；YAML 表达力有边界；gitlab.com 国内网络不稳（注册/拉代码走代理）
- 面试怎么答：**禁止**说"Jenkins 配置漂移/插件地狱所以换 GitLab"（一句"你们没用 Jenkinsfile 吗"即被追问穿）。标准答法见「修正记录」第 3 条

## 四、环境备注

- 家里网络走 nano 代理（127.0.0.1:65532）；gitlab.com 注册/Runner 注册若失败先查代理是否在跑（`netstat -ano | grep 65532`）
- Runner 装宿主机（Windows, shell executor）；D3 需要 docker build 时再评估是否挪到 master VM
- 断更问责：10/20、10/21 无提交（断更 2 天），纪律恢复起点 = 本文件 + 10/22 任务包

---

## 修正记录（2026-10-21 夜，用户当面抓出，保留示众）

1. **错误原文**："它解决什么问题：Jenkins 的配置漂移、插件地狱、job 散落" / "为什么需要它：流水线即代码"——把 GitLab CI 的平台特性讲成了 Jenkins 的缺陷。
2. **错因**：PaC 不是 GitLab CI 差异点。Jenkins 2.0（2016）起 Pipeline as Code 即核心能力，用户 Week 2 已用 Jenkinsfile 实操验证。教程谬误来源：Jenkins 旧形象停留在 FreeStyle 时代，被营销话术继承。
3. **面试标准答法（修正版）**："两者都支持流水线即代码，我用 Jenkins + Jenkinsfile 落地过。差异在架构和表达力：Jenkins 是独立 CI 系统，靠 webhook 或轮询与仓库桥接，Jenkinsfile 用 Groovy 表达力强但需要代码规范兜底；GitLab CI 内生于代码平台，声明式 YAML 和 MR 深度集成，触发走 git 事件零延迟。选型看团队形态：编排复杂、有平台团队维护选 Jenkins；小团队要低总拥有成本选 GitLab CI。"
4. **方法论教训**：讲任何对比前，先核对对方的实操基线（用户 Week 2 笔记明写"弃 FreeStyle 选 Jenkinsfile"），再决定哪些"痛点"对对方真实存在。把推断当事实 = 信任赤字。
