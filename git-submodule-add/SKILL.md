---
name: git-submodule-add
description: >
  Add a GitHub repository as a Git submodule under the ref/ directory.
  Handles submodule initialization, URL normalization, and cleanup. Use
  this skill when the user wants to add a submodule, reference an external
  repo, manage submodules, or mentions "添加子模块", "add submodule",
  "引用仓库", "reference repo", "放到 ref 下". Also covers submodule
  update, initialization, and removal.
compatibility: Requires git installed, GitHub access.
metadata:
  author: skillshub
  version: "1.0"
---

# 添加 GitHub 仓库为子模块

将 GitHub 仓库添加为 Git 子模块，统一存放在 `ref/` 目录下，方便管理外部依赖。

## 目录约定

```
root/
├── ref/                 # 所有子模块统一入口
│   ├── project-a/       # git submodule
│   ├── project-b/       # git submodule
│   └── ...
```

---

## 检查清单

- [ ] Step 1: 确认仓库 URL 和分支
- [ ] Step 2: `git submodule add` 到 `ref/`
- [ ] Step 3: 验证 `.gitmodules` 和提交状态
- [ ] Step 4: 提交变更

---

## Step 1: 确认仓库 URL

向用户确认：
1. **仓库完整 URL**（如 `https://github.com/org/repo`）
2. **目标分支**（默认 `main`）
3. **子模块目录名**（默认取仓库名，如有冲突需重命名）

URL 格式要求：
- 必须用 `https://` 前缀（兼容性好，不依赖 SSH key）
- 格式：`https://github.com/<owner>/<repo>`

---

## Step 2: 添加子模块

```bash
# 确保 ref/ 目录存在
mkdir -p ref

# 添加子模块
git submodule add https://github.com/<owner>/<repo> ref/<repo>
```

**指定分支**（如需）：

```bash
git submodule add -b <branch> https://github.com/<owner>/<repo> ref/<repo>
```

**重命名目录**（避免与已有子模块冲突）：

```bash
git submodule add https://github.com/<owner>/<repo> ref/<custom-name>
```

---

## Step 3: 验证

```bash
# 查看子模块状态
git submodule status

# 确认 .gitmodules 内容
cat .gitmodules

# 确认工作区状态
git status
```

`.gitmodules` 示例：

```
[submodule "ref/repo-name"]
	path = ref/repo-name
	url = https://github.com/owner/repo-name
[submodule "ref/another"]
	path = ref/another
	url = https://github.com/owner/another
```

---

## Step 4: 提交

```bash
git add ref/<repo> .gitmodules
git commit -m "chore: add submodule ref/<repo>"
```

> `.gitmodules` 和子模块目录本身都需要提交。

---

## 常见操作

### 克隆含子模块的项目

```bash
git clone --recurse-submodules <project-url>
```

或已克隆后初始化：

```bash
git submodule update --init --recursive
```

### 更新子模块到最新

```bash
cd ref/<repo>
git pull origin main
cd ../..
git add ref/<repo>
git commit -m "chore: update submodule ref/<repo>"
```

或一次性更新所有：

```bash
git submodule update --remote
```

### 切换子模块分支

```bash
cd ref/<repo>
git checkout <branch>
cd ../..
git add ref/<repo>
git commit -m "chore: switch ref/<repo> to <branch>"
```

### 移除子模块

```bash
# 1. 删除 .gitmodules 中的条目
git submodule deinit -f ref/<repo>

# 2. 删除工作区目录
rm -rf ref/<repo>

# 3. 清理 .git/modules/ 中的缓存
rm -rf .git/modules/ref/<repo>

# 4. 提交
git add .gitmodules
git commit -m "chore: remove submodule ref/<repo>"
```

---

## Gotchas

- 子模块 URL 必须用 `https://`，不能用 SSH（`git@github.com:`），否则没有配置 SSH key 的人 clone 会失败
- `git submodule add` 前确保项目已 `git init`，否则会报 `fatal: not a git repository`
- 子模块目录名冲突时 `git submodule add` 会失败，需要手动指定不同的目标路径
- 子模块加载后，`ref/<repo>` 内是一个独立的 git 仓库（有自己的 `.git`）
- 主仓库只记录子模块的 commit hash，不记录分支名。切换子模块分支后需显式 commit
- `.gitmodules` 文件必须提交到主仓库，否则其他协作者无法初始化子模块
- 子模块目录本身 `.gitmodules` 变更后需要 `git add` 和 `git commit`
- 移除子模块的三个步骤缺一不可，尤其是 `.git/modules/ref/<repo>` 清理，否则重新 add 同路径会失败
