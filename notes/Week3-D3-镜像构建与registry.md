# Week3 D3：把镜像构建接进流水线（build → test → package）

> 日期：2026-10-23
> 环境：自建 GitLab（node1 `192.168.245.150`）+ Runner（shell executor，concurrent=2）+ 私有 registry（`registry:2`，端口 5000）
> 项目：`devops/gitlab-ci-lab`（本地 `Claw\2026-09-21-16-30-29\gitlab-ci-lab`）

## 一、D3 要解决的问题

前两天的流水线只在"跑脚本"，没有产物。真实项目的 CI 最终产物通常是**一个可部署的镜像**——部署时运维不拷代码，只 `docker pull`。

一句话：**代码 push → 自动构建镜像 → 推到私有仓库 → 任何机器 pull 下来就能跑。**

## 二、三个核心概念

### 1. Dockerfile = 镜像的配方

```dockerfile
FROM nginx:alpine
COPY app/index.html /usr/share/nginx/html/index.html
```

配方进仓库 → 任何人、任何 CI 都能构建出完全相同的镜像（可复现构建）。

### 2. Registry = 镜像仓库

node1 上跑着 `registry:2`（`:5000`），就是私有 Docker Hub。镜像不能只留在构建机上，部署侧必须能拉到。

### 3. Tag = 版本号

| tag | 例子 | 用途 |
|---|---|---|
| commit sha | `hello:a38117e5` | 每个提交唯一，**部署用这个**，可精确回溯 |
| latest | `hello:latest` | 只表示最新，**不要用于生产部署** |

## 三、流水线结构

```yaml
stages: [build, test, package]

variables:
  REGISTRY: 192.168.245.150:5000
  IMAGE: $REGISTRY/devops/hello

package-job:
  stage: package
  script:
    - docker build -t $IMAGE:$CI_COMMIT_SHORT_SHA -t $IMAGE:latest .
    - docker push $IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE:latest
```

**关键设计点**：

1. **不需要 DooD**：runner 是 shell executor，直接跑在 node1 上，`docker` 命令天然可用。这正是 shell executor 相比 Jenkins 容器化 DooD（挂 socket + 注入 CLI + `--group-add`）省掉的"税"
2. **双 tag**：sha 用于部署与回溯，latest 只做展示
3. **`$CI_COMMIT_SHORT_SHA`** 是 GitLab 预定义变量（对应 Jenkins 的 `env.GIT_COMMIT`）

## 四、实测日志证据（2026-10-23 21:5x）

```
#7 naming to 192.168.245.150:5000/devops/hello:a38117e5 done
#7 naming to 192.168.245.150:5000/devops/hello:latest done
$ docker push $IMAGE:$CI_COMMIT_SHORT_SHA
a38117e5: digest: sha256:32a2047005e1... size: 2196
$ docker push $IMAGE:latest
c86f6810da0b: Layer already exists      ← 同一 digest，层不重复上传
latest: digest: sha256:32a2047005e1... size: 2196
$ curl -s $REGISTRY/v2/_catalog
{"repositories":["devops/hello","devops-demo"]}
$ curl -s $REGISTRY/v2/devops/hello/tags/list
{"name":"devops/hello","tags":["a38117e5","latest"]}
Job succeeded
```

**两个知识点**：
- 两个 tag 指向**同一个 digest** → 镜像内容完全相同，只是名字不同
- 第二次 push 全是 `Layer already exists` → 分层复用的价值

## 五、variables 详解

`variables` 用于定义流水线变量，`script` 里用 `$名字` 引用，GitLab 自动注入为环境变量。

| 位置 | 作用范围 |
|---|---|
| 顶层 `variables:` | 所有 job 可见 |
| job 内 `variables:` | 仅该 job，且覆盖同名全局变量 |

变量可以嵌套引用：`IMAGE: $REGISTRY/devops/hello` → 展开成完整地址。

