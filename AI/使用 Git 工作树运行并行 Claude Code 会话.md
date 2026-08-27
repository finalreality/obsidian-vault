---
title: "使用 Git 工作树运行并行 Claude Code 会话_use git worktrees to run multi"
source: "https://blog.csdn.net/polanpan/article/details/150959595"
author:
  - "[[破烂pan]]"
published: 2025-08-28
created: 2026-08-27
description: "文章浏览阅读2.1k次，点赞26次，收藏18次。摘要：使用 Git 工作树实现并行开发 Git 工作树（Worktree）允许在同一个仓库中创建多个独立的工作目录，每个目录可检出不同分支，实现并行开发。 核心操作： 创建：git worktree add <路径> [分支] 管理：git worktree list 查看，git worktree remove 删除 隔离性：各工作树文件状态独立，适合同时运行多个 Claude Code 会话 优势： 共享 Git 历史但隔离文件修改 无需克隆多份仓库，节省空间 支持锁定、移动等高级管理 典型_use git worktrees to run multiple claude sessions in parallel."
tags:
  - "clippings"
---
假设您需要同时处理多个任务，在 Claude Code 实例之间完全隔离代码。

### 1 了解 Git 工作树

Git 工作树允许您将同一存储库的多个分支检出到单独的目录中。每个工作树都有自己的工作目录和隔离的文件，同时共享相同的 Git 历史。在 [官方 Git 工作树文档](https://git-scm.com/docs/git-worktree) 中了解更多。

> Git 工作树相关的教程在文章下方

### 2 创建新的工作树

```bash
# 使用新分支创建新工作树
git worktree add ../project-feature-a -b feature-a

# 或使用现有分支创建工作树
git worktree add ../project-bugfix bugfix-123
bash
```

这创建一个新目录，其中包含 存储 库的单独工作副本。

### 3 在每个工作树中运行 Claude Code

```bash
# 导航到您的工作树
cd ../project-feature-a

# 在这个隔离环境中运行 Claude Code
claude
bash
```

### 4 在另一个工作树中运行 Claude

```bash
cd ../project-bugfix
claude
bash
```

### 5 管理您的工作树

```bash
# 列出所有工作树
git worktree list

# 完成后删除工作树
git worktree remove ../project-feature-a
bash
```

> 提示：
> 
> - 每个工作树都有自己独立的文件状态，使其非常适合并行 Claude Code 会话
> - 在一个工作树中所做的更改不会影响其他工作树，防止 Claude 实例相互干扰
> - 所有工作树共享相同的 Git 历史和远程连接
> - 对于长时间运行的任务，您可以让 Claude 在一个工作树中工作，同时在另一个工作树中继续开发
> - 使用描述性目录名称轻松识别每个工作树用于哪个任务
> - 记住根据项目的设置在每个新工作树中初始化您的开发环境。根据您的技术栈，这可能包括：
> 	- JavaScript 项目：运行依赖项安装（ `npm install` 、 `yarn` ）
> 		- Python 项目：设置虚拟环境或使用包管理器安装
> 		- 其他语言：遵循项目的标准设置过程

## 附：

以下是关于 **Git 工作树（Worktree）** 的从入门到精通的详细教程文档，基于官方文档（https://git-scm.com/ docs /git-worktree）整理，结合实际使用场景和示例。

---

### Git 工作树（Worktree）教程

#### 目录

1. **概述**
2. **安装与配置**
3. **基本操作**
	- 创建工作树
		- 列出工作树
		- 删除工作树
4. **高级操作**
	- 锁定与解锁工作树
		- 移动工作树
		- 修复工作树
5. **最佳实践**
6. **常见问题与解决方案**
7. **附录：命令参考**

---

### 1\. 概述

Git 工作树（Worktree）允许一个 Git 仓库（repository）同时关联多个工作目录（working directory）。每个工作目录可以独立地检出不同的分支或提交，从而实现 **多分支并行开发** 的需求。这对于需要同时处理多个功能、修复或实验性开发的场景非常有用。

#### 核心概念

- **主工作树（Main Worktree）** ：通过 `git init` 或 `git clone` 创建的默认工作目录。
- **关联工作树（Linked Worktree）** ：通过 `git worktree add` 创建的额外工作目录，共享主仓库的 Git 数据（`.git` 目录）。
- **裸仓库（Bare Repository）** ：没有工作目录的仓库，仅包含 Git 数据。

---

### 2\. 安装与配置

#### Git 版本要求

Git 2.5+（2014 年发布）支持工作树功能。可通过以下命令检查版本：

```bash
git --version
bash1
```

#### 配置选项

- **启用工作树配置** ：
	```bash
	git config extensions.worktreeConfig true
	bash1
	```
	启用后，每个工作树可以拥有独立的配置文件（`.git/worktrees/<name>/config.worktree` ）。
- **设置默认远程仓库** ：
	```bash
	git config checkout.defaultRemote origin
	bash1
	```
	在创建新分支时，优先从 `origin` 远程拉取。

---

### 3\. 基本操作

#### 3.1 创建工作树

##### 语法

```bash
git worktree add [-f] <path> [<commit-ish>]
bash1
```

##### 示例

1. **创建新分支的工作树** ：
	```bash
	git worktree add ../feature-1 feature-1
	bash1
	```
	- 在 `../feature-1` 路径下创建名为 `feature-1` 的工作树，并检出该分支。
2. **创建临时工作树（脱离 HEAD）** ：
	```bash
	git worktree add -d ../hotfix
	bash1
	```
	- 创建一个脱离 HEAD 的工作树，用于临时修改（如紧急修复）。
3. **强制覆盖现有路径** ：
	```bash
	git worktree add -f ../feature-1 feature-1
	bash1
	```
	- 如果 `../feature-1` 已存在，强制覆盖。

##### 参数说明

- `-f` ：强制创建，覆盖已存在的路径。
- `-d` ：创建脱离 HEAD 的工作树。
- `<commit-ish>` ：可指定提交哈希、分支名或标签。若省略，默认使用当前分支。

---

#### 3.2 列出工作树

##### 语法

```bash
git worktree list [-v | --porcelain]
bash1
```

##### 示例

1. **默认输出** ：
	```bash
	git worktree list
	bash1
	```
	输出示例：
	```
	/path/to/repo (main)
	/path/to/feature-1  abc1234 [feature-1]
	/path/to/hotfix    def5678 (detached HEAD)
	```
2. **详细输出（含状态）** ：
	```bash
	git worktree list --verbose
	bash1
	```
	输出示例：
	```
	/path/to/feature-1  abc1234 [feature-1]
	/path/to/locked-worktree  def5678 (brancha) locked: 暂存开发环境
	/path/to/prunable-worktree  1234abc (detached HEAD) prunable
	```
3. **机器可读格式（Porcelain）** ：
	```bash
	git worktree list --porcelain
	bash1
	```
	输出示例：
	```
	worktree /path/to/repo
	bare
	worktree /path/to/feature-1
	HEAD abc1234abcd1234abcd1234
	branch refs/heads/feature-1
	123456
	```

---

#### 3.3 删除工作树

##### 语法

```bash
git worktree remove <path> [-f]
bash1
```

##### 示例

1. **删除干净的工作树** ：
	```bash
	git worktree remove ../feature-1
	bash1
	```
	- 要求工作树中没有未提交的修改或未跟踪文件。
2. **强制删除** ：
	```bash
	git worktree remove -f ../feature-1
	bash1
	```
	- 即使工作树不干净，也强制删除。

---

### 4\. 高级操作

#### 4.1 锁定与解锁工作树

##### 锁定工作树

防止工作树被自动清理（如 `git gc` ）：

```bash
git worktree lock --reason "暂存开发环境" ../feature-1
bash1
```

##### 解锁工作树

```bash
git worktree unlock ../feature-1
bash1
```

---

#### 4.2 移动工作树

##### 语法

```bash
git worktree move <worktree> <new-path>
bash1
```

##### 示例

```bash
git worktree move ../feature-1 /new/path/feature-1
bash1
```

##### 注意事项

- 主工作树不能被移动。
- 若目标路径已存在，需使用 `-f` 强制覆盖。

---

#### 4.3 修复工作树

当工作树路径被手动移动或损坏时，使用 `git worktree repair` 修复 链接 关系。

##### 示例

```bash
git worktree repair /new/path/feature-1
bash1
```

---

### 5\. 最佳实践

#### 场景 1：并行开发与紧急修复

- **需求** ：在开发 `feature-1` 时，需要紧急修复生产环境问题。
- **操作** ：
	```bash
	# 创建临时工作树处理紧急修复
	git worktree add -b hotfix ../hotfix main
	# 在临时工作树中完成修复并提交
	cd ../hotfix
	# ... 修改代码 ...
	git commit -am "Fix critical bug"
	# 删除临时工作树
	git worktree remove ../hotfix
	bash12345678910
	```

#### 场景 2：多分支测试

- **需求** ：同时测试 `feature-a` 、 `feature-b` 和 `main` 分支。
- **操作** ：
	```bash
	git worktree add ../test-feature-a feature-a
	git worktree add ../test-feature-b feature-b
	git worktree add ../test-main main
	bash
	```

#### 场景 3：隔离实验性开发

- **需求** ：尝试新功能但不想污染当前分支。
- **操作** ：
	```bash
	git worktree add -d ../experiment
	cd ../experiment
	# ... 实验性开发 ...
	bash
	```

---

### 6\. 常见问题与解决方案

#### Q1: 工作树被删除后如何清理？

- **问题** ：手动删除工作树后，Git 会保留元数据，导致 `git worktree list` 显示“prunable”。
- **解决方案** ：
	```bash
	git worktree prune
	bash1
	```

#### Q2: 如何修复损坏的链接？

- **问题** ：手动移动工作树后，Git 无法识别新路径。
- **解决方案** ：
	```bash
	git worktree repair /new/path/to/worktree
	bash1
	```

#### Q3: 如何配置相对路径？

- **问题** ：工作树路径使用绝对路径，导致迁移后失效。
- **解决方案** ：
	```bash
	git config worktree.useRelativePaths true
	bash1
	```

---

### 7\. 附录：命令参考

| 命令 | 说明 |
| --- | --- |
| `git worktree add <path> <commit-ish>` | 创建新工作树 |
| `git worktree list` | 列出所有工作树 |
| `git worktree remove <path>` | 删除工作树 |
| `git worktree lock <path>` | 锁定工作树 |
| `git worktree unlock <path>` | 解锁工作树 |
| `git worktree move <path> <new-path>` | 移动工作树 |
| `git worktree repair <path>` | 修复工作树 |
| `git worktree prune` | 清理无效工作树 |

---

### 总结

Git 工作树是管理多分支开发的强大工具，尤其适合需要并行处理多个任务的场景。通过合理使用 `git worktree` ，可以避免频繁 切换 分支的麻烦，提升开发效率。掌握其基本操作和高级技巧后，能够显著优化开发流程。