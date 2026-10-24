# Week3 D4：最小生产版流水线（门禁 / 不可变 / 审批 / 回滚）

> 日期：2026-10-24
> 环境：自建 GitLab（node1）+ shell executor Runner（concurrent=2）+ 私有 registry（:5000）
> 项目：`devops/gitlab-ci-lab`

## 一、D4 的核心命题

> **"流水线能跑通" ≠ "能安全上线"。** 两者差的不是技术，是四个门禁。

| # | 门禁 | 解决的真实事故 | 实现方式 |
|---|---|---|---|
| ① | 质量门禁 | 跳过测试直接上线，故障进生产 | `scripts/test.sh` 失败即 `exit 1`，后续 stage 不执行 |
| ② | 制品不可变 | 用 latest 部署，想回滚时发现旧版本已被覆盖 | 每次构建打 sha tag；tag 已存在则跳过推送 |
| ③ | 发布审批 | 自动部署把未评审代码推上生产 | `when: manual`，必须人点 ▶ |
| ④ | 一键回滚 | 上线出错，手工重部署耗时几十分钟 | 独立 manual job，用旧 tag 重新拉起容器 + 冒烟验证 |

## 二、流水线结构

```
build-job ──► test-job ──► package-job ──► [deploy-job ▶] / [rollback-job ▶]
  自动          自动           自动              手动（deploy 阶段两个并列）
```

| job | 阶段 | 干什么 | 关键机制 |
|---|---|---|---|
| build-job | build | 产出 `dist/build-info.txt` | artifacts 传给下游 |
| test-job | test | 跑 `scripts/test.sh` | 失败即停 |
| package-job | package | 查重 → docker build → push | 标记文件 `.skip_push` |
| deploy-job | deploy | 用 sha tag 起容器 + 冒烟 | `when: manual` + environment |
| rollback-job | deploy | 用指定 tag 起容器 + 冒烟 | 变量 `TARGET_TAG` 可覆盖 |

## 三、四个门禁的实现细节

### ① 质量门禁：test.sh

```bash
set -e                       # 任何命令失败立即退出
for f in Dockerfile app/index.html; do
  if [ ! -f "$f" ]; then echo "FAIL: 缺少 $f"; exit 1; fi
done
if ! grep -q "Hello from GitLab CI" app/index.html; then exit 1; fi
if [ ! -f dist/build-info.txt ]; then exit 1; fi
```

`exit 1` → job 返回码非 0 → GitLab 判定失败 → **package 阶段根本不启动**。

### ② 制品不可变：check-tag.sh + 标记文件

```bash
EXIST=$(curl -s --max-time 5 "$REGISTRY/v2/devops/hello/tags/list" | grep -o "\"$TAG\"" || true)
if [ -n "$EXIST" ]; then
  touch .skip_push     # 创建标记
  exit 0               # 注意：跳过是"成功"，不是失败
fi
```

yml 里：

```yaml
- rm -f .skip_push                                     # 清上一轮残留
- bash scripts/check-tag.sh $CI_COMMIT_SHORT_SHA $REGISTRY
- 'if [ -f .skip_push ]; then 跳过; else build+push; fi'
```

**关键认知**：`.skip_push` 不是"记忆"，是"这一轮的传话筒"。真正的状态在 **registry**（唯一真相源），每轮都重新去问。
**为什么必须 `rm -f`**：shell executor 工作目录复用，上一轮的标记会残留，导致新提交被误跳过。

### ③ 发布审批：`when: manual`

job 出现在流水线图上但不执行，等人点 ▶。配合 `environment` 可在 Environments 页看到部署记录。

### ④ 一键回滚：rollback-job

```yaml
rollback-job:
  when: manual
  variables:
    TARGET_TAG: latest      # 手动触发时可覆盖成具体 sha
  script:
    - docker rm -f hello-app ...
    - docker run -d --name hello-app -p 8081:80 $IMAGE:$TARGET_TAG
    - sleep 3
    - 冒烟 curl（失败打印容器日志并 exit 1）
    - docker inspect --format '{{.Config.Image}}' hello-app   # 证明真的换了镜像
```

**回滚的本质**：不是重新构建旧代码，而是**用已知可用的旧镜像替换当前容器**。不 build、不碰代码。

## 四、回滚的镜像选择逻辑（重点）

| 策略 | 怎么定 tag | 评价 |
|---|---|---|
| ① 人工指定 | 触发时手填 `TARGET_TAG=a38117e5` | 简单但依赖人记，易填错 |
| ② 上一个成功部署的版本 | 从部署历史取 | **生产主流**，精准可自动化 |
| ③ 上一个 commit | `HEAD~1` 的 sha | **不可靠**：上一提交 ≠ 上一部署过的版本 |

**生产做法**：
- K8s：`kubectl rollout undo deployment/hello`（Deployment 自带 ReplicaSet 历史，默认保留 10 个）
- GitLab：查 environment 的 deployments API 取上一版
- 自研发布系统：自己记录"当前版本"指针

**三个必须理解的配套条件**：
1. 记录时机 = **健康检查通过后**，不是部署命令成功
2. 记录的是**历史栈**，不是单个快照（可连续回滚）
3. 前提是**制品不可变**——旧 tag 不覆盖，旧镜像才一直在

> 一句话：不可变保证"存得住"，部署历史保证"选得准"，冒烟测试保证"回滚成功"。

## 五、关键坑：镜像构建成功 ≠ 能运行

**现象**：deploy-job 报 `部署后 HTTP=000`，容器 `Exited (1)`。

**根因**（容器日志）：

```
[crit] 1#1: pwrite() "/run/nginx.pid" failed (1: Operation not permitted)
```

`nginx:alpine` 最新版（1.31.6）在 **CentOS 7 内核 3.10** 上写 pid 文件失败，进程直接退出。

**修复**：`FROM nginx:1.24-alpine`（兼容老内核，node1 本地也有缓存）。

**教训**：
- build 绿、push 成功，容器照样可能启动即退出
- 所以生产流水线**必须有部署后冒烟测试**——这正是门禁④的价值
- 进一步：部署 job 失败时应自动打印容器日志（已加入），否则排查要登机器

## 六、Bash 语法要点（今天用到的）

| 语法 | 含义 |
|---|---|
| `set -e` | 任何命令失败立即退出脚本 |
| `for x in a b; do … done` | 循环 |
| `if …; then … fi` | 条件判断 |
| `[ ]` | test 命令，**两侧必须空格** |
| `-f` / `-n` / `-z` | 是文件 / 字符串非空 / 为空 |
| `$1` `$2` | 脚本的第 1、2 个参数 |
| `$(命令)` | 命令替换 |
| `\|` | 管道；`\|\|` 或；`&&` 与 |
| `exit 0` / `exit 1` | 成功 / 失败（GitLab 看返回码） |
| `touch` | 创建空文件 |

**YAML 里写命令的坑**：`script` 的每一项只要含 `: `（冒号+空格），YAML 会把它解析成 key:value → 报 `script config should be a string`。**解法：整行用单引号包起来**。

## 七、脚本放哪里：仓库 vs 服务器

**放仓库**（`scripts/` 目录，与代码同 commit）：

- 改动有 commit 记录，可 review、可回滚
- 脚本与代码**版本严格对齐**，结果可复现
- 换 runner / 加 runner，脚本自动跟着走
- 面试可讲"基础设施即代码"

**放服务器**（老做法）→ 配置漂移的温床：代码更新了脚本还是旧的，换台机器就废。

**与 Jenkins 的对应**：FreeStyle 网页配置 vs Jenkinsfile 进仓库，同一个道理。

**CRLF 坑**：Windows 上编辑 `.sh` 会变成 CRLF，Linux 上跑报 `$'\r': command not found`。解法：仓库根加 `.gitattributes` 写 `*.sh text eol=lf`。

**可执行位**：用 `bash scripts/xxx.sh` 调用不依赖 `+x`，比 `./xxx.sh` 更稳（Windows 上可执行位容易丢）。

## 八、面试话术

> "生产流水线的核心不是 stage 多，而是每段都有门禁和回滚：单测门禁不过就不打包；制品按 commit sha 打 tag 且不可变，保证能精确回溯和回滚；部署走 manual 审批；回滚用已验证可用的旧镜像重新拉起并冒烟验证。回滚选版本靠部署历史而不是 git 顺序，K8s 场景直接用 rollout undo。"

## 九、下一步（D5）

- Jenkins vs GitLab CI 对比文（1000 字，含六语法对照 + 票据管理 + 短板）
- 周复盘 + 自测题
