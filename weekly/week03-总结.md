# Week 3 总结：GitLab CI（10/22 - 10/25）

> 原计划 10/19-10/25，因 10/19-10/21 修复 VM 网络迁移 + 断更，实际执行 10/22-10/25。
> 环境：自建 GitLab CE（node1 `192.168.245.150`，Docker，6G 内存）+ gitlab-runner 19.5.0（shell executor）+ 私有 registry（`:5000`）

## 一、目标与达成

| 开题目标 | 状态 | 证据 |
|---|---|---|
| `.gitlab-ci.yml` 跑通 | ✅ | build → test → package → deploy 四阶段全绿 |
| 与 Jenkins 对比话术 | ✅ | `notes/Week3-Jenkins与GitLabCI对比.md` |
| 每日提交不断更 | ✅ | 10/22、10/23、10/24、10/25 每天至少 1 commit |

## 二、五天实际进度

| 天 | 计划 | 实际完成 |
|---|---|---|
| D1 10/22 | 概念 + Runner 注册 + hello pipeline | ✅ 但先绕了一圈：gitlab.com 验证墙（中国手机号不在列表）→ 改自建 GitLab CE |
| D2 10/23 | stages/rules/artifacts/cache 四实验 | ✅ 全做完，附实测数据（57s→28s、MISS→HIT） |
| D3 10/24 上午 | devops-demo 复刻流水线 | ✅ 改为 lab 内最小应用（nginx + 静态页），双 tag + push registry |
| D4 10/24 晚 | — | ✅ 最小生产版：四门禁（门禁/不可变/审批/回滚） |
| D5 10/25 | 对比文 + 周复盘 + 自测 | ✅ 自测 10 题 + 对比文模板成稿 |

## 三、关键实测数据（面试可直接引用）

| 指标 | 数据 |
|---|---|
| 并发效果 | `concurrent=1`：同 stage 两 job 排队，**57 秒**；`=2`：并行，**28 秒** |
| 槽位排队实证 | 3 个 test job 抢 2 槽位 → 第三个 `Queued 28s / Duration 0s` |
| artifacts 传递 | 日志 `Downloading artifacts for build-job` —— 经 GitLab 中转 |
| cache 命中 | 第一次 `CACHE MISS`，第二次 `CACHE HIT`（上次写入时间戳），5s → 1s；日志 `stored only locally` |
| 镜像 | 双 tag（sha + latest）指向**同一 digest**；第二次 push 全 `Layer already exists` |
| 不可变 | 重跑同提交 → package 显示 `SKIP: 已存在` |

## 四、踩坑清单（8 条，全部真实踩过）

1. **gitlab.com 身份验证墙**：新建账号要手机验证才能跑流水线，国家列表无中国；GitHub OAuth 也只是登录进原账号 → 改自建
2. **Runner token 不能 OCR**：截图读 token 错字符 → `Verifying runner... is not valid`，必须复制原文
3. **`type` vs `cat`**：Linux shell executor 里 `type` 是 bash 内建命令，读文件要用 `cat`
4. **YAML 的 `: `**：plain scalar 里出现"冒号+空格"会被解析成 key:value → `script config should be a string`；解法=整行单引号包裹
5. **代理与内网**：git 全局代理指向 65532，代理一关连内网 GitLab 都连不上 → `NO_PROXY` 或 `git config http.http://<ip>.proxy ""`
6. **registry 换 IP 后 `insecure-registries` 忘了改**：HTTP registry 未登记 → `server gave HTTP response to HTTPS client`
7. **镜像构建成功 ≠ 能运行**：`nginx:alpine`（1.31.6）在 CentOS 7 内核 3.10 上 `pwrite nginx.pid` EPERM → 容器启动即退出；换 `nginx:1.24-alpine` 解决
8. **受保护分支 cache key 自动加 `-protected` 后缀**（GitLab 的隔离机制，不是 bug）

## 五、能力盘点（现在会什么）

**能独立完成**：

- 自建 GitLab CE（Docker）+ Runner 注册 + shell executor 配置 + 并发调优（concurrent/limit）
- 编写完整 `.gitlab-ci.yml`：stages / variables / rules / artifacts / cache / when: manual / environment
- 把 docker build + push 接进流水线，双 tag 策略，私有 registry 管理
- 实现四个生产门禁：质量门禁、制品不可变、发布审批、一键回滚
- 读懂 job 日志：executor 准备、detached HEAD、artifacts 下载、step_script、cleanup

**能讲清原理**：

- stage 与 job 的分工；pull 模型（runner 抢活）+ 原子锁
- 并发槽位公式：`min(job 数, Σconcurrent, Σlimit)`
- artifacts（经 GitLab，保正确性）vs cache（Runner 本地，只提速）
- tag 是名字、digest 是身份；sha tag 支撑可追溯与回滚
- 回滚选版本靠**部署历史**（健康检查通过才算），不是 `HEAD~1`

## 六、自测暴露的薄弱点（W4 要复习）

1. `$CI_PIPELINE_SOURCE` 的语义 —— 是**触发来源**（push/web/schedule/MR），不是"在哪看"
2. `rules` 管**存在性**（job 进不进流水线），`when: manual` 管**时机**（job 在图上等人点）
3. Jenkins 六语法 → GitLab 对应表，还没能自己完整复述

## 七、W4 方向

按计划进入下一模块（K8s 部署 / 监控方向，具体以《技术转型计划-1003》为准）。D4 的回滚只在 Docker 层面，W4 应升级到 **K8s 层面**：Deployment + rollout undo + 滚动更新，把 CI 的最后一公里真正打通。

**下一步最小目标**：流水线 package 后自动 `kubectl set image` 部署到 master 集群，并用 `kubectl rollout undo` 完成回滚。
