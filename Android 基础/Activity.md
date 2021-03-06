[TOC]

# Activity的启动模式

默认情况下，Activity都存放在同一个任务栈中。

Activity有4中启动方式：分别为：

- standard 标准模式
- singleTop 栈顶复用模式
- singleTask 栈内复用模式
- singleInstance 单实例模式

默认情况下为standard。

## standard

标准模式。每次启动一个Activity都会重新创建一个新的实例，如果A和B都是standard，那么如果启动A再启动B再启动A，那么栈中的情景就是ABA。

这个模式下的activity被谁启动，他就会加入到谁的栈中，所以用ApplicationContext去启动一个这样的activity的时候系统会报错，因为ApplicationContext并不是一个activity，他没有自己所属的栈。若非要这么干，可以指定Intent.FLAG_ACTIVITY_NEW_TASK标记，这样启动的时候就会为这个activity指定一个新的任务栈。

## singleTop

顶栈复用模式。如果被启动的activity已经在栈顶，那么这个activity就不会被再次创建，否则创建一个新的实例，保证不会产生两个相同的activity相邻。

如果被启动的activity在栈顶，那么再次启动它，它的onNewIntent()方法会被调用，onCreate, onStart不会被调用。

## singleTask

栈内复用模式。如果栈中已经存在这个activity的实例，那么多次启动也不会被创建新的实例，只会调用onNewIntent方法，否则创建新的实例。举例来说栈中有ABCD，D为栈顶，此时D启动B，则栈中情况就变成了AB。是的……中间的就都销毁了……

任务栈中唯一，它的实例在任务栈中是唯一的。它在被Intent的时候，会先在系统中查找属性值affinty与它的属性值taskAffinity相同的任务栈是否存在，如果存在，则在这个任务启动，如果不在，则在新任务栈中启动。

之前还遇到过这样一个问题，B activity为singleTask模式，A使用startActivityForResult()方法启动B，奇怪的事情发生了，A的onActivityResult()在B启动的时候立即就被执行了，确切的说应该是除第一次启动完全正常没有被执行以外，其后每次启动B，A都会被立即调用onActivityResult()，将B的启动方式改为standard就没毛病了，   回立即调用onActivityResult()方法。搜了下发现了这个：

*“Ams内部原理“10.1.3中有这样的一段话：请注意：SINGLE_TASK标识以及SINGLE_INSTANCE两个标识必须在r.result==0的条件中，即这两个标识只能用在startActivity()的方法中，而不能使用在startActivityForResult方法中。因为从Task的角度看，Android认为不同Task之间的Activity是不能传递数据的，所以不能使用NEW_TASK标识，但还是要调用forResult方法。”*

也就是说这个模式不建议或者说不支持你使用startActivityForResult()启动再使用onActivityResult()方法接收数据，因为这个模式默认是为被启动的activity创建一个新的task，或者说将其加入到和他TaskAffinity相同的activity所存在的task中，只会出现这两种情况：1、系统中没有和他taskAffinity相同的activity，系统为其新建一个task。2、系统中有与他taskAffinity值相同的activity存在并已经有了所属栈，则将这个activity加到该栈中， 而Android认为不同Task之间的Activity是不能传递数据的，So……

这里有一个不知道对不对的地方就是，如果一个S1栈为ABCD，D为栈顶，此时D启动A，并且指定FLAG_ACTIVITY_NEW_TASK标记，那么此时是不是S1为ABCD，S2为A？不知道这种模式下指定FLAG_ACTIVITY_NEW_TASK管不管用。到目前为止我认为不管用。

你问什么是TaskAffinity？他是任务相关性。这个参数表示了一个activity所需要的任务栈的名字，默认情况下所有activity所需任务栈的名字都是应用的包名，我们可以为每个activity指定taskAffinity，记住不要和包名相同。

## singleInstance

单实例模式。这是singleTask的加强版，你可以这么认为：他是全局唯一的，它的实例在全局（即在众多任务栈中）是唯一的，它单独地存在于属于自己的任务栈中，而且这个任务栈没有其他实例。例如你在应用A中打开了百度地图的地图activity，并定位到了一个地区，如果这个activity是singleInstance模式，此时切换到百度地图，打开这个activity，他仍然是定位在这个地区的。

此模式下的activity被启动后系统会为其创建一个新的栈，在其他栈中启动这个activity，系统仍会来这个栈中找他。

需要注意的是下面这个例子：栈S1中为AB，栈S2也就是此类模式activity的聚集地中有CD，左底右顶，此时B启动D，则S1中就变成了ABCD。如果B启动了C，则S1中就是ABC，D被销毁了。

# 设置启动模式

**方法一：通过AndroidManifest文件**

```xm
       <activity android:name=".MainActivity"
            android:launchMode="singleTask">
```

**方式二：通过代码**

```java
        Intent intent = new Intent(ActivityA.this , ActivityB.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
```

第二种方式优先级高于第一种.

# **Activity常用的Flags**

**FLAG_ACTIVITY_NEW_TASK**

指定为singleTask模式

**FLAG_ACTIVITY_SINGLE_TOP**

指定为singleTop模式

**FLAG_ACTIVITY_CLEAR_TOP**

如果已经存在了,调用他的onNewIntent(),他之上的activity全部出栈.

**FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS**

在用户使用最近任务的时候,该Task不会出现在最近任务中,等同于在XML中设置的android:excludeFromRecents="true"

# Activity生命周期

百年不变的基础问题 , 无非是在跟几个不同时期的回调方法的做周旋 .

- **onCreate** 一直到onDestroy为activity的全部生命周期 , 创建时调用 , 此时Activity不可见 , 带一个Bundle参数 saveInstanceState , 如果你的activity被系统回收了 , 再次打开需要重新创建 , 这个参数可以用来恢复一些必要数据 . 


- **onStart** 一直到onStop都为activity的可见生命周期 , 这说明了到了这里activity已经为可见了 . activity交替的对用户可见或隐藏时 , 也会多次调用onStart和onStop.
- **onResume** 一直到onPause都为activity的前台生命周期 , 该方法执行后activity完全加载完毕 , 已经可见并可以与用户交互了.
- **onPause** 退到后台时调用 , 比如被一个Dialog覆盖了 ,Dialog消失调用onResume , 所以说这个方法可以说是用户离开activity的必经之路.
- **onStop** 当activity不会长时间显示的时候调用 , 也就是说 , 只要这个activity还可见 , 那就不会调用该方法.
- **onDestroy** activity被销毁的时候调用.注意 , 这里的调用时机是不确定的 , 有时候并不会马上调用这个方法.
- **onRestart** 当activity从后台重新返回前台的时候回调用该方法.

其他:

- **onSaveInsanceState** 在上面的onCreate方法中提到了saveInstanceState 参数 , 用来恢复必要数据 , 那什么时候存呢? 就在这个方法里了 , 当遇到内存不足/屏幕旋转或其他情况需要由系统而不是用户来销毁一个Activity的时候 , 就会调用该方法 , 你可以在这个方法中保存这个页面当前的状态 , 在下次onCreate的时候通过saveInstanceState拿到你保存过的这些东西.


- **onRestoreInstanceState** 这个方法中也带有一个saveInstanceState , 并且和onCreate附带的那个参数是一样的 , 这个方法也是用来恢复activity的状态 , 需要注意的是 , 较恢复数据而言 , onCreate是在被销毁后从新创建时被调用 , 而onRestoreInstanceState则是在activity 没有被销毁且重新出现的时候被调用的 . 


- **onNewIntent** 若一个activity已经在栈中 , 且该activity的启动模式又不需要重新创建一个新的实例 , 在调用startActivity时这个activity的方法调用则为 onNewIntent()---->onResart()------>onStart()----->onResume().


- **onRetainNonConfigurationInstance** 到这里 , 你已经知道了在onSaveInsanceState中保存数据 , 然后再onRestoreInstanceState 或 onCreate中恢复数据 , onRetainNonConfigurationInstance 方法同样可以保存数据 , 不同的是他可以返回一个Object .


- **getLastNonConfigurationInstance** 上面已经提到了 , onRetainNonConfigurationInstance 方法返回一个Object , 这个方法则可以得到那个Object .
- **onWindowFocusChanged** 当Activity焦点发生变化时被调用 .

> 由A启动B 声明周期如下:
> A:onPause() 
> B:onCreate() - onStart() - onResume()
> A:onStop()
> 按返回由B回到A:
> B:onPause()
> A:onRestart() - onStart() - onResume()
> B:onStope() - onDestroy()
>
> 如果B的`Theme`设置为`Dialog`or`Translucent` , 那么A只会调用onPuse()
>
> 另外 , Dialog和Toast都是由`WindowManager.addView()`来实现的 , 所以他们不会影响Activity的生命周期.

# Activity的状态

- **running**  活动状态一切正常,在栈顶 
- **paused**  失去焦点,不在栈顶,并不是被销毁,可能被回收
- **stopped**  被完全覆盖 , 完全不可见 , 也可能被销毁 .
- **killed**   已经被回收了.

**onStart()**的时候activity已经可以看见了,但无法交互. 

**onResume()** 可见并可交互. 

**onPause()**调用之后activity就不能与用户交互了 , 

**onStop()**调用之后,activity就有可能被回收. 

**onRestart()**从消失到可见时候调用.

# Activity与Window

Activity只负责生命周期和事件处理
Window只控制视图
一个Activity包含一个Window，如果Activity没有Window，那就相当于Service.

