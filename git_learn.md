## git  几个常用的命令 ##

###添加远程库###
    $ git remote add origin git@github.com:xxx/xxx.git
###从远程库克隆###
    $ git clone git@github.com:xxx/xxx.git
###添加版本控制###
    $ git add readme.txt
###查看状态###
    $ git status
###提交文件###
    $ git commit -m "your commit message show in log"
###同步到远程库###
    $ git push -u origin master
###查看log###
    $ git log
    如果嫌输出信息太多，看得眼花缭乱的，可以试试加上--pretty=oneline参数：
    $ git log --pretty=oneline
###回退版本###
    回退到上一个版本$ git reset --hard HEAD^
    上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100
    $ git reflog 用来记录你的每一次命令 被回滚的版本号可以找回
    HEAD指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令git reset --hard commit_id
###查看修改###
    $ git diff HEAD -- readme.txt 查看工作区和版本库里面最新版本的区别：
###撤销###
    $ git checkout
    恢复暂存区的指定文件到工作区
    $ git checkout [file]
    恢复某个commit的指定文件到暂存区和工作区
    $ git checkout [commit] [file]
    恢复暂存区的所有文件到工作区
    $ git checkout .
    重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
    $ git reset [file]
    重置暂存区与工作区，与上一次commit保持一致
    $ git reset --hard
    重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
    $ git reset [commit]
    重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
    $ git reset --hard [commit]
    重置当前HEAD为指定commit，但保持暂存区和工作区不变
    $ git reset --keep [commit]
###分支###
    列出所有本地分支
    $ git branch
    列出所有远程分支
    $ git branch -r
    列出所有本地分支和远程分支
    $ git branch -a
    新建一个分支，但依然停留在当前分支
    $ git branch [branch-name]
    新建一个分支，并切换到该分支
    $ git checkout -b [branch]
    新建一个分支，指向指定commit
    $ git branch [branch] [commit]
    新建一个分支，与指定的远程分支建立追踪关系
    $ git branch --track [branch] [remote-branch]
    切换到指定分支，并更新工作区
    $ git checkout [branch-name]
    切换到上一个分支
    $ git checkout -
    建立追踪关系，在现有分支与指定的远程分支之间
    $ git branch --set-upstream [branch] [remote-branch]
    合并指定分支到当前分支
    $ git merge [branch]
    选择一个commit，合并进当前分支
    $ git cherry-pick [commit]
    删除分支  
    $ git branch -d [branch-name]
    删除远程分支
    $ git push origin --delete [branch-name]
    $ git branch -dr [remote/branch]

    删除已经被删除的分支
    git remote prune origin
    git remote show origin 来查看有关于origin的一些信息，包括分支是否tracking。

    $ ssh -T git@gitlab.ds.XX.com.cn
    ssh-keygen -t rsa -C "XX@XXX.com"