**与预定义变量的区别**：`$REGISTRY` 是自定义的；`$CI_COMMIT_SHORT_SHA`、`$CI_PIPELINE_SOURCE`、`$CI_COMMIT_BRANCH` 是 GitLab 自动注入的。

**优先级（高 → 低）**：手动/API 触发传入 > 项目 CI/CD Variables（UI）> 组变量 > yml 全局 > yml job 内。

**必须记住的坑**：写在 yml `variables` 里的值会**原样回显在 job 日志**里（script 每行都回显）。所以密码/token 只能放 UI 的 CI/CD Variables，并勾 **Masked**（日志打码）+ **Protected**（仅受保护分支可用）。

## 六、踩坑记录

**registry 换 IP 后 `insecure-registries` 忘了改**：

```json
// /etc/docker/daemon.json —— 迁移前
"insecure-registries": ["192.168.2.150:5000"]   // 旧 IP
```

HTTP registry 必须在 `insecure-registries` 登记，否则 Docker 会尝试 HTTPS 并报
`http: server gave HTTP response to HTTPS client`。

修正为 `192.168.245.150:5000` 后 `systemctl restart docker`，用 `docker pull` 验证通过。
（注意：重启 docker 会重启所有容器，包括 GitLab 本身，需等 1-2 分钟恢复）

## 七、生产流水线 vs 实验室流水线

```
提交/MR → 静态检查 → 单测+覆盖率 → 构建镜像 → 漏洞扫描 → 推 registry(sha tag)
→ 部署 dev → 冒烟 → 部署 staging → 集成/E2E → 人工审批 → 生产金丝雀(5%→50%→100%)
→ 指标验证 → 失败自动回滚 → 通知
```

| 维度 | 实验室 | 生产 |
|---|---|---|
| stage 数 | 3 | 10+ |
| Runner | 1 台 shell executor | Runner 池，按 tags 分池 |
| registry | 匿名可 push | 认证 + 签名 + GC |
| 凭据 | 无 | Vault / protected variables / 短期 token |
| 审批 | 无 | protected environment + 人工卡点 |
| 回滚 | 无 | 一键或自动回滚 |
| 扫描 | 无 | SAST + 依赖 + 镜像 CVE |
| 观测 | 无 | 成功率/时长/DORA 四指标 |

**最小可用生产流水线的 5 条**：
1. sha tag + 制品不可变
2. 单测门禁
3. 镜像高危漏洞阻断（trivy）
4. 受保护分支 + 发布审批
5. 一键回滚

## 八、为什么运维必须掌握 CI/CD（职业认知）

开发"顺手做"的只是 yml 里的业务构建步骤；一条流水线要在生产稳定跑一年，靠的是：

| 层面 | 谁管 |
|---|---|
| 业务构建步骤 | 开发 |
| Runner 部署/扩容/并发/资源 | **运维** |
| 构建环境（基础镜像、依赖源、缓存） | **运维** |
| registry 存储/GC/认证 | **运维** |
| CI 变量与权限隔离 | **运维** |
| 发布与回滚 | **运维** |
| 可观测与安全扫描 | **运维** |

本质变化：**传统运维交付服务器，DevOps 运维交付"从代码到上线的通道"**。交付物变了，技能树必须变。

## 九、技术要点

> "CI 的最终产物是镜像：代码提交后自动 build，按 commit sha 打 tag 推到 registry，部署侧只认 sha tag，latest 只做展示。我做过 Runner 的部署与并发调优、私有 registry 搭建与认证、CI 变量权限隔离和发布回滚，所以既能写流水线，也能保证它在生产上不出事。"

## 十、下一步（D4-D5）

- D4：把 lab 流水线升级成"最小生产版"（sha tag 不可变 + 单测门禁 + trivy 扫描 + 受保护分支 + 回滚）
- D5：Jenkins vs GitLab CI 对比文（1000 字）+ 周复盘 + 自测题
