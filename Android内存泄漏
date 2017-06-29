# Android内存泄漏
## 内存分配
* 静态存储区（方法区）：主要存放静态方法，静态常量等静态数据。这块内存在编译时期已经分配好内存，伴随应用程序整个生命周期
* 栈区 ：当方法执行的时候，方法体中的常量都被栈中创建，方法执行结束这些内存空间都会被释放，new出来的对象的地址也是存在栈中。
* 堆区：又称为动态分配区，在应用程序中值new出来的对象，这部分内存在不使用的时候将会被垃圾回收器(GC)回收。
## 内存泄漏
> 进程中某些对象没有被使用了，但是没有被回收，一直占据着内存空间，时间越长，剩余内存空间就会越来越少，就是说内存泄漏了。

##常见的内存泄漏
#####1. 非静态内部类容易造成内存泄漏
~~~ java
public class MainActivity extends AppCompatActivity {

    static Demo demo;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        demo = new Demo();
        finish();
    }
    class Demo {
    }
}
~~~
上述过程中当Activity执行finish后，因为demo 一直持有Activity类的引用，所以导致Activity结束之后并GC回收器并不能回收。发生内存泄漏.

##### 2.使用Handler出现的问题
Handler使用造成的内存泄漏最为常见，Handler,Message,MessageQueue是相互关联的，万一Handler发送的消息一直很长一段时间或者一直没有被处理，则该Handler以及Handler一直被线程MessageQueue引用，如果此时Activity已经finish但是GC并不能回收它从而导致内存泄漏。
~~~java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
            }
        }, 1000 * 6);

        finish();
    }
    Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
~~~
上面这个例子就是因为handler发送的消息还没有被处理当前的Activtiy已经finish所以导致Activity不能被回收发生内存泄漏,所以推荐使用静态内部类 + WeakReference 这种方式。

##### 3.没有反注册
比如我们使用广播的时候没有进行反注册，也或者使用EventBus等等没有反注册都会引起内存泄漏，正确的做法应该在Activity或者Fragment的onDestroy方法进行反注册
##### 4.资源对象没关闭造成的内存泄露
资源性对象比如(Cursor，File文件等)往往都用了一些缓冲，我们在不使用的时候，应该及时关闭它们，以便它们的缓冲及时回收内存。它们的缓冲不仅存在于Java虚拟机内，还存在于Java虚拟机外.如果我们仅仅是把它的引用设置为null,而不关闭它们，往往会造成内存泄露。因为有些资源性对象，比如SQLiteCursor(在析构函数finalize(),如果我们没有关闭它，它自己会调close()关闭)，如果我们没有关闭它，系统在回收它时也会关闭它，但是这样的效率太低了。因此对于资源性对象在不使用的时候，应该立即调用它的close()函数，将其关闭掉，然后再置为null.在我们的程序退出时一定要确保我们的资源性对象已经关闭

##### 5.Bitmap使用不当
Bitmap在使用的过程总会占用相当大的内存，所以在使用完毕的时候应该调用recycle方法。
##### 6.构造Adapter时，没有使用缓存的 convertView
这个主要在使用ListView的时候初学者可能会犯的错误。当然现在更多的使用Recyclerview，它已经封装好了所以忽略这个问题
##### 7.单例造成的内存泄漏
单例模式非常受开发者的喜爱，不过使用的不恰当的话也会造成内存泄漏，由于单例的静态特性使得单例的生命周期和应用的生命周期一样长，这就说明了如果一个对象已经不需要使用了，而单例对象还持有该对象的引用，那么这个对象将不能被正常回收，这就导致了内存泄漏
~~~
public class AppManager {
private static AppManager instance;
private Context context;
private AppManager(Context context) {
this.context = context;
}
public static AppManager getInstance(Context context) {
if (instance != null) {
instance = new AppManager(context);
}
return instance;
}
~~~
这是一个普通的单例用法，但是要注意的是传入的context必须是Application。否则如果传入的是activity的context，当finish的时候就发生了内存泄漏,最好在写单利的时候直接传入getApplicationContext()
## 内存泄漏检测工具
我在工作中使用过最多的就是LeakCanary，使用方法特别简单
1.首先在工程下的build.gradle下面
​~~~java
debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5.1'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.1'
~~~
2.在Application初始化
​~~~java
@Override
    public void onCreate() {
        super.onCreate();
        LeakCanary.install(this);
    }
~~~
3然后运行你的项目手机会自动安装一个app

![icon](C:\Users\DAOPENG1\Desktop\icon.png)
4.点击的你项目如果发现了内存泄漏的地方通知栏会弹出来一条消息点击打开就可以看到具体发生内存泄漏的地方

![1](C:\Users\DAOPENG1\Desktop\1.png)

## 小结

对 Activity 等组件的引用应该控制在 Activity 的生命周期之内； 如果不能就考虑使用 getApplicationContext 或者 getApplication，以避免 Activity 被外部长生命周期的对象引用而泄露
对于生命周期比Activity长的内部类对象，并且内部类中使用了外部类的成员变量，可以这样做避免内存泄漏： 将内部类改为静态内部类 
静态内部类中使用弱引用来引用外部类的成员变量 
Handler 的持有的引用对象最好使用弱引用
正确关闭资源，对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销。

## 联系我
如果你发现我的文章有出现错误的地方请联系我一起交流，共同学习~~~
[Github](https://github.com/icuihai)

[weibo](https://weibo.com/icuihai)

[Gmail](icuihai@gmail.com)

