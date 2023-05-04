# code review: gerrit实战 - 知乎
[code review: gerrit 实战 - 知乎](https://zhuanlan.zhihu.com/p/69311610) 

 gerrit 是 git 的超集，实操过程中会遇到很多问题，解决起来学习曲线比较陡峭。因此本节整理了一下常见问题，给出解决方案。过程中会涉及一些原理。

### 1. 处理 gerrit 和 gitlab 不同步

gerrit 做 code review 的原理：当开发者提交一个 change 时，相当于在 gerrit 上的**临时分支**提交代码。比如在 master 修改后提交 change 等待 review，实际上代码提交到了 refs/for/master 分支上。review 完 submit 的时候，代码才会真正提交到 master 上。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/bfc208f2-c339-4394-b581-ac76af4538c7.jpeg?raw=true)

gerrit 的 gitlab 插件会将 gitlab 作为 gerrit 的 slave，每隔几秒就把最新的**真实分支**的代码提交到 gitlab 上。当有人绕过 gerrit 向 gitlab 提交代码，在插件工作时，会认为两个仓库不同步，从而不能合并。 如果要解决这个问题，我们需要把 gitlab 最新的改动手动同步到 gerrit 上。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/f8d7d98a-f9eb-43b5-bb21-eb17d7680605.jpeg?raw=true)

在本地执行：

```text
在gitlab上查看最新一次提交的分支
▶ git checkout {不同步的分支}
▶ git remote add upstream https://gitlab.com/{仓库名}/{项目名}.git
▶ git fetch upstream
▶ git merge upstream/{不同步的分支}
向gerrit提交变更，review后将自动同步到gitlab
```

### 2. 缺少 change id

change id 是 gerrit change 的唯一标识。每一个 change id 代表一个 code review 的请求。在向 gerrit push 代码时，我们经常会遇到 missing Change-Id in message footer 的报错。主要有两个原因：

### 2.1. git clone 的时候没有选择带 git hook 的方式

每次 git commit 的时候 git 会自动执行. git/hooks/commit-msg 脚本，为 commit 添加 change id。这个. git/hooks/commit-msg 脚本是在 clone 仓库的时候一起下载下来的。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/b9856bdd-1015-45c4-896e-1282c49802ba.jpeg?raw=true)

如果没有使用带 hook 的地址 clone 代码，push 时会报错：

```text
▶ git push origin HEAD:refs/for/master
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 268 bytes | 268.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, done
remote: ERROR: commit e74dfd3: missing Change-Id in message footer
remote:
remote: Hint: to automatically insert a Change-Id, install the hook:
remote:   gitdir=$(git rev-parse --git-dir); scp -p -P 29418 mengnan@172.17.202.23:hooks/commit-msg ${gitdir}/hooks/
remote: and then amend the commit:
remote:   git commit --amend --no-edit
remote:
To http://172.17.202.23:29419/a/test-webhook2
 ! [remote rejected] HEAD -> refs/for/master (commit e74dfd3: missing Change-Id in message footer)
error: failed to push some refs to 'http://mengnan@172.17.202.23:29419/a/test-webhook2'
```

如果想正确提交代码，需要根据报错提示执行操作：

```text
▶ gitdir=$(git rev-parse --git-dir);scp -p -P 29418 {username}@172.17.202.23:hooks/commit-msg${项目名}/hooks/
commit-msg                                                                                                                                                                100% 1521    70.3KB/s   00:00
▶ git commit --amend --no-edit #手动给前一个请求加上change id
[master a5822f8] add one line
 Date: Sun May 26 15:37:56 2019 +0800
 1 file changed, 1 insertion(+)
▶ git push origin HEAD:refs/for/master
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 311 bytes | 311.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, new: 1, done
remote:
remote: SUCCESS
remote:
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook2/+/24 add one line
To http://172.17.202.23:29419/a/test-webhook2
 * [new branch]      HEAD -> refs/for/master
```

### 2.2. 某些操作不会自动添加 change id

比如 pull + 自动 merge 的时候 git hook 加上了 change id，然后又自动追加了 merge 冲突的文件名，这样 change id 不在最后一行，gerrit 识别不出来会报错。

