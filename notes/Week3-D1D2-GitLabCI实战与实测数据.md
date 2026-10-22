# Week3 D1-D2：GitLab CI 从零到跑通（含实测数据）

> 日期：2026-10-22
> 环境：自建 GitLab CE（node1 `192.168.245.150`，Docker 容器，6G 内存）+ gitlab-runner 19.5.0（shell executor）
> 项目：`devops/gitlab-ci-lab`（本地 `Claw\2026-09-21-16-30-29\gitlab-ci-lab`）

## 一、为什么自建 GitLab（而不是用 gitlab.com）

gitlab.com 对新建账号强制**身份验证**才能跑流水线，而验证页的手机号国家列表里**没有中国**；GitHub OAuth 登录也只是登录进原账号，验证墙仍在（流水线报 `This pipeline failed due to the user not being verified`）。

改走自建：
- 完全绕开验证墙，内网访问零代理依赖
- 面试价值更高："装过、配过、注册过 Runner"

自建命令（node1，`memsize` 已从 4G 调到 6G）：

```bash
docker run -d --hostname 192.168.245.150 --name gitlab \
  --restart always --shm-size 256m \
  -p 80:80 -p 2222:22 \
  -v /srv/gitlab/config:/etc/gitlab \
  -v /srv/gitlab/logs:/var/log/gitlab \
  -v /srv/gitlab/data:/var/opt/gitlab \
  -e GITLAB_OMNIBUS_CONFIG="external_url 'http://192.168.245.150'; gitlab_rails['gitlab_shell_ssh_port'] = 2222; prometheus_monitoring['enable'] = false; puma['worker_processes'] = 2; sidekiq['max_concurrency'] = 10;" \
  gitlab/gitlab-ce:latest
```

镜像走 `docker.m.daocloud.io`（国内源）。首次启动约 13 分钟到 healthy。

## 二、一条流水线从 push 到 passed 的完整链路

```
git push
 → GitLab 读 .gitlab-ci.yml，解析校验
 → 生成 pipeline，job 入队（pending）
 → Runner 长轮询 GitLab 要活（pull 模型，不是 GitLab 派活）
 → Runner 领到 job → git fetch（detached HEAD，精确检出某个 commit）
 → 下载上游 artifacts
 → 执行 before_script → script（逐行，任一行非 0 即失败）
 → 上传 artifacts → 回报状态
 → 下一 stage 解锁
```

真实日志片段（web-only-job）：

```
Preparing the "shell" executor...
Running on k8s-node1...
Fetching changes with git depth set to 20...
Checking out 75c4ff6c as detached HEAD (ref is main)...
Downloading artifacts for build-job (17)...
Executing "step_script" stage of the job script
$ echo "触发来源 = $CI_PIPELINE_SOURCE"
触发来源 = web
Job succeeded
```

## 三、并发模型：concurrent / limit / 槽位

| 概念 | 含义 | 位置 |
|---|---|---|
| `concurrent` | 一台机器上**所有 runner 合计**最多同时跑几个 job（默认 1） | `config.toml` 顶部 |
| `limit` | 单个 runner 的上限（默认 0 = 不单独限制） | `[[runners]]` 块内 |
| `request_concurrency` | 同时向 GitLab 要几个活（影响派发速度，不影响执行数） | 全局 |

```
实际并行 = min( 待跑 job 数, Σ各机器槽位 )
每台机器槽位 = min( 该机 concurrent, 该机各 runner 的 limit 之和 )
```

**实测**：`concurrent = 1` 时两个 test job 排队，流水线 **57 秒**；改成 2 后并行，**28 秒**。
**旁证**：某次 test stage 有 3 个 job（槽位只有 2），第三个 `Queued 28 秒 / Duration 0 秒`——排队等槽位的现场证据。

**调大并发的代价**：每个 job 是独立进程，吃 CPU/内存/IO；2 核机器跑 4 个 job 会互相拖慢，甚至 OOM。扩容优先加机器。

## 四、Jenkins 六语法 → GitLab CI 对照

| Jenkinsfile | GitLab CI | 说明 |
|---|---|---|
| `agent { label 'linux' }` | `tags: [linux]` | Runner 注册时打标签，job 用 tags 挑 Runner |
| `agent { docker { image 'maven:3' } }` | `image: maven:3` | 需 docker executor |
| `environment { X = '1' }` | `variables: { X: '1' }` | 全局或 job 级 |
| `when { branch 'main' }` | `rules: - if: '$CI_COMMIT_BRANCH == "main"'` | rules 取代 only/except，还支持 `changes:` |
| `parallel { ... }` | 同 stage 多 job 并行；`parallel: N` 分片；`needs:` 做 DAG | 机制不同 |
| `post { always/success/failure }` | `after_script:` / 独立 job + `when: on_failure` | **GitLab 的短板**，after_script 在新 shell 里跑 |
| `script { sh 'make' }` | `script: [ 'make' ]` | 逐行执行 |

