git clone http.....

创建一个文件...

git add .

git status

git commit -m 提交说明

git push origin master

git remote -v

git remote add [name] http.....



将远程库文件同步到本地

git pull --rebase origin master





git init 是必须的

如果你已经在git上有了一个项目，那么需要建立通道 git remote [name] http://......

建立通道后，使用git remote updata [name]来更新通道

然后git pull [name]_[master]将git上的代码同步到本地

- pull提交的时候出现了fatal: refusing to merge unrelated histories错误，网上说是因为他们是两个不同的项目，要把两个不同的项目合并，git需要添加一句代码，在`git pull`，这句代码是在git 2.9.2版本发生的

  最新的版本需要添加`--allow-unrelated-histories` 命令变成了这样：

  git pull origin master --allow-unrelated-histories