```text
commit 7104c75d78562df974d6b60183d2b8d27cbced43 (HEAD)
Merge: e7a3fb5 6d6fa16
Author: mengnan <supernan1994@gmail.com>
Date:   Sun Jun 9 16:15:45 2019 +0800

    Merge remote-tracking branch 'origin/master' into HEAD

    Change-Id: I3b8e9ce79bbfd1e7ca698be597316e8e4f78b376

    # Conflicts:
    #       main.go
```

具体报错信息如下：

```text
▶ git push origin HEAD:refs/for/master                                                                        
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 572 bytes | 572.00 KiB/s, done.
Total 4 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, done    
remote: ERROR: commit c532445: missing Change-Id in message footer
remote: 
remote: Hint: run
remote:   git commit --amend
remote: and move 'Change-Id: Ixxx..' to the bottom on a separate line
remote: 
To ssh://172.17.202.23:29418/test-webhook
 ! [remote rejected] HEAD -> refs/for/master (commit c532445: missing Change-Id in message footer)
error: failed to push some refs to 'ssh://mengnan@172.17.202.23:29418/test-webhook'
```

如果想正确提交代码，需要根据报错提示执行操作：

```text
▶ git commit --amend          
[detached HEAD b266908] Merge remote-tracking branch 'origin/master' into HEAD
 Date: Sun May 26 15:25:59 2019 +0800
▶ git push origin HEAD:refs/for/master
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 555 bytes | 555.00 KiB/s, done.
Total 4 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, new: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/25 Merge remote-tracking branch 'origin/master' into HEAD
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

### 2.3. 批量提交多个 commit，中间一个 commit 没加 change id

经常会出现这样的情况：在 pull 代码之后，没有直接提交，直接基于 pull 的结果进行了修改。最终向 gerrit 提交代码时发现中间有一个 commit 没有 change id。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/7a8b1446-1f13-4873-9a21-e8e4a88c92e6.jpeg?raw=true)

如下图所示，在这个例子中一共有三个 commit 需要提交，分别是 ea24f1d，4f99a0e，37958d1。其中 4f99a0e 没有 change id。

```text
▶ git checkout local_branch2
▶ git log --oneline
37958d1 (HEAD -> master) 第三次提交
4f99a0e Merge remote-tracking branch 'origin/master'
7a1edf0 (origin/master, origin/HEAD) 第二次提交
ea24f1d 第一次提交
```

处理这样的情况，一般有两种方式：

### 2.3.1. 多个 commit 需要分开提交到多个 change

我们需要 check 到没有 change id 的 commit，添加 change id 后手动提交。我们先提交 ea24f1d 和 4f99a0e 两个 commit。

```text
▶ git checkout 4f99a0e
# 创建一个新的分支对4f99a0e进行改动
▶ git checkout -b local_branch3
▶ git log --oneline
4f99a0e (HEAD) Merge remote-tracking branch 'origin/master'
7a1edf0 (origin/master, origin/HEAD) 第二次提交
ea24f1d 第一次提交
▶ git commit --amend --no-edit
[detached HEAD 2b9dbee] Merge remote-tracking branch 'origin/master'
 Date: Sun Jun 9 18:40:13 2019 +0800

▶ git push origin HEAD:refs/for/master
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 569 bytes | 569.00 KiB/s, done.
Total 4 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, new: 2, done    
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/64 第一次提交
remote:   http://172.17.202.23:29419/c/test-webhook/+/65 Merge remote-tracking branch 'origin/master'
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

看一下 merge 这个 change 的最后一个 commit id，我们发现已经不是原来的 4f99a0e，已经变成了 2b9dbee。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/e2e8d382-942c-4847-98a0-19482ef14b3e.jpeg?raw=true)

```text
▶ git fetch "ssh://mengnan@172.17.202.23:29418/test-webhook" refs/changes/65/65/1 && git checkout FETCH_HEAD
▶ git log
commit 2b9dbeedda57338bee6799781fa26722247df36b (HEAD)
Merge: ea24f1d 7a1edf0
Author: mengnan <supernan1994@gmail.com>
Date:   Sun Jun 9 18:40:13 2019 +0800

    Merge remote-tracking branch 'origin/master'

    Change-Id: I66766de4aa2dcbae836e4bec13b84ce66d7f3efc
```

