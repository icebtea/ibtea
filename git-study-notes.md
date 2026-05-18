# Git 学习笔记

## 一、初始配置

```bash
# 设置用户名和邮箱（全局配置，只需一次）
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"

# 查看当前配置
git config --list
```

## 二、SSH 密钥配置（连接 GitHub）

```bash
# 1. 生成 SSH 密钥
ssh-keygen -t ed25519 -C "你的邮箱"
# 一路回车使用默认设置

# 2. 查看公钥，复制到 GitHub → Settings → SSH and GPG keys → New SSH key
cat ~/.ssh/id_ed25519.pub

# 3. 测试连接
ssh -T git@github.com
# 成功会显示：Hi 用户名! You've successfully authenticated...
```

## 三、仓库操作

### 克隆仓库

```bash
# SSH 方式（推荐，配置好密钥后无需重复输入密码）
git clone git@github.com:用户名/仓库名.git

# HTTPS 方式（简单，但每次推送需要登录）
git clone https://github.com/用户名/仓库名.git
```

### 创建新仓库

```bash
# 方式一：在本地初始化后关联远程
mkdir 项目名 && cd 项目名
git init
git remote add origin git@github.com:用户名/仓库名.git

# 方式二：直接克隆 GitHub 上已创建的空仓库
git clone git@github.com:用户名/仓库名.git
```

## 四、日常工作流（核心）

```
编辑文件 → git status → git add → git commit → git push
```

### 4.1 查看状态

```bash
git status              # 查看哪些文件被修改/新增/删除
git diff                # 查看未暂存的具体改动
git diff --cached       # 查看已暂存但未提交的改动
git log --oneline -10   # 查看最近10条提交记录
```

### 4.2 暂存文件

```bash
git add 文件名          # 暂存指定文件（推荐）
git add file1.md file2.py  # 暂存多个文件
git add .               # 暂存所有改动（注意不要误提交不需要的文件）
```

### 4.3 提交

```bash
git commit -m "提交说明"
```

### 4.4 推送和拉取

```bash
git push                # 推送到远程仓库
git pull                # 拉取远程最新代码并合并
```

## 五、分支管理

```bash
# 查看分支
git branch              # 查看本地分支
git branch -a           # 查看所有分支（含远程）

# 创建和切换
git branch 分支名       # 创建新分支
git checkout 分支名     # 切换到分支
git checkout -b 分支名  # 创建并切换（合二为一）

# 合并分支
git checkout main       # 先切回主分支
git merge 分支名        # 把分支合并到当前分支

# 删除分支
git branch -d 分支名    # 删除已合并的本地分支
```

## 六、撤销和回退

```bash
# 撤销工作区的修改（未 add）
git checkout -- 文件名

# 撤销暂存（已 add，未 commit）
git reset HEAD 文件名

# 回退到上一个提交（已 commit）
git reset --soft HEAD~1    # 保留改动在暂存区
git reset --hard HEAD~1    # 丢弃所有改动（危险！）

# 查看操作历史（用于找回误删的提交）
git reflog
```

## 七、删除文件

```bash
# 方式一：用 git rm 一步完成删除+暂存（推荐）
git rm 文件名
git commit -m "删除 文件名"
git push

# 方式二：先 rm 删除，再 git add 暂存删除操作
rm 文件名
git add 文件名
git commit -m "删除 文件名"
git push

# 只从 Git 追踪中移除，保留本地文件（适合误提交敏感文件后清理）
git rm --cached 文件名
git commit -m "从仓库中移除 文件名"
git push
```

> 删除文件后必须 commit + push 才能同步到远程仓库，流程和普通修改一样。

## 八、.gitignore 文件

在仓库根目录创建 `.gitignore` 文件，列出不需要提交的文件：

```gitignore
# 编译产物
*.pyc
*.o
__pycache__/

# 环境和配置
.env
venv/
node_modules/

# IDE 文件
.vscode/
.idea/

# 系统文件
.DS_Store
Thumbs.db
```

## 八、最佳实践

### 提交规范

1. **一个 commit 只做一件事** — 不同类型的改动分开提交
2. **commit message 要有意义** — 写清楚做了什么：
   - 好：`修复用户登录时密码校验失败的问题`
   - 差：`修改代码`、`更新`、`fix bug`
3. **用 `git add 指定文件`**，少用 `git add .`，避免误提交

### 安全习惯

4. **推送前先拉取** — `git pull` → `git push`，避免冲突

   > **为什么先 pull 不会覆盖本地修改？**
   >
   > `git pull` 不是用远程覆盖本地，而是**合并**远程和本地的改动：
   > - 没有冲突 → 自动合并，本地修改完好保留
   > - 有冲突（别人改了你正在改的同一行）→ Git 标记冲突，手动解决后再 push
   >
   > **什么时候需要先 pull？** 主要是多人协作场景——别人在你开发期间推送了新代码，你直接 push 会被拒绝，必须先 pull 合并。
   >
   > **如果只有你一个人用这个仓库？** 直接 push 就行，不需要先 pull。
5. **推送前检查暂存区** — `git diff --cached` 确认改动正确
6. **不要提交敏感信息** — 密码、密钥、token 等放入 `.gitignore`
7. **不要 force push 到 main** — `git push --force` 会覆盖他人的提交

### 分支策略

8. **不在 main 上直接开发** — 新功能开分支开发，完成后合并
9. **分支命名要有意义** — `feature/登录功能`、`fix/密码校验`、`docs/更新说明`

### 推荐的完整工作流

```bash
# 开始新功能
git checkout -b feature/新功能

# 开发过程中，频繁提交
git add 具体文件
git commit -m "实现XX功能"

# 开发完成，合并回主分支
git checkout main
git pull                    # 先拉取最新代码
git merge feature/新功能    # 合并

# 解决冲突（如果有）
# 打开冲突文件，手动选择保留的内容，然后：
git add 冲突文件
git commit -m "解决合并冲突"

# 推送到远程
git push

# 清理已合并的分支
git branch -d feature/新功能
```

## 九、常用速查表

| 场景 | 命令 |
|------|------|
| 克隆仓库 | `git clone 地址` |
| 查看状态 | `git status` |
| 查看改动 | `git diff` |
| 暂存文件 | `git add 文件名` |
| 提交 | `git commit -m "说明"` |
| 推送 | `git push` |
| 拉取 | `git pull` |
| 新建分支 | `git checkout -b 分支名` |
| 合并分支 | `git merge 分支名` |
| 查看日志 | `git log --oneline` |
| 撤销修改 | `git checkout -- 文件名` |
| 回退提交 | `git reset --soft HEAD~1` |
