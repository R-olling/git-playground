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

## 14. 解决冲突 = 给冲突一个结果（拍板）

冲突解决**不是**"必须二选一"，而是**由人决定最终内容**：可以选当前分支的、选对方的、或写一个全新的值（教程用 `color=purple` 演示"第三个值"）。

- 教程选 purple 只是为了教学：全新值一眼能看出"是你拍板的"，不会碰巧等于任何一方
- git 只关心三件事：① 文件是想要的最终内容 ② 没有残留冲突标记（`<<<<<<<` / `=======` / `>>>>>>>`）③ `git add` 标记已解决
- 选 red、green、purple 都合法——谁对由人判断，这就是"拍板"；问题不是"真正解决"，而是"让流程继续、由你决定"

## 15. `git -c core.editor=true rebase --continue` 详解

整句效果：**继续被中断的 rebase，并且不弹出编辑器确认消息**。

拆解：
- `git rebase --continue` = 恢复被打断的 rebase（好比传送带重新启动：把解决好的内容重新打包成新提交，继续重放剩余提交）
- `-c` = config 的缩写，**临时**设置一项配置（只对这次命令生效，不改永久设置）
- `core.editor` = 配置项路径，和 `user.name` 是同类（都是"分组.选项"格式）；作用：git 需要确认提交消息时用哪个编辑器（默认 vim）
- `= true` = 把编辑器设成 `true` 这个"什么都不干、立即成功退出"的程序 → git "打开编辑器"时瞬间结束 = 自动接受原消息，不弹窗

记忆：**`-c core.editor=true` = 别弹编辑器**。
忘了怎么办：裸敲 `git rebase --continue`，弹出 vim 时按 `Esc`，输入 `:q`（不保存退出）或 `:wq`（保存退出）回车，一样能过。

## 16. rebase 冲突处理的两轮练习（第 6 节）

- **第一轮练"撤退"**：`git rebase --abort` → 回到 rebase 前原样（settings.txt = green，子分支自己的版本）——验证"后悔药"有效
- **第二轮练"解决"**：重新 `git rebase main` 触发冲突 → 覆盖成 purple → `git add` → `--continue`

两个易混点：
- 冲突解决时改的是工作区里**"无主"的冲突现场文件**（同时装着双方版本，不属于任何分支）；purple 最终归 rebase 后的 "Choose green" 提交所有
- rebase **不需要你站在 main 上**：当前分支是执行者，main 只是"新底座"

## 17. 第 7 节：Pull Request 全流程（自己的话版）

**本地半场（终端）**：建分支 → 写文件 → add（暂存）→ commit（提交并命名）→ push -u（推送到 GitHub 对应分支）

**远程半场（网页）**：创建 PR（base = 目的地 main，compare = 来源 pr-notes）→ 审查 Files changed → 点 Merge 合并

关键理解：
- **PR 不是 git 命令，是 GitHub 的"审批流程"**：写代码的人通过 PR 申请把分支合进主分支，审查后批准
- `-u`（--set-upstream）= 建立跟踪：之后在这个分支上 `git push` / `git pull` 免参数
- `base: main` = 改动最终合到哪；`compare: pr-notes` = 改动从哪来（方向别记反）
- **Create a merge commit ≠ 揉成一个**：它保留原始提交，只产生一个"接点提交"（像绳子打结，结是新的、绳子都在）；**Squash and merge** 才是把所有提交压成一个
- 合并发生的是**远程** main；本地电脑不知情，要靠 fetch / pull 同步

## 18. 第 8 节：fetch vs pull（修正版故事）

① 网页上：PR 已合并 → 远程 main 有了 review.txt
② 本地：`git switch main`（此刻本地还没有 review.txt）
③ `git fetch`：下载"远程状态信息"——本地 main 没动、文件夹不变
④ `git log --oneline main..origin/main`：显示"远程有、本地没有"的提交（`A..B` = A 缺的提交）
⑤ `git diff main origin/main`：显示具体差在哪（文件内容差异）
⑥ `git pull --ff-only`：真正把远程更新合并到本地 → review.txt 出现
⑦ `cat review.txt`：确认内容
⑧ `git branch -d pr-notes`：**删除**已合并的本地分支（-d = delete；内容没丢，提交还在 main 历史里）

一句话：**fetch = 我知道了；pull = 我拿到了。**

## 19. GitHub 网页英文术语速查（PR 相关）