而第三次提交 37958d1 还是基于 4f99a0e，而不是基于 2b9dbee。我们需要基于 2b9dbee 重做第三次提交。

```text
▶ git checkout local_branch2
▶ git rebase local_branch3
First, rewinding head to replay your work on top of it...
Applying: 第三次提交

▶ git push origin HEAD:refs/for/master
Enumerating objects: 12, done.
Counting objects: 100% (12/12), done.
Delta compression using up to 8 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (8/8), 1.01 KiB | 1.01 MiB/s, done.
Total 8 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2)
remote: Processing changes: refs: 1, new: 1, done    
remote: warning: no changes between prior commit 2b9dbee and new commit f360771
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/66 第三次提交
remote: 
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

### 2.3.2. 多个 commit 提交到一个 change

我们可以批量把 commit rebase 到一个 commit。rebase 多个 commit 的操作见 3.2 节讲解。

### 3. 合并 commit

### 3.1. 修改上一个 commit

假设现在完成了一个需求，准备提交 code review:

```text
▶ git commit -m 'main commit'
▶ git log --oneline
674f8ca (HEAD -> master) main commit
66c6677 (origin/master, origin/HEAD) Merge "等待被修改的change"
```

在 push 之前突然发现了一个 bug。如果正常提交，在 gerrit 上就会有两个 change。reviewer 在看第一个 change 的时候，不知道哪些是第二个已经修复的 bug，哪些是没修复的，对比起来很麻烦。 我们提倡同一个需求、相互依赖的两个改动尽量放在一个 change 里。方便 reviewer review，也会减少一些不必要的冲突。 第二个 commit 我们用 --amend 参数来修改上一个 commit：

```text
▶ git commit -m 'main commit and fix' --amend
[master 81598cd] main commit and fix
 Date: Sun May 26 16:51:35 2019 +0800
 1 file changed, 1 insertion(+)
 ▶ git log --oneline
81598cd (HEAD -> master) main commit and fix
66c6677 (origin/master, origin/HEAD) Merge "等待被修改的change"
```

这样就会作为一个 change 提交了

```text
▶ git push origin HEAD:refs/for/master
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 604 bytes | 604.00 KiB/s, done.
Total 4 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, new: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/29 main commit and fix
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

### 3.2. 合并多个 commit

有时当我们 git pull/git merge 的时候，解决冲突后，会自动生成没有加 change id 的 commit，夹在多个 commit 中间（见 2.3 节的情况）；或者 commit 忘记加 --amend 的时候，在提交 gerrit 的时候会变成多个 changes。我们可以用 rebase 操作合并多个 commit。

假如现在有三个 commit(19e38cb, 4bea9f9, 710b142) 我们希望合并为一个 commit:

```text
▶ git log --oneline
19e38cb (HEAD -> master) 等待被合并的提交3
4bea9f9 等待被合并的提交2
710b142 等待被合并的提交1
66c6677 (origin/master, origin/HEAD) Merge "等待被修改的change"
bfe60ee 等待被修改的change
743bc04 Merge "Merge branch 'master' of ssh://172.17.202.23:29418/test-webhook"
12e2855 Merge changes Ic0e4291c,I178ede01
460190e Merge remote-tracking branch 'origin/master' into HEAD
b266908 Merge remote-tracking branch 'origin/master' into HEAD
0b732d7 Merge branch 'master' of ssh://172.17.202.23:29418/test-webhook
51fde7e Merge "pl"
```

可以用 rebase 批量合并最近 3 个 commit

```text
▶ git rebase -i HEAD~3
pick 710b142 等待被合并的提交1
pick 4bea9f9 等待被合并的提交2
pick 19e38cb 等待被合并的提交3

# Rebase 66c6677..19e38cb onto 66c6677 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
"~/Works/programs/SaaS/test-webhook/.git/rebase-merge/git-rebase-todo" 29L, 1253C
```

把 pick 修改为 s(squash)

```text
pick 710b142 等待被合并的提交1
s 4bea9f9 等待被合并的提交2
s 19e38cb 等待被合并的提交3
```

然后保存。本地会由上至下重做 3 个 commit。

