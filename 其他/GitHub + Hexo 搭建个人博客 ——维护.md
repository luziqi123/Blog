---
title: GitHub + Hexo 搭建个人博客 ———维护
date: 2016-2-20
---
上一章节我们已经搭建好了自己的个人博客，感觉逼格一下就高到九霄云外的你一定觉得这个界面有点儿不应景了。拉磨，这节我们就来讨论一下如何维护你的博客，但无论怎么捣鼓，持之以恒的维护和满载干活的博客才是好博客。

<!--more-->

# 主题

目前来说这个样子你肯定是没脸拿出去让别人看的，我们需要一个更好看的主题，这里给出一个下载主题的地方：[HexoThemes](https://hexo.io/themes/)  。

拿这个网站举例，点击喜欢主题的图片是预览，点击名称是进入主题的GitHub，通常来说每个主题的GitHub中都有配置的步骤了，这个并不是每个都一样，所以一定要仔细阅读他们提供的配置步骤。

但是第一步总是这样，进入到你的博客目录，将你喜欢的主题从GitHub上Clone到本地 ，这里我们clone到theme文件下的maupassant：

```
git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
```

待命令执行完毕，在你的博客文件中的theme文件中，多了一个maupassant文件。

然后打开_config.yml文件，拖动到最下面，看到theme标签，改成这样：

```
theme: maupassant
```

maupassant就是你刚才下载的主题。

按说到这里就可以了，但是该主题的README中说:

Install theme and renderers:

```java
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
$ npm install hexo-renderer-jade --save
$ npm install hexo-renderer-sass --save
```

so,你知道该怎么做了，对 ，接着执行下面两行命令。

至此，主题配置完毕了。

不。还有，打开这个主题所在的文件夹，会看到也有一个_config.yml文件，这个文件是作用于这个主题的，在这个主题的GitHub上已经写了这个文件中每个标签的意思，这个看着配置就可以了。各位看官说了，这里为什么你说的这么笼统？那我就告诉你们，我也不太会！……

至此，主题真的配置完毕了。

赶快运行看一下效果吧！

```java
// hexo s 打开本地预览 -p 5000 为选输项，如果不输入默认端口为4000，输入后可以自行指定端口。
hexo s -p 5000
```



#_config.yml 文件

```yaml
# Hexo Configuration
## Docs: http://zespia.tw/hexo/docs/configure.html
## Source: https://github.com/tommy351/hexo/

# Site 这里的配置，哪项配置反映在哪里，可以参考我的博客
title: Xiaomiya's blog #站点名，站点左上角
subtitle: Walk steps step by step #副标题，站点左上角
description: Walk steps step by step #给搜索引擎看的，对站点的描述，可以自定义
author: xiaomiya#在站点左下角可以看到
email: #你的联系邮箱
language: zh-CN #中国人嘛，用中文

# URL #这项暂不配置，绑定域名后，欲创建sitemap.xml需要配置该项
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
tag_dir: tags
archive_dir: archives
category_dir: categories

# Writing 文章布局、写作格式的定义，不修改
new_post_name: :title.md # File name of new posts
default_layout: post
auto_spacing: false # Add spaces between asian characters and western characters
titlecase: false # Transform title into titlecase
max_open_file: 100
filename_case: 0
highlight:
  enable: true
  backtick_code_block: true
  line_number: true
  tab_replace:

# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Archives 默认值为2，这里都修改为1，相应页面就只会列出标题，而非全文
## 2: Enable pagination
## 1: Disable pagination
## 0: Fully Disable
archive: 1
category: 1
tag: 1

# Server 不修改
## Hexo uses Connect as a server
## You can customize the logger format as defined in
## http://www.senchalabs.org/connect/logger.html
port: 4000
logger: false
logger_format:

# Date / Time format 日期格式，不修改
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: MMM D YYYY
time_format: H:mm:ss

# Pagination 每页显示文章数，可以自定义，我将10改成了5
## Set per_page to 0 to disable pagination
per_page: 5
pagination_dir: page

# Disqus Disqus插件，我们会替换成“多说”，不修改
disqus_shortname:

# Extensions 这里配置站点所用主题和插件，暂默认，后面会介绍怎么修改
## Plugins: https://github.com/tommy351/hexo/wiki/Plugins
## Themes: https://github.com/tommy351/hexo/wiki/Themes
theme: light
exclude_generator:
plugins:
- hexo-generator-feed
- hexo-generator-sitemap
```

上一篇只提到这个文件的作用，这一章给出了一个该文件的详细说明，对照这个吧想该的地方改成你喜欢的就可以了。

# 文章管理

这里应该算是最重要的了，其实也简单，你只需要将你的md文件放到 博客目录/source/_posts 下就OK了，hexo server一下，看是不是多了一篇文章？

需要注意的是，这里只能放md文件，顺便安利一下Typora，Mac下一款超好用的MarkDown编辑器，完全没有了预览窗口，直接用，回车，效果就出来，爽到屙血尿浓啊~

# 更新博客

到这里主题文章都有了，本地看着也差不多了，提交到GitHub吧！

这里比较重要的命令：

```
1. hexo clean 或 hexo c
2. hexo generate 或 hexo g
3. hexo deploy 或 d
```

1. 清理一下hexo项目。
2. 生成静态页面
3. 部署Hexo项目

在提交博客或更改主题的时候会用到这些命令，挨个执行，不知道能不能一下复制到终端上一次性执行。可爱的你可以试试……

好了，终于大功告成了……



# 总结

## 主题

找到喜欢的主题，然后仔细看github中README，很简单就下载并配置好了。

然后更改_config.yml中theme标签： theme:主题的文件名。

## 文章

将md博文文件放到 "博客目录/source/_posts "下，然后执行：

```
1. hexo clean 或 hexo c
2. hexo generate 或 hexo g
3. hexo deploy 或 d
```

完事儿。



最后，来一个比较正式的结束语。



至此，你的个人博客已经全部配置完毕。











