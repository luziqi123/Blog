[TOC]

# Context是什么?

在程序中，我们可以理解为当前对象在程序中所处的一个环境，一个与系统交互的过程。比如微信聊天，此时的“环境”是指聊天的界面以及相关的数据请求与传输，Context在加载资源、启动Activity、获取系统服务、创建View等操作都要参与。

那Context到底是什么呢？一个Activity就是一个Context，一个Service也是一个Context。Android程序员把“场景”抽象为Context类，他们认为用户和操作系统的每一次交互都是一个场景，比如打电话、发短信，这些都是一个有界面的场景，还有一些没有界面的场景，比如后台运行的服务（Service）。

源码中的注释是这么来解释Context的：Context提供了关于应用环境全局信息的接口。它是一个抽象类，它的执行被Android系统所提供。它允许获取以应用为特征的资源和类型，是一个统领一些资源（应用程序环境变量等）的上下文。就是说，它描述一个应用程序环境的信息（即上下文）；是一个抽象类，Android提供了该抽象类的具体实现类；通过它我们可以获取应用程序的资源和类（包括应用级别操作，如启动Activity，发广播，接受Intent等）。

# Context体系

Context是一个抽象类 , 有两个实现类 ContextImle和ContextWrapper , 其中ContextWrapper持有一个ContextImple的实例 , 他只作为一个包装类 , 而ContextThemeWrapper / Activity / Service / Application继承了ContextWrapper , ContextThemeWrapper类，如其名所言，其内部包含了与主题（Theme）相关的接口，这里所说的主题就是指在AndroidManifest.xml中通过android：theme为Application元素或者Activity元素指定的主题。

所以 , 每创建一个Activity / Service / Application , 相当于随之创建了一个Context , So 一个应用的Context个数就是Activity+Service+1 . 

为什么没有提到BradcastReceiver 和 ContentProvider ? 因为他们两个并不是Context的子类 , 他们所拥有的Context其实都是从其他地方传递进来的 .



# Context能干什么?

并不是随便拿到一个Context实例就可以为所欲为 , 从Activity / Service / Application 中获取到的Context的作用于是不同的 , 比如Application的Context就不能弹出Dialog . 下面是一张Context的作用域示例:

![](http://upload-images.jianshu.io/upload_images/1187237-fb32b0f992da4781.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图我们可以发现Activity所持有的Context的作用域最广，无所不能。因为Activity继承自ContextThemeWrapper，而Application和Service继承自ContextWrapper，很显然ContextThemeWrapper在ContextWrapper的基础上又做了一些操作使得Activity变得更强大 . 

[Context都没弄明白，还怎么做Android开发？](http://www.jianshu.com/p/94e0f9ab3f1d)

















