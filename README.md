# CommonNote
欢迎来到使用 Git 和 GitHub 进行团队协作的世界。这套流程是现代软件开发的基石，一旦掌握，你会发现它非常强大和高效。

下面我为你分解一下完整、规范的协作开发流程，从准备工作到完成一个任务的全过程。

------



### 第零步：准备工作 (只需做一次)

在开始之前，确保你已经完成了以下准备：

1. **安装 Git**: 如果你的电脑还没有安装 Git，请先从 [官网下载](https://www.google.com/url?sa=E&q=https%3A%2F%2Fgit-scm.com%2Fdownloads) 并安装。

2. **拥有 GitHub 账号**: 确保你有一个 GitHub 账号。

3. **获得仓库访问权限**: 这是最关键的一步。你需要联系仓库的所有者 (owner)，也就是 Anainoyume，**让他们将你的 GitHub 账号添加为这个私有仓库的协作者 (Collaborator)**。否则，你连仓库的代码都无法下载。

4. **配置 Git**: 在本地设置你的用户名和邮箱，这会记录在你的每一次提交中。

   code Bash

   downloadcontent_copyexpand_less

   

   ```bash
   git config --global user.name "你的名字"
   git config --global user.email "你的GitHub邮箱"
   ```

5. **配置认证 (推荐使用 SSH) (可选)**: 由于你会频繁地与 GitHub 交互，每次都输入密码会很麻烦。强烈建议配置 SSH 密钥认证。

   - [GitHub 官方教程：生成新的 SSH 密钥并添加到 ssh-agent](https://www.google.com/url?sa=E&q=https%3A%2F%2Fdocs.github.com%2Fzh%2Fauthentication%2Fconnecting-to-github-with-ssh%2Fgenerating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
   - [GitHub 官方教程：将 SSH 密钥添加到你的 GitHub 帐户](https://www.google.com/url?sa=E&q=https%3A%2F%2Fdocs.github.com%2Fzh%2Fauthentication%2Fconnecting-to-github-with-ssh%2Fadding-a-new-ssh-key-to-your-github-account)
   - 配置好后，你需要使用 SSH 格式的 URL 来克隆仓库，而不是 HTTPS。这个 URL 在仓库主页的 "Code" 按钮下可以找到，格式是 git@github.com:Anainoyume/Re_UnderDawn.git。

------



### 第一步：将远程仓库复制到你的本地 (Clone)

这是你参与项目的第一步，将云端的代码完整地下载到你的电脑上。打开你的终端或命令行工具：

```bash
# 如果你配置了 SSH (推荐)
git clone git@github.com:Anainoyume/Re_UnderDawn.git

# 如果你仍想用 HTTPS (每次推送可能需要输入密码或个人访问令牌)
git clone https://github.com/Anainoyume/Re_UnderDawn.git

# 这会在当前目录下创建一个名为 Re_UnderDawn 的文件夹
cd Re_UnderDawn
  
```

现在，你的电脑上就有了一份完整的项目代码了。

------



### 第二步：日常开发循环 (为你分配的每个任务重复此流程)

假设你要开始开发一个新功能或修复一个 Bug，**永远不要直接在主分支 (main 或 master) 上进行修改**。这非常重要！

#### 1. 同步最新代码

在开始任何新工作之前，务必确保你本地的代码是最新版本，与远程主分支保持一致。

```bash
# 首先，切换到主分支 (可能是 main 或 master，根据你们团队的约定)
git switch main

# 从远程仓库拉取最新的更新
git pull origin main
```

#### 2. 创建一个属于你的新分支 (Branch)

为你的新任务创建一个独立的分支。分支的作用是隔离你的工作，确保不会影响到主分支和其他同事的工作。分支的命名最好能清晰地描述这个分支的用途。

```bash
# 创建并立即切换到这个新分支
# 好的命名习惯: feature/功能名, bugfix/问题描述, hotfix/紧急修复 等
git switch -c feature/user-login-system
```

现在，你就像进入了一个“平行时空”，可以自由地进行编码、修改、实验，而不用担心弄乱主分支。

#### 3. 编写代码并保存进度 (Add & Commit)

这是你实际的工作环节。

- **编写代码**：修改文件、添加新文件、删除旧文件。
- **保存进度**：当你完成了一个小阶段的工作时，应该创建一个“存档点”，在 Git 里这叫做 **提交 (Commit)**。

```bash
# 查看你做了哪些修改
git status

# 将你想要保存的文件添加到“暂存区”
# 添加某个特定文件
git add src/components/LoginForm.js
# 或者添加所有修改过的文件
git add .

# 提交你的修改，并写下清晰的说明
git commit -m "Feat: 完成用户登录表单的基本结构和样式"
```

**良好实践**：一次提交只做一件相关的事情，并写清楚提交信息（Commit Message），这会让代码历史非常清晰，方便以后回溯和排查问题。

你可以多次重复“编码 -> git add -> git commit”这个过程。

#### 4. 将你的分支推送到远程仓库 (Push)

当你觉得这个功能开发得差不多了，或者希望同事能看到你的代码时，就需要将你本地的分支推送到 GitHub。

```bash
# 第一次推送这个新分支时，需要用 -u 参数来建立本地分支和远程分支的联系
git push -u origin feature/user-login-system

# 之后，如果你在这个分支上有了新的 commit，再次推送时只需要：
git push
```

------



### 第三步：请求合并代码 (Pull Request)

推送完成后，你的分支和代码就已经在 GitHub 云端了。但它还没有进入主分支。你需要发起一个 **Pull Request (PR)**，这相当于一个正式的申请：“我完成了我的工作，请检查一下，如果没问题，就把它合并到主分支里吧。”

1. 打开你的浏览器，访问 https://github.com/Anainoyume/Re_UnderDawn。
2. GitHub 通常会自动检测到你刚刚推送的新分支，并显示一个黄色的提示条，上面有一个 "Compare & pull request" 按钮，点击它。
3. **填写 PR**:
   - **标题**: 清晰地概括你的工作内容。
   - **描述**: 详细说明你做了什么、为什么这么做、如何测试等。如果这个 PR 修复了某个特定的 Issue，可以在描述里写 Closes #123 来关联它们。
   - **审查者 (Reviewers)**: 在右侧选择你的同事或项目负责人来审查你的代码。
4. 点击 "Create pull request"。

------



### 第四步：代码审查与合并 (Code Review & Merge)

你的同事会收到通知，他们会在线上查看你的代码，可能会提出一些修改建议或问题。

- **讨论与修改**: 你们会在 PR 的评论区进行讨论。如果需要修改代码，你只需在**本地的同一个分支**上继续修改、git commit、然后 git push。你的 PR 会自动更新。
- **批准与合并**: 当你的代码通过审查后，拥有权限的同事（通常是项目负责人）会在 GitHub 页面上点击 "Merge pull request" 按钮，将你的所有提交一次性地合并到 main 主分支中。

**恭喜！你的代码已经成功地贡献给了项目！**

------



### 第五步：清理工作

合并后，为了保持仓库整洁，可以做一些清理工作。

```bash
# 1. 切换回主分支
git switch main

# 2. 拉取最新的主分支代码 (包含了你刚刚被合并进去的改动)
git pull origin main

# 3. 删除已经合并的本地分支 (因为它已经没用了)
git branch -d feature/user-login-system

# 4. 你也可以在 GitHub 的 PR 页面上点击按钮删除远程分支。
```

然后，你就可以从“第二步：日常开发循环”开始你的下一个任务了。

### 总结一下流程图

**本地操作**
git pull (更新) -> git switch -c new-branch (创建分支) -> 编码 -> git add & git commit (保存) -> git push (推送)

**GitHub 网页操作**
-> 创建 Pull Request -> Code Review -> Merge PR -> 删除远程分支

**回到本地**
-> git switch main -> git pull -> git branch -d new-branch (清理) -> **开始下一个循环**

这个流程看起来步骤很多，但当你熟练之后，就会发现它非常清晰和安全，能够极大地提升团队协作的效率和代码质量。祝你编码愉快！