| 英文 | 中文 | 含义 |
|---|---|---|
| base: main | 目的地分支 | 改动最终合到哪 |
| compare: 分支 | 来源分支 | 改动从哪来 |
| ✓ Able to merge | 可以合并 | 两边没冲突，能自动合 |
| Create pull request | 创建合并请求 | 递交"合并申请书" |
| Conversation | 讨论页 | 状态、评论、合并按钮所在 |
| Files changed | 文件变更页 | 审查这次改了哪些文件 |
| Commits | 提交列表 | 这个 PR 包含哪些提交 |
| Merge pull request | 合并按钮 | 点它执行合并 |
| Create a merge commit | 生成合并提交 | 保留原始提交，加一个接点 |
| Squash and merge | 压扁合并 | 所有提交压成一个 |
| Rebase and merge | 变基合并 | 提交重放成直线（哈希变） |
| Confirm merge | 确认合并 | 最终确认 |
| Merged | 已合并 | 合并成功状态 |
| Delete branch | 删除分支 | 删 GitHub 上的来源分支（选填） |
| Reviewers / Labels | 审阅人 / 标签 | 全是选填，不填不影响 |

## 20. 术语澄清：是"暂存区"，不是"缓存"

`git add` 把改动放进的是 **暂存区（staging area）**，准备提交用的；不要叫"缓存"（cache，指另一个概念）。术语记准，看文档不懵。

## 21. 第 9 节：克隆队友 + 历史分叉（push 被拒）

- 用 `git clone 仓库地址 新文件夹名` 在**原仓库旁边**建一个兄弟副本（模拟队友）；副本的 VS Code 里看不到是正常的——VS Code 只显示你打开过的文件夹
- 副本里 `git config user.name / user.email`（不带 --global）只改**这个仓库**的身份 → 提交作者显示 Teammate，两个作者一目了然
- 队友推了新提交后，你自己本地也提交、**不拉取直接 push** → **被拒**：
  `! [rejected] main -> main (non-fast-forward)` / "remote contains work that you do not have locally"
- 原因 = **历史分叉（history divergence）**：远程有你本地没有的提交（哪怕改的是不同文件），git 一律拒绝覆盖式推送
- 解决：`git pull --rebase`（把远程提交拉下来 + 把你的提交重放到它上面）→ `git log --oneline -4`（看到一条直线）→ 再 `git push` 成功

## 22. 第 10 节：pull --rebase vs pull --no-rebase

| | pull --rebase | pull --no-rebase（merge） |
|---|---|---|
| 你的提交 | **被重写**（新哈希） | **保留原样**（哈希不变） |
| 历史形状 | 一条直线 | 分叉 + 合并提交（`git log --graph` 可见交叉） |
| 场景 | 想保持历史干净 | 想保留协作分叉的真实痕迹 |

`--no-edit` = 接受 git 自动生成的合并消息、不弹编辑器（和 `core.editor=true` 一个思路）。

## 23. 第 11 节：stash（把干一半的活"存起来"）

- `git stash push -m "说明"` = 把未提交的改动存进"保险柜"并清空工作区 → 可以随便 `git switch` 切分支
- `git stash list` = 看保险柜里有什么；`git stash pop` = 取回（成功时自动销毁存单，输出 `Dropped refs/stash@{0}`）
- ⚠️ 默认**只存已跟踪的改动**：新建的、从没 add 过的文件不会被存；要连它们一起存：`git stash push -u`
- ⚠️ pop 时如果目标文件已变，可能冲突——冲突时 stash 保留，先解决冲突再说

## 24. 第 12 节 revert vs 第 13 节 reset（撤销的两种方式）

- **revert（撤销已推送的提交）**：追加一个**反向提交**抵消效果——**历史完整保留**（`d1e06ec` 错误提交还在，后面跟一个 `Revert` 提交删掉文件）；适合别人可能已拉过的提交
- **reset（重做未推送的提交）**：把提交**撤下来重做**——`git reset --soft HEAD~1` 后，草稿提交从 log 消失、**改动还留在暂存区**，改写后重新 commit
- 关键区别：**revert 留着历史，reset 抹掉重来**；`HEAD~1` = HEAD 的上一个提交（~1 = 往前数 1）
- `--soft` = 只退提交指针，工作区和暂存区都不动

## 25. 第 14 节：restore 的两种用法

- `git restore --staged 文件` = **取消暂存**（add 的反操作）——改动还在文件里
- `git restore 文件` = **丢弃工作区改动**——文件回到最近一次提交的状态，**改动基本找不回来**（没提交没 stash 的改动 git 救不了，丢弃前三思）

## 26. 撤销三兄弟对比 + 收尾（第 15 节）

| 命令 | 对象 | 效果 | 历史 |
|---|---|---|---|
| `git revert` | 已推送的提交 | 追加反向提交抵消 | 保留 |
| `git reset --soft` | 未推送的提交 | 提交撤下、改动留在暂存区 | 抹掉 |
| `git restore` | 未提交的改动 | 工作区/暂存区回退 | 无 |

收尾检查（第 15 节）：`git push` → `git status` 干净 → `git log --graph --oneline --decorate --all -25` 能看到这一天折腾出来的完整历史：分支提交、PR 合并提交、merge 合并提交、rebase 重放的提交、显式 revert，全在图里。

**教程 15 节全通关！**