```text
# This is a combination of 3 commits.
# This is the 1st commit message:

等待被合并的提交1

Change-Id: Ic34f22bbd03ff1f482fdbb559dce6ad86e9a9524

# This is the commit message #2:

等待被合并的提交2

Change-Id: I1ac2cd32b9894e0a7e21da726e031f96ecfeca2b

# This is the commit message #3:

等待被合并的提交3

"~/Works/programs/SaaS/test-webhook/.git/COMMIT_EDITMSG" 37L, 913C
```

成功合并：

```text
▶ git rebase -i HEAD~3             
[detached HEAD 83835ff] 等待被合并的提交1
 Date: Sun May 26 16:05:13 2019 +0800
 1 file changed, 1 deletion(-)
Successfully rebased and updated refs/heads/master.
▶ git log --oneline
83835ff (HEAD -> master) 等待被合并的提交1
66c6677 (origin/master, origin/HEAD) Merge "等待被修改的change"
bfe60ee 等待被修改的change
743bc04 Merge "Merge branch 'master' of ssh://172.17.202.23:29418/test-webhook"
12e2855 Merge changes Ic0e4291c,I178ede01
460190e Merge remote-tracking branch 'origin/master' into HEAD
```

### 4. 修改已提交的 change

开发者正常提交了一个 change 到 gerrit，reviewer 看过之后觉得有些地方需要修改，这个时候开发者需要修完之后继续提交新的 patchset 到原来的 change 里，方便 reviewer 对比。 主要分三种情况：  
1）修改的 change 是最后一个 change，后面再没有修改了  
2）修改的 change 后面还有其他 change，改完需要把这个修改同步到后面的 change 上，这种情况比较复杂  
3）修改的 change 后面还有其他 change，改完需要把这个修改同步到后面的 change 上，但最新的改动和后面的 change 有冲突，复杂程度 MAX

### 4.1 修改最新提交的 change

假设现在正常提交了一个 change：

```text
▶ git commit -m '等待被修改的commit'
[master 68e2e71] 等待被修改的commit
 1 file changed, 1 deletion(-)
▶ git push origin HEAD:refs/for/master
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 331 bytes | 331.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, new: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/31 等待被修改的commit
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

在 gerrit 上能看到这个新的 change。我们可以点击右下角 update change 获取修改 change 的提示。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/d95aff33-3165-4dc5-ab81-95419638b95a.jpeg?raw=true)

按照提示操作：

```text
▶ git fetch "ssh://mengnan@172.17.202.23:29418/test-webhook" refs/changes/31/31/1 && git checkout FETCH_HEAD
From ssh://172.17.202.23:29418/test-webhook
 * branch            refs/changes/31/31/1 -> FETCH_HEAD
Note: checking out 'FETCH_HEAD'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 68e2e71 等待被修改的commit
▶ git add main.go &&   git commit --amend --no-edit
[detached HEAD ca2b767] 等待被修改的commit
 Date: Sun May 26 17:17:08 2019 +0800
 1 file changed, 2 insertions(+)
▶ git push origin HEAD:refs/for/master
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 338 bytes | 338.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, updated: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: Updated Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/31 等待被修改的commit
remote: 
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

可以看到提交了新的 patchset

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/4e718626-43c8-4495-abdd-a5dec77bf392.jpeg?raw=true)

可以点击对比，查看第二次修改的内容

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/58d1272d-22ad-4d83-9a6b-e2a8c25a5d69.jpeg?raw=true)

### 4.2 有其他 change 依赖当前修改的 change（没有冲突）

假设我们连续提交了三个 change，第一个有 bug。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/97a89ff7-64b6-4788-861c-ec1071bc05fc.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/73cc2177-47db-4ea3-b60a-8e8f3c1c3bca.jpeg?raw=true)

