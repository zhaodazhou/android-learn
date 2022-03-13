# Handler

来源：https://mp.weixin.qq.com/s/7PAMm_FPrA0P3jf0tn3yyQ

## Handler 如何运行

### Handler角色分配



Handler中存在四种角色。



#### Handler



Handler用来向Looper发送消息，在Looper处理到对应的消息时，Handler在对消息进行具体的处理。上层关键API为handleMessage()，由子类自行实现处理逻辑。



#### Looper



Looper运行在目标线程里，不断从消息队列MessageQueue读取消息，分配给Handler处理。Looper起到连接的作用，将来自不同渠道的消息，聚集在目标线程里处理。也因此Looper需要确保线程唯一。



#### MessageQueue



存储消息对象Message，当Looper向MessageQueue获取消息，或Handler向其插入数据时，决定消息如何提取、如何存储。不仅如此，MessageQueue还维护与Native端的连接，也是解决Looper.loop() 阻塞问题的 Java 端的控制器。



#### Message



Message包含具体的消息数据，在成员变量target中保存了用来发送此消息的Handler引用。因此在消息获得这行时机时，能知道具体由哪一个Handler处理。此外静态成员变量sPool，则维护了消息缓存池以复用。



## 运行过程

首先，需要构建消息对象。获取消息对象从Handler.obtainMessage()系列方法可以获取Message，这一系列的函数提供了相应对应于Message对象关键成员变量对应的函数参数，而无论使用哪一个方法获取，最终通过Message.obtain()获取具体的Message对象。

```
  // 缓存池
  private static Message sPool;
  // 缓存池当前容量
  private static int sPoolSize = 0;
  // 下一节点
  Message next;

  public static Message obtain() {
    // 确保同步
    synchronized (sPoolSync) {
      if (sPool != null) {
        // 缓存池不为空
        Message m = sPool;
        // 缓存池指向下一个Message节点
        sPool = m.next;
        // 从缓存池拿到的Message对象与缓存断开连接
        m.next = null;
        m.flags = 0; // clear in-use flag
        // 缓存池大小减一
        sPoolSize--;
        return m;
      }
    }
    // 缓存吃没有可用对象，返回新的Message()
    return new Message();
  }
```

创建出消息之后，通过Handler将消息发送到消息队列，发送方法有很多，不一一陈列。发送发上有两种：

1. 将Message对象发送到
2. 发送Runnable，通过getPostMessage()将Runnable包装在Message里，表现为成员变量callback

```
private static Message getPostMessage(Runnable r) {
    // 获取Message
    Message m = Message.obtain();
    // 记住Runnale，等消息获得执行时回调
    m.callback = r;
    return m;
  }
```

不管哪种方式发送，最终消息队列MessageQueue只知接受到了消息对象Message。而将消息加入到消息队列，最终通过enqueueMessage()加入。

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    // Message.target 记住 Handler 以明确是由哪一个Handler来处理这个消息的
    msg.target = this;
    if (mAsynchronous) {
      msg.setAsynchronous(true);
    }
    // 消息入队
    return queue.enqueueMessage(msg, uptimeMillis);
  }
