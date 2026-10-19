# Git 常用命令速查（含今日实操解释）

> 建立于 2026-10-05，对应 `devops-demo` 首次推送到 GitHub 的全过程。
> 环境：CentOS 7 / git 1.8.3.1 / 用户 root / 项目路径 `/root/devops-demo`

---

## 零、先分清两个概念

| | 是什么 | 在哪 | 能否离线 |
|---|---|---|---|
| **git** | 版本控制**工具**（软件） | 装在你机器上 | ✅ 能 |
| **GitHub** | 托管 git 仓库的**网站**（服务） | 云端 | ❌ 不能 |

git 2005 年就有，GitHub 2008 年才有。**今天敲的 12 条命令里只有 2 条真正联网。**

---

## 一、文件的四段旅程

```
工作区               暂存区              本地仓库            远程仓库
你编辑的文件   ──▶   待提交清单   ──▶   .git 目录   ──▶   GitHub
             add              commit           push
```

| 阶段 | 位置 | 说明 |
|---|---|---|
| 工作区 | `/root/devops-demo/` | 你实际编辑文件的地方 |
| 暂存区 | `.git/index` | "这次要提交哪些文件"的清单 |
| 本地仓库 | `.git/` | 所有历史版本都存在这里 |
| 远程仓库 | GitHub | 云端，别人能看到 |

**`git init` 之前，`devops-demo` 只是个普通文件夹；之后它变成了"被 git 跟踪的仓库"。**
删掉 `.git` 目录 = 删掉全部历史（但文件还在）。

### 表 1 · 四个阶段的属性对照

| 阶段 | 文件存在哪里 | 别人能看到吗 | 进入这个阶段的命令 |
|---|---|---|---|
| 工作区 | `/root/devops-demo/` | 不能 | `git init` |
| 暂存区 | 仓库内 `.git/index` | 不能 | `git add .` |
| 本地仓库 | 仓库内 `.git/` | 不能 | `git commit -m ""` |
| 远程仓库 | GitHub 服务器上 | **能**（公开仓库） | `git push` |

> **看最后一列：前三行全是"不能"。**
> 工作区、暂存区、本地仓库**全都在你自己的机器上**，只有第四阶段在云端。
> 所以 `add` 和 `commit` 折腾半天，外面一无所知；只有 `push` 那一下别人才第一次看到。
> 这也是为什么断网也能 commit。

### 表 2 · 命令在阶段之间的读写关系

| 命令 | 工作区 | 暂存区 | 本地仓库 | 远程仓库 |
|---|---|---|---|---|
| `git init` | — | — | **创建** | — |
| `git add .` | 读取 | **写入** | — | — |
| `git commit` | — | 读取 | **写入** | — |
| `git push` | — | — | 读取 | **写入** |
| `git pull` | **写入** | — | **写入** | 读取 |

**读这张表的方法**：每一行只有 2 个格子有内容 —— 每条命令的本质就是
**在两个阶段之间搬运数据**。"—" 表示这条命令根本不碰那个阶段。

看清这一点，git 就不再是"一堆要背的命令"，而是"数据在四个区域之间流动"。

**顺带回答一个常见疑问：为什么 commit 之前必须先 add？**
因为 `git commit` 只从**暂存区**读取，它不直接读工作区。
不 add 就没有东西进暂存区，commit 自然没内容可提交。

---

## 二、今日 12 条命令逐条解释

### 第 1 组 · 告诉 git 你是谁

```bash
git config --global user.name "oushilei"
git config --global user.email "ou1879099080@gmail.com"
```

- `config` = 配置
- `--global` = 全局，对本机**所有**仓库生效（不加则只对当前仓库）
- **作用**：每次 `commit` 都会把这两个信息写进提交记录，GitHub 靠它认人
- **不联网、不动文件**，只是写进 `~/.gitconfig`

```bash
git config --global --list
```
把当前所有配置打印出来，用来验证。

---

### 第 2 组 · SSH 免密认证

```bash
ssh-keygen -t ed25519 -C "ou1879099080@gmail.com"
```

- `ssh-keygen` = 生成 SSH 密钥对
- `-t ed25519` = 类型选 ed25519（现代算法，比 rsa 更短更安全）
- `-C` = 注释，通常写邮箱，**只是给人看的标签**
- 生成两个文件：
  - `~/.ssh/id_ed25519` → **私钥，永远不能外传**
  - `~/.ssh/id_ed25519.pub` → 公钥，可以随便公开

```bash
cat ~/.ssh/id_ed25519.pub
```
`cat` = 打印文件内容。把这一整行贴到 GitHub，GitHub 就能认出你。

