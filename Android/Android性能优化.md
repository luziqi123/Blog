---
title: Android性能优化
date: 2017-2-26
---



[TOC]

# 内存优化

## 防止内存泄漏——夜用大片防侧漏

JAVA是在JVM所虚拟出的内存环境中运行的，JVM的内存可分为三个区：堆(heap)、栈(stack)和方法区(method)。

**栈(stack)**

是简单的数据结构，但在计算机中使用广泛。栈最显著的特征是：LIFO(Last In, First Out, 后进先出)。比如我们往箱子里面放衣服，先放入的在最下方，只有拿出后来放入的才能拿到下方的衣服。栈中只存放基本类型和对象的引用(不是对象)。

**堆(heap)**

堆内存用于存放由new创建的对象和数组。在堆中分配的内存，由java虚拟机自动垃圾回收器来管理。JVM只有一个堆区(heap)被所有线程共享，堆中不存放基本类型和对象引用，只存放对象本身。

**方法区(method)**

又叫静态区，跟堆一样，被所有的线程共享。方法区包含所有的class和static变量。当然这个区在这里不会参与讨论了 .

在JAVA中**JVM的栈记录了方法的调用**，每个线程拥有一个栈。在线程的运行过程当中，执行到一个新的方法调用，就在栈中增加一个内存单元，即帧(frame)。在frame中，保存有该方法调用的参数、局部变量和返回地址。然而JAVA中的局部变量只能是基本类型变量(int)，或者对象的引用。所以在**栈中只存放基本类型变量和对象的引用。引用的对象保存在堆中。**

当某方法运行结束时，该方法对应的frame将会从栈中删除，frame中所有局部变量和参数所占有的空间也随之释放。线程回到原方法继续执行，当所有的栈都清空的时候，程序也就随之运行结束。

如果我们不停的创建新对象，堆(heap)的内存空间就会被消耗尽。所以JAVA引入了垃圾回收(garbage collection，简称GC)去处理堆内存的回收，但如果对象一直被引用无法被回收，造成内存的浪费，无法再被使用。所以**对象无法被GC回收就是造成内存泄露的原因.**

GC会从根节点（GC Roots）开始对heap进行遍历 , 到最后，部分没有直接或者间接引用到GC Roots的就是需要回收的垃圾，会被GC回收掉。

**实现思想如下**：JVM中有一个栈用于存放引用 , 一个堆用于存放对象 . 遍历栈中所有的对象的引用，再遍历一遍堆中的对象。因为栈中的对象的引用执行完毕就删除，所以我们就可以通过栈中的对象的引用，查找到堆中没有被指向的对象，这些对象即为不可到达对象，对其进行垃圾回收。

> **思考 :** 话说堆内存是一个二叉树的数据结构 , 为什么用二叉树 ? 首先查找方便 , 在一个就是 , 如果占内存中的引用被删除了 , 那么堆内存中的对象将永远访问不到了 , 这时需要一个自己的引用 , 而二叉树恰好能通过这种引用让每一个对象都不会被遗漏 . 但每一次垃圾回收的动作都要遍历这颗树 ? 而对于垃圾回收的时机 , 什么时候才去执行一次垃圾回收 ? 记录对象分配的次数么 ? 到达一定值就清理一次 , 还是别的什么 ?

