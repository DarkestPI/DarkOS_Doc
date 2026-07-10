

# 常用命令

## 暂存更改（Stash）

```bash
# 暂存所有未提交的更改（包括暂存区和工作区）
git stash push -m "描述信息"

# 只暂存未跟踪的新文件
git stash push -u -m "描述信息"

# 查看暂存列表
git stash list

# 恢复最近一次暂存（并删除该暂存）
git stash pop

# 恢复指定暂存
git stash apply stash@{0}

# 删除指定暂存
git stash drop stash@{0}
```

## 变基拉取（Rebase Pull）

```bash
# 变基拉取（推荐用于特性分支）
git pull --rebase origin main

# 或者先 fetch 再 rebase
git fetch origin
git rebase origin/main
```

**注意**：如果变基过程中出现冲突，解决后执行：

```bash
git add .
git rebase --continue
# 或者跳过当前提交
git rebase --skip
# 或者中止变基
git rebase --abort
```

## 合并代码（Merge）

```bash
# 合并特性分支（产生合并提交）
git merge feature-branch

# 快进合并（如果无分叉）
git merge --ff-only feature-branch

# 禁用快进，强制生成合并提交
git merge --no-ff feature-branch
```

# 工作场景

## 变基拉取合并代码

```bash
# 1. 暂存本地修改
git stash push -m "暂存"

# 2. 变基拉取最新代码
git pull --rebase origin main

# 3. 恢复本地修改
git stash pop

# 4. 解决可能的冲突后提交
git add .
git commit -m "feat: 添加CAN FD 2M速率测试"

# 5. 推送到远程（若之前已推送过该分支，需强制推送）
git push --force-with-lease origin feature-branch
```

## 冲突解决过程

### 冲突
```bash
warning: Cannot merge binary files: zs_work/lib/libptzctrl.so (Updated upstream vs. Stashed changes)
Auto-merging zs_work/lib/libptzctrl.so
CONFLICT (content): Merge conflict in zs_work/lib/libptzctrl.so
On branch nlj
Your branch is ahead of 'origin/nlj' by 10 commits.
  (use "git push" to publish your local commits)

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   zs_work/module/ptzctrl/ptz_base.hpp
        modified:   zs_work/module/ptzctrl/ptz_preset_cruise.cpp
        modified:   zs_work/module/ptzctrl/ptz_preset_cruise.hpp

Unmerged paths:
  (use "git restore --staged <file>..." to unstage)
  (use "git add <file>..." to mark resolution)
        both modified:   zs_work/lib/libptzctrl.so

The stash entry is kept in case you need it again.
```

#### 方案 1：保留本地版本（重新编译你自己的）
```bash
# 强制使用 stash 中的版本（你本地的）
git checkout --theirs zs_work/lib/libptzctrl.so
git add zs_work/lib/libptzctrl.so
```

#### 方案 2：保留上游版本（同事的最新编译）

```bash
# 强制使用上游版本
git checkout --ours zs_work/lib/libptzctrl.so
git add zs_work/lib/libptzctrl.so
```
#### 方案 3：两个都不要，重新编译（推荐）

```bash
# 删除冲突标记，不保留任何一方的 .so
git rm -f zs_work/lib/libptzctrl.so
# 然后重新编译项目，生成新的 libptzctrl.so
make  # 或你的编译命令
git add zs_work/lib/libptzctrl.so
```

## 清理 stash 并完成

```bash
# 确认冲突解决后，删除 stash（因为 pop 失败保留了）
git stash drop

# 查看状态确认干净
git status

# 提交合并结果（如果处于 rebase 中，继续）
git rebase --continue
# 或者普通提交
git commit -m "resolve: merge libptzctrl.so binary conflict"
```