```bash
ssh -T git@github.com
```
- `-T` = 不分配终端（只是测连接，不需要交互 shell）
- 用来验证认证配置是否成功

> **为什么用 SSH 不用密码**：GitHub 从 2021 年起不再接受密码推送。
> 而且 SSH 配一次，以后永久免密。

---

### 第 3 组 · 建立本地仓库

```bash
git init
```
- `init` = initialize（初始化）
- 在**当前目录**创建 `.git` 隐藏目录 —— 这个目录就是"本地仓库"
- 只做一次，之后不用再跑

```bash
git status
```
查看当前状态：哪些文件被改过、哪些已放进暂存区。
**这是以后用得最多的命令**，不确定就先跑它。

---

### 第 4 组 · 提交

```bash
git add .
```
- `add` = 把文件放进**暂存区**
- `.` = 当前目录下所有文件
- 可以理解为"勾选要提交的文件"
- **为什么要有这一步**：让你能精确控制这次提交包含什么（改了 3 个文件也可以只提交 1 个）

```bash
git commit -m "feat: Go HTTP 服务容器化，多阶段构建 884MB → 14.1MB"
```
- `commit` = 把暂存区内容打包成一个**快照**存进本地仓库
- `-m` = message（提交说明）
- **这一步只在本地发生，不联网**
- 每个 commit 有一个哈希值（本次是 `9758ae6`）

---

### 第 5 组 · 关联远程仓库

```bash
git branch -m main
```
- `branch` = 分支；`-m` = move / rename
- 把当前分支从 `master` 改名为 `main`
- **为什么**：git 1.8 默认叫 `master`，GitHub 现在默认叫 `main`，统一一下
- ⚠️ git 1.8.3.1 只支持小写 `-m`，大写的 `-M` 是 git 2.30+ 才有

```bash
git remote add origin git@github.com:oushilei/devops-demo.git
```
- `remote` = 远程仓库；`add` = 添加一个
- `origin` = 给这个远程地址起的**别名**（约定俗成的名字，也可以叫别的）
- **这条命令只是在本地的 `.git/config` 里记了一行地址，不联网**

---

### 第 6 组 · 推送

```bash
git push -u origin main
```
- `push` = 把本地提交推送到远程
- `origin` = 推到哪个远程（第 5 组起的别名）
- `main` = 推哪个分支
- `-u` = 建立跟踪关系。**建立之后，以后只用 `git push` 就行**，不用再写 `origin main`

---

## 三、关键认知：只有 2 条命令联网

| 命令 | 联网 | 作用 |
|---|---|---|
| `ssh -T git@github.com` | ✅ | 测试 SSH 认证 |
| `git push` | ✅ | 上传到 GitHub |
| 其余 10 条 | ❌ | 全在本地完成 |

所以：**git 没网也能用**。你可以在飞机上 commit 二十次，落地再 push。

---

## 四、以后日常只用 3 条

```bash
cd /root/devops-demo
git add .
git commit -m "说明这次改了什么"
git push
```

这个循环会重复几百次。**commit message 要认真写**——面试官会看你的提交历史。

### commit message 的约定写法

```
feat:  新功能
fix:   修 bug
docs:  只改文档
chore: 杂项（配置、依赖）
```

例：`fix: 修复 nginx 容器因内核不兼容无法启动的问题`

---

## 五、其他会用到但今天没敲的

| 命令 | 作用 |
|---|---|
| `git log --oneline` | 看提交历史（一行一条） |
| `git diff` | 看改了什么（还没 add 的） |
| `git diff --staged` | 看已放进暂存区的改动 |
| `git remote -v` | 看当前关联了哪些远程仓库 |
| `git clone <地址>` | 把别人的仓库整个下载下来 |
| `git pull` | 把远程的新提交拉到本地 |
| `git checkout -- <文件>` | 撤销某个文件的修改 |

---

## 六、踩过的坑

| 现象 | 原因 | 解决 |
|---|---|---|
| `git: command not found` | CentOS 7 未预装 git | `sudo yum install -y git` |
| `git branch -M main` 报错 | git 1.8.3.1 不支持 `-M` | 改用小写 `git branch -m main` |
| `git push` 报 `rejected - fetch first` | GitHub 建仓库时勾了初始化文件，远程已有提交 | 先 `git pull --rebase origin main` 再 push |
| push 提示要输密码 | SSH 密钥没配好 | 检查公钥是否贴到 GitHub，用 `ssh -T git@github.com` 验证 |
| 提示 `| 0 bytes/s` | git 1.8 的显示缺陷（传输太快算不出速率） | 无害，忽略 |
