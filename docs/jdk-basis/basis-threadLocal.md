ThreadLocal实现原理详解

介绍
​ThreadLocal大家应该不陌生，经常在一些同步优化中会使用到它。很多地方叫线程本地变量，ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。也就是对于同一个ThreadLocal，每个线程通过get、set、remove接口操作只会影响自身线程的数据，不会干扰其他线程中的数据。
ThreadLocal是怎么实现的呢？
ThreadLocal又有哪些误区呢？
源码分析
从ThreadLocal的set方法说起，set是用来设置想要在线程本地的数据，可以看到先拿到当前线程，然后获取当前线程的ThreadLocalMap，如果map不存在先创建map，然后设置本地变量值。
![threal-set](/images/threal-set.png)
那ThreadLocalMap又是什么？跟线程有什么关系？可以看到ThreadLocalMap其实是线程自身的一个成员属性threadLocals的类型。也就是线程本地数据都存在这个threadLocals应用的ThreadLocalMap中。
![threal-map](/images/thread-map.png)
我们再接着看看ThreadLocalMap，跟想象中的Map有点不一样，它其实内部是有个Entry数组，将数据包装成静态内部类Entry对象，存储在这个table数组中，数组的下标是threadLocal的threadLocalHashCode&(INITIAL_CAPACITY-1)，因为数组的大小是2的n次方，那其实这个值就是threadLocalHashCode%table.length，用&而不用%，其实是提升效率。只要数组的大小不变，这个索引下标是不变的，这也方便去set和get数据。
![ThreadLocalMap](/images/ThreadLocalMap.png)
我们再看看Entry的定义，Entry继承自WeakReference（这么做目的是什么，后面会讲到），构造方法有两个参数一个是threadLocal对象，一个是线程本地的变量。
![Entry](/images/Entry.png)
看到这里大家应该就明白了，每个线程自身都维护着一个ThreadLocalMap，用来存储线程本地的数据，可以简单理解成ThreadLocalMap的key是ThreadLocal变量，value是线程本地的数据。就这样很简单的实现了线程本地数据存储和交互访问。
误区
上文也提到了，Entry继承自WeakReference，大家都知道WeakReference（弱引用）的特性，只要从根集出发的引用中没有有效引用指向该对象，则该对象就可以被回收，这里的有效引用并不包含WeakReference，所以弱引用不影响对象被GC。
这里被WeakReference引用的对象是哪个呢？可以看Entry的构造方法，很容易看出指的是ThreadLocal自身，也就是说ThreadLocal自身的回收不受ThreadLocalMap的这个弱引用的影响，让用户减轻GC的烦恼。
但是不用做些什么吗？这么简单？
其实不然，ThreadLocalMap还做了其他的工作，试想一下，ThreadLocal对象如果外界没有有效引用，是能够被GC，但是Entry呢？Entry也能自动被GC吗，当然不行，Entry还被ThreadLocalMap的table数组强引用着呢。
所以ThreadLocalMap该做点什么？
我看看ThreadLocalMap的expungeStaleEntry这个方法，这个方法在ThreadLocalMap get、set、remove、rehash等方法都会调用到，看下面标红的两处代码，第一处是将remove的entry赋空，第二次处是找到已经被GC的ThreadLocal，然后会清理掉table数组对entry的引用。这样entry在后续的GC中就会被回收。

![ThreadLocalMap](/images/ThreadLocalMap.png)
是不是这样就万事大吉了呢，不用担心GC问题了呢？
没那么简单，还是有点坑：
这里的坑与WeakHashMap垃圾回收原理中所说的类似，如果数据初始化好之后，一直不调用get、set等方法，这样Entry就一直不能回收，导致内存泄漏。所以一旦数据不使用最好主动remove。
有些同学问，如果线程资源回收了，还会泄漏吗？
当然不会，线程回收后，存在线程相关的ThreadLocalMap也会被回收，线程相关的本地数据自然也会回收。但是不要忘记ThreadLocal的使用场景，就是用来存储线程本地变量，大部分场景中，线程都是一直存活或者长时间存活。
场景
一般在连接池优化上会使用到ThreadLocal，在多线程获取连接池时，会有同步操作，影响性能。如果使用ThreadLocal，每个线程使用自己的独立连接，性能会有一定的提升。
总结
希望这遍文章能够提升大家对ThreadLocal的理解，避免踩坑，使用起来也更加了然于胸。



















