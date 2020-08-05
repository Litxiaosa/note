```shell
git init 初始化git

git branch 查看分支

git branch <branchName> 新建分支

git branch -d <branchName> 删除分支

git push -u origin master 推送到master分支

git push -u -f origin master 强制推送到远程仓库

git pull 远程拉取

git branch -r 查看远程分支

git push origin HEAD -u 分支推送到远程

git status 查看未添加的文件

git add . 添加文件

git commit -m '项目初始化' 提交到本地仓库

git remote add origin https://gitee.com/maxlangod/motown_mall.git 提交到远程仓库

git reset –hard 回退本地版本

git reset –hard, git push -f 回退远程分支

git push origin feature-branch:feature-branch //推送本地的feature-branch(冒号前面的)分支到远程origin的feature-branch(冒号后面的)分支(没有会自动创建)


mkdir ceshi_ce
cd ceshi_ce
git init
touch README.md
git add README.md
git commit -m "first commit"
git remote add origin https://gitee.com/gean_807/ceshi_ce.git
git push -u origin master

```

