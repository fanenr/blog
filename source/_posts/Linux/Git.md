---
title: Git
date: 2023-10-24 21:48:08
updated: 2023-11-07 17:18:16
tags:
  - Git
categories:
  - [Linux, Tool]
---

&emsp;&emsp;一些必备且常用的 Git 知识。

<!-- more -->

## 基础

&emsp;&emsp;Git 对待数据的方式类似快照流，不同于其他很多基于差异流的版本控制系统。Git 的每次 commit 都是一次对所有文件的快照：

![](01.png)

&emsp;&emsp;每当用户提交更新或保存项目状态时，Git 就会对当时的全部文件创建一个快照并保存这个快照的索引 (提交对象)。为了效率，如果文件没有修改，Git 不再重新存储该文件，而是只保留一个链接指向之前存储的文件。

### 安装

&emsp;&emsp;安装 Git 很简单：

```bash
sudo dnf in git-all
```

### 配置

&emsp;&emsp;安装 Git 后需要配置一些变量，这些变量将控制 Git 的外观和行为，它们存储在 3 个不同的位置：

1. /etc/gitconfig

&emsp;&emsp;该文件包含系统上每一个用户及他们仓库的通用配置。 如果在执行 git config 时带上 --system 选项，那么它就会读写该文件中的配置变量。

2. \~/.gitconfig or \~/git/config

&emsp;&emsp;该文件只针对当前用户。可以传递 --global 选项让 Git 读写此文件，这会对系统上所有的仓库生效。

3. anyrepo/.git/config

&emsp;&emsp;每个 Git 仓库都有一个该文件，它只对当前仓库生效。如果在执行 git config 时带上 --local 选项，那么就会读写该文件，这也是默认行为。 

&emsp;&emsp;每一个级别的配置会覆盖上一级别的配置，所以 .git/config 的配置会覆盖 ~/.gitconfig 中的配置，后者会继续覆盖 /etc/gitconfig 中的配置。

-----

&emsp;&emsp;使用 Git 前必须配置用户名和邮箱：

```bash
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
```

&emsp;&emsp;默认的文本编辑器是可选的：

```bash
git config --global core.editor vim
```

&emsp;&emsp;创建一个新仓库时其默认分支名称：

```bash
git config --global init.defaultBranch main
```

-----

&emsp;&emsp;也可以使用 config 命令查看信息：

```bash
git config user.name
```

&emsp;&emsp;这会列出当前操作最近一个配置中的变量值，建议使用下面的操作，它能显示值来自哪个文件：

```bash
git config --show-origin user.name
```

-----

&emsp;&emsp;可以配置 Git 命令的别名来减少输入：

```bash
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.unstage 'restore --staged'
```

### 创建仓库

&emsp;&emsp;Git 的大部分操作都是在本地执行的，所以使用 Git 必须在本地拥有一个仓库。

1. 初始化本地仓库

&emsp;&emsp;使用 init 子命令在本地初始化一个新仓库：

```bash
cd anydir
git init

# or 

git init anydir
```

&emsp;&emsp;它会在仓库目录下创建一个 .git 目录用于存放仓库信息。

2. 克隆远程仓库

&emsp;&emsp;使用 clone 子命令克隆一个远程仓库到本地：

```bash
git clone https://github.com/libgit2/libgit2
```

&emsp;&emsp;这将在当前目录中创建一个 libgit2 子目录来存放该仓库。也可以手动指定仓库目录：

```bash
git clone https://github.com/libgit2/libgit2 mydir
git clone https://github.com/libgit2/libgit2 .
```

&emsp;&emsp;Git 支持很多数据传输协议：https，git，ssh 等。

### 查看状态

&emsp;&emsp;一个 Git 仓库中所有文件只有两种状态：未跟踪和已跟踪。具体来说，已跟踪状态又分为 3 种，一个文件如果已被跟踪，则说明它已被纳入版本管理中。

&emsp;&emsp;可以使用 status 子命令查看仓库详细信息：

```bash
git status
```

&emsp;&emsp;具体来说，所有文件都只属于下列状态中的一种：

![](02.png)

&emsp;&emsp;status 命令还支持简短形式的信息展示：

```bash
git status -s
```

&emsp;&emsp;一个输出例子：

```bash
 M README
MM Rakefile
A lib/git.rb
M lib/simplegit.rb
?? LICENSE.txt
```

&emsp;&emsp;新添加的未跟踪文件前面有 ?? 标记，新添加到暂存区中的文件前面有 A 标记，修改过的文件前面有 M 标记。输出中有两栏，左栏指明了暂存区的状态，右栏指明了工作区的状态。例如，上面的状态报告显示： README 文件在工作区已修改但尚未暂存，而 lib/simplegit.rb 文件已修改且已暂存。Rakefile 文件已修改，暂存后又作了修改，因此该文件的修改中既有已暂存的部分，又有未暂存的部分。

