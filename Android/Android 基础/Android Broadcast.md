[TOC]

# Broadcast使用

1. 定义

   ```java
   public class MyBroadcast extends BroadcastReceiver {
       @Override
       public void onReceive(Context context, Intent intent) {

       }
   }
   ```

2. 注册

   **静态注册**

   ```xml
           <receiver android:name=".MyBroadcast">
               <intent-filter>
                   <action android:name="android.intent.action.PACKAGE_FIRST_LAUNCH"/>
               </intent-filter>
           </receiver>
   ```

   **动态注册**

   ```java
   IntentFilter intentFilter = new IntentFilter();
   intentFilter.addAction("android.provider.Telephony.SMS_RECEIVED");
   registerReceiver(new MyBroadcast() , intentFilter);
   ```

3. 发送

   ```java
   Intent intent = new Intent();
   intent.setAction("android.provider.Telephony.SMS_RECEIVED");
   sendBroadcast(intent);
   ```



# 工作流程

AMS:  ActivityManagerService , 

1. **静态注册**

   静态注册是在应用安装时由系统自动完成注册 , 具体是由PackageManagerService来完成的 , 其他三大组件也都是通过PMS解析并注册的. 具体我们需要分析PMS的工作过程了.

2. **动态注册**

   动态注册过程是从ContextWrapper的registerReceiver方法开始的 , 真正的实现在ContextImpl的registerReceiver()里面 , 在这个方法中, 系统首先从mPackageInfo或者从LoadedApk.ReceiverDispatcher获取IIntentReceiver对象 , 再由AMS发送广播注册请求 . 

# 注意事项

- 因为静态注册耗电、占内存、不受程序生命周期影响，所以Google在Android 8.0上禁止了大部分广播的静态注册，以此来减少耗电、增加待机时间、节省内存空间、提升性能。







































