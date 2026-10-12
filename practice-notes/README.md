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
| `git addsettings.txt` | `git add settings.txt` | `add` 和文件名之间必须有空格，否则被当成一个命令 |
| `-c core.editor=ture` | `-c core.editor=true` | `true` 是四个字母；拼错则 git 找不到该程序，`rebase --continue` 无法提交 |

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

## 10. 分支名 / 文件名拼错

**分支名**：`git switch -c color-updata`（少打一个 e）→ 创建出名叫 color-updata 的分支。之后按教程正确名字 `git switch color-update` 会报 `fatal: invalid reference: color-update`。
修复：`git branch -m color-updata color-update`（旧名在前、新名在后；未推送过的本地分支改名安全，内容/历史不变）。
教训：分支名是精确标识，拼错 = 生成一个错误名字的分支，所有按正确名字的操作都会找不到它。

**文件名**：settings 这个名字踩过两次——
- `settings.text`（.txt 打成 .text）→ 生成新文件，目标文件没改动
- `setting.txt`（settings 少个 s）→ 生成新文件，冲突文件 settings.txt 原样未动，"解决"无效

症状：目标文件内容不变、commit 报 "nothing to commit" 或冲突标记仍在。
修复：`Remove-Item 拼错的文件`（未跟踪，删除安全），再对正确文件名重做。

## 11. 概念：分支是指针，不是文件夹

- 分支 = 指向某个提交的指针；文件夹里永远只有一份工作区文件
- 新建分支 = 在当前位置复制一个指针（内容一样）→ 切过去文件夹看不出变化，这是正常的
- 分支切换对文件的影响，只在"提交产生分叉后"才可见（例：切回 main 时 settings.txt 变回 color=blue）
- 指针链：`HEAD` → 当前分支 → 分支指向的最新提交；`git log --oneline` 里的 `(HEAD -> 分支名)` 就是这条链
- `git log --oneline -N`：查看最近 N 条提交，每条一行
- `git branch -m 旧名 新名`：分支改名

## 12. rebase 是什么

`git rebase main`（当前在子分支上）= 把子分支独有的提交一个个摘下来，重放到 main 的最新提交之后。

- 结果：历史变直线；提交内容/消息不变，但**哈希必然变**（父提交变了）
- 旧提交变成"孤儿提交"（无人引用，git gc 会清理）——不会重复，log/文件只显示新链
- **不需要站在 main 上**：当前分支是执行者，main 只是"新底座"

### 12.1 为什么 rebase 会冲突（Python 直觉的误区）

- 有先后：`red 之后再 commit green` → 覆盖，无冲突（和 Python 变量覆盖一样）
- 无先后（并行分支）：两条分支从同一行出发改成不同值 → git 无法决定 → 冲突
- Python 覆盖的是"当前值"，git 合并的是"并行变更"；冲突不是错误，是 git 把"需要人拍板"的情况暴露出来

### 12.2 rebase 冲突处理三选一

| 命令 | 含义 |
|---|---|
| `git rebase --abort` | 整场放弃，回到 rebase 前状态（后悔药） |
| `git add 文件` + `git -c core.editor=true rebase --continue` | 解决后继续 |
| `git rebase --skip` | 跳过这条提交（不轻易用） |

### 12.3 冲突标记怎么读

```
<<<<<<< HEAD      上半区 = 当前分支的版本（rebase 场景 = 新底座）
=======           分界线
>>>>>>> 提交说明  下半区 = 对方提交带来的版本
```

## 13. cat 显示"锘縞"乱码（BOM 显示问题）

`Set-Content -Encoding utf8` 会在文件开头写 UTF-8 BOM（EF BB BF）。当冲突标记把 BOM 挤到文件中间时，PowerShell 的 cat 检测不到文件开头的 BOM，改用 GBK 解码 → BOM 显示成"锘縞"。

- 不是文件坏了：按 UTF-8 解码内容完全正常（git 一直能正确识别）
- 验证：VS Code 打开，或 `Get-Content 文件 -Encoding UTF8`
- 覆盖写之后 BOM 回到文件开头，cat 又恢复正常显示
