

DI（依赖注入）设计模式已经不是什么新鲜技术了，但是最近频繁在 Android 开发中用到，主要是因为它提供了一些很棒的框架。

在开始一个例子之前, 我们需要先弄清楚大致的用法 , 依赖注入说白了就是将对象的初始化拿到另外的地方去做 , 既然需要单独抽出来做初始化的功能 , 那么必将有一个可以配置的地方 , 不然谁知道你要实例化哪个对象 ? Spring框架是通过XML来配置对象初始化的 , 而Dagger2则通过Java注解的方式来完成 , 既然是注解 , 那么就看看常用的几个注解意思:

- @Inject 标明所需要实例化的对象和该对象的构造方法.
- @Component 
- @Module 
- @Provides 

话不多说 , 看个例子先.

- 添加APT插件

  ```xml
  dependencies {
          classpath 'com.android.tools.build:gradle:2.1.2'
          //添加apt插件
          classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
      }
  ```

- 添加库

  ```xml
  apply plugin: 'com.android.application'
      //添加如下代码，应用apt插件
      apply plugin: 'com.neenbedankt.android-apt'
      ...
      dependencies {
      ...
      compile 'com.google.dagger:dagger:2.4'
      apt 'com.google.dagger:dagger-compiler:2.4'
      //java注解
      compile 'org.glassfish:javax.annotation:10.0-b28'
      ...
  }
  ```

- 代码

  我们来实现一个MVP模式下的Home页的显示 , 为了突出重点 , 我们省去了接口 , 涉及到的类有HomeActivity , HomePresenter , HomeModle ,  分别简称为 A , P , M我们简单用普通方式实现一下:

  ```java
  public class HomeActivity extends AppCompatActivity {
  
      HomePresenter mPresenter;
      TextView textView;
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          textView = (TextView) findViewById(R.id.page_content);
          // 创建Presenter 并开始获取内容
          mPresenter = new HomePresenter(this);
          mPresenter.getPage();
      }
  
      /**
       * Presenter用来设置数据的方法
       * @param content
       */
      public void setPageContent(String content){
          textView.setText(content);
      }
  
  }
  ```



  public class HomePresenter {

      private HomeActivity homeView;
      private HomeModel mModel;
    
      // 构造器中创建Model
      public HomePresenter(HomeActivity homeView){
          this.homeView = homeView;
          mModel = new HomeModel();
      }
    
      // 获取主页内容
      public void getPage(){
          mModel.getPage(pageText -> homeView.setPageContent(pageText));
      }
  }



  public class HomeModel {
      /**
       * 假装从服务器获取了主页的内容
       * @param callback
       */
      public void getPage(Callback callback){
          callback.call("This is a homeActivity");
      }

      /**
       * 一个回调 不用管它
       */
      public interface Callback{
          void call(String pageText);
      }
  }
  ```

  省略了接口 , 一个超简易MVP结构的主页完成了 . 现在我们把焦点放在对象的创建上 , A创建P , P创建M , 他们为什么要创建对象 ? 因为要使用 , 那么创建的职责是被强迫的 , 并且这些创建在MVP架构中几乎是必须的 , 所以创建对象这个职责是不是可以抽离出来 ? 那么依赖注入就解决了这个问题. 但是在这里我们不再说依赖注入了 , 直接看看Dagger2为我们提供了那些便利 . 

  使用Dagger2之后的代码:

  先更正一下名字 :HomeActivity , HomePresenter , HomeData , 这三个类是MVP框架中的类 , HomeModule , HomeComponent是Dagger2框架所需要的类.

  首先 , 我们需要新增两个类:

  HomeModule  /  HomeComponent :

  ```java
  @Module
  public class HomeModule {
      private final HomeActivity view;
      public HomeModule(HomeActivity view){
          this.view = view ;
      }

      @Provides
      HomeActivity provideHomeView(){
          return view ;
      }

      @Provides
      HomeData provideHomeData(){
          return new HomeData();
      }
  }

  @Component(modules = HomeModule.class)
  public interface HomeComponent {
      void inject(HomeActivity activity);
  }
  ```

  HomeActivity:

  ```java
  public class HomeActivity extends AppCompatActivity {

      // 此处使用了Inject 同时 onCreate中的创建工作可以删掉了
      @Inject
      HomePresenter mPresenter;

      TextView textView;

      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          textView = (TextView) findViewById(R.id.page_content);
          // 通过这句完成依赖注入
          DaggerHomeComponent.builder().homeModule(new HomeModule(this)).build().inject(this);
          mPresenter.getPage();
      }

      /**
       * Presenter用来设置数据的方法
       * @param content
       */
      public void setPageContent(String content){
          textView.setText(content);
      }
  }
  ```

  HomePresenter:

  ```java
  public class HomePresenter {

      private HomeActivity mHomeView;
      private HomeData mHomeData;

      // 此处也使用了注解 Inject
      // 同时初始化参数变成了两个 , 多了一个HomeData
      @Inject
      public HomePresenter(HomeActivity homeView , HomeData model){
          this.mHomeView = homeView;
          mHomeData = model;
      }

      // 获取主页内容
      public void getPage(){
          mHomeData.getPage(pageText -> mHomeView.setPageContent(pageText));
      }
  }
  ```

  至此 , 使用Dagger2将代码改造完毕 , Dagger2会根据你的HomeComponent和HomeModule生成DaggerHomeComponent类 , 那么里面到底做了什么? 看下他的代码:

  ```java
  @SuppressWarnings("unchecked")
    private void initialize(final Builder builder) {

      this.provideHomeViewProvider = HomeModule_ProvideHomeViewFactory.create(builder.homeModule);

      this.provideHomeDataProvider = HomeModule_ProvideHomeDataFactory.create(builder.homeModule);

      this.homePresenterProvider =
          HomePresenter_Factory.create(provideHomeViewProvider, provideHomeDataProvider);

      this.homeActivityMembersInjector = HomeActivity_MembersInjector.create(homePresenterProvider);
    }
  ```

  provideHomeViewProvider , provideHomeDataProvider , homePresenterProvider , 分别就是HomeActivity,HomeData 和HomeProvider了 , 刚才我们创建的Module类 , 其实就是一堆创建实例的方法 , 然后DaggerHomeComponent类中为我们将这些实例都创建了出来 , 同时也将创建实例时候的依赖关系囊括了.





# 热修复

## 热修复的原理

热修复的使用场景是什么样的? 在上线项目中, 一个类中的某个方法出了问题需要修改 , 这时候就可以重新写一个同名的方法 , 修复已存在的问题 , 然后局部修复 . 怎么做到的? 那么就思考一个问题 , 方法是如何加载到内存中的 . 据不可靠消息称 , Android程序中所有的方法都维护在一张表中 , 待用到某个方法到时候 , 该方法将会加载到内存的栈中, 然后从栈顶依次执行 . 

看到这里就差不多找到了偷梁换柱的地方机会 , 这张表怎么来的? 如何在生成这张表的时候将出BUG的方法替换为修复后的方法 ?

android sdk build-tools-

dx --dex  --output=输出路径 从哪读取class