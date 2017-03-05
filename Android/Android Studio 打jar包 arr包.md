---
title: Android Studio 下导出带混淆Jar包
date: 2017-3-3
categories: Android
---



# Android Studio 下导出带混淆Jar包 

偶然间遇到了将Module打成jar包的需求 , 并且需要混淆 , 网上很多资料都有些不尽人意 , 这里分享一下操作流程 .  

<!-- more -->

1. 首先你要有一个Android Library的Module , 并已经跟主Module为依赖关系 .
2. 然后在这个Module下的build.gradle中加入如下代码 , 注释已经很清楚了 . 

```xml
task clearJar(type: Delete) {
    delete 'build/libs/XXX.jar'////这行表示如果你已经打过一次包了，再进行打包则把原来的包删掉
}

task makeJar(type: Copy) {
    from('build/intermediates/bundles/release/') //这行表示要打包的文件的路径，根据下面的内容，其实是该路径下的classes.jar
    into('build/libs/')  //这行表示打包完毕后包的生成路径，也就是生成的包存在哪
    include('classes.jar')  //看到这行，如果你对分包有了解的话，你就可以看出来这行它只是将一些类打包了
    rename ('classes.jar', 'XXX.jar')
}

makeJar.dependsOn(clearJar, build)

task proguard(type: proguard.gradle.ProGuardTask, dependsOn: makeJar) {
//  输入路径
    injars "build/libs/XXX.jar"
//  输出路径
    outjars 'libs/XXX.jar'
//  添加配置信息
    configuration 'proguard-rules.pro'
}
```



```xml
	buildTypes {
        release {
            minifyEnabled true // 这里需要改为true, 启用混淆功能
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```

3. 因为开启了混淆功能 , 你提供的接口或某些类不应该参与混淆 , 所以需要配置`proguard-rules.pro`文件 :

```xml
-libraryjars 'C:\Program Files\Java\jdk1.8.0_20\jre\lib\rt.jar'
-libraryjars 'C:\Users\AAAA\AppData\Local\Android\aaa\sdk\platforms\android-21\android.jar'
-optimizationpasses 5 // 混淆等级
-keep class com.ooo.toothbrush.api.** {*;}  // com.ooo.toothbrush.api下的所有类都不参与混淆
-keep class com.ooo.toothbrush.callback.** {*;} // 同上
```

3. 在Android Studio下的Terminal命令行窗口中执行 `gradlew makeJar`命令 , 稍等片刻......
4. 成功之后会显示 `BUILD SUCCESSFUL` . 在你Module下的build/libs 就会出现一个jar包了 , 是他是他就是他 . 

至此 jar包导出完毕 . 