# Git笔记

0. PRE  
工作区（working tree）——暂存区（index/stage）——仓库（repository）

1. 创建仓库  
git init  
会根据当前的目录创建一个本地仓库  
git remote add  
创建仓库后可以关联空的远程仓库  
git push -u  
将本地创建的仓库推送至远程仓库

2. 克隆远程仓库  
git clone

3. 提交至暂存区  
git add  
git status
查看状态  
git diff  
对比工作区、暂存区  
git rm
提交删除

4. 提交至仓库  
git commit  
git status  
查看状态  
git diff --cached  
对比暂存区、仓库  
git diff HEAD  
对比工作区、仓库

5. 撤销工作区修改  
git checkout -- \<filename>

6. 撤销暂存区修改  
git reset HEAD

7. 撤销本地仓库修改  
git reset --hard
git log  
查看提交日志  
git reflog  
查看命令日志

8. 创建分支  
git branch dev

9. 切换分支  
git switch dev（git checkout dev）  
创建并切换分支  
git switch -c dev（git checkout -b dev）

10. 查看分支  
git branch

11. 合并分支  
git merge dev  
git merge --no-ff dev  
忽略ff模式(若分支没有冲突，默认启动）

12. 删除分支  
git branch -d dev

