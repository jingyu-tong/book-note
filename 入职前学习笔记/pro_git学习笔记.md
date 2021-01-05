 # 1. git 基础

1. 文件状态

   * 未跟踪：没有被添加到 git 记录里的文件
   * 已跟踪：已经被 git 追踪的文件
     * Unmodified：上次提交后，未被修改的文件
     * modified：上次提交后，被修改的文件
     * staged：存入暂存区的文件(修改了，利用 git add 加入暂存区，但是没有被 commit，statu 显示 changes to be commited)
2. 提交更新

   * git commit：提交所有暂存区的内容，因此，每次提交前，先用 git status 看下，是不是都已经存起来了
   * -m(message): 补充提交说明
   * 删除时，可以 git rm + filename 或者在文件中删除后，再使用 git add 更新
3. 远程仓库
* 抓取数据：git fetch [remote-name]

# 2. git 分支

## 2.1 git 系统实现机理

![img](/Users/jingyu/Documents/GitHub/book-note/assets/18333fig0301-tn.png)

git 利用 blob 来表示一个存储一个文件的某个快照，利用 tree 对象来维护每个目录，commit 则记录了提交相关的信息外，还存储了指向树对象的指针，以便之后可以重现某次提交。在提交记录后，再次提交，会包含一个指向上次提交对象的指针(commit 中的 parent 指针)，形成一条链

当调用 `git branch branch-name`的时候，会创建一个分支指针指向一个 commit 记录，利用`git checkout branch-name`可以将 HEAD 指针切换到另一个分支。这种操作是的 git 会基于各种操作，形成一个有向无环图。

## 2.2 分支合并(merge)

![img](/Users/jingyu/Documents/GitHub/book-note/assets/18333fig0317-tn.png)

假设 master 分支为 c4，iss53 分支为 c5，合并会基于他两的共同祖先，和两个分支末端进行三方合并计算，并形成一个新的提交 c6。

有时候会遇到合并冲突，可以用可视化工具解决完，在提交

## 2.3 工作流程

1. 长期分支

   在 master 分支中，保留完全稳定或者要发布的代码，develop、next 等平行分支用于后续开发，当测试完某一分支后，就可以合并到主干分支。

2. 远程分支

   <img src="/Users/jingyu/Documents/GitHub/book-note/assets/18333fig0324-tn.png" alt="img" style="zoom:50%;" />

   有时候会遇到这样的问题，在本地修改本地的 master 后，远程仓库的 master 已经被更新了，可以用 git fetch origin 来更新master，合并后才能推送。

   ## 2.4 分支衍合(rebase)

   除了 merge 之外，还可以用 rebase 来整合两个分支

   ```bash
   git checkout experiment
   git rebase master
   ```

   ![img](/Users/jingyu/Documents/GitHub/book-note/assets/18333fig0329-tn.png)

   上述操作，会在 master 分支，重演 experiment 分支所进行的更改，然后合并后就可以得到一条平整的修改链（**原理就是回到二者的公共祖先，分析当前分支后续的提交对象，生成一系列文件补丁，然后以基地分支最后一个对象为新的出发点，逐步应用，改写分支的历史**）

   **一旦分支中的提交对象发布到公共仓库，就千万不要对该分支进行衍合操作（多人合作可能造成 rebase 生成的替代和本身都存在，给人造成异或）。**应该把 rebase 当成一种在推送之前清理提交历史的手段，而且仅仅衍合那些尚未公开的提交对象

# 3. 一些其他命令

1. 移动(利用 checkout 可以移动 HEAD 指针的位置)
   * checkout 除了可以移动到分支，还可以移动到某一次具体的提交
   * 使用^向上移动一个记录，可以加在引用名称后面，表示指定位置开始的移动
   * 使用~num 向上移动多个记录，同&，也可以加在名称之后

2. 撤销
   * git reset：向上移动分支，好像从来没有提交过一样
   * git revert：相当于新提交一个更改，抵消上一次更改，git 会记录这次变更
3. cherry-pick
   * 可以选择想要的提交记录，移动到当前指针之后
4. git rebase -i：可以提供交互式的方式，进行 rebase
5. git fetch：拉取远程仓库最新的更新，会修改 origin/master 引用，但是不会修改本地 master
6. git pull：由于通常的操作是 fetch 之后，将 origin/master 和本地 master 合并，因此提出了这个命令，相当于 git fetch + git merge
7. 