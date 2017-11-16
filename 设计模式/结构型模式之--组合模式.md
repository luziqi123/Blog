[TOC]

#结构型模式之 组合模式

##前言

对于树形结构，在代码中有容器节点和叶子节点之分，容器节点可以有子节点，而叶子节点没有，所以两者是有区分的，而在实际使用中，我们更希望一致的对待他们，因为如若区别对待，在程序上会非常复杂。组合模式则是为了解决此类问题而生的，它可以让叶子对象和容器对象的使用具有一致性。他是组合多个对象形成树形结构以表示具有“整体—部分”关系的层次结构，故称为组合模式（这个是我猜的……）。

----

##模式定义

组合模式(Composite Pattern)：组合多个对象形成树形结构以表示具有“整体—部分”关系的层次结构。组合模式对单个对象（即叶子对象）和组合对象（即容器对象）的使用具有一致性，组合模式又可以称为“整体—部分”(Part-Whole)模式，它是一种对象结构型模式。
在组合模式中引入了抽象构件类Component，它是所有容器类和叶子类的公共父类，客户端针对Component进行编程。

----

##模式结构

**Component**  抽象构件，叶子构件和容器构件的接口或抽象类


**Leaf**  叶子构件，叶子节点没有子节点


**Composite**  容器构件，容器节点可以有子节点，子节点也可以是容
器构件。

----

##UML类图

![](http://img.blog.csdn.net/20160817224905662)

----

##适用场景

###在以下情况下可以使用组合模式：

**1**  在具有整体和部分的层次结构中，希望通过一种方式忽略整体与部分的差异，客户端可以一致地对待它们。
**2**  在一个使用面向对象语言开发的系统中需要处理一个树形结构。
**3** 在一个系统中能够分离出叶子对象和容器对象，而且它们的类型不固定，需要增加一些新的类型。

###看看大神怎么用

组合模式使用面向对象的思想来实现树形结构的构建与处理，描述了如何将容器对象和叶子对象进行递归组合，实现简单，灵活性好。由于在软件开发中存在大量的树形结构，因此组合模式是一种使用频率较高的结构型设计模式，Java SE中的AWT和Swing包的设计就基于组合模式，在这些界面包中为用户提供了大量的容器构件（如Container）和成员构件（如Checkbox、Button和TextComponent等）

----


##高清有码

> 前戏：我们就Android的UI做一个简单的例子，界面控件分为两大类：View和ViewGroup。View是单元控件，如TextView，ImageView。ViewGroup是容器节点，如LinearLayout...我们用组合模式来设计这个库。
> 正文开始，未满18岁的可以离场了。

**Demo角色预览：**

*AbstractView*  控件节点接口

*ViewGroup*  容器节点父类

*View* 子节点父类

*LinearLayout* 父节点实现类

*RelativeLayout* 父节点实现类

*TextVeiw* 子节点实现类

*ImageView* 子节点实现类

**AbstractView**

```
/**
 * 控件抽象类
 * Created by R on 2016/8/17.
 */
public interface AbstractView {

    void addView(AbstractView view);

    void removeView(AbstractView view);

    void show();

}

```
**View**

```
/**
 * 子控件的父类
 * Created by R on 2016/8/18.
 */
public abstract class View implements AbstractView{

    String id;

    public View(String id){
        this.id = id;
    }

    @Override
    public void addView(AbstractView view) {
        System.out.println("该控件不支持此方法");
    }

    @Override
    public void removeView(AbstractView view) {
        System.out.println("该控件不支持此方法");
    }

}
```

**ViewGroup**

```
/**
 * 容器控件的父类
 * Created by R on 2016/8/18.
 */
public abstract class ViewGroup implements AbstractView{

    String id;

    public ViewGroup(String id){
        this.id = id;
    }

    protected LinkedList<AbstractView> mChilds = new LinkedList<>();

    @Override
    public void addView(AbstractView view) {
        mChilds.add(view);
    }

    @Override
    public void removeView(AbstractView view) {
        mChilds.remove(view);
    }

}
```

**LinearLayout**

```
/**
 * Created by R on 2016/8/18.
 */
public class LinearLayout extends ViewGroup{

    public LinearLayout(String id) {
        super(id);
    }

    @Override
    public void show() {
        System.out.println("线性布局容器：" + id + " 显示");
        for (AbstractView mChild : mChilds) {
            mChild.show();
        }
    }
}
```

**RelativeLayout**

```
/**
 * Created by R on 2016/8/18.
 */
public class RelativeLayout extends ViewGroup{

    public RelativeLayout(String id) {
        super(id);
    }

    @Override
    public void show() {
        System.out.println("相对布局容器：" + id + " 显示");
        for (AbstractView mChild : mChilds) {
            mChild.show();
        }
    }
}
```

**TextView**

```
/**
 * Created by R on 2016/8/18.
 */
public class TextView extends View{

    public TextView(String id) {
        super(id);
    }

    @Override
    public void show() {
        System.out.println("文字控件：" + id + " 显示");
    }
}
```

**ImageView**

```
/**
 * Created by R on 2016/8/18.
 */
public class ImageView extends View{

    public ImageView(String id) {
        super(id);
    }

    @Override
    public void show() {
        System.out.println("图片控件：" + id + " 显示");
    }
}

```

**Client 使用**

```
/**
 * Created by R on 2016/8/17.
 */
public class Client {

    public void main(){
        AbstractView view1 , view2 , view3, view4 , view5 , view6 , viewGroupRoot, viewGroup2, viewGroup3;

        viewGroupRoot = new LinearLayout("根布局");
        viewGroup2 = new RelativeLayout("容器1");
        viewGroup3 = new LinearLayout("容器2");

        view1 = new TextView("文本1");
        view2 = new TextView("文本2");
        view3 = new TextView("文本3");
        view4 = new ImageView("图片1");
        view5 = new ImageView("图片2");
        view6 = new ImageView("图片3");

        viewGroupRoot.addView(view1);
        viewGroupRoot.addView(viewGroup2);
        viewGroupRoot.addView(viewGroup3);

        viewGroup2.addView(view2);
        viewGroup2.addView(view3);

        viewGroup3.addView(view4);
        viewGroup3.addView(view5);
        viewGroup3.addView(view6);

        viewGroup3.removeView(view6);

        viewGroupRoot.show();
    }
}
```

**输出：**

![输出内容](http://img.blog.csdn.net/20160818005954206)

----

##总结

> 树形结构由容器节点和叶子节点组成，简单来说组合模式就是将容器节点和叶子节点转化为一种节点来使用的设计模式，这样一来便可将精力更专注于树的结构，而不是节点的类型上了。
据我不成熟的猜想，Android中的View系列就用到了组合模式，并且用的淋漓尽致。所有View都是节点，View就是View和ViewGroup的Component，只不过他不是抽象类或接口。而我们在构件界面的时候并不是特别关注节点的类型，而是节点之间的关系。并且在Touch事件中尤为明显，Touch事件可以针对某一个节点，也可以针对某一节点之下的所有节点。

##感谢

[跪在地上谢的站点-所有文章均参考这里](https://quanke.gitbooks.io/design-pattern-java/content/)

[所有文章全部参考了这个网站](http://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/bridge.html#id5)