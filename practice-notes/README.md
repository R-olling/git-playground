# Git 练习踩坑笔记

> 按《git-guide-and-practice》练习过程中遇到的问题与解决方案汇总。

## 0. 最重要的环境坑：PowerShell 写文件的编码问题

Windows PowerShell 里 `echo "内容" > 文件` / `echo "内容" >> 文件` 默认按 **UTF-16** 写入，每个字符后带一个空字节（NUL）。Git 检测到 NUL 字节就会把文件判定为**二进制**，症状是：

- `git diff` / `git diff --staged` 显示 `Binary files ... differ`
- `git commit` 显示 `0 insertions(+)`

**解决：写文件一律带 `-Encoding utf8`。**

## 1. `>` 和 `>>` 的区别（对应 Set / Add）

教程命令的**大于号个数**是关键：

| 教程写法 | 符号含义 | PowerShell 替代命令 |
|---|---|---|
| `echo "x" > 文件` | **覆盖**整个文件内容 | `Set-Content -Path 文件 -Value "x" -Encoding utf8` |
| `echo "x" >> 文件` | **追加**到文件末尾 | `Add-Content -Path 文件 -Value "x" -Encoding utf8` |

记忆方法：

- 一个尖 `>`：Set = **设成**（覆盖）
- 两个尖 `>>`：Add = **添加**（追加）
- **文件不存在时两个命令效果一样**（都会创建文件）；文件已存在时必须按符号选择，否则内容会不对（例如覆盖写成追加，文件会变成两行，冲突的样子也和教程对不上）。

## 2. 命令参数拼写错误（遇到过的）

| 错误写法 | 正确写法 | 说明 |
|---|---|---|
| `-Vlue` | `-Value` | 参数名少字母 |
| `- Encoding utf8` | `-Encoding utf8` | `-` 和参数名之间不能有空格 |
| `git merge --no-off` | `git merge --no-ff` | `--no-ff` = 禁止快进，强制生成合并提交 |
| `git --oline-3` | `git log --oneline -3` | `log` 是命令名，不是 `--oline` |

## 3. 改文件之后忘了 `git add`

流程是 **改文件 → `git add` 更新暂存区 → `git diff --staged` 才能看到新内容**。

- `git diff` 比较：工作区（磁盘上的文件） vs 暂存区
- `git diff --staged` 比较：暂存区 vs 上次提交（HEAD）

如果改了文件但没重新 add，`git status` 会显示 `AM`：`A` = 已暂存，`M` = 暂存后又改过。此时 `git diff --staged` 看到的还是旧内容。

## 4. `git remote add origin` 报 "already exists"

说明之前已经配置过远程地址了，不是错误。用 `git remote -v` 查看即可，不需要重复添加。

```
origin  https://github.com/你的用户名/git-playground.git (fetch)
origin  https://github.com/你的用户名/git-playground.git (push)
```

- `origin` 只是远程仓库的**别名**（快递地址的"小名"）
- `(fetch)` = 拉取地址，`(push)` = 推送地址

## 5. 提交消息（commit message）只是标签

`git commit -m "xxx"` 里的 `xxx` 不会写进任何文件，只是给这次历史快照贴的说明标签，显示在 `git log` 里。Git 合并靠的是**内容**，不是消息。

（练习中就出现过一条消息是 "Use a friendly greeting"、内容却是 tasks.txt 的提交——消息和内容对不上，但合并照常工作。）

## 6. 合并冲突（merge conflict）

冲突是 Git 的正常机制，两边改了同一处内容才会发生，**不是错误**。

冲突标记长这样：

```
<<<<<<< HEAD
当前分支的版本
=======
对方分支的版本
>>>>>>> 对方分支名
```

处理流程：

```powershell
git status                 # 查看冲突文件（显示 UU / both modified）
cat 文件                    # 查看冲突标记
git merge --abort          # 取消合并（练习用）
git merge 分支名            # 重新合并
# 解决：把文件改成最终内容（用 Set-Content 覆盖，保证删掉所有冲突标记）
Set-Content -Path 文件 -Value "最终内容" -Encoding utf8
git add 文件
git commit -m "Resolve ... conflict"
```

## 7. 登录 GitHub（Credential Manager）

第一次 `git push` 会弹出登录窗口（或跳转浏览器），登录一次后凭据自动保存到 Windows 凭据管理器，**以后 push / pull 不再需要重复登录**。

## 8. 概念小抄

- **工作区 → 暂存区 → 提交 → 远程**：`git add`（登记）→ `git commit`（存快照）→ `git push`（上传）
- `git switch -c 分支名`：新建并切换到分支；`git switch main`：切回主分支
- `--no-ff`：即使能快进也强制生成一个合并提交（历史里看得见分叉和合并）
- `git log --graph --oneline --all`：以图形方式看全部分支历史
- `git push -u origin main`：推送并建立跟踪，之后直接 `git push` 即可
- 提交只保存在本地，**push 才会同步到 GitHub**

## 9. 以后写文件的最优选择

1. **VS Code 编辑**：点开文件 → 编辑 → Ctrl+S（默认 UTF-8，最省心）
2. **PowerShell**：`Set-Content` / `Add-Content` + `-Encoding utf8`
3. **切到 Git Bash**：VS Code 终端下拉框可选，`echo` 可直接按教程原样使用，不会有编码问题
