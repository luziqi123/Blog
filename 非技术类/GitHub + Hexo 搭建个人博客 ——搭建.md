---
title: GitHub + Hexo 搭建个人博客 ———搭建
date: 2016-2-20

---
说来你们可能不信，这破玩意儿我捣鼓了一天……知道为什么？因为我至少看到了N种配置方法，然后我把这N种配置方法融会贯通，取之精华，弃之拖沓。含泪写了这么一篇博客。
希望你们看完能少走弯路，希望我不会成为你们的弯路……

<!--more-->

#配置环境

- mac OS
- Git
- Hexo 3.0+
- Note.js
- npm
- GitHub账号
- xcode

这里所有的操作流程都是在mac上做的，但是Windows也可以参考。

#你需要知道的

在配置之前我想先把全部流程简要叙述一下，以免在某个环节懵逼， 同时你几乎可以按照如下的说明完成部分的准备工作：
1.  创建GitHub账号，并创建一个项目，项目的名称格式必须是：你的用户名.github.io，这将会是你今后访问博客的域名，不要妄想使用自己喜欢的名字。在创建项目的时候会让你填写或勾选一些东西，除项目名称之外，其他默认就好。
2.  安装Xcode -> 安装Note.js -> 安装Hexo （注意这里的顺序, xcode中集成了git，所以只装这个就可以了 ）。
3.  在本地创建文件夹：你的用户名.github.io 接下来对博客所有的相关操作都是在这里完成的，为了方便描述，在后面我将称这个文件夹为博客文件。
4.  进入博客文件，初始化Hexo。
5.  将Hexo部署到你的GitHub。

xcode安装命令：

```
xcode-select --install
```



接下来我们将直接从Hexo的安装开始，其余操作都可以轻松并准确的在百度得到答案。


#开始Hexo的安装

到了这里你应该已经成功安装了Git、Note.js，同时拥有了自己的GitHub账号，并且按照上面的命名格式创建了一个项目。现在开始正式对Hexo进行配置了。

##安装Hexo
打开终端，进入root模式
```
su
```
安装Hexo
```
npm install hexo-cli -g
```
命令执行完毕后执行
```
npm install --save
```
至此，Hexo已经安装完毕。

##初始化你的博客文件

进入到你的博客文件，记住，这个文件的名字要跟你GitHub上的项目名称一样。(其实这里我不是太确定是不是要一样……)
```
cd yourBlogFilePath
```
初始化Hexo目录
```
hexo init
```
待命令执行完毕，你会发现你的目录下多了好多文件。
如：_config.yml 、themes等，他们的主要作用如下：

- _config.yml ——是博客主要配置文件
- db.json——是博客数据库
- node_modules——是NodeJS依赖模块
- source——是博客内容以及其他页面（page）存在的目录，这个目录里面有个_post目录就是我们存放博客内容的地方，也就是存放博客内容markdown文档地方，输入hexo new “newPage”就会在这个目录建立一个名为newPage的子目录，然后在里面放入md文档，并在主题的配置文件里面添加相应栏目newPage，这样就会显示在主页面的目录上。（将在后续有所mark）
- themes——是主题存放文件
  生成静态页面：
```
hexo generate 或 hexo g
```

现在你的博客已经可以在本地预览了：
```
hexo server 或 hexo s
```
你会看到一个发光的地球，
至此，初始化完毕。

##部署到GitHub

现在我们要将博客部署到GitHub上了。
打开我们上面提到的_config.yml，拉到最下面，会看到这样的字样：
```
deploy:
     type:
```
把他改成如下这样（示例）：
```
deploy:
     type: git 
     repo: https://github.com/你的GitHub账号/你的项目名称.git
     branch: master
```

- type：如果你是Hexo3.0以上的版本，就写git就OK，如果是3.0之前的版本，要写成github。这是我听说的。

至此，你的本地博客文件从某种意义上来讲已经和Git关联了，你需要做的就是提交了。

执行命令：
```
npm install hexo-deployer-git --save
```
字面上意思貌似又安装了一个hexo-deployer-git，应该是可以让Hexo支持提交到git的操作。我也不知道……

待执行完毕，接着执行下面的命令：
```
hexo deploy
```
将项目部署到GitHub。
访问：https://你的GitHub账号.github.io/ 就可以和那个发光的地球重逢了。
至此，部署完毕。

下一节我们将继续讲解如何管理你的博客，在那里你可以知道如何更换自己想要的博客主题。


##总结

说来你们可能不信，这破玩意儿我捣鼓了一天……知道为什么？因为我至少看到了N种配置方法，然后我把这N种配置方法融会贯通，取之精华，弃之拖沓。含泪写了这么一篇博客。
希望你们看完能少走弯路，希望我不会成为你们的弯路……

[主要参考](http://blog.csdn.net/u010053344/article/details/50689446)
[辅助参考](http://www.jianshu.com/p/465830080ea9)