[自己动手写GC](http://it.deepinmind.com/gc/2014/03/26/babys-first-garbage-collector.html)

垃圾回收器不会回收GC Roots以及那些被它们间接引用的对象。GC Roots有如下几种类型 , 也就是说每一种类型都对应一个root , 这意味着会有以下这些类型的root持有你的对象 , 再简单点说就是你创建的对象可能被存储在以下几个类型的二叉树中 . 需要注意的是 一个对象可以同时属于一个以上的root .  **被这些root持有的对象不会被当做垃圾回收 .** 

- **Class** 

  应用运行过程中非动态加载的类都是通过`dalvik.system.PathClassLoader`的实例加载到虚拟机中的。这些类对象是GC root的一种，它们带来的静态变量永远不会被垃圾回收。因此，静态变量持有的“过期”对象将会造成内存泄漏。

  例如 : 单例时候的 getInstance(Context context) , 如果你传进来的是一个Activity , 那这个Activity可就没机会投胎了 .

- **Thread** 

  激活状态的线程是不会被GC回收的，所以它持有的对象也不会被回收。

  例如 : 你创建了一个Runnable或AsyncTask之类的线程 , 并在线程中引入了一个Context , 如果恰巧这个Context是一个Activity...它会和上面提到的那个一样惨 .


- **Stack Local** 

  栈中的对象。上面已经说过了 , JVM中有一个栈来记录方法的调用 , 而在这个栈中也存放着一些局部变量和参数 . 他们是不会被回收的 , 因为随时可能会用到 .


- **JNI Local**  / **JNI Global** 

  JNI中的局部 / 全局变量和参数引用的对象 , 可能在JNI中定义的，也可能在虚拟机中定义 . 

  JNI Local  和 JNI Global 这些对象不止被Java代码中的引用持有，也会被虚拟机中的底层代码持有。**在将持有它们的引用设置为null之前，要先将他们`close()`掉**。还有一个特殊的类是`Bitmap`。在Android系统3.0之前，它的内存一部分在虚拟机中，一部分在虚拟机外。因此它的一部分内存不参与垃圾回收，需要我们主动调用`recycler()`才能回收。以及IO等一些流操作 . 


- **Monitor Used** 

  用于保证同步的对象，例如wait()，notify()中使用的对象、锁等。


- **Held by JVM** 

  JVM持有的对象。JVM为了特殊用途保留的对象，它与JVM的具体实现有关。比如有System Class Loader, 一些Exceptions对象，和一些其它的Class Loader。对于这些类，JVM也没有过多的信息。

  [Android开发从GC root分析内存泄漏](http://www.jianshu.com/p/f5582d9a0f73)

  [What are the roots?](http://stackoverflow.com/questions/6366211/what-are-the-roots)

这些是关于GC的一些简单介绍 ,  接下来将介绍一些常见的优化手段 . 

---

### Handler的正确写法

正常情况下，本着方便快捷，省时省力的思想，我们会将Handler写成这副德性：
```java
    private Handler handler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            // TODO you have message
        }
    };
```
但是如果你用的是Android Studio，那么他会报一片屎黄屎黄的颜色，那是因为Android Studio的Inspect Code检出出了这里可能会造成内存泄漏，Inspect Code可以检查你的代码并标记出可能出现问题的地方，后面的优化工具篇会讲到，回到正题，鼠标放在小灯泡的图标上，你就会看到这些东西：
>Since this Handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler; Make all references to members of the outer class using the WeakReference object.

他主要是告诉你，这里你声明了一个内部类，他会阻碍GC回收垃圾，哔哩哔哩……
用代码说话，错误示例：

```java
public class SampleActivity extends Activity {
  private final Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // 发送一个10分钟后执行的一个消息
    mHandler.postDelayed(new Runnable() {
      @Override
      public void run() { }
    }, 600000);

    // 结束当前的Activity
    finish();
  }
}
```
当Activity结束后，在 Message queue 处理这个Message之前，它会持续存活着。这个Message持有Handler的引用，而Handler有持有Activity(SampleActivity)的引用，这个Activity所有的资源，在这个消息处理之前都不能也不会被回收，所以发生了内存泄露。

辣么，正确的写法应该是这样的：

```java
private MyHandler handler = new MyHandler(this);

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 当参数为null的时候，可以清除掉所有跟次handler相关的Runnable和Message，我们在onDestroy中调用次方法也就不会发生内存泄漏了。
        handler.removeCallbacksAndMessages(null);
    }

    private void todoHandler(Message msg){
        // TODO handlerMessage
    }

    public static class MyHandler extends Handler{

        private final WeakReference<MainActivity> mActivity;

        public MyHandler(MainActivity activity) {
            mActivity = new WeakReference(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            MainActivity mainActivity = mActivity.get();
            if (mainActivity != null){
                mainActivity.todoHandler(msg);
            }
        }
        
    }
```

如果你觉得这么写太TM麻烦了，不用担心，嫌麻烦的不止你一个，总有受不了的，这里提供一个WeakHandler库 ： [GitHub](https://github.com/badoo/android-weak-handler)

**这里再提供一个我感觉比较好的写法，自己觉得可以，但是不知道对不对**：

```java
// 实现Handler.Callback接口
public class ActivitySigin extends BaseActivity implements Handler.Callback{

	private Handler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sigin_new);
		
		// 创建Handler，将自己的实例传进去
		mHandler = new Handler(this);
    }

	@Override
    public boolean handleMessage(Message msg) {
		// TODO 处理handler消息
        return false;
    }
```

-------

### 单例，不仅仅存在线程安全问题

单例模式深受广大开发者的喜爱，但从相对论的角度来看，肉吃多了也能死人。所以，在适合的情况下使用单例。而在单例的实现方式上，更多人关注线程安全方面的问题，但是内存泄漏问题同样重要。
在用LeakCanary检测APP内存泄漏的时候，发现N多地方出现了单例造成的内存泄漏。
下面来看一段反面教材：
``` java
public class XXXManager{  

	private static XXXManager instance;
	private Context context;
	
    private XXXManager(Context context) {
	    this.context = context;
    }
    
    public static XXXManager getInstance(Context context) {
      if (instance != null) {
        instance = new XXXManager(context);
      }
      return instance;
    } 
}
```
XXXManager实例是存在于整个app生命周期的，如果是传入的Application完全没有问题，但如果你传入的是某个Activity，那么这个不幸的Context就跟死了没法儿投胎一样的跟着XXXManager，况且如果你在调用getInstance()之前这个实例已经存在了，那么你传入的这个Context并没有什么卵用。下面给出两个方案，我认为第二个会好一点。
方案1：
```java
private XXXManager(Context context) {
	// 这样你持有的就是Application的Context了，没毛病~
	this.context = context.getApplication();
}
```
方案2：
```java
	private static Context mContext;

    public static XXXManager getInstance(){
        if (mContext != null){
           if (instance == null){
               instance = new XXXManager(mContext);
           }
            return instance;
        }
        // 如果mContext = null 抛出异常提醒没有注册过
        try {
            throw new Exception("not register!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 在MyApplication中先注册
    public static void register(Context context){
        mContext = context;
    }
```
这里为了突出重点，忽略了线程安全，使用的时候需要注意到这一点。

---

### 多线程的滥用——你用多少内存换来懒惰？

我相信很多人都这么干过：
```java
 new Thread(new Runnable() {
      @Override
      public void run() {
        SystemClock.sleep(10000);
      }
    }).start();
```
这里的Runnable和前面提到的错误书写handler一样，是一个匿名内部类，因此它们对当前Activity都有一个隐式引用。如果Activity在销毁之前，任务还未完成，那么将导致Activity的内存资源无法回收，造成内存泄漏。
同Handler的解决方法一样，你需要使用一个静态的内部类：
```java
static class MyRunnable implements Runnable{
	@Override
	public void run() {
		SystemClock.sleep(10000);
	}
}
```

---

### 内存抖动

在Android studio中我们可以方便的看到App运行时的内存使用情况，下面这张图就是典型的内存抖动了， 内存抖动是因为大量的对象被创建又在短时间内马上被释放。
瞬间产生大量的对象会严重占用Young Generation的内存区域，当达到阀值，剩余空间不够的时候，也会触发GC。即使每次分配的对象占用了很少的内存，但是他们叠加在一起会增加Heap的压力，从而触发更多其他类型的GC。这个操作有可能会影响到帧率，并使得用户感知到性能问题。

![Alt text](http://img.blog.csdn.net/20170224132942416?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

解决方法也比较简单，当出现问题时，多半是在for循环中分配对象占用内存，或者在自定义的onDraw()中做了构建对象的操作或其他复杂操作，如若真的无法避免，你可以考虑使用对象池来管理这些需要被大量创建的对象，我们在工具篇也介绍了如何定位到问题所在。

---

### 选择合适的容器

####ArryaMap
ArrayMap是Android提供的用来替代HashMap的集合
对象个数的数量级在千以内
数据组织形式包含Map结构
这时使用ArrayMap就相比HashMap节省内存，并且在遍历速度上也更胜一筹。
具体为什么，请看下图：
![Alt text](http://img.blog.csdn.net/20170224133249318?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
[原文地址](https://liuzhichao.com/p/832.html)

####Sparse系列
为了避免HashMap的自动装箱（autoboxing）行为，Android系统提供了SparseBoolMap，SparseIntMap，SparseLongMap，LongSparseMap等容器。

另外这些容器的使用场景也和ArrayMap一致，需要满足数量级在千以内，数据组织形式需要包含Map结构。

----

###for循环的秘密
遍历容器是编程里面一个经常遇到的场景。在Java语言中，使用Iterate是一个比较常见的方法。可是在Android开发中，大家却尽量避免使用Iterator来执行遍历操作。下面我们看下在Android上可能用到的三种不同的遍历方法：
![Alt text](http://img.blog.csdn.net/20170224133014096?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

他们的执行速度分别是这样的：
![Alt text](http://img.blog.csdn.net/20170224133024315?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上面可以看到***for index的方式有更好的效率，但是因为不同平台编译器优化各有差异，我们最好还是针对实际的方法做一下简单的测量比较好，拿到数据之后，再选择效率最高的那个方式***。

---

### 你所不知道的枚举真面目

枚举往往需要超过一倍多的内存静态常量。在Android中你应该严格避免使用枚举。
总的来说枚举更耗内存，并且在编译之后的dex文件也会占用更多的空间。

>Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.

----

# UI优化

## 真的是越小越好

在Android里面一个相对操作比较繁重的事情是对Bitmap进行旋转，缩放，裁剪等等。例如在一个圆形的钟表图上，我们把时钟的指针抠出来当做单独的图片进行旋转会比旋转一张完整的圆形图的所形成的帧率要高56%。
![Alt text](http://img.blog.csdn.net/20170224133005900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

另外尽量减少每次重绘的元素可以极大的提升性能，假如某个钟表界面上有很多需要显示的复杂组件，我们可以把这些组件做拆分处理，例如把背景图片单独拎出来设置为一个独立的View，通过setLayerType()方法使得这个View强制用Hardware来进行渲染。

---

## 16ms

Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，这称作一帧，如果每次渲染都成功，这样就能够达到流畅的画面所需要的60fps（1000 / 16 = 62.6），这意味着程序的大多数操作都必须在16ms内完成。
![Alt text](http://img.blog.csdn.net/20170224132838493?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
例如在滑动ListView的时候经常出现卡顿的现象，很可能就是因为item的布局太过复杂，加载时间超过了16ms，这种情况就是丢帧了。

也许是因为你的layout太过复杂，无法在16ms内完成渲染，有可能是因为你的UI上有层叠太多的绘制单元，还有可能是因为动画执行的次数过多。这些都会导致CPU或者GPU负载过重。

我们可以通过一些工具来定位问题，比如可以使用HierarchyViewer来查找Activity中的布局是否过于复杂，也可以使用手机设置里面的开发者选项，打开Show GPU Overdraw等选项进行观察。你还可以使用TraceView来观察CPU的执行情况，更加快捷的找到性能瓶颈。

---

## 小心Bitmap

在开发过程中，我们或多或少的会用到Bitmap，但是如果处理不当，就会发生OOM或是让你的程序变的卡顿并异常的消耗内存，对于Bitmap，我们应该清楚以下几点：

####  **-如何高效的加载大图**

在大多数情况下，图片的实际大小都比需要呈现的尺寸大很多，考虑到应用是在有限的内存下工作的，***理想情况是我们只需要在内存中加载一个低分辨率的照片即可***。这一小结我们会介绍如何通过加载一个缩小版本的图片，从而避免超出程序的内存限制。

***Android 提供了现成的API createScaledBitmap()来缩放一张图片，可是这个方法能够执行的前提是，原图片需要事先加载到内存中，如果原图片过大，很可能导致OOM。***下面介绍其他几种缩放图片的方式。

***inSampleSize***

他能够等比的缩放显示图片，同时还避免了需要先把原图加载进内存的缺点。BitmapFactory提供了一些解码（decode）的方法（decodeByteArray(), decodeFile(), decodeResource()等），用来从不同的资源中创建一个Bitmap。 我们应该根据图片的数据源来选择合适的解码方法。 这些方法在构造位图的时候会尝试分配内存，因此会容易导致OutOfMemory的异常。每一种解码方法都可以通过BitmapFactory.Options设置一些附加的标记，以此来指定解码选项。

![Alt text](http://img.blog.csdn.net/20170224132819306?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


***设置 inJustDecodeBounds 属性为true可以在解码的时候避免内存的分配，它会返回一个null的Bitmap，但是可以获取到 outWidth, outHeight 与 outMimeType。该技术可以允许你在构造Bitmap之前优先读图片的尺寸与类型。***
***为了避免OOM 的异常，我们需要在真正解析图片之前检查它的尺寸***

```
/**
 *在不加载原图到内存的情况下获取到图片的实际宽高
 */ 
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

>例如，如果把一个大小为1024x768像素的图片显示到大小为128x96像素的ImageView上吗，就没有必要把整张原图都加载到内存中。

>为了告诉解码器去加载一个缩小版本的图片到内存中，需要在BitmapFactory.Options 中设置 inSampleSize 的值。例如, 一个分辨率为2048x1536的图片，如果设置 inSampleSize 为4，那么会产出一个大约512x384大小的Bitmap。加载这张缩小的图片仅仅使用大概0.75MB的内存，如果是加载完整尺寸的图片，那么大概需要花费12MB（前提都是Bitmap的配置是 ARGB_8888）。下面有一段根据目标图片大小来计算Sample图片大小的代码示例：

*使用下面这个方法可以简单地加载一张任意大小的图片。这里显示了一个接近 20X20像素的缩略图：*
```
main(){
	// 调用
	imageView.setImageBitmap(
                decodeSampledBitmapFromResource(
                getResources() , 
                R.mipmap.ic_launcher , 
                20 , 
                20)
        );
}


/**
     * 从resource中解码一个图片
     * @param res 图片的来源
     * @param resId 图片的ID
     * @param reqWidth 期望的宽度
     * @param reqHeight 期望的高度
     * @return 缩小版的Bitmap
     */
    public static Bitmap decodeSampledBitmapFromResource
            (Resources res, int resId, int reqWidth, int reqHeight) {
        // 首先你需要解码并将inJustDecodeBounds设置为true,检查图片的大小
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(res, resId, options);

        // 计算inSampleSize的值
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

        // 根据inSampleSize的值来解码图片
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(res, resId, options);
    }

    /**
     * 根据你所期望的宽高获取inSampleSize
     * @param options 一个经过设置的BitmapFactory.Options
     * @param reqWidth 期望图片的宽
     * @param reqHeight 期望图片的高
     * @return 一个合适的inSampleSize值
     */
    public static int calculateInSampleSize
            (BitmapFactory.Options options, int reqWidth, int reqHeight) {
        // 记录未加工前的图片的宽高 初始化inSampleSize为1
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;

        if (height > reqHeight || width > reqWidth) {

            final int halfHeight = height / 2;
            final int halfWidth = width / 2;

            // 当高度和宽度大于所要求的高度和宽度，将inSampleSize以2的幂增加，得到最大inSampleSize值
            // 设置inSampleSize为2的幂是因为解码器最终还是会对非2的幂的数进行向下处理，获取到最靠近2的幂的数。
            while ((halfHeight / inSampleSize) > reqHeight
                    && (halfWidth / inSampleSize) > reqWidth) {
                inSampleSize *= 2;
            }
        }

        return inSampleSize;
    }
```

除此之外，我们还可以使用inScaled，inDensity，inTargetDensity的属性来对解码图片做处理。

####  **-提升bitmap的循环效率**
// TODO 需要单独写一篇
####  **-非UI线程处理Bitmap**
// TODO 需要单独写一篇
####  **-Bitmap缓存**
// TODO 需要单独写一篇

----

## 优化你的布局

用事实说话就是，曾经一个界面的运行内存是160M左右，单修改了布局的结构就下降到了140M左右。当然，优化布局最终要的是让你的界面变的更流畅，我们可以通过HierarchyViewer这个工具来查看布局(工具篇有讲到)，使得布局尽量扁平化，移除非必需的UI组件，这些操作能够减少Measure，Layout的计算时间。

对于布局优化，有以下几点建议：

#### ***-merge和include——减少节点***

在一些无需太多属性的根节点上，你可以使用merge，他会为你减少一层节点，如果你想优化你的布局，减少节点是第一步要做的，通过include将这些merge为根节点的布局引入进来，再用HierarchyViewer看一下你的层级吧，它变的更扁平化了。

*示例代码：*
```
<?xml version="1.0" encoding="utf-8"?>
<merge
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ImageView
        android:layout_width="30dp"
        android:layout_height="30dp"
        android:src="@mipmap/ic_launcher"/>
    
    <ImageView
        android:layout_marginTop="30dp"
        android:layout_width="30dp"
        android:layout_height="30dp"
        android:src="@mipmap/ic_launcher"/>

</merge>
```


#### ***-viewstub——优化预加载***

这是一个神奇的控件，像公关小姐一样召之即来挥之即去，在你不需要她的时候绝对不会给你添麻烦，viewstub引入的布局默认不会扩张，不会占用显示位置，不参与任何的布局和绘制过程，从而在解析layout时节省cpu和内存。它的意义在于按需加载布局文件，比如网络异常的界面正常情况下是不会显示的，我们就没有必要在初始化的时候就把它加载进来。

需要注意的是:
1. ViewStub只能用来Inflate一个布局文件，而不是某个具体的View，当然也可以把View写在某个布局文件中。
2. ViewStub只能Inflate一次，之后ViewStub对象会被置为空。按句话说，某个被ViewStub指定的布局被Inflate后，就不会够再通过ViewStub来控制它了。

*示例代码：*

```
	private View mStubView;
    
    /**
     * 显示这个布局
     */
    public void comeBaby(){
        // 避免重复加载
        if (mStubView == null){
            ViewStub stub = (ViewStub) findViewById(R.id.view_stub);
            mStubView = stub.inflate();
        }
        mStubView.setVisibility(View.VISIBLE);
        Button btn1 = (Button) mStubView.findViewById(R.id.btn1);
        Button btn2 = (Button) mStubView.findViewById(R.id.btn2);
    }
    
```

#### **-别让你的APP太多层——过渡绘制**

在开发者选项中打开调试GPU绘制，模拟器上叫Show GPU Overdraw ，不同机型名字也不一样，大概就这个意思，然后你就会看到你的屏幕跟坏了似得花花绿绿。这时候，你就可以去换一台手机了O(∩_∩)O。
开玩笑……这些了花花绿绿的分别代表了某个区域的绘制情况。
![Alt text](http://img.blog.csdn.net/20170224132854071?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

简而言之就是，红色严重，粉色严重，绿色说得过去，蓝色正常。

有时候是因为你的UI布局存在大量重叠的部分，还有的时候是因为非必须的重叠背景。例如某个Activity有一个背景，然后里面的Layout又有自己的背景，同时子View又分别有自己的背景。仅仅是通过移除非必须的背景图片，这就能够减少大量的红色Overdraw区域，增加蓝色区域的占比。这一措施能够显著提升程序性能。


### ***硬件加速 Hardware Accelerated***

 Android从3.0（API Level 11）开始，在绘制View的时候支持硬件加速，充分利用GPU的特性，使得绘制更加平滑，但是会多消耗一些内存。

**开启硬件加速：**

Application级别:
```
<application android:hardwareAccelerated="true" ...>
```

Activity级别:
```
<application android:hardwareAccelerated="true">
    <activity ... />
    <activity android:hardwareAccelerated="false" />
</application>
```

Window级别：
```
getWindow().setFlags(
    WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
    WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
```

View级别：
```
myView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

**开启后的绘制区别：**
1. 没有硬件加速：invalidate the view hierarchy ------> draw the view hierarchy

2. 有硬件加速：invalidate the view hierarchy ------> record and update the display list ------> draw the display list

**开启硬件加速之后的异常反应：**

1. 某些UI元素没有显示：可能是没有调用invalidate

2. 某些UI元素没有更新：可能是没有调用invalidate

3. 绘制不正确：可能使用了不支持硬件加速的操作， 需要关闭硬件加速或者绕过该操作

4. 抛出异常：可能使用了不支持硬件加速的操作， 需要关闭硬件加速或者绕过该操作

**下表为硬件加速不支持的绘制操作：**

![Alt text](http://img.blog.csdn.net/20170224132955447?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![Alt text](http://img.blog.csdn.net/20170224134117218?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![Alt text](http://img.blog.csdn.net/20170224134117218?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### ***需要慎用的透明度***

这小节会介绍如何减少透明区域对性能的影响。通常来说，对于不透明的View，显示它只需要渲染一次即可，可是如果这个View设置了alpha值，会至少需要渲染两次。原因是包含alpha的view需要事先知道混合View的下一层元素是什么，然后再结合上层的View进行Blend混色处理。在某些情况下，一个包含alpha的View有可能会触发该View的父控件都被额外重绘一次。

大多数情况下，屏幕上的元素都是由后向前进行渲染的。如果后渲染的元素有设置alpha值，那么这个元素就会和屏幕上已经渲染好的元素做blend处理。

如何渲染才能够得到我们想要的效果呢？我们可以先按照通常的方式把View上的元素按照从后到前的方式绘制出来，但是不直接显示到屏幕上，而是使用GPU预处理之后，再又GPU渲染到屏幕上，GPU可以对界面上的原始数据直接做旋转，设置透明度等等操作。使用GPU进行渲染，虽然第一次操作相比起直接绘制到屏幕上更加耗时，可是一旦原始纹理数据生成之后，接下去的操作就比较省时省力。

```

convertView.setLayerType(View.LAYER_TYPE_HARDWARE , null);
```

对于一些不存在层叠关系的View，我们可以重写hasOverlappingRendering()方法来让渲染器知道这种情况。
```
Override
hasOverlappingRendering(){
	return false;
}
```

下面这张图是设置后的情况：

![Alt text](http://img.blog.csdn.net/20170224133033596?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)




## **电量优化**
- AP 和 BP
- 唤醒屏幕
## **网络优化**
- 打包发送网络请求
## **关于生命周期**
- 在onCreate中处理你的复活状态
## **代码结构**
- 全局通知机制


## **优化工具**

***Android SDK包含了许多工具来帮助我们开发Android App，这些工具呢大概分为两类：SDK工具和平台工具。SDK工具是独立于平台之外的，而平台工具是为了支持最新的Android开发平台而定制的。***

### ***Hierarchy Viewer —— 检查你的布局***
Hierarchy Viewer是我们的布局文件的层级结构变的可见，并且在每一个节点标注此节点的性能相关的信息。通过此工具可以详细的理解当前界面的控件布局以及某个控件的属性（name、id、height等）。同时，我们可以借助Hierarchy Viewer学习别人优秀的布局方式，也能更深入更全面更整体的把握xml布局文件。
Android studio中，你更是可以方便的使用他：
![Alt text](http://img.blog.csdn.net/20170224132718149?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
然后点击这里：
![Alt text](http://img.blog.csdn.net/20170224132901900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
在弹框中选择Hierarchy Viewer。
之后我们就可以看到这么一张图：
![Alt text](http://img.blog.csdn.net/20170224132910103?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
每一个小方格就代表了一个View，从左至右的顺序代表了层级关系，上面最显眼的三个点分别代表了：计算宽高时间、放置布局时间、绘制时间。和你想象的一样，绿色合格，黄色警告，红色超时。
点击对应的控件，可以详细的看到他分别在绘制流程的每一步花费具体花费了多长时间。

*Android系统出于安全考虑，Hierarchy Viewer只能连接开发版手机或模拟器*，我们普通的商业手机是无法连上的（老版本的Hierarchy Viewer可以），这一限制在
frameworks/base/services/java/com/android/server/wm/WindowManageService.java
cmd下执行:
```
adb shell service call window 3
```
若返回值是：Result: Parcel(00000000 00000000 '........') 说明View Server处于关闭状态
若返回值是：Result: Parcel(00000000 00000001 '........') 说明View Server处于开启状态
如果不嫌麻烦可以参照这里[连接真机教程](http://blog.csdn.net/autumn_xl/article/details/40741835)
但是我比较懒，就直接装了一个小米的开发版系统，执行adb开启：
```
// 开启
adb shell service call window 1 i32 4939
// 关闭
adb shell service call window 2 i32 4939
```
如果你连系统也不想刷就下载个Genymotion来用吧，很棒的虚拟机。
如果这个也不想下，那你的布局肯定没毛病~

### ***TraceView——关心你的CPU占用时长***
TraceView可以用图表的方式告诉你，哪些线程在CPU中占用了太长时间，但是界面之复杂程度……着实不是一般人能看懂的。
而Android studio中对此做了优化，我们最常看的LogCat的右边：
![Alt text](http://img.blog.csdn.net/20170224132916165?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 

点开后可以看到你的CPU使用情况，点击下图中的小闹钟开始，再隔一段时间后再次点击它结束：
![Alt text](http://img.blog.csdn.net/20170224132931150?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


然后你就会看到下面这个界面了：
![Alt text](http://img.blog.csdn.net/20170224132923040?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

选中Color by inclusive time 就可以快速的预览各个线程占用CPU的时间。谁是最慢的短板？想办法优化它吧！


### ***一只鼻子灵敏的DOG——MAT内存分析***
MAT这个工具可以将内存空间的现状展现的淋漓尽致，这里会让你快速认识并使用它找到内存泄露的地方。
首先说下运行环境，这里使用的是Android studio + MAT。
[用力戳我下载MAT](http://pan.baidu.com/s/1cp1PpW)
你需要完成以下步骤：
1. **在Android studio的内存监测中获取某个时刻的内存情况。**
   打开App，开始正常使用吧，最后，我建议你回到你进入到app的第一个界面。
   然后在内存监测中调用GC回收掉可以被回收的垃圾，此时内存中只剩下无法被回收的一部分了，不管他是否应该被回收。
   ![Alt text](http://img.blog.csdn.net/20170224133105675?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   然后点这里，Dump Java Heap:
   ![Alt text](http://img.blog.csdn.net/20170224133044128?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   你会看到一个这样的图标出现在了内存显示区![Alt text](http://img.blog.csdn.net/20170224133052690?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   可能需要等一会儿，上个厕所什么的，小的就可以 ，大的就过长了…这期间他会将这一时刻的内存情况分析出来并保存成一个.hprof的文件。
   你可以在这里找到他，在第三项Captures：
   ![Alt text](http://img.blog.csdn.net/20170224133113941?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   然后右键—>Export to standard .hprof 导出，你需要选择一个路径，我建议新建一个文件夹，如果你听了我的建议，用过之后你会感谢我的。
2. **在MAT中打开刚才导出的文件**
   至此我们已经得到了你的内存分析报告，那给医生去看吧，打开MAT，File—>Open Heap Dump,就可以看到分析结果了。
3. **定位**
   网上很多资料，但是对于如何定位问题很笼统，今天就让本菜手把手 嘴对嘴的告诉你。
   首先点击这里，可以看到更详细的统计：
   ![Alt text](http://img.blog.csdn.net/20170224133121010?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   一看都不认识，脸一黑，都啥****东西（*号部分自由发挥），如果你这么想，你需要点这里：
   ![Alt text](http://img.blog.csdn.net/20170224133141988?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


选中最后一项，就可以以包名分类来看这些条目了，应该知道怎么做了，找到你的项目com.ex...巴拉巴拉，你会看到刚才涉及到的对象都在这里出现了，分别有4个tab, 第一个是name第二个是Object，就是这个对象当前存在实例的个数，右边还有两个这里不说了，因为我没用到，也不敢瞎BB……
如果你有内存泄露的情况，就会看到这样的现象：

![Alt text](http://img.blog.csdn.net/20170224133232114?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


![Alt text](http://img.blog.csdn.net/20170224135135895?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我的BrushActivity明明早就隔儿屁了，怎么还有一个实例存在，你可以右键—>Merge Shortest Paths to GC Roots  —> exclude weak/soft references ，看是谁脱离了Root的节点：
![Alt text](http://img.blog.csdn.net/20170224135037426?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

被我画红线的地方写了包名，你可以很容易的找到问题所在了。这里可以清楚的知道，这是因为一个线程持有了activity的实例，而线程又是一个AnimationSet动画线程，在MoveParticleView中，自己挖的坑儿，应该会想到问题所在了。

至此，就没了……没了……

### ***分配追踪，Allocation Tracking***
看完了上一小节，再来看下一小节……戳完了上节的按钮，肯定有不少人想戳下面的，这里我就好奇了，一戳惊喜不断呀。
![Alt text](http://img.blog.csdn.net/20170224133218426?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
使用很简单，点第一下开始，点第二下结束，然后他会分析出在这期间你的内存分配情况。正好项目里有个地方出现了内存抖动，检测一下吧。
看到检测结果，都不认识，脸一黑，都啥****东西（*号部分自由发挥），如果你这么想，你需要点这里：
![Alt text](http://img.blog.csdn.net/20170224133156448?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

第一个是以方法分组，第二个是以包名，想都不想肯定用包名。

![Alt text](http://img.blog.csdn.net/20170224135514021?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这里看到brush包下居然分配了600个对象，点开，定位到了一个叫Renderer的类，右键，Jump to Source 打开源码，发现有一个方法一直在频繁调用，但是问题是从native中报出来的，So……放弃……但是总之是定位到问题所在了。
在上图中还看到一个饼状图一样的按钮，点他会出现一个很炫酷的饼状图，并且可以在柱状和饼状之间切换，可以点着看看，一目了然。

### ***Android Lint——为你的Code体检***

从字面上我是这么理解的，Android Lint，Lint嘛！线头！安卓线头！它帮你找出你代码中可能出问题，书写不规范，可以优化的地方，不知道理解的对不对，求调教……没图我说个JB，直接上图：
![Alt text](http://img.blog.csdn.net/20170224133332198?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在这里你可以选择检查的范围，从上到下分别是，整个项目
![Alt text](http://img.blog.csdn.net/20170224133339303?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjk4NDA1NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



# 关于性能优化的一些建议

以下摘自<Android开发艺术探索>  和 <Android群英传>

- 避免创建过多的对象
- 不要过多使用枚举 , 枚举占用的内存空间要比整形大
- 常量使用static final来修饰
- 使用一些Android特有的数据结构 , 比如SparseArray 和 Pair等 , 他们具有更好的性能 
- 适当使用软引用和弱引用 (软引用比较常用)
- 采用内存缓存和磁盘缓存
- 尽量采用静态内部类 , 这样可以避免由于内部类而导致的内存泄露
- 任何Java类 , 都将占用大约500字节的内存空间 , 创建一个实例大约消耗15字节的内存 .
- 可以定义为局部的变量不要定义为成员
- 避免频繁创建短作用域的变量
- 使用RenderScript , OpenGL来进行非常复杂的绘图操作
- 使用SurfaceView来完成频繁的绘图操作



# **参考资料**
[Android 性能优化列表](http://hanks.xyz/2016/01/20/android-optimization)
[Android APP内存优化之图片优化](http://www.jianshu.com/p/5bb8c01e2bc7)
[Android 性能优化（谷歌官方）](http://www.kancloud.cn/kancloud/android-performance/53238)
[Android性能优化1~4](http://hukai.me/android-performance-patterns-season-2/)
[Android 布局优化](http://blog.csdn.net/sdkfjksf/article/details/50939425)

​    







