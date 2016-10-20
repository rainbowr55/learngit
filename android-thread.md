####Android消息处理机制 之 Looper、Handler、Message 学习
在很多时候，线程不仅仅是线性执行一系列的任务就结束那么简单的，我们会需要增加一个任务队列，让线程不断的从任务队列中获取任务去进行执行，另外我们还可能在线程执行的任务过程中与其他的线程进行协作。如果这些细节都交给我们自己来处理，这将会是件极其繁琐又容易出错的事情，Android中提供了Looper，Handler，MessageQueue来帮助实现线程任务模型，这也是Android中必须理解的重要内容，下面我们来分析Android中的多线程任务模型的实现。
  
我们知道，Android应用程序是通过消息来驱动的，即在应用程序的主线程（UI线程）中有一个消息循环，负责处理消息队列中的消息。我们也知道，Android应用程序是支持多线程的，即可以创建子线程来执行一些计算型的任务，那么，这些子线程能不能像应用程序的主线程一样具有消息循环呢？这些子线程又能不能往应用程序的主线程中发送消息呢,下面将分析Android应用程序线程消息处理模型。

我们知道Android的主线程是从ActivityThread类的 main(String[] args)方法启动的
```java

    public static void main(String[] args) {
      //······
       Looper.prepareMainLooper();

       ActivityThread thread = new ActivityThread();
       thread.attach(false);

       if (sMainThreadHandler == null) {
           sMainThreadHandler = thread.getHandler();
       }

       if (false) {
           Looper.myLooper().setMessageLogging(new
                   LogPrinter(Log.DEBUG, "ActivityThread"));
       }

       // End of event ActivityThreadMain.
       Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
       Looper.loop();

       throw new RuntimeException("Main thread loop unexpectedly exited");

    }

```
在Main函数中调用了Looper.prepareMainLooper方法具体看看它都做了什么
```java
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

 private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
在prepareMainLooper方法中调用了 prepare()方法，prepare()方法从sThreadLocal重get一个Looper对象
```java
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
我们看到 sThreadLocal是一个ThredLocal对象，我们之前学习过ThreadLocal可以为不同线程提供不同的数据副本，可以在不同线程中互不干扰地存储并提供数据，prepare方法为线程准备或者新建一个当前主线程专有的Looper对象，并可以通过ThreadLocal获取到当前线程的Looper,也就是mylooper()方法 
```java
  public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
如果还不了解ThreadLocal的机制可以参阅[https://github.com/rainbowr55/learngit/blob/master/threadLocal.md](https://github.com/rainbowr55/learngit/blob/master/threadLocal.md "关于ThreadLocal") ，我们知道在prepare()方法中为线程创建了Looper对象
```java
 private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
Looper创建后持有了当前线程的对象，并且创建了一个MessageQueue对象,MessageQueue 顾名思义就是消息队列，我们接着看Main方法里调用的Looper.loop()方法
```java
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
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
     //·····
    }
```
在loop方法中 再次使用myLooper方法获取当前线程的Looper对象及该Looper对象维护的消息队列MessageQueue对象
紧接着进入一个循环不断地从消息队列中取出消息Message,如果没有消息则阻塞，阻塞是在Message msg = queue.next(); // might block中实现的，阻塞主要是epoll_wait
```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        //注意当 MessageQueue 没有消息时，是不会返回 null 的，只会一直循环等待消息。
        //只有当调用 quit 方法退出时，才会返回 null。
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        //···········
    }

```
我们再回到loop方法，从线程取出消息之后调用消息目标事务  msg.target.dispatchMessage(msg); 处理消息的部分已经分析完了，那么我们现在就可以往looper里的消息队列里添加消息并且为message的target赋值来完成把一些工作放到线程中去执行了，Android为我们提供了Handler这个类来完成这些工作。下面看看它是怎么工作的
```java

   /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     * 使用looper来获取当前线程并设置特定的回调
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     * Handler默认初始化为同步的，除非使用特定的构造函数来严格设定其为异步的
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     * 异步消息相对于同步消息表现出中断或者不要求全局的顺序
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```
Handler的初始化中通过Looper.myLooper()获取了当前线程保存的Looper实例 又获取了这个Looper实例中保存的MessageQueue 这样就保证了handler的实例与我们Looper实例中MessageQueue关联上了，并且可以给线程发消息了
我们接着Handler的SendMessage方法
```java
 public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```
sendMessageDelayout最终调用了sendMessageAtTime
```java
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```
enqueueMessage方法最终会给当前线程中的looper中的消息队列添加一个消息，并且这个消息的target设置成Hadler本身，介绍Looper的loop方法的时候我们知道Looper的loop方法会取出每个msg然后交给msg,target.dispatchMessage(msg)去处理消息，现在已经很清楚了Looper会调用prepare()和loop()方法，在当前执行的线程中保存一个Looper实例，这个实例会保存一个MessageQueue对象，然后当前线程进入一个无限循环中去，不断从MessageQueue中读取Handler发来的消息。然后再回调创建这个消息的handler中的dispathMessage方法
```java
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }


     /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
```
dispathMessage方法会调用 handleMessage()方法，这个方法交给子类去实现，我们就可以在Handler的子类中实现该方法并调用 sendMessage方法后就能实现在线程中处理一些工作啦。

比如，我们知道View的更新不能放在子线程中，如果要对一些文件进行下载，这个下载的工作很繁重，为了不阻塞主线程，肯定要放在子线程中，在下载完成之后，更新UI就必须放在主线程，这时可以使用在主线程中创建的Handler来通知主线程更新UI 

在上面的dispatchMessage方法中还判断了一个msg的callback对象，我们来看看callback对象究竟是什么鬼，在Message类中,callback是一个Runnable对象
```java
 /*package*/ Runnable callback;
```
也就是说 handler还能够将其他的Runnable的事务发送到当前线程中去处理
```java
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```
使用Handler的post方法来实现callback的赋值
```java 
mHandler.post(new Runnable()  
        {  
            @Override  
            public void run()  
            {  
                Log.e("TAG", Thread.currentThread().getName());  
                mTxt.setText("yoxi");  
            }  
        });  
```
其实Handler是把Runnable对象包装成了Message
唯一区别的是线程处理消息的回调会区分回调的方式，如果是Runnable 则回调Runnable 的run方法。
```java
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }


  private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```
可以看到，在getPostMessage中，得到了一个Message对象，然后将我们创建的Runable对象作为callback属性，赋值给了此message.
注：产生一个Message对象，可以new  ，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new 重新分配内存

总结 在Looper类中可以看到对该类的说明 ，同时还给出了典型的应用例子，这段英文注释很好地解释了Looper Thread Message Handler的关系
```java
/**
  * Class used to run a message loop for a thread.  Threads by default do
  * not have a message loop associated with them; to create one, call
  * {@link #prepare} in the thread that is to run the loop, and then
  * {@link #loop} to have it process messages until the loop is stopped.
  *
  * <p>Most interaction with a message loop is through the
  * {@link Handler} class.
  *
  * <p>This is a typical example of the implementation of a Looper thread,
  * using the separation of {@link #prepare} and {@link #loop} to create an
  * initial Handler to communicate with the Looper.
  */
    class LooperThread extends Thread {
        public Handler mHandler;
   
         public void run() {
            Looper.prepare();
   
             mHandler = new Handler() {
                 public void handleMessage(Message msg) {
                     // process incoming messages here
                 }
             };
   
             Looper.loop();
       }
  
```
![](http://upload-images.jianshu.io/upload_images/1903766-0864d85a80072fbc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