### 暂存文件

&emsp;&emsp;add 是一个多功能命令：可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态等。

1. 跟踪一个新文件

```bash
git add newfile
```

&emsp;&emsp;newfile 将被加入暂存区。

2. 将一个已跟踪文件加入暂存区

```bash
git add modfile
```

&emsp;&emsp;modfile 是一个已跟踪的文件，但是它被修改了，使用 add 将它加入暂存区中。

### 创建提交

&emsp;&emsp;commit 用于向仓库提交一条记录，它只会提交暂存区中的所有内容：

```bash
git commit
```

&emsp;&emsp;它会启动默认编辑器来获取一些提交说明，编辑器中会包含一个空行以供输入，后面是一些注释。这些注释会在提交时被删除，即使在编辑器中删除它们也没问题。

&emsp;&emsp;使用 commit 命令的 -m 选项可以直接在命令行中说明注释信息，而不调用编辑器：

```bash
git commit -m "Update version"
```

&emsp;&emsp;使用 commit 命令的 -a 选项可以直接跳过暂存区，而直接将所有已跟踪且修改过的文件提交：

```bash
git commit -a -m "Update version"
```

&emsp;&emsp;注意：-a 选项只会提交已跟踪过的文件。

&emsp;&emsp;使用 commit 命令的 --amend 选项可以修改上一次的提交：

```bash
git commit --amend
```

&emsp;&emsp;它的效果就像：在原记录的基础上增加一条提交记录，然后删除原记录。如果修改提交时暂存区非空，则暂存区的内容会随着 --amend 一起被提交。

### 撤销操作

&emsp;&emsp;包含两种撤销操作：撤销暂存区中的文件和撤销工作区中的文件。两种命令都可以使用 restore 命令完成，也可以使用 reset 或 checkout 命令完成。

1. 撤销暂存区中的文件

&emsp;&emsp;作用：将暂存区中的文件恢复为某次提交中的状态，默认是最近一次提交。

&emsp;&emsp;使用 restore 命令的 --staged 选项：

```bash
git restore --staged filename
```

&emsp;&emsp;撤销暂存区中的所有文件：

```bash
git resotre --staged .
```

1. 撤销工作区中的文件

&emsp;&emsp;作用：将工作区中的文件恢复为暂存区中的状态。

&emsp;&emsp;使用不带选项的 restore 命令：

```bash
git restore <file>
```

&emsp;&emsp;一次性撤销所有文件：

```bash
git restore .
```

&emsp;&emsp;注意：这两个操作的交集是暂存区，它们都不影响 HEAD 指向和分支指向。当在暂存区撤销文件时，不会影响工作区。当在工作区撤销文件时，不会影响暂存区。如果要同时撤销暂存区和工作区的所有文件，可以使用 checkout 命令或 reset 命令，但要注意它们的风险：

![](06.png)

### 远程仓库

&emsp;&emsp;clone 一个远程仓库时 Git 会自动保存远程仓库的信息，可以使用 remote 命令查看所有远程仓库的信息：

```bash
git remote
```

&emsp;&emsp;它只会列出远程仓库在本地的 alias，加上 -v 选项可以查看读写情况和仓库地址。

```bash
git remote -v
```

&emsp;&emsp;可以使用 remote 的 show 选项来显示更详细的信息：

```bash
git remote show origin
```

&emsp;&emsp;如果不从远程仓库 clone，或者想添加别远程仓库，可以使用 add 选项：

```bash
git remote add alias url
```

&emsp;&emsp;这将在本地添加一个远程仓库记录，并且设置该远程仓库的别名。

&emsp;&emsp;可以使用 rename 选项改变一个远程仓库的别名：

```bash
git remote rename origin newname
```

&emsp;&emsp;使用 remvoe 选项删除一个远程仓库的信息：

```bash
git remote remove origin
```

### 拉取远程仓库

&emsp;&emsp;fetch 命令用于拉取远程仓库的数据到本地：

```bash
git fetch origin
```

&emsp;&emsp;fetch 会访问远程仓库，从中拉取所有本地还没有的数据。执行完成后，本地将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看。

&emsp;&emsp;注意：fetch 只是简单的拉取远程仓库的数据，它不会修改本地仓库的信息，也不会自动合并。

&emsp;&emsp;fetch 默认拉取远程仓库的所有分支，可以手工指定拉取哪些分支：

```bash
git fetch origin master
```

### 推送本地分支

&emsp;&emsp;push 命令用于将本地记录推送到上游：

```bash
git push origin master
```

&emsp;&emsp;只有当用户有所克隆服务器的写入权限，并且之前没有人推送过时，这条命令才能生效。当和其他人在同一时间克隆，但他们先推送到上游然后用户再推送到上游时，用户的推送就会毫无疑问地被拒绝。用户必须先抓取他们的工作并将其合并进本地仓库后才能推送。