```text
▶ git log --oneline
96dc449 (HEAD -> master) 第三次提交
8b256f6 第二次提交
7abedfb 第一次提交，有bug
5fda3c0 (origin/master, origin/HEAD) Merge remote-tracking branch 'origin/master'

▶ git push origin HEAD:refs/for/master
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 8 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (9/9), 900 bytes | 900.00 KiB/s, done.
Total 9 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3)
remote: Processing changes: refs: 1, new: 3, done    
remote: 
remote: SUCCESS
remote: 
remote: New Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/79 第一次提交，有bug
remote:   http://172.17.202.23:29419/c/test-webhook/+/80 第二次提交
remote:   http://172.17.202.23:29419/c/test-webhook/+/81 第三次提交
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/19ef8db7-f92c-4cfd-a9b8-803f7475074c.jpeg?raw=true)

我们按照 4.1 节的操作，修复这个 bug。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/7e74713e-c58d-4df2-a5de-24a91dd41024.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/7d96f753-4847-4300-a102-6cb048805035.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/99a2527a-3d2e-4151-b23b-417efd31f2ea.jpeg?raw=true)

会发现剩下的 change2, change3 旁边出现了神奇的绿色字体 (Indirect ancestor)，点击进入 change2 也会看到神奇的红色字体 (Not current)。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/d0264471-5eeb-4de8-9ecd-bb13dde7706b.jpeg?raw=true)

如果这时合并了 change1，会发现 change2 冲突了，即使 change1 和 change2 没有修改同一行代码。这是因为对于 change1 patch2 修改的代码行，已经合并的 change1 patch2 的代码和 change2 上的代码是不一样的。我们可以在合并 change1 之前把 change2 和 change3 调整为基于 change1 的 patch2，否则合并后解决冲突就请参照 4.3 吧。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/27750f18-541f-4782-a5de-0e768d149671.jpeg?raw=true)

我们可以进入 change2 页面，点击右上角 rebase，大功告成。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/879c31ca-c008-4895-ae4d-4e2a79dca1ed.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/2da64580-80c2-4c49-8805-c7d130e4cad7.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/109bf5d6-545e-456b-8234-0e5be6629ecb.jpeg?raw=true)

同样处理 change3：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/c6a40854-d05a-4de8-9559-c08d33de5613.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/0eefce6c-0f2a-41cc-8125-ff062e8c2906.jpeg?raw=true)

### 4.3 有其他 change 依赖当前修改的 change（发生冲突）

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/8c1939f3-7d0d-4323-ab1d-0c4d927eb6f7.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/54c300df-bf46-4bc2-9f9b-291715cbfbcf.jpeg?raw=true)

只要 change1 的 patch2 和 change2 的 patch1 同时对一行做了修改，gerrit 会认为是冲突的。

-   当 change1 还没有 merge 的情况下，在 gerrit 页面上基于 change2 rebase 会提示冲突。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/e13b7414-927a-46ea-a4e3-8d0f266dd7b5.jpeg?raw=true)

-   当 change1 已经 merge 的情况下，change2 页面上会直接显示冲突。submit 时提示无法合并。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/8f724989-c077-4055-89e0-227f2e9f399b.jpeg?raw=true)

我们需要分别在 change2 和 change3 上 rebase change1 最新的改动，在 rebase 过程中解决冲突。 先解决 change2 的冲突：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/251012ea-70dd-4314-b35d-f9fe48f35992.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/fd0af8eb-279e-4443-abe3-43d7e33f03d5.jpeg?raw=true)

```text
# check到change2上，创建一个新本地分支local_branch3
▶ git fetch "ssh://mengnan@172.17.202.23:29418/test-webhook" refs/changes/74/74/1 && git checkout FETCH_HEAD
From ssh://172.17.202.23:29418/test-webhook
 * branch            refs/changes/74/74/1 -> FETCH_HEAD
Note: checking out 'FETCH_HEAD'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at a343549 第二次提交
▶ git checkout -b local_branch3
Switched to a new branch 'local_branch3'

# 在change2上应用change1的fix
▶ git rebase -i local_branch2
# 此处省略一系列merge的操作
▶ git rebase --continue
Successfully rebased and updated refs/heads/local_branch3.

▶ git push origin HEAD:refs/for/master
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 327 bytes | 327.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: Processing changes: refs: 1, updated: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: Updated Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/74 第二次提交
remote: 
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

第二次提交的冲突已经解决：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/b1bc3cdb-4830-4b27-859f-009f19f13840.jpeg?raw=true)

