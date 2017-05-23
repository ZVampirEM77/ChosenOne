1. 只想修改最近一次提交说明：\
> $ git commit --amend

2. 发现有漏掉的文件没有提交：\
这里需要说明的一点是，\
> $ git commit --amend\
此命令将使用当前的暂存区快照提交。如果刚才提交完没有作任何改动，直接运行此命令的话，相当于有机会重新编辑提交说明，但将要提交的文件快照和之前的一样。\
如果刚才提交时忘了暂存某些修改，可以先补上暂存操作，然后再运行--amend提交：
> $ git commit -m 'initial commit'
> $ git add forgotten_file
> $ git commit --amend

3. git reset命令的三种方式：\
- git reset --mixed     此为默认方式，不带任何参数的git reset，就是这种方式，它回退到某个版本，**只保留源码**，回退commit和stage信息\
- git reset --soft      回退到某个版本，只回退了commit的信息，不会回复stage (如果还要提交，直接commit即可)\
- git reset --hard      彻底回退到某个版本，本地的源码也会变为上一个版本的内容\

4. git add -p
执行git add -p会进入一个交互界面，git会针对每个修改的hunk询问你是否stage this hunk or not stage this hunk。将一次commit拆分为多个commit的精髓在于此。

