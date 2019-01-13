# Otto
## 1.Otto是什么
Otto是Event Bus，模块间通过它来通信可以高效、解耦。但目前已被官方**Deprecated**，目的是为了支持**RxJava**与**RxAndroid**。但这不代表它不能被使用，或被学习。
[Github地址](https://github.com/square/otto)
集成方式为：`implementation 'com.squareup:otto:1.3.8'` 
## 2.使用注意事项
1. 需要新建一个Bus单例类，用来注册对象、发送消息等功能；
2. 使用了register方法的类，一定需要使用unregister方法反注册自己；

## 3.源码解析
1. 类Produce，此类可以将函数标识为生产者，用来主动发送消息。源码如下：
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Produce {
}
```
RUNTIME：该注解会被保留在class文件中，同时运行期间也会被识别，所以可以使用反射获取注解的信息；
METHOD：表示Target的注解为方法。
比如：
```
    @Produce
    public BusData sendMessage() {
        return new BusData("Hello World");
    }
```
::此功能需要先将实现该函数的类register进Bus单例类中。::

2. 类Subscribe，此类可以将函数标识为订阅者，用来接收消息。源码如下： ```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Subscribe {
} ```
使用如下：
```
	  @Subscribe
    public void receiveMessage(String msg) {
    }
```

3. 类AnnotatedHandlerFinder，此类用来查找某类中所有用Produce或Subscribe注释的方法，并保存在ConcurrentMap类型的结构中。 此类中核心方法是loadAnnotatedMethods，来加载参数类listenerClass中所有Annotation为Produce或Subscribe的方法。
```
private static void loadAnnotatedMethods(Class<?> listenerClass,
      Map<Class<?>, Method> producerMethods, Map<Class<?>, Set<Method>> subscriberMethods) {
    for (Method method : listenerClass.getDeclaredMethods()) {// 1 获取方法
      // The compiler sometimes creates synthetic bridge methods as part of the
      // type erasure process. As of JDK8 these methods now include the same
      // annotations as the original declarations. They should be ignored for
      // subscribe/produce.
      if (method.isBridge()) { // 2 判断是否为桥接方法
        continue;
      }
      if (method.isAnnotationPresent(Subscribe.class)) { // 3 是否注释为Subscribe方法
        Class<?>[] parameterTypes = method.getParameterTypes(); // 获取参数列表
        if (parameterTypes.length != 1) { // 判断参数个数
          throw new IllegalArgumentException("Method " + method + " has @Subscribe annotation but requires "
              + parameterTypes.length + " arguments.  Methods must require a single argument.");
        }

        Class<?> eventType = parameterTypes[0];
        if (eventType.isInterface()) { // 判断参数类型
          throw new IllegalArgumentException("Method " + method + " has @Subscribe annotation on " + eventType
              + " which is an interface.  Subscription must be on a concrete class type.");
        }

        if ((method.getModifiers() & Modifier.PUBLIC) == 0) { // 判断函数访问性声明
          throw new IllegalArgumentException("Method " + method + " has @Subscribe annotation on " + eventType
              + " but is not 'public'.");
        }

        Set<Method> methods = subscriberMethods.get(eventType);
        if (methods == null) {
          methods = new HashSet<Method>();
          subscriberMethods.put(eventType, methods);
        }
        methods.add(method);
      } else if (method.isAnnotationPresent(Produce.class)) { // 4 是否注释为Produce方法
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (parameterTypes.length != 0) {
          throw new IllegalArgumentException("Method " + method + "has @Produce annotation but requires "
              + parameterTypes.length + " arguments.  Methods must require zero arguments.");
        }
        if (method.getReturnType() == Void.class) {
          throw new IllegalArgumentException("Method " + method
              + " has a return type of void.  Must declare a non-void type.");
        }

        Class<?> eventType = method.getReturnType();
        if (eventType.isInterface()) {
          throw new IllegalArgumentException("Method " + method + " has @Produce annotation on " + eventType
              + " which is an interface.  Producers must return a concrete class type.");
        }
        if (eventType.equals(Void.TYPE)) {
          throw new IllegalArgumentException("Method " + method + " has @Produce annotation but has no return type.");
        }

        if ((method.getModifiers() & Modifier.PUBLIC) == 0) {
          throw new IllegalArgumentException("Method " + method + " has @Produce annotation on " + eventType
              + " but is not 'public'.");
        }

        if (producerMethods.containsKey(eventType)) {
          throw new IllegalArgumentException("Producer for type " + eventType + " has already been registered.");
        }
        producerMethods.put(eventType, method);
      }
    }

    PRODUCERS_CACHE.put(listenerClass, producerMethods);
    SUBSCRIBERS_CACHE.put(listenerClass, subscriberMethods);
  }
```
在1处，通过getDeclaredMethods()方法获取类listenerClass中所有声明的方法，包括public、protected、private，但不包括继承的方法；
在2处，过滤掉[桥接方法](https://blog.csdn.net/qq_32647893/article/details/81071336)；
在3处，如果方法被Annotation为Subscribe类型，则进入if语句逻辑，通过getParmeterTypes()方法获取参数，对参数的个数（只能是1个）、参数类型（只能是concrete class type）、方法访问类型（只能是public）进行判断，都通过，则将该方法存入`Map<Class<?>, Set<Method>>`类型的结构中；
在4处：如果方法被Annotation为Produce类型，则进入相应逻辑处理，对该方法的参数（只能是0个）、函数返回类型（不能为Void、只能为concrete class type）、方法访问类型（只能是public）、是否被注册过进行判断，都通过，则将该方法存入`Map<Class<?>, Method>`中。
由各自存入的结构中看，一个类中，Produce方法，只能有一个，而Subscribe方法可以有多个。 
5. 类Bus，一般继承此类，实现单例类。用来register、unregister、post等操作。

**方法register**主要内容分如下几部分：
```
Map<Class<?>, EventProducer> foundProducers = handlerFinder.findAllProducers(object); // 1 查找类中所有的Produce注释的方法
    for (Class<?> type : foundProducers.keySet()) {

      final EventProducer producer = foundProducers.get(type);
      EventProducer previousProducer = producersByType.putIfAbsent(type, producer);
      //checking if the previous producer existed
      if (previousProducer != null) { // 2 若存在，则说明已经注册过，抛出异常
        throw new IllegalArgumentException("Producer method for type " + type
          + " found on type " + producer.target.getClass()
          + ", but already registered by type " + previousProducer.target.getClass() + ".");
      }
      Set<EventHandler> handlers = handlersByType.get(type); // 3 获取type类型对应的订阅者集合
      if (handlers != null && !handlers.isEmpty()) {
        for (EventHandler handler : handlers) {
          dispatchProducerResultToHandler(handler, producer); // 4 执行Produce方法，将其返回值作为handler的参数来执行
        }
      }
    }
```
在1处，查找类中所有的Produce方法；
在2处，判断此方法是否已经注册过，若重复注册，则抛出异常；
在3处，获取订阅了此type的Subscribe方法集合；
在4处，对Subscribe集合中每一个handler遍历，先执行Produce方法以获取其返回值，然后将该返回值作为参数执行Subscribe方法，以此实现模块间Produ对Subscribe的主动通信。

处理完Produce方法，接着处理类中注释的Subscribe方法，这与处理Produce方法类似（此处不粘贴代码），首先查找所有的Subscribe方法；然后查找是否有与Subscribe方法对应的Produce方法，若有的话，则触发对应的订阅者处理此事件。

**方法post**源码如下：
```
public void post(Object event) {
    ...
    Set<Class<?>> dispatchTypes = flattenHierarchy(event.getClass()); // 1 生成订阅对象及其父类的集合

    boolean dispatched = false;
    for (Class<?> eventType : dispatchTypes) {
      Set<EventHandler> wrappers = getHandlersForEventType(eventType);

      if (wrappers != null && !wrappers.isEmpty()) {
        dispatched = true;
        for (EventHandler wrapper : wrappers) { 
          enqueueEvent(event, wrapper);// 2 将事件与订阅者放入队列中
        }
      }
    }

    if (!dispatched && !(event instanceof DeadEvent)) {
      post(new DeadEvent(this, event)); // 3 如果该事件无订阅者，则生成DeadEvent再post一次
    }

    dispatchQueuedEvents(); // 4 分发事件队列中的事件
  }
```
在1处，将事件及其父类放入集合中；
在2处，如果存在对订阅该事件的订阅者，则将事件与其对应的订阅者一起放入事件队列中；
在3处，如果存在某事件无对应的订阅者，则将其生成DeadEvent事件后再post一次；
在4处，对事件队列进行处理。

**方法unregister**
此方法中首先将类中对Produce注释的函数进行移除，然后对Subscribe注释的函数进行移除。

**其他**
```
ThreadLocal<ConcurrentLinkedQueue<EventWithHandler>> eventsToDispatch;
```
[ThreadLocal](https://www.jianshu.com/p/98b68c97df9b)，为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。多线程并发时就安全了。
一个ThreadLocal只能保存一个变量副本，若要保存多个变量，则需要创建多个ThreadLocal；
ThreadLocal存在内存泄露的风险；
适用于无状态，副本独立后，对业务逻辑无影响的高并发场景。


