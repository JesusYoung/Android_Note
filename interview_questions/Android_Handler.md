# Android中的Handler

常见问题：

- 说一说Android中的Handler机制；
- 在子线程中可以创建多个Handler吗，如何确保Handler发送的消息能被对应的Handler处理；
- 如何确保线程中Looper的唯一；
- 子线程中使用Handler；



## Handler处理消息顺序

Handler将消息发送到MessageQueue保存，通过Looper去轮询MessageQueue取出消息，然后调用消息对应的Handler的dispatchMessage()方法去处理消息；

```java
# Looper
public static void loop() {
  ...
  for (;;) {
    Message msg = queue.next(); // might block
    ...
    try {
      msg.target.dispatchMessage(msg);
      ...
    }
    ...
  }
}
```

由此可见当Looper轮询取出Message消息时，会去找到该消息对应的Handler，执行该Handler的dispatchMessage()方法去处理消息；

```java
public void dispatchMessage(@NonNull Message msg) {
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
```

首先会判断该Message消息是否自带有Callback，如果有，则将任务交给Message的Callback去执行，如果消息不带有Callback，则会判断Handler本身的Callback对象是否为空，如果不为空，则调用Handler本身的Callback对象的handleMessage()方法去处理任务，当该Callback的handleMessage()方法处理完任务，如果返回true，则表示该消息Message已经完全消费完了，如果Handler的Callback为空，或者Callback的handleMessage()方法返回false，则消息还会继续交给Handler的handleMessage()方法去处理；

### 1、Message的Callback

Message消息类中，有个Runnable类型的成员变量callback，当创建消息的时候，如果初始化了该Runnable对象，则表明该Message消息会通过自带的callback对象去执行，不再由Handler的方法去执行；

```java
@UnsupportedAppUsage
/*package*/ Runnable callback;

# Handler
private static void handleCallback(Message message) {
  message.callback.run();
}
```

初始化消息的时候，可以将Runnable对象传入，也可以通过setCallback()方法为Message设置Callback；

```java
public static Message obtain(Handler h, Runnable callback) {
  Message m = obtain();
  m.target = h;
  m.callback = callback;
  return m;
}

@UnsupportedAppUsage
public Message setCallback(Runnable r) {
  callback = r;
  return this;
}
```

### 2、Handler的Callback

Handler内部有一个Callback接口，并且Handler有个成员变量Callback对象；

```java
final Callback mCallback;
public interface Callback {
  boolean handleMessage(@NonNull Message msg);
}
```

当Message消息不带有Runnable对象时，会判断Handler的成员变量mCallback是否为空，如果不为空，则将消息交给该Callback对象去处理，Callback对象的handleMessage()方法的返回值是boolean类型，表示如果返回为true则该消息已经处理完毕，如果返回false，则该消息最终还是会交给Handler的handleMessage()方法去处理；

创建Handler的时候，可以通过构造函数传入Callback对象；

```java
public Handler(@Nullable Callback callback) {
  this(callback, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback) {
  this(looper, callback, false);
}

public Handler(@Nullable Callback callback, boolean async) {
  ...
  mQueue = mLooper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}

public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
  ...
  return new Handler(looper, callback, true);
}
```

### 3、Handler的handleMessage()方法

当Message中不带有Runnable，并且Handler本身没有Callback处理或者Handler的Callback对象处理消息返回为false时，Message消息最终会交给Handler的handleMessage()方法去处理，该方法为一个空实现，需要由开发者去实现处理消息的逻辑，通常创建Handler时需要复写该方法；

```java
public void handleMessage(@NonNull Message msg) {
}
```



## 子线程中使用Handler

通常在主线程中使用Handler时，可以直接创建Handler，但是在子线程中使用Handler时，需要手动去创建Looper对象，并且调用Looper对象的loop()方法去轮询消息，因为子线程中默认没有创建Looper，在主线程即UI线程中不需要手动创建Looper对象是因为在App启动的时候，在ActivityThread的main()方法中就已经创建出了主线程的Looper对象，并调用了Looper的loop()方法；

```java
# ActivityThread
public static void main(String[] args) {
  ...
  Looper.prepareMainLooper();
  ...
  ActivityThread thread = new ActivityThread();
  thread.attach(false, startSeq);
  if (sMainThreadHandler == null) {
    sMainThreadHandler = thread.getHandler();
  }
  ...
  Looper.loop();
  ...
}
```

在App启动的时候，Zygote进程fork出App进程之后，即ActivityThread进程，执行其main()方法，在这里调用Looper.prepareMainLooper()方法去创建出主线程的Looper，最后执行Looper的loop()方法去轮询主线程的MessageQueue队列；

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

