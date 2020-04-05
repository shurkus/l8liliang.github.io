---
layout: article
tags: Shell
title: Linux Shell脚本攻略 -- Git
mathjax: true
key: Git
---

## 配置    

```
配置Git的时候，加上--global是针对当前用户起作用的，如果不加，那只针对当前的仓库起作用。
配置文件放哪了？每个仓库的Git配置文件都放在.git/config文件中：
$ cat .git/config
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
    ignorecase = true
    precomposeunicode = true
[remote "origin"]
    url = git@github.com:michaelliao/learngit.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
    remote = origin
    merge = refs/heads/master
[alias]
    last = log -1
别名就在[alias]后面，要删除别名，直接把对应的行删掉即可。
而当前用户的Git配置文件放在用户主目录下的一个隐藏文件.gitconfig中：
$ cat .gitconfig
[alias]
    co = checkout
    ci = commit
    br = branch
    st = status
[user]
    name = Your Name
    email = your@email.com

```

```
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
$ git config --global color.ui true
忽略某些文件时，需要编写.gitignore；
.gitignore文件本身要放到版本库里，并且可以对.gitignore做版本管理！
$ git check-ignore -v App.class//检查文件是否被忽略
.gitignore:3:*.class    App.class

//配置别名
$ git config --global alias.co checkout
$ git config --global alias.ci commit
$ git config --global alias.br branch
```
注意git config命令的--global参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。
{:.info}

## 创建SSH Key
在用户主目录下，看看有没有.ssh目录，如果有，再看看这个目录下有没有id_rsa和id_rsa.pub这两个文件，如果已经有了，可直接跳到下一步。如果没有，打开Shell（Windows下打开Git Bash），创建SSH Key：
```
$ ssh-keygen -t rsa -C "youremail@example.com"
```
你需要把邮件地址换成你自己的邮件地址，然后一路回车，使用默认值即可，由于这个Key也不是用于军事目的，所以也无需设置密码。
如果一切顺利的话，可以在用户主目录里找到.ssh目录，里面有id_rsa和id_rsa.pub两个文件，这两个就是SSH Key的秘钥对，id_rsa是私钥，不能泄露出去，id_rsa.pub是公钥，可以放心地告诉任何人。

## 初始化仓库    
初始化一个Git仓库，使用git init命令。
添加文件到Git仓库，分两步：
```
    第一步，使用命令git add <file>，注意，可反复多次使用，添加多个文件；
    第二步，使用命令git commit，完成。
```

## 克隆远程仓库
要克隆一个仓库，首先必须知道仓库的地址，然后使用git clone命令克隆。    
Git支持多种协议，包括https，但通过ssh支持的原生git协议速度最快。    
```
git clone git@github.com:michaelliao/gitskills.git
```

## 暂存区
我们把文件往Git版本库里添加的时候，是分两步执行的：    
第一步是用git add把文件添加进去，实际上就是把文件修改添加到暂存区(.git/stage)；    
第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。    
因为我们创建Git版本库时，Git自动为我们创建了唯一一个master分支，所以，现在，git commit就是往master分支上提交更改。    
你可以简单理解为，需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。    

## 撤销修改
场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令git checkout -- file。    
场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令git reset HEAD file，就回到了场景1，第二步按场景1操作。    
场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库。    

## 版本回退    
首先，Git必须知道当前版本是哪个版本，在Git中，用HEAD表示当前版本，也就是最新的提交3628164...882e1e0，    
上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100。    
现在，我们要把当前版本“append GPL”回退到上一个版本“add distributed”，就可以使用git reset命令：
```
git reset --hard HEAD^
git reset --hard 3628164xxxxxxxcommit
```

## 删除文件
一般情况下，你通常直接在文件管理器中把没用的文件删了，或者用rm命令删了：rm test.txt    
这个时候，Git知道你删除了文件，因此，工作区和版本库就不一致了，    
git status命令会立刻告诉你哪些文件被删除了     
现在你有两个选择，    
一是确实要从版本库中删除该文件，那就用命令git rm删掉，并且git commit;    
另一种情况是删错了，因为版本库里还有呢，所以可以很轻松地把误删的文件恢复到最新版本：
```
$git checkout -- test.txt
```


## 分支
Git鼓励大量使用分支：    
```
查看分支：git branch     
创建分支：git branch name     
切换分支：git checkout name    
创建+切换分支：git checkout -b name
合并某分支到当前分支：git merge name
删除分支：git branch -d name
开发一个新feature，最好新建一个分支；
如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name>强行删除。
```

## 合并分支，解决冲突
```
git checkout -b branch1
git branch
修改文件txt,git add , git commit
git checkout master
修改文件txt,git add , git commit
git merge branch1:冲突，需要手动修改txt文件
修改文件（里面有冲突提示，用箭头标识的）
git add , git commit
git branch -d feature1
git log --graph --pretty=oneline --abbrev-commit命令可以看到分支合并图。

通常，合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。
如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。
git merge --no-ff -m "merge with no-ff" dev
```

## 查看修改记录
```
git log --stat
//在提交记录我们可以看到每一次commit，都有哪些文件发生了改变，这里简洁的列出了相关文件基本信息。

git show <hashcode>
//查看某次提交修改详情

git show <hashcode> <filename>
//查看某个文件在某次提交中的修改

git whatchanged <filename>
//查看某个文件的包含提交人员，日期、版本号等记录信息，不包括修改详情

git log --pretty=oneline <filename>
//查看仅某个文件的所有历史记录

git diff <hashcode-before-right> <hashcode> <filename>
//查看目标文件两个版本之间的差异

git log $old_tag..$new_tag --oneline           // 打印连个tag之间的所有commit
git log commit^..commit --oneline --name-only  // 打印两个commit之间修改的文件的名字
```

## 保存现场
```
在分支dev中工作时，如果不git stash 就 git checkout master，会导致dev分支中的修改也带到mster中。使用git stash可以避免这种情况。
一般情况下，如果有bug要在master修改，但是我在dev分支的工作没有完成不能提交的情况下，可以使用git stash保存现场。
$ git stash list
一是用git stash apply恢复，但是恢复后，stash内容并不删除，你需要用git stash drop来删除；
另一种方式是用git stash pop，恢复的同时把stash内容也删了。
git stash apply stash@{0}
```

## 多人协作
```
当你从远程仓库克隆时，实际上Git自动把本地的master分支和远程的master分支对应起来了，并且，远程仓库的默认名称是origin。
要查看远程库的信息，用$ git remote origin
或者用git remote -v显示更详细的信息：
$ git remote -v
origin  git@github.com:michaelliao/learngit.git (fetch)
origin  git@github.com:michaelliao/learngit.git (push)
上面显示了可以抓取和推送的origin的地址。如果没有推送权限，就看不到push的地址。

可以手动添加新的远程仓库：
git remote add origin git@server-name:path/repo-name.git    

git push -u origin master                //-u选项把远程仓库的分支于本地分支关联上，以后再push就不需要-u了
git push origin master                   //把master分支的变化更新到远程仓库
git push origin dev                      //把dev分支的变化更新到远程仓库，注意需要事先关联上。
git checkout -b dev origin/dev           //clone后，创建远程origin的dev分支到本地
git branch --set-upstream dev origin/dev //指定本地dev分支与远程origin/dev分支的链接

总结：
1)查看远程库信息，使用git remote -v；
2)本地新建的分支如果不推送到远程，对其他人就是不可见的；
3)从本地推送分支，使用git push origin branch-name，如果推送失败，先用git pull抓取远程的新提交；
4)在本地创建和远程分支对应的分支，使用git checkout -b branch-name origin/branch-name，本地和远程分支的名称最好一致；
5)建立本地分支和远程分支的关联，使用git branch --set-upstream branch-name origin/branch-name；
6)从远程抓取分支，使用git pull，如果有冲突，要先处理冲突。
```

## 标签管理
tag就是一个让人容易记住的有意义的名字，它跟某个commit绑在一起
```
git tag -a v0.1 -m "version 0.1 released" 3628164
git show v0.1//查看tag信息
	tag v0.1
	Tagger: Michael Liao <askxuefeng@gmail.com>
	Date:   Mon Aug 26 07:28:11 2013 +0800
	version 0.1 released
git tag -s <tagname> -m "blablabla..."可以用PGP签名标签；

命令git push origin <tagname>可以推送一个本地标签；
命令git push origin --tags可以推送全部未推送过的本地标签；
命令git tag -d <tagname>可以删除一个本地标签；
命令git push origin :refs/tags/<tagname>可以删除一个远程标签。先从本的删除，再删除远程标签。
```

## 更新github fork的别人的repo
```
首先要先确定一下是否建立了主repo的远程源：
git remote -v
如果里面只能看到你自己的两个源(fetch 和 push)，那就需要添加主repo的源：
git remote add upstream URL
git remote -v
然后你就能看到upstream了。
如果想与主repo合并：
git fetch upstream
git merge upstream/master
```

## 还原commit后的某个文件
```
直接用 git-checkout 即可。理解起来稍微有点奇怪就是了。

$ git checkout ${commit} /path/to/file

http://stackoverflow.com/questions/215718/reset-or-revert-a-specific-file-to-a-specific-revision-using-git
https://www.kernel.org/pub/software/scm/git/docs/git-checkout.html
```

## 查看文件中每一行是谁修改的
```
git blame -L 1,10 nic_info
```

## patch
```
git checkout master
git checkout -b b1
edit,commid
git format-patch master

git blame -L 1,10 nic_info
git apply --check 001-patch...
git apply 001-patch...

git format-patch -1 master //为master上最近一次变更打patch，patch包含master最近一次变更的内容
git format-patch -2 b1     //为b1上最近两次变更打patch，patch包含b1最近两次变更的内容
```

## 历史命令记录
Git提供了一个命令git reflog用来记录你的每一次命令
