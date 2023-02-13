# (14条消息) git与gerrit基础概念_Kris_u的博客-CSDN博客_git gerrit
[(14 条消息) git 与 gerrit 基础概念\_Kris_u 的博客 - CSDN 博客\_git gerrit](https://blog.csdn.net/qq_33690342/article/details/115870697) 

-   本文记录了 git 与 gerrit 学习所得
-   重点关注于当前所用到的实际操作部分，其余理论部分以及更复杂用法留待将来用到时继续补充


-   Git 是当前全世界流行的[分布式](https://so.csdn.net/so/search?q=%E5%88%86%E5%B8%83%E5%BC%8F&spm=1001.2101.3001.7020)版本控制工具，但是只适用于纯文本文件，包括 markdown、网页、代码等，一般不用于图片、视频、.doc 文档等
-   实际上是在当前目录下新建一个名为 .git 的**隐藏文件夹**，作为本地仓库 / 版本库（Repository），**切记不可手动直接修改内容**
-   Gerrit = Git + 强制代码审核
-   只有通过了代码审核，才能提交至远程仓库然后合并（merge）

## 2.1 一些概念

### 2.1.1 四块区域

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/f28b6f2c-ecb5-4c1b-bc3f-0b9277fe2644.png?raw=true)

git 在具体使用中，可分为三个区域：

-   工作区（Working Directory）：实际代码文件等存放的地方
-   暂存区（Stage / Index）：已经完成修改，等待最终提交文件的存放之处
-   本地仓库 / 版本库（Repository）：当前分支的最终更改提交处

加上远程仓库，实际可视为使用不同命令，实现文件在四个区域之间来回传输

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/5e5a769c-ceeb-452f-a5f2-1c39585be5c6.png?raw=true)

### 2.1.2 本地与远程

-   相较于传统的 SVN 等集中式版本控制系统，git 的最大优势在于无论有无网络，皆可工作：即可以把仓库建在本地，也可以建在远程服务器
-   可以先在本地修改，完成后推送到远程服务器，或者是不通过服务器直接互相交换
-   远程仓库既可以作为备份，又可以让其他人通过该仓库来协作
-   GitHub 上免费托管的 Git 仓库，任何人都可以看到（但只有你自己才能改）因此不能把敏感信息放进去

### 2.1.3 分支

-   每一次 commit 可视为增加一个时间节点，因此整个写代码、改代码、提交代码的过程可以串成一条时间线
-   这条时间线即为默认状态下所用的 master 分支
-   当多人合作，或者避免在主线上做改动，可以新建分支，例如 develop 分支：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/47f413ec-9cf6-4449-b684-61d8f4b6d054.jpeg?raw=true)

-   在 develop 分支中的修改完成后可以与 master 中的代码 合并（merge），然后删掉 develop 分支
-   也可以继续增加分支适应不同删改需求：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/0aef6dea-c8be-4a6c-b7cf-d377e5a8bee3.jpeg?raw=true)

## 2.2 流程

### 2.2.1 初始化

-   用户配置

    -   git config --global user.name "Your Name"
    -   git config --global user.email "[email@example.com](mailto:email@example.com)"
    -   加了 “--global” 后，本台机器所有仓库都使用这个用户配置
-   仓库建立

    -   git init：建立仓库
    -   只能监控当前目录与子目录下的文件更改

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/250660a8-fee8-4f78-a0aa-15b4eb9b69f7.jpeg?raw=true)

### 2.2.2 提交修改

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/4f647a52-8810-4c30-8ea4-3a0704b811c0.jpeg?raw=true)

### 2.2.3 撤回修改

1.  文件在版本库 / 仓库需要撤销：

-   即，版本回退，流程如下：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-13%2009-54-27/260bf054-ff07-4690-98a1-095ca8c82eb6.jpeg?raw=true)

1.  文件在暂存区，需要撤销更改且需要退回工作区

-   git reset HEAD &lt;file>
-   撤销一次 git add，即将暂存区中的文件撤回到工作区

1.  文件在暂存区，需要撤销更改但不需要退回工作区

-   git check --&lt;file>
-   撤销回到最近一次 git add 状态，即刚 add 时的状态

1.  文件在工作区，需要撤销更改

-   git check --&lt;file>
-   撤销回到最近一次 git commit 状态，即与版本库 / 仓库中相同的版本

## 2.3 命令

-   git init：在当前目录下新建本地仓库
-   git pull：取回远程仓库代码，并于本地的合并

    -   实际上 git pull = git fetch + git merge


-   git add &lt;file>：将 file 加入监控
-   git status：查看暂存区状态
-   git diff：查看修改（缓存区文件 V.S. 原始文件）
-   git reset：将暂存区的文档退回至工作区
-   git commit -m "&lt;说明>"：将暂存区中的文件提交至本地仓库，并附上 &lt; 说明 >
-   git log：查看上传记录（只有当次的操作记录）

    -   \--pretty=oneline：查看上传记录，且每条记录缩写至一行内
-   git reflog：查看所有历史命令与版本提交号
-   git checkout：查看当前分支

    -   \-a：查看所有分支
    -   &lt;分支>：切换到 &lt; 分支 >