public static @Nullable Looper myLooper() {
  return sThreadLocal.get();
}
```

调用Looper的prepare()方法创建Looper对象，参数传入false表示主线程的loop()循环不允许退出，需要一直轮询消息保持主线程一直工作，当主线程退出也就意味着该App进程结束了，创建出Looper对象保存达到ThreadLocal中，在创建时首先会从ThreadLocal中去获取当前线程是否已经有Looper对象，如果有则会抛出异常，一个线程只能创建一个Looper对象，这样保证了线程和Looper的一对一关系；

创建出的主线程的Looper赋值给sMainLooper对象，在主线程中可以直接通过Looper.myLooper()方法获取主线程的Looper对象；

测试一：子线程中创建Handler不调用Looper的prepare()方法

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      
      // 在主线程中直接获取主线程的Looper
      Looper mainLooper = Looper.myLooper();
      Log.e("tag", "current thread-" + Thread.currentThread().getName() + "'s looper is : " + mainLooper.toString());
      
      new Thread(new Runnable() {
        @Override
        public void run() {
          // 在子线程中直接获取子线程的Looper
          Looper looper = Looper.myLooper();
          if (looper == null) {
            Log.e("tag", "current thread-" + Thread.currentThread().getName() + "'s looper is null");
          } else {
            Log.e("tag", "current thread-" + Thread.currentThread().getName() + "'s looper is : " + looper.toString());
          }
          // 在子线程中获取获取该子线程对应的主线程的Looper对象
          Looper mainL = Looper.getMainLooper();
          Log.e("tag", "current thread-" + Thread.currentThread().getName() + "'s main looper is : " + mainL.toString());
        }
      }).start();
    }
}

## 日志：
E/tag: current thread-main's looper is : Looper (main, tid 2) {3ba7e5e}
E/tag: current thread-Thread-2's looper is null
E/tag: current thread-Thread-2's main looper is : Looper (main, tid 2) {3ba7e5e}
```

由此可见：

- 在主线程中可以直接获取到主线程的Looper对象，在子线程获取主线程的Looper对象，两者打印出是同一个对象；
- 在子线程中直接获取子线程的Looper对象，打印出为Looper对象为null，所以子线程中默认是没有Looper对象；

测试二：子线程中创建Handler并且调用Looper的prepare()方法

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      new Thread(new Runnable() {
        @Override
        public void run() {
          // 在子线程中直接获取子线程的Looper
          Looper.prepare();
          Looper looper = Looper.myLooper();
          ...
        }
      }).start();
    }
}

## 日志：
E/tag: current thread-main's looper is : Looper (main, tid 2) {3ba7e5e}
E/tag: current thread-Thread-2's looper is : Looper (Thread-2, tid 4964) {39ca23f}
E/tag: current thread-Thread-2's main looper is : Looper (main, tid 2) {3ba7e5e}
```

由此可见，在子线程中调用Looper.prepare()方法，会为子线程创建对应的Looper对象；

因此在子线程中使用Handler时，需要手动去调用Looper的prepare()方法去为该线程创建Looper，并且执行Looper的loop()方法去轮询该Handler对应的MessageQueue消息队列，Looper的prepare()方法默认调用自己的prepare()方法，参数为true，表示该Looper可以退出轮询，因此在子线程结束时，需要去终止looper的轮询；



## 多个Handler发送消息，Looper取到如何给对应Handler处理

在一个线程中可以创建出多个Handler对象，通过Handler对象去发送消息，但是一个线程中的Looper是唯一的，当Looper轮询MessageQueue队列取出消息时，该如何通知对应的Handler去处理消息呢？

Handler发送消息，创建消息的方法有多种；

- obtainMessage

调用Handler的obtainMessage()方法，会传入一个this，也就是传入了Handler本身，最终都会调用到Message类的obtain()方法，并将Message的成员变量target赋值为传过来的Handler对象；

```java
public final Message obtainMessage() {
  return Message.obtain(this);
}

public final Message obtainMessage(int what) {
  return Message.obtain(this, what);
}

# Message
public static Message obtain(Handler h) {
  Message m = obtain();
  m.target = h;
  return m;
}
```

- 调用Message的构造函数直接创建

直接调用Message的构造函数创建Message对象，并为其设置相关参数；

```java
public Message() {
}

Message msg = new Message();
msg.what = xx;
```

发送消息，当创建出消息之后，Handler去发送消息；

- sendMessageXXX

- postXXX

Handler的sendMessageXXX()方法和postXXX()方法最终都会调用到enqueueMessage()方法；

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
  msg.target = this;
  msg.workSourceUid = ThreadLocalWorkSource.getUid();
  if (mAsynchronous) {
    msg.setAsynchronous(true);
  }
  return queue.enqueueMessage(msg, uptimeMillis);
}
```

在该方法中，会为Message对象的target属性赋值为this，即Handler本身，所以是哪个Handler发送的消息，都会绑定在Message对象中，Message类中有个成员变量Handler类型的target对象；

当Looper的loop()方法从MessageQueue队列中取出Message时，会通过该Message的target对象的dispatchMessage()方法去执行，即交给了对应的Handler对象去执行；

```java
# Looper
public static void loop() {
	...
  for (;;) {
    Message msg = queue.next(); // might block
    ...
    try {
      // 调用Message对应的Handler去执行
      msg.target.dispatchMessage(msg);
      ...
    }
    ...
  }
}
```

通过这样来保证Message消息会交给对应的Handler去处理，不会错乱；