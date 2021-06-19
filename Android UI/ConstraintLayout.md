[TOC]

### XXX_toXXXOf

`layout_constraintXXX_toXXXOf` 将控件的某个边和指定控件或父容器连接, 例如 : 

```xml
app:layout_constraintBottom_toBottomOf="@id/text_view"
app:layout_constraintEnd_toEndOf="parent"
```



### 参考线  Guideline 

`android.support.constraint.Guideline` 参考线控件, 就是一个用来标记的线 , 和其他View一样 , 这样其他控件就可以依照他来定位. 例如:

```xml
<android.support.constraint.Guideline
        android:id="@+id/guideline"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintGuide_percent="0.33" />
```

`android:orientation="horizontal"` 是指这条线是横着还是竖着 . 

`app:layout_constraintGuide_percent` 属性则是他在父控件中33%的位置. 也可以不用这个属性而使用marginTop之类的.



### 线性布局权重

```xml
app:layout_constraintHorizontal_weight=“1” //水平方向权重
app:layout_constraintVertical_weight=“1” //垂直方向权重
```



### 百分比

```java
layout_constraintVertical_bias：垂直偏移率。
layout_constraintHorizontal_bias：水平偏移率。
  
layout_constraintHeight_percent：高度百分比，占父类高度的百分比
layout_constraintWidth_percent：宽度百分比，占父类宽度的百分比
  
app:layout_constraintDimensionRatio="16:6"这个属性，这里设置宽高比为16：6
```



### 线性排列模式

```java
app:layout_constraintXXXX_chainStyle="spread" 				// 不占满/ 平局分布
app:layout_constraintXXXX_chainStyle="packed"					// 挤在一起
app:layout_constraintXXXX_chainStyle="spread_inside"	// 占满 / 平均分布
```



### Baseline 基线

```java
app:layout_constraintBaseline_toBaselineOf="id"  // 与其他View水平基线对齐
```



### 圆形布局

```java
app:layout_constraintCircle   // 引用控件的id
app:layout_constraintCircleAngle   // 据引用id的角度，从0度到360度
app:layout_constraintCircleRadius  // 半径(据引用id圆心的距离，可变)
```



### 组
将btn1和btn2放到了一个组里 , 如果想隐藏这两个控件只需要将这个组设置为GONE .
```xml
<androidx.constraintlayout.widget.Group
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:constraint_referenced_ids="btn1 , btn2" />
```



### 占位

Placeholder控件是一个占位控件 , 通过 `app:content = "@id/btn3"` 设置他的内容是btn3这个控件 , 然后btn3就会直接套用Placeholder的宽高和位置属性了 , 代码里可以通过 `setContentId(R.id.btn1)` 设置 , 但是具体啥用没遇到过...

```xml
<androidx.constraintlayout.widget.Placeholder
    android:layout_width="300dp"
    android:layout_height="300dp"
    app:content = "@id/btn3"
    app:layout_constraintLeft_toLeftOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
 <Button
    android:id="@+id/btn3"
    android:layout_width="20dp"
    android:layout_height="wrap_content"
    android:background="@android:color/black" />
```



### ConstraintSet

如果你想在ConstraintLayout中用代码动态添加一个View怎么办? 或者想用代码改变控件的约束关系 , 其他布局都在五行中 , 但是这个是后期扩展的ViewGroup , 且核心机制都变化很大 , 所以就新增了一个用于设置约束布局属性的类 `ConstraintSet` (我猜的) ,  他的工作流程大致就这样:

```java
// 先创建一个实例
ConstraintSet constraintSet = new ConstraintSet();
ConstraintLayout root = v.getRoot();

// 创建一个TextView
TextView textView = new TextView(this);
int gId = View.generateViewId();
textView.setId(gId);
textView.setText("啊啊啊");
textView.setTextColor(Color.parseColor("#ffffff"));
root.addView(textView);

// 然后用clone()方法 , 将当前现有的一个ConstraintLayout的布局拷贝过来
constraintSet.clone(root);
// 这个方法是连接两个小部件的 , 其实就是建立约束关系
constraintSet.connect(R.id.btn1 , ConstraintSet.RIGHT , gId , ConstraintSet.LEFT);
// 调用这个方法生效 , 相当于把改过的约束关系写回某个ConstraintLayout节点
constraintSet.applyTo(root);
```

但是实际操作了一下 , 不好用.....



