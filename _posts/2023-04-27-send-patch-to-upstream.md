---
layout: article
tags: Linux
title: send patch to upstream
mathjax: true
key: Linux
---

[Reference](https://blog.csdn.net/jcf147/article/details/123719000)
{:.info}

## send patch to linux upstream

```
# install git and setup git
yum install git
yum install git-email

git config --global user.name "xxx"
git config --global user.email "xxx@xxx.com"

# 配置 ~/.gitcommit文件
selftests: bonding: add tests for ether type changes
 
Why I do these changes and how I do it.
 
Signed-off-by: Liang Li <liali@redhat.com>

# update code
git status
git add .
git commit -s /* -s：自动在commit中添加Signed-off-by行 */

# generate patch
git  format-patch  --subject-prefix='PATCH'  -1
git format-patch -n --subject-prefix="PATCH net" -1 
    -n, --numbered
           Name output in [PATCH n/m] format, even with a single patch.
    Extract three topmost commits from the current branch and format them as e-mailable patches:
           $ git format-patch -3

a.获取文件作者：
git blame [filename] 查看文件的每一行作者是谁
b. 获取maintainer信息，内核源码树中执行如下命令：
./scripts/get_maintainer.pl [patchname]
c. 检查patch, 提交patch前检查是否有错误
./scripts/checkpatch.pl [patchname]

# have a test first
git send-email --to "haliu@redhat.com" --cc "liali@redhat.com" 0001-selftest-bonding-delete-unnecessary-line.patch

# 既然 Patch 已经测试完毕，那么是时候发送给上游维护者了。运行以下命令（二选一均可）找出你应该把 Patch 发给谁。
./scripts/get_maintainer.pl  -f  drivers/vfio/vfio_iommu_type1.c
./scripts/get_maintainer.pl  0001-xxxx.patch

# send patch
git send-email --to alex.williamson@redhat.com \
-cc cohuck@redhat.com \
-cc kvm@vger.kernel.org \
-cc linux-kernel@vger.kernel.org
git send-email --to "netdev@vger.kernel.org" --cc "liali@redhat.com" --cc "j.vosburgh@gmail.com" --cc razor@blackwall.org 0001-selftests-bonding-delete-unnecessary-line.patch

# 静静的等待维护者的邮件通知吧，一般几天之内就会回复邮件然后表示Apllied，Thanks或告知预计要合入到下一版本的如linux-5.18，有时第二天就回复一般是patch有问题。
#如果patch有问题，需要回复邮件说明疑问，或直接按maintainer的要求修改补丁变成V2版本再次提交。再次提交V2版本需要注意在补丁说明中添加v1->v2的变化（patch中---分隔符之后）：
<commit message>
...
Signed-off-by: Author <author@mail>
---
V2 -> V3: Removed redundant helper function
V1 -> V2: Cleaned up coding style and addressed review comments
 
path/to/file | 5+++--
...

# 如果是回复补丁的话，可以按照如下格式发送新版patch或说明txt
 git send-email \
    --in-reply-to=20220615073348.6891-1-xxx@xxx.com \                  (Message-ID)
    --to=xxx@xxx.com \
    --cc=xxx@xxx.org \
    /path/to/YOUR_REPLY

其中，Message-ID可以在任一带有官方性质的邮件记录网址如（Project List - Patchwork，All of lore.kernel.org，Projects | Patchew 等等）查看，一般是“时间戳+邮件地址”的形式，最后是你的补丁或文本注释。

```