**关键认知**：Jenkins + Jenkinsfile **也是**流水线即代码（PaC 不是 GitLab 的差异点）。真差异是：
1. 表达力：Groovy 程序（图灵完备）vs 声明式 YAML（有限关键字）
2. 系统边界：Jenkins 的 CI 与仓库是两个系统，靠 poll SCM/webhook 桥接；GitLab CI 内生于平台，push 是内部事件
3. 服务端残留配置：两边都有（Jenkins credentials/工具/节点 vs GitLab CI/CD Variables）

## 五、票据管理对照

| | Jenkins | GitLab |
|---|---|---|
| 存储 | Jenkins 服务器 credentials | Settings → CI/CD → Variables |
| 使用 | `withCredentials` 块内可见 | 注入环境变量，整个 job 可见 |
| 安全 | 块级作用域，更窄 | masked（日志打码）+ protected（仅受保护分支可用）+ environment scope |
| 文件类 | file credential | File 类型变量（值是临时文件路径） |

GitLab 变量作用域更宽，泄露面更大，靠 protected + masked 补偿。

## 六、D2 四个实验的实测结果

| 实验 | 做法 | 结果 |
|---|---|---|
| stages 串行/并行 | 3 stage，test 内两个 job 各 sleep 25 | concurrent=1 时 57s，=2 时 28s |
| rules 条件 | `web-only-job` 用 `rules: if: $CI_PIPELINE_SOURCE == "web"` | push 时 4 个 job（不出现），手动跑时出现且日志 `触发来源 = web` |
| artifacts | build 产出 build.txt，test 里 cat | 日志 `Downloading artifacts for build-job`——**经 GitLab 中转** |
| cache | `cache: key/paths`，两次运行对比 | 第一次 `CACHE MISS`、第二次 `CACHE HIT`，5s → 1s；日志 `stored only locally`——**不经过 GitLab** |

**artifacts vs cache 一句话**：artifacts 承载正确性（丢了流水线错），cache 承载性能（丢了只是慢）；artifacts 经 GitLab，cache 存 Runner 本地。

## 七、踩坑记录（都是真踩的）

1. **gitlab.com 验证墙**：中国手机号不在列表；GitHub OAuth 不绕过 → 改自建
2. **Runner token 不能靠截图 OCR**：读错字符 → `Verifying runner... is not valid`，必须复制原文
3. **`type` 是 Windows 写法**：Linux shell executor 里 `type` 是 bash 内建命令，读文件要用 `cat`
4. **YAML 的 `: `（冒号+空格）**：plain scalar 里出现会被解析成 key: value，导致 `script config should be a string` 报错。修法：整行用单引号包起来
5. **`- |` 多行块**：GitLab 19 校验下容易出问题，复杂逻辑写成脚本文件更稳
6. **代理干扰**：git 全局代理指向 127.0.0.1:65532，代理一关连内网 GitLab 都连不上 → `NO_PROXY=192.168.245.150`
7. **受保护分支的 cache key 自动加 `-protected` 后缀**（GitLab 的隔离机制）
8. **`kubeadm join` 没有 `--image-repository` flag**（v1.23 实测），镜像名不符要用 `docker tag` 本地改名复用

## 八、面试话术

> "我们用 Jenkins + Jenkinsfile 落地过流水线即代码，也踩过 DooD、poll SCM 这些细节。GitLab CI 的差异在于 CI 内生于代码平台：push 是内部事件不靠轮询，Runner 注册即用不需要独立 CI 服务器和插件维护。它的执行模型是 pull——Runner 主动向 GitLab 长轮询要活，原子锁保证不重复；并行度由 Runner 的 concurrent 和 limit 决定，实测 concurrent=1 时同 stage 的 job 会排队。artifacts 经 GitLab 中转保证跨机器可达，cache 存在 Runner 本地只做加速。"

## 九、下一步（D3-D5）

- D3：用 devops-demo 复刻完整流水线（build → test → docker build → push 到 registry）
- D4：Jenkins vs GitLab CI 对比文（1000 字）
- D5：周复盘 + 自测题