```

在将消息加入消息队列时，有时需要提供延迟信息delayTime，以期未来多久后执行，这个值存于 uptimeMillis。



之后，等待Looper轮询从消息队列中读取消息进行处理。见Looper.loop()。

```
public static void loop() {
     // 拿到Looper
    final Looper me = myLooper();
    if (me == null) {
      // 没调用prepare初始化Looper，报错
      throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 拿到消息队列
    final MessageQueue queue = me.mQueue;
    ......

​    for (;;) {
​      // 从消息队列取出下一个信息
​      Message msg = queue.next();
​      if (msg == null) {
​        // 消息为空，返回
​        return;
​      }
​      .......
​      try {
​        // 分发消息到Handler
​        msg.target.dispatchMessage(msg);
​        end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
​      }
​      // 消息回收，放入缓存池
​      msg.recycleUnchecked();
  }
}
```

Looper从MessageQueue里取出Message，Message.target则是具体的Hander，Handler.dispatchMessage()将触发具体分配逻辑。此后，将Message回收，放入缓存池。

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
      // 这个情况说明了次消息为Runnable，触发Runnable.run()
      handleCallback(msg);
    } else {
      if (mCallback != null) {
        // 指定了Handler的mCallback
        if (mCallback.handleMessage(msg)) {
          return;
        }
      }
      // 普通消息处理
      handleMessage(msg);
    }
  }
```

Handler分配消息分三种情况：



1. 可以通过Handler发送Runnable消息到消息队列，因此handleCallback()处理这种情况
2. 可以给Handler设置Callback，当分配消息给Handler时，Callback可以优先处理此消息，如果Callback.handleMessage()返回了true，不再执行Handler.handleMessage()
3. Handler.handleMessage()处理具体逻辑



回收则是通过Message.recycleUnchecked()。



```
void recycleUnchecked() {
    // 这里是将Message各种属性重置操作
    ......

​    synchronized (sPoolSync) {
​      if (sPoolSize < MAX_POOL_SIZE) {
​        // 缓存池还能装下，回收到缓存池

​        // 下面操作将此Message加入到缓存池头部
​        next = sPool;
​        sPool = this;
​        sPoolSize++;
​      }
​    }
  }
```

![img](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt5bLSgfvn73SgylhDOFxz94Pkfel9C77YR3xbDZo4VmZ2Y0TjY173Y51H232HljElan6VLpMVK2Fw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

1. Handler 从缓存池获取Message，发送到MessageQueue
2. Looper不断从MessageQueue读取消息，通过Message.target.dispatchMessage()触发Handler处理逻辑
3. 回收Message到缓存池



目前来看，可以算是了解了Handler的运行机制，但是对于解答开篇提出的问题，捉襟见肘，需要深入Handler。



## Java端与Native端建立连接

实际上，不仅仅是Java端存在Handler机制，在Native端同样存在Handler机制。他们通过MessageQueue建立了连接。



一般来说，Looper通过prepare()进行初始化。

```
private static void prepare(boolean quitAllowed) {
    // 保证Looper在线程唯一
    if (sThreadLocal.get() != null) {
      throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 将Looper放入ThreadLocal
    sThreadLocal.set(new Looper(quitAllowed));
  }
```

在实例化Looper时，需要确保Looper在线程里是唯一的。Handler知道自己的具体Looper对象，而Looper运行在具体的线程里并在此线程里处理消息。这也是为什么Looper能达到切换线程的目的。Looper线程唯一需要ThreadLocal来确保，ThreadLocal的原理，简单来说Thread里有类型为ThreadLocalMap的成员threadLocals，通过ThreadLocal能将相应对象放入threadLocals里通过K/V存储，如此能保证变量在线程范围内存储，其中Key为ThreadLocal< T > 。

```
private Looper(boolean quitAllowed) {
    // 初始化MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    // 记住当前线程
    mThread = Thread.currentThread();
  }



  MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    // 与Native建立连接
    mPtr = nativeInit();
  }
```

在MessageQueue创建时，通过native方法nativeInit()与Native端建立了连接，mPtr为long型变量，存储一个地址。方法实现文件位于frameworks/base/core/jni/android_os_MessageQueue.cpp。

```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
  NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
  if (!nativeMessageQueue) {
    jniThrowRuntimeException(env, "Unable to allocate native queue");
    return 0;
  }

  nativeMessageQueue->incStrong(env);

  // 返回给Java层的mPtr, NativeMessageQueue地址值
  return reinterpret_cast<jlong>(nativeMessageQueue);
}

NativeMessageQueue::NativeMessageQueue() :
    mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
  mLooper = Looper::getForThread();
  // 检查Looper 是否创建
  if (mLooper == NULL) {
    mLooper = new Looper(false);
    // 确保Looper唯一
    Looper::setForThread(mLooper);
  }
}
```

在Native端创建了NativeMessageQueue，同样也创建了Native端的Looper。在创建NativeMessageQueue后，将它的地址值返回给了Java层MessageQueue.mPtr。实际上，Native端Looper实例化时做了更多事情。Nativ端Looper文件位于system/core/libutils/Looper.cpp。



```
Looper::Looper(bool allowNonCallbacks) :
    mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
    mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
    mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
  // 添加到epoll的文件描述符，线程唤醒事件的fd
  mWakeEventFd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
  LOG_ALWAYS_FATAL_IF(mWakeEventFd < 0, "Could not make wake event fd: %s",
            strerror(errno));

  AutoMutex _l(mLock);
  rebuildEpollLocked();
}

void Looper::rebuildEpollLocked() {
  .....

  // Allocate the new epoll instance and register the wake pipe.
  // 创建epolle实例，并注册wake管道
  mEpollFd = epoll_create(EPOLL_SIZE_HINT);
  LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));

  struct epoll_event eventItem;
  // 清空，把未使用的数据区域进行置0操作
  memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
  // 监听可读事件
  eventItem.events = EPOLLIN;
  // 设置作为唤醒评判的fd
  eventItem.data.fd = mWakeEventFd;
  // 将唤醒事件（mWakeEventFd）添加到epoll实例，意为放置一个唤醒机制
  int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);
  LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
            strerror(errno));

  // 添加各种事件的fd到epoll实例，如键盘、传感器输入等
  for (size_t i = 0; i < mRequests.size(); i++) {
    const Request& request = mRequests.valueAt(i);
    struct epoll_event eventItem;
    request.initEventItem(&eventItem);

​    int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
​    if (epollResult < 0) {
​      ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
​         request.fd, strerror(errno));
​    }
  }
}
```

### 如何理解epoll机制？

文件、socket、pipe(管道)等可以进行I/O操作的对象可以视为流。既然是I/O操作，则有read端读入数据，有write端写入数据。但是两端并不知道对方进行操作的时机。而epoll则能观察到哪个流发生了了I/O事件，并进行通知。



这个过程，就好比你在等快递，但你不知道快递什么时候来，那这时你可以去睡觉，因为你知道快递送来时一定会打个电话叫醒你，让你拿快递，接着做你想的事情。

epoll有效地降低了CPU的使用，在线程空间时令其休眠，等有事件到来时再讲它唤醒。



在知道了epoll之后，再来看上面的代码，就可以理解了。在Native端创建Looper时，会创建用来唤醒线程的fd —— mWakeEventFd，创建epoll实例并注册管道，清空管道数据，监听可读事件。当有数据写入mWakeEventFd描述的文件时，epoll能监听到此事件，并通知将目标线程唤醒。



在Java端MessageQueue.mPrt存储了Native端NativeMassageQueue的地址，可以利用NativeMassageQueue享用此机制。



## 发送数据的具体过程