我们用同样的方法解决 change3 的冲突，不过这次是需要 change3 上应用 change2。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/fdacf691-fa4b-4b2a-8f60-f7b931df830d.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/99b013a8-9334-48ad-9c30-be9901375723.jpeg?raw=true)

```text
# check到change3上，创建一个新本地分支local_branch4
▶ git fetch "ssh://mengnan@172.17.202.23:29418/test-webhook" refs/changes/75/75/1 && git checkout FETCH_HEAD
From ssh://172.17.202.23:29418/test-webhook
 * branch            refs/changes/75/75/1 -> FETCH_HEAD
Note: checking out 'FETCH_HEAD'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at fa14773 第三次提交
▶ git checkout -b local_branch4
Switched to a new branch 'local_branch4'

# 在change3上应用change2的fix
▶ git rebase -i local_branch3
# 此处省略一系列merge的操作
▶ git rebase --continue
Successfully rebased and updated refs/heads/local_branch4.

▶ git push origin HEAD:refs/for/master                                                                      
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 8 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (5/5), 620 bytes | 620.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0)
remote: Processing changes: refs: 1, updated: 1, done    
remote: 
remote: SUCCESS
remote: 
remote: Updated Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/75 第三次提交
remote: 
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

去 gerrit 的页面上看，第三次提交也不冲突了。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/72349604-3c2c-4c86-92a5-93bcb4d66304.jpeg?raw=true)

### 5. 发生冲突

4.2 节中我们主要讨论了对 parent change 改动引起的冲突，本节我们主要讨论多人改动引起的冲突。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/79398490-2a53-41f5-b091-9b9ea1f4ce87.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/e630fafc-88a0-411e-9362-49e7237f704d.jpeg?raw=true)

当两个人同时基于一个分支进行修改，提交了两个冲突的 change。

-   未合并时，gerrit 会提示：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/e6afa548-a605-4ae1-910e-905a87ea0e55.jpeg?raw=true)

-   当其中一个合并之后，另一个会提示冲突：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/0f07536b-a9f8-49a4-8956-ec45819a41a0.jpeg?raw=true)

不管有没有合并，我们都可以用同样的方式解决冲突。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/86e24a5d-8946-446a-ba1c-6abceb0f40a7.jpeg?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/85b93616-ddc9-4942-a56f-3af6ee1c3f27.jpeg?raw=true)

```text
# 下载change1（未合并）或master（已合并）
▶ git fetch "ssh://mengnan@172.17.202.23:29418/test-webhook" refs/changes/88/88/1 && git checkout FETCH_HEAD
From ssh://172.17.202.23:29418/test-webhook
 * branch            refs/changes/88/88/1 -> FETCH_HEAD
Note: checking out 'FETCH_HEAD'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 514d1c5 第一个人的改动
▶ git checkout -b local_branch1  
Switched to a new branch 'local_branch1'

# 在change2上应用change1
▶ git rebase local_change1            
# 省略merge操作
▶ git rebase --continue
Applying: 第二个人的改动

▶ git push origin HEAD:refs/for/master
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 352 bytes | 352.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2)
remote: Processing changes: refs: 1, updated: 1, done    
remote: warning: 15787e8: no files changed, was rebased
remote: 
remote: SUCCESS
remote: 
remote: Updated Changes:
remote:   http://172.17.202.23:29419/c/test-webhook/+/87 第二个人的改动
remote: 
To ssh://172.17.202.23:29418/test-webhook
 * [new branch]      HEAD -> refs/for/master
```

成功解决冲突：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-4%2015-34-51/cc1025a1-53ce-4df8-b2c7-93160ed04f98.jpeg?raw=true)

## 参考资料

-   [http://wincent.github.io/gerrit-best-practices-tech-talk/assets/fallback/index.html](https://link.zhihu.com/?target=http%3A//wincent.github.io/gerrit-best-practices-tech-talk/assets/fallback/index.html)
-   [https://www.algoclinic.com/gerrit-best-practices.html](https://link.zhihu.com/?target=https%3A//www.algoclinic.com/gerrit-best-practices.html)
-   [https://juejin.im/post/5c2d7377518825544d43dfa5#heading-39](https://link.zhihu.com/?target=https%3A//juejin.im/post/5c2d7377518825544d43dfa5%23heading-39)