&emsp;&emsp;push 的完整语法是：

```bash
git push <remote> <local_branch>:<remote_branch>
```

&emsp;&emsp;可以省略远程分支名，这代表远程分支名和本地分支同名。

&emsp;&emsp;如果远程仓库是一个空仓库，那么 push 很可能失败，因为 push 不知道要推送到远程仓库的哪个分支。可以给 push 加上 -u 选项，这将为本地分支关联上游分支：

```bash
git push -u origin master
```

&emsp;&emsp;如果远程仓库不存在 master，那么 -u 会自动创建。此后，本地分支会跟踪这个远程分支。

## 分支

&emsp;&emsp;分支是 Git 的核心概念，它和 commit 一样，都是很轻量化的概念。分支本质上仅仅是指向提交对象的可变指针。Git 的默认分支名字是 master，并且 master 分支会在每次提交时自动向前移动。

![](03.png)

&emsp;&emsp;可以看到：每次 commit 都会创建一个快照，而分支就是指向快照索引 (提交对象) 的可变指针。

&emsp;&emsp;在本地仓库中，始终存在一个 HEAD 指针，它说明当前所在的位置，HEAD 一般情况下都指向分支，此时 HEAD 相当于分支的别名。

### 创建分支

&emsp;&emsp;branch 命令基于当前提交对象创建一个新分支：

```bash
git branch testing
```

&emsp;&emsp;如果没有特殊操作，现在 testing 和 master 都指向同一个提交对象。

![](04.png)

&emsp;&emsp;另一种创建分支的方法是使用带 -b 选项的 checkout 命令：

```bash
git checkout -b testing
```

&emsp;&emsp;它包含两个操作：创建分支并自动切换到该分支上。

### 切换分支

&emsp;&emsp;checkout 另一个作用就是切换分支：

```bash
git checkout testing
```

&emsp;&emsp;checkout 做两个工作：修改 HEAD 指针，以及更新工作区。由于切换分支会加载该分支关联的文件快照到工作目录中，所以要注意在切换分支前保存当前工作内容。

### 移动分支

&emsp;&emsp;branch 的 -m 选项用来移动分支，经常用它来重命名一个分支：

```bash
git branch -m old new
```

### 分支合并

&emsp;&emsp;由于分支非常轻量，所以一个项目中可能同时有很多分支，它们可能只关注一个很小的问题，当工作完成后，这个分支的使命就终了。接下来，主分支就可以使用 merge 命令合并它们的工作：

```bash
git checkout master
git merge issue53
```

&emsp;&emsp;先切换到主分支，然后合并 issue 分支的内容。这将在主分支上创建一个新记录，这个新纪录有两个祖先：之前 master 的最新记录，以及 issue 的最新记录。

&emsp;&emsp;因为 merge 涉及 3 个文件快照，所以它也叫三方合并：

![](05.png)

### 查询分支

&emsp;&emsp;branch 命令用于查询本地分支：

```bash
git branch
git branch -v
```

&emsp;&emsp;二者只列出所有本地仓库中的分支，加上 -v 选项时，它会附加一些提交信息。

&emsp;&emsp;branch 的 -vv 选项会显示本地分支的上游情况：

```bash
git branch -vv
```

&emsp;&emsp;如果要查看所有分支，包括远程仓库的分支，可以使用 -a 选项：

```bash
git branch -a
```

### 合并冲突

&emsp;&emsp;合并两个分支时很可能遇到冲突，例如两个分支都对同一文件的相同部分作了修改，Git 无法确定要保留哪个，所以它将暂时终止合并，只有冲突被解决后，才能继续合并。

&emsp;&emsp;冲突发生时，冲突所在文件会被修改以标识出冲突部分：

```bash
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```

&emsp;&emsp;用户可以选择保留一个分支中的内容，也可以做额外修改，但最后必须删除`<<<<`，`====`以及`>>>>`。

&emsp;&emsp;解决冲突文件后，需要使用 add 命令标识该文件的冲突已被解决，就和暂存它一样。

&emsp;&emsp;最后，需要使用 commit 命令手动为这次发生冲突的合并提交记录，就和普通的 commit 一样。

### 删除分支

&emsp;&emsp;当一个分支已被其他分支合并后，就可以安全的删除它：

```bash
git branch -d issue53
```

&emsp;&emsp;但是，如果一个分支还没有被任何分支合并，-d 选项时无法删除它的，因为这将导致信息丢失。如果要强行删除它，可以使用：

```bash
git branch -D  notmerged
```

### 跟踪上游

&emsp;&emsp;由于本地仓库和远程仓库时分隔的，一种联系本地分支和远程的方式就是让本地分支跟踪远程分支。

1. 一个已有的分支

&emsp;&emsp;可以使用 branch 的 -u 选项让一个本地分支跟踪远程分支：

```bash
git checkout master
git branch -u origin/master
```

2. 直接从远程仓库创建分支

&emsp;&emsp;从远程分支上直接创建一个本地分支也会自动跟踪它：

```bash
git checkout origin/master
git checkout -b master
```

3. 其他方式

&emsp;&emsp;还有很多其他方式：

```bash
git checkout -b sf origin/serverfix
git push -u origin master
```

&emsp;&emsp;后一种方式会在推送的同时改变本地分支跟踪的上游。

-----

&emsp;&emsp;当一个本地分支有了上游 (跟踪了远程分支)，那么它和远程仓库交互就会很方便。

&emsp;&emsp;之前拉取远程分支并合并到本地需要两个操作 fetch 和 merge，但是现在可以直接使用 pull 命令，它会自动从当前分支的上游拉取信息，如果有更新，它会自动合并到本地：

```bash
git checkout master
git pull
```

### 贮藏

&emsp;&emsp;stash 命令以及其子命令用于完成贮藏工作。贮藏会将当前工作目录中的变动 (相对于上一次提交) 保存起来，并打包成一个镜像放入栈中。然后工作目录就被恢复成上一次提交时的样子。

1. 创建贮藏

```bash
git stash
git stash push
```

&emsp;&emsp;上面两条命令等价。但贮藏默认只会保存已跟踪文件，如果目录中有未跟踪文件，贮藏是不会保存的，必须使用 -u 或者 -a 选项：

```bash
git stash -u
git stash -a
```

&emsp;&emsp;二者的区别在于：-u 只会保存未跟踪文件，但是 -a 还会保存 .gitignore 中忽略的文件。

2. 查看贮藏

&emsp;&emsp;使用 stash 的 list 子命令，可以查看分支上所有贮藏镜像：

```bash
git stash list
```

3. 删除贮藏

&emsp;&emsp;使用 stash 的 drop 子命令，可以删除指定贮藏镜像：

```bash
git stash drop 2
```

4. 恢复贮藏

&emsp;&emsp;使用 stash 的 apply 子命令，可以恢复指定贮藏到工作区 (不指定则为最近一次记录)：

```bash
git stash apply 2
```

&emsp;&emsp;注意：apply 默认不会恢复贮藏时的暂存状态，如果需要，使用 --index 选项：

```bash
git stash apply --index 2
```

&emsp;&emsp;一种常见的模式是恢复完后立即删除贮藏镜像 apply 和 drop，它们可以组合成一个操作 pop：

```bash
git stash pop
```

&emsp;&emsp;pop 会取出最近一次贮藏记录，然后立即删除它。

### 标签

&emsp;&emsp;标签和分支一样轻量：它只是指向某个提交的标记，但标签不随着提交而移动。

&emsp;&emsp;标签有两种：轻量标签和附注标签。轻量标签就是一个简单的不动指针，它简单的指向某个提交。而附注标签不同，后者是存在于 Git 中的一个数据对象：附注标签中包含打标签者的名字、电子邮件地址、日期时间， 此外还有一个标签信息，并且可以使用 GPG 签名并验证。

1. 创建标签

&emsp;&emsp;使用 tag 子命令为一个提交创建轻量标签：

```bash
git tag tag-name
git tag tag-name <commit-hash>
```

&emsp;&emsp;第一种写法会为当前提交创建标签，第二种写法可以为指定提交创建标签。

&emsp;&emsp;使用 tag 子命令的 -a 选项创建一个附注标签：

```bash
git tag -a tag-name
git tag -a tag-neme -m "message" <commit-hash>
```

&emsp;&emsp;创建附注标签必须提供附注信息，如果没有通过 -m 选项直接提供信息，Git 就会启动默认编辑器。

2. 删除标签

&emsp;&emsp;使用 tag 子命令的 -d 选项删除一个标签：

```bash
git tag -d tag-name
```

3. 列出标签

&emsp;&emsp;使用 tag 子命令查看仓库所有标签：

```bash
git tag
```

&emsp;&emsp;空的 tag 命令会按字母顺序列出所有标签。如果需要按模式列出标签，可以使用 -l 选项：

```bash
git tag -l "v1.0*"
```

4. 远程标签

&emsp;&emsp;可以使用 push 子命令将本地标签推送到远程仓库：

```bash
git push <remote> tag-name
git push <remote> <local-tag>:<remote-tag>
```

&emsp;&emsp;如果要一次性推送所有的本地标签到远程仓库，可以使用 --tags 选项：

```bash
git push <remote> --tags
```

&emsp;&emsp;和在远程仓库中删除分支一样，删除远程仓库中标签的语法是：

```bash
git push <remote> :tag-name
```

