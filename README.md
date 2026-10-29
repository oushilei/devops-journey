# DevOps 学习记录

一个运维工程师在 **CI/CD + 云原生** 方向上的完整学习记录。

这里不是教程合集，是每天动手做实验、踩坑、复盘之后留下的原始笔记。

---

## 我在做什么

用一条完整的项目链路串起 DevOps 的核心技术栈，而不是孤立地学工具：

```
代码提交 → CI 构建 → 镜像推送 → 容器编排 → 监控告警
   Git      Jenkins      Registry      K8s      Prometheus
```

配套项目仓库：[oushilei/devops-demo](https://github.com/oushilei/devops-demo)（Go HTTP 服务 + 多阶段构建 + 流水线）

---

## 进度

| 阶段 | 主题 | 状态 | 关键产出 |
|---|---|---|---|
| W1 | Docker | ✅ 完成 | 镜像 884MB → 14.1MB（缩小 62.7 倍）；compose 多容器编排；自定义网络服务发现 |
| W2 | Jenkins | ✅ 完成 | DooD 容器内构建；Jenkinsfile 六大语法；凭据管理；参数化 + poll SCM 自动触发 |
| W3 | GitLab CI | ✅ 完成 | 自建 GitLab CE + Runner；stages / rules / artifacts / cache 四实验；镜像构建与私有 registry；四道生产门禁（单测、制品不可变、manual 审批、一键回滚） |
| W4 | K8s 部署 | ⏳ 进行中 | Deployment / Service / 探针 / 滚动更新 / rollout 回滚 |
| W5 | Prometheus | ⬜ 待开始 | 监控栈 + 告警规则 |
| W6 | 全链路整合 | ⬜ 待开始 | 完整 CI/CD + 监控链路 |

---

## 目录

### `notes/` —— 技术笔记

| 文件 | 内容 |
|---|---|
| [Week1-技术要点](notes/Week1-技术要点.md) | 镜像优化 / 故障排查 / 容器网络，三段可复述的技术表述 |
| [Week1-Docker自测8题](notes/Week1-Docker自测8题.md) | 闭卷自测 + 批改 + 标准答案 |
| [复习提纲-1005](notes/复习提纲-1005.md) | W1 复习材料 |
| [Week2-Day2-第一条流水线-原理讲解](notes/Week2-Day2-第一条流水线-原理讲解.md) | DooD 原理：容器里的 Jenkins 如何操作宿主机 Docker |
| [Week2-Day2-复习提纲](notes/Week2-Day2-复习提纲.md) | W2 Day2 自测 + 标准答案 |
| [Week2-Day3-Jenkinsfile语法](notes/Week2-Day3-Jenkinsfile语法.md) | 六大语法点全部实机验证：agent / environment / when / parallel / post / script |
| [Week2-技术要点](notes/Week2-技术要点.md) | 完整 CI 流水线 / DooD / 凭据管理 / 自动触发 |
| [Week3-D1D2-GitLabCI实战与实测数据](notes/Week3-D1D2-GitLabCI实战与实测数据.md) | 自建 GitLab + 并发模型 + 四个实验的实测数据 |
| [Week3-D3-镜像构建与registry](notes/Week3-D3-镜像构建与registry.md) | docker build / push / 双 tag / variables |
| [Week3-D4-最小生产流水线](notes/Week3-D4-最小生产流水线.md) | 四道门禁的实现与实测 |
| [Week3-Jenkins与GitLabCI对比](notes/Week3-Jenkins与GitLabCI对比.md) | 三个真实差异 + 各自短板 + 选型 |
| [Week4-D3-回滚与ReplicaSet](notes/Week4-D3-回滚与ReplicaSet.md) | rollout undo / --to-revision / RS 编号重排机制 |
| [K8s实验自测与要点](notes/K8s实验自测与要点.md) | 主节点删除与重新加入实验记录 |
| [Git常用命令速查](notes/Git常用命令速查.md) | 日常命令 + 排错速查 |

### `k8s/` —— 集群清单

kube-flannel manifest 归档（10-19 NAT 迁移后集群重建用）。

### `assets/` —— 图示

CI/CD 与容器化部署架构图。

### `weekly/` —— 周总结

按周沉淀「做了什么 / 遇到什么问题 / 怎么解决的」。

---

## 两个值得看的坑

这些不是抄来的，是我实际踩过的。

### 1. 参数在 shell 里取不到值

Jenkins 里 `parameters` 定义的参数，在 `sh` 步骤里**首次运行可能为空** —— 因为 job 还没被注册为参数化任务，环境变量尚未注入。

```
Invalid argument "devops-demo:" for "-t, --tag" flag: invalid reference format
```

**解法**：所有参数在 `environment` 里显式映射一次，并用 Elvis 运算符兜底。

```groovy
environment {
    IMAGE_TAG = "${params.TAG ?: 'v1'}"
}
```

**规则**：shell 里出现的一切变量，必须先在 `environment` 里定义过。

### 2. 轮询成功 ≠ 构建能拉下来

同一台机器、同一个仓库、同一分钟内：

| 操作 | 数据量 | 结果 |
|---|---|---|
| `git ls-remote`（轮询用） | 几百字节 | ✅ 0.97 秒完成 |
| `git fetch`（构建用） | 几 MB | ❌ `GnuTLS recv error (-110)` |

**结论：小数据量查询能通，不代表大数据量传输能通。** 判断网络问题时，"上次能这次不能"比"能 ping 通"更有参考价值。

---

## 方法论

学每个模块都问三个问题，答不上来就是没学会：

1. **我做了什么操作？**（命令级别，能复现）
2. **遇到了什么问题、怎么解决的？**（现象 → 排查 → 根因 → 解法）
3. **在真实工作里解决什么问题？**（价值翻译）

---

## 联系

GitHub：[@oushilei](https://github.com/oushilei)
