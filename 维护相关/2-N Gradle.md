#### 什么是Gradle?

> > **Gradle**是一个基于[Apache Ant](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Apache_Ant)和[Apache Maven](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Apache_Maven)概念的项目[自动化建构](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/%E8%87%AA%E5%8B%95%E5%8C%96%E5%BB%BA%E6%A7%8B)工具。它使用一种基于[Groovy](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Groovy)的[特定领域语言](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/w/index.php%3Ftitle%3D%E7%89%B9%E5%AE%9A%E9%A2%86%E5%9F%9F%E8%AF%AD%E8%A8%80%26action%3Dedit%26redlink%3D1)来声明项目设置，而不是传统的[XML](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/XML)。当前其支持的语言限于[Java](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Java)、[Groovy](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Groovy)和[Scala](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Scala)，计划未来将支持更多的语言。

在linux上，有一个工具叫make。我们可以通过编写Makefile来执行工程的构建。windows上相应的工具是nmake。这个工具写起来比较罗嗦，所以从早期，Java的构建就没有选择它，而是新建了一个叫做ant的工具。ant的思想和makefile比较像。定义一个任务，规定它的依赖，然后就可以通过ant来执行这个任务了。

但是ant有一个很致命的缺陷，那就是没办法管理依赖。我们一个工程，要使用很多第三方工具，不同的工具，不同的版本。每次打包都要自己手动去把正确的版本拷到lib下面去，不用说，这个工作既枯燥还特别容易出错。为了解决这个问题，maven闪亮登场。

maven最核心的改进就在于提出仓库这个概念。我可以把所有依赖的包，都放到仓库里去，在我的工程管理文件里，标明我需要什么什么包，什么什么版本。在构建的时候，maven就自动帮我把这些包打到我的包里来了。我们再也不用操心着自己去管理几十上百个jar文件了。

maven已经很好了，可以满足绝大多数工程的构建。那为什么我们还需要新的构建工具呢？第一，maven是使用xml进行配置的，语法不简洁。第二，最关键的，maven在约定优于配置这条路上走太远了。就是说，maven不鼓励你自己定义任务，它要求用户在maven的生命周期中使用插件的方式去工作。这有点像设计模式中的模板方法模式。说通俗一点，就是我使用maven的话，想灵活地定义自己的任务是不行的。基于这个原因，gradle做了很多改进。

gradle并不是另起炉灶，它充分地使用了maven的现有资源。继承了maven中仓库，坐标，依赖这些核心概念。文件的布局也和maven相同。但同时，它又继承了ant中target的概念，我们又可以重新定义自己的任务了。(gradle中叫做task)

[原文出处](https://zhuanlan.zhihu.com/p/24429133)

#### Gradle学些路线

1. Gradle的基础.
   包括Groovy语言语法及概念
   Gradle的生命周期

2. Android编译流程.
3. Android Gradle插件API.
4. 开发一款自己的Gradle插件.

[参考内容](https://mp.weixin.qq.com/s/2FTpDA5jSrrojVIVuoYOSQ)

#### Gradle基础

