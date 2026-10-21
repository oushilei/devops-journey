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

- 它解决什么问题：Jenkins 的配置漂移、插件地狱、job 散落各处
- 为什么需要它：流水线即代码（pipeline as code），与代码同仓同版本同评审
- 怎么工作：`.gitlab-ci.yml` → Runner 认领 job → executor 执行 → 结果回传 GitLab
- 代价/局限：gitlab.com 免费 CI 分钟额度、Runner 机器要常开、YAML 表达力边界
- 面试怎么答：先讲"为什么声明式"，再讲四要素，最后主动对比 Jenkins

## 四、环境备注

- 家里网络走 nano 代理（127.0.0.1:65532）；gitlab.com 注册/Runner 注册若失败先查代理是否在跑（`netstat -ano | grep 65532`）
- Runner 装宿主机（Windows, shell executor）；D3 需要 docker build 时再评估是否挪到 master VM
- 断更问责：10/20、10/21 无提交（断更 2 天），纪律恢复起点 = 本文件 + 10/22 任务包
