## Hanlder源码分析
> Handler是android中引入的一种让开发者参与处理线程中消息循环的机制,本文主要是对Handler表述一下个人的理解，如有错误还请指教~~~
#### 主要类
 Handler的内部实现主要涉及到如下几个类: Thread、MessageQueue和Looper,下面我给大家一一介绍这几个相关类
##### MessageQueue
消息队列,顾名思义就是存放线程的队列~~，安卓把一些事件处理成message，将其放到MessageQueue中，也就是说一个Message对象就是一件需要处理的事情,消息队列就是一些message池，MessageQueue中有两个比较重要的方法，一个是enqueueMessage方法，一个是next方法。enqueueMessage方法用于将一个Message放入到消息队列MessageQueue中，next方法是从消息队列MessageQueue中阻塞式地取出一个Message。
##### Looper
消息队列MessageQueue只是存储Message的地方，真正让消息队列循环起来的是Looper，可以说MessageQueue是水，那Looper就是大自然的搬运工~~~线程Thread和Looper是一对一绑定的，也就是一个线程中最多只有一个Looper对象，当我们创建一个线程的时候，是没有MessageQueue的，需要调用Looper两个静态方法Looper.prepare()和Looper.loop()
  ``` java
  
      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                
              }
          };

          Looper.loop();
      }
```
Looper通过如下代码保存了对当前线程的引用：
  ``` java
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
所以说在Looper对象中通过hreadLocal可以找到其绑定的线程,set()和get()方法进行读取,
我们再来看一下Looper.prepare(),
``` java
  private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
}
```
对的，前面提到的这个方法就是把Looper对象通过ThreadLocal传如到线程中
上面的代码执行了Looper的构造函数
``` java
 private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
}
```
我们可以看到在其构造函数中实例化一个消息队列MessageQueue
需要注意的是，在一个线程中，只能调用一次Looper.prepare()，因为在第一次调用了Looper.prepare()之后，当前线程就已经绑定了Looper，在该线程内第二次调用Looper.prepare()方法的时候，sThreadLocal.get()会返回第一次调用prepare的时候绑定的Looper，不是null，这样就会走的下面的代码throw new RuntimeException(“Only one Looper may be created per thread”)，从而抛出异常，告诉开发者一个线程只能绑定一个Looper对象。
在调用了Looper.prepare()方法之后，当前线程和Looper就进行了双向的绑定，这时候我们就可以调用Looper.loop()方法让消息队列循环起来了。
Looper.loop()的源码如下:
``` java
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        1.
       变量me是通过静态方法myLooper()获得的当前线程所绑定的Looper，me.mQueue是当前线程所关联的消息队列。
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        //注意下面这行
        for (;;) {
        2.
           我们通过消息队列MessageQueue的next方法从消息队列中取出一条消息，如果此时消息队列中有Message，那么next方法会立即返回该Message，如果此时消息队列中没有Message，那么next方法就会阻塞式地等待获取Message。 
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

         3.该代码的意思是让Message所关联的Handler通过dispatchMessage方法让Handler处理该Message
            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
}
```
#####  Handler
Handler具有多个构造函数
在Handler中所有可以直接或间接向消息队列发送Message的方法最终都调用了sendMessageAtTime方法，该方法的源码如下：
``` java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //注意下面这行代码
        return enqueueMessage(queue, msg, uptimeMillis);
}
```
enqueueMessage方法
``` java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
      1.代码将Message的target绑定为当前的Handler 
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
      2.变量queue表示的是Handler所绑定的消息队列MessageQueue，通过调用queue.enqueueMessage(msg, uptimeMillis)我们将Message放入到消息队列中。
        return queue.enqueueMessage(msg, uptimeMillis);
}
```
Handler提供了三种途径处理Message，而且处理有前后优先级之分：首先尝试让postXXX中传递的Runnable执行，其次尝试让Handler构造函数中传入的Callback的handleMessage方法处理，最后才是让Handler自身的handleMessage方法处理Message。
##### 总结
我们借助网上一个比较恰当的比喻，传送带运输货物.我们可以把传送带上的货物看做是一个个的Message，而承载这些货物的传送带就是装载Message的消息队列MessageQueue。我们可以把发送机滚轮看做是Looper。我们可以把发送机滚轮看做是Looper，开关就是Looper的loop方法，当我们按下开关的时候，我们就相当于执行了Looper的loop方法，此时Looper就会驱动着消息队列循环起来。Hanlder相当于管道,endMessageXXX方法就相当于放入货物的过程,当货物划出管道的时候就相当于调用了Hanlder的dispatchMessage方法，在该方法中我们完成对Message的处理。
可能我理解的会有所偏差，如有错误请大佬指出~~共同学习.
[Github](https://github.com/icuihai)
email：icuihai@gmail.com
