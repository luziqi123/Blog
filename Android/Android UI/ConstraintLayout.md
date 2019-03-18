`layout_constraintXXX_toXXXOf` 将控件的某个边和指定控件或父容器连接, 例如 : 

```xml
app:layout_constraintBottom_toBottomOf="@id/text_view"
app:layout_constraintEnd_toEndOf="parent"
```



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



