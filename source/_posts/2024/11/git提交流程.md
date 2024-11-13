---
title: git提交流程
tags: [git]
categories: [git]
permalink: posts/1.html
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-11 22:12:51
topic: project
description:
cover:
banner:
references:
---

## 一、Git下载及配置

我们第一次用git或者是新电脑上重新安装git工具的时候，都需要重新配置一下这个工具。

### Windows安装git

官网网址[https://git-scm.com/downloads](https://git-scm.com/downloads)下载速度慢，且有可能安装不成功。 附快速下载地址（国内下载站）：[https://github.com/waylau/git-for-win](https://github.com/waylau/git-for-win) 。 下载完之后，Windows中你在桌面上或者文件管理器中鼠标右键就可以看见Git Bash here，就是用来打开git bash的。

### Linux安装git

linux在终端中，输入`sudo apt-get install git`

下载完可以用命令`git --version`打印当前的git版本验证是否成功。下面正式开始Git的配置。

### 配置git基本信息

接下来就是不管我们是第一次使用git工具，还是后来换电脑了，还是换成linux系统了，要想使用git都按照下面的方法配置一遍，才可以使用。 安装成功之后，在命令行中敲下如下命令 `git config --list`,显示当前的配置信息。 接下来设置提交仓库时的用户名信息 `git config --global user.name "张三"` 设置提交仓库是的邮箱信息 `git config --global user.email "xxxxxxxx@qq.com"` git设置关闭自动换行`git config --global core.autocrlf false` 为了保证文件的换行符是以安全的方法，避免windows与unix的换行符混用的情况，最好也加上这么一句 `git config --global core.safecrlf true`

其实这些信息都在一个配置文件中，就在当前用户的主目录下边的**.gitconfig**文件中，也可以直接打开这个文件`cd ~,vim .gitconfig`进行编辑。

### git协议及秘钥配置

git有四种协议：Git协议，http协议，本地协议，ssh协议。使用https除了速度慢以外，还有个最大的麻烦是每次推送都必须输入口令，但是在某些只开放http端口的公司内部就无法使用ssh协议而只能用https。大部分都是用ssh协议。这个不仅速度快，而且不用每次提交输入密码，可谓是省心省力。 下面就说一下ssh配置过程。(这个协议配置的过程不可缺少，不然就用不了这种协议。)

首先生成 RSA 密钥对 :

`ssh-keygen -t rsa -C "xxxxxxxx@qq.com"`注意格式，一定要正确。 ssh和-keygen无空格 此时在用户主目录下就会有一个.ssh隐藏文件，进入该目录有一个id_rsa.pub文件，cat命令查看这个文件，复制下来然后在 github网站添加公钥 ，方法如下 在 Github 网站添加公钥：在右上角头像处点settings进入设置，然后点SSH and GPG keys,进入之后点击New SSH key 粘贴进去，随便给这个秘钥命个名，方便管理就行了。钥匙显示黑色即可。 此时配置就完成了。接下来就可以使用git了。 执行此命令验证是否成功`ssh -T git@github.com` 成功显示为：Hi XXX! You've successfully authenticated, but GitHub does not provide shell access.

## 二、先有本地库，后有远程库

### 创建版本库

可以理解为版本库就是本地文件的一个目录，也叫仓库。可以用git来管理和回退等 创建版本库：找到一个想被管理的文件夹，进入到文件夹里，输入命令`git init`,此时git就可以管理这个目录了，并且在文件夹下多出来了一个.git的隐藏文件夹。这个.git就是版本库。 Git的版本库里存了很多东西，其中最重要的就是称为**stage（或者叫index）的暂存区**，还有Git为我们自动创建的**第一个分支master**，以及指向master的一个指针叫HEAD。 工作区：就是你在电脑里能看到的目录，比如刚才的文件夹就是一个工作区

### 第一步：将文件添加到暂存区

当我们想添加文件或者修改文件是需要添加到版本库中的，否则无法被git跟踪管理呀，所以当我们添加或者修改文件时，先要用`git add filename`添加到暂存区中，filename为`.`的时候代表当前目录下所有文件都添加到暂存区

### 第二步：将文件提交到分支

`git commit -m "message"`，将文件提交到了分支。

### 添加远程库

我们已经在本地创建了一个Git仓库后，又想在GitHub创建一个Git仓库，并且让这两个仓库进行远程同步，这样，GitHub上的仓库既可以作为备份，又可以让其他人通过该仓库来协作，真是一举多得。 首先，登陆GitHub，然后，在右上角找到“Create a new repo”按钮，创建一个新的仓库，在Repository name填入项目名字，比如我们叫learngit，其他保持默认设置，点击“Create repository”按钮，就成功地创建了一个新的Git仓库 目前，在GitHub上的这个learngit仓库还是空的，GitHub告诉我们，可以从这个仓库克隆出新的仓库，也可以把一个已有的本地仓库与之关联，然后，把本地仓库的内容推送到GitHub仓库。

现在，我们根据GitHub的提示，在本地的learngit仓库下运行命令： `git remote add origin git@github.com:mengchao666/learngit.git` 请千万注意，把上面的mengchao666替换成你自己的GitHub账户名，否则，你在本地关联的就是我的远程库，你以后推送是推不上去的，因为你的SSH Key公钥不在我的账户列表中。

添加后，远程库的名字就是origin，这是Git默认的叫法，也可以改成别的，但是origin这个名字一看就知道是远程库。下一步，就可以把本地库的所有内容推送到远程库上： `git push -u origin master`把本地库的内容推送到远程，用git push命令，实际上是把当前分支master推送到远程。 由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。 推送成功后，可以立刻在GitHub页面中看到远程库的内容已经和本地一模一样,从现在起，只要本地作了提交，就可以通过命令： `git push origin master`

## 三、先有远程库，后有本地库

现在，远程库已经准备好了，下一步是用命令git clone克隆一个本地库： `git clone git@github.com:mengchao666/learngit.git`

可以使用`git clone -b branch`克隆指定的分支
后续修改可以使用如下命令提交
```shell
git add .
git commit -m "message"
git push
```

## 四、企业开发流程

在公司中做项目，一般项目代码都在公共仓库中，我们将其称为远程仓，一般按照如下步骤开发
1、将远程仓fork一份到个人仓
2、git clone个人仓代码到本地
3、使用`git remote -v`命令查看，此时本地关联的origin为个人仓
4、将本地代码关联到公司的远程仓，方便拉取最新代码，命令如下：
```shell
git remote add upstream 公共仓地址
```
此时再次使用`git remote -v`可以看到已经关联了upstream为远程仓
5、在开发需求和问题单修改之前，一般使用`git pull upstream`命令将远程仓代码更新至本地
6、修改代码后提交
```shell
git add .
git commit -m "message"
git push origin branch
```
7、在github/gitlab页面创建MR申请，一般此时就可以了，找人加分就合入了
8、但是如果在创建MR申请时，提示冲突，此时需要解决冲突，解决如下：
```shell
git pull upstream
```
此时会提示哪些文件有冲突，使用ctrl + F搜索`>>>`,有此标志的即为冲突的地方，保留自己想要的代码，重新提交add commit push即可。

企业开发流程中，通常有远程仓新建了一个分支，个人远程仓没有此分支，需要更新，可按照如下步骤更新个人仓代码分支
```shell
git checkout -b 新分支名称 upstream/新分支名称
git pull upstream 新分支名称
git push origin 新分支名称
```

## 五、代码回退等操作
### 工作区的恢复(此时还没有add，代码回退)

使用checkout恢复工作区 `git checkout .` （全部修改），`git checkout --file`改回一个文件,工作区--->还没add

### add的撤销
git reset就是回退到指定的commitID,,使用git commit --amend时追加，不会生成新的commitID,是在原来的commitID基础上进行修改的。

HEAD指向当前最新的commitID，所以仅仅add,没有commit，此时的最新的commitID还是之前的

```shell
git reset --hard HEAD  // 不保留本地修改，回退
git reset --mixed HEAD //保留本地修改，可以重新git add
```
简单理解git reset --xxx HEAD命令，就是将代码回退到了最新的一次commitID的代码状态，hard不保留本地代码工作空间的修改，而mixed保留
### commit后的撤销 

```c
git reset --hard HEAD^ // 不保留本地修改，回退到上一次的commitID状态
git reset --mixed HEAD^ // 保留本地修改，撤销git commit,并且撤销git add
git reset --soft HEAD^ //保留本地修改，撤销commit,不撤销add
```

## 六、git的一些其他操作

查看分支：`git branch` 
创建分支：`git branch <name>` 
切换分支：`git checkout <name>`或者`git switch <name>` 
创建+切换分支：`git checkout -b <name>`或者`git switch -c <name>` 
合并某分支到当前分支：`git merge <name>` 
删除分支：`git branch -d <name>`

当代码仓有子仓情形下的一些命令：
`git submodule update --init --remote`
`git submodule foreach git checkout branch`
`git submodule foreach --recursive`

## 七、同时提交两个MR

当我们同时修改多个问题单，或者同时处理需求时，可能需要同时提交多分不同代码，此时处理方法如下：

1、本地创建一个新的分支来处理第一个MR
```shell
git checkout -b mr1-branch
```
2、提交第一个MR。
修改代码，git add、git commit，git push
```shell
git push -u origin mr1-branch
```
3、切换回原来的分支，以main分支为例
```shell
git checkout main
```
4、创建第二个分支处理第二个MR
```shell
git checkout -b mr2-branch
```
5、提交第二个MR。
修改代码，git add、git commit，git push
```shell
git push -u origin mr2-branch
```