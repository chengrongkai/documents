# Java中有哪些锁？ 

> 在Java中存在很多的锁，比如ReentrantLock、Synchronized，这些锁根据其特性和使用场景，可以划分为很多的类型，比如乐观锁和悲观锁、可重入锁和不可重入锁，本文将结合源码具体分析下这些锁的设计思想以及其应用场景

## 为什么要有锁

为什么要有锁，结合我们现实生活就不难理解，锁存在的意义就是为了保护某样东西，对于Java编程中，具体来讲就是在多线程环境，对于同步资源的一种保护，保证同步资源的访问不会因为多线程的访问出现不可预期的错误。

举个例子，现在张三有一张银行卡，可以通过柜员机转账和网上银行转账，假设现在卡内余额为200元，张三同时在柜员机和网上银行操作转账100元给李四（可能是提交了多次请求），如果银行没对账户余额的访问进行锁住，势必会出现两笔转账都成功的情况，这种情况下违背了用户操作的初衷，所以在这种场景下就需要用锁机制来保护账户余额。

当然，在实际的场景中，肯定有多种手段来实现，本文只关注Java编程中在多线程环境下，保证线程安全的方式。

## 锁的分类
<img src="https://telegraph-image-aed.pages.dev/file/f578fc00481356c273281.png" style="margin: auto" />
![](https://telegraph-image-aed.pages.dev/file/f578fc00481356c273281.png)

## 乐观锁和悲观锁

### 乐观锁

乐观锁是指在访问同步资源时，先不加锁，如果出现获取同步资源失败的话，进行重试，也就是乐观的认为访问时能够成功获取到同步资源，这种锁通常用在竞争频率不高的场景下，乐观锁会导致重试，从而影响系统性能，如果某些资源竞争频率很高，不建议采用乐观锁。

Java中乐观锁的常见实现方式是CAS（compare and swap）即在场景操作资源时，先比较资源的当前值是否符合预期，如果符合预期则执行操作，否则返回执行失败，CAS在Java中由Usafe类实现，常见的方法如下

![](https://telegraph-image-aed.pages.dev/file/d457a9370d4220ae20f8f.png)

在Java中，有很多的锁都是基于CAS实现，比如AQS的锁（底层）、ReentrantLock、CountDownLatch、Semaphore等，还有一些原子类比如AtomicInteger，使用CAS也不能完全保证线程安全，也要注意一些问题，比如典型ABA问题，解决这类问题可以通过增加版本号之类的方式处理，在尝试操作资源时，不但要比较预期值，也要比较预期版本。

### 悲观锁

悲观锁是指在访问同步资源时，认为会有其他线程与其竞争该资源，因此需要先对其进行加锁才能操作，Java中的ReentrantLock、Synchronized等都属于悲观锁，在访问同步资源时，都需要先执行上锁操作

## 自旋锁和自适应自旋锁

### 自旋锁

自旋锁是指在访问同步资源失败的情况下，线程采用自旋的方式来等待获取锁，典型的实现方式如在AbstractQueuedSynchronizer（ReentrantLock的抽象父类实现）中尝试加锁的操作，可以看到当tryAcquire方法执行失败时，会通过for循环自旋来等待获取锁，而不是让线程进入阻塞状态。

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
### 自适应自旋锁

自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

## 无锁VS偏向锁VS轻量级锁VS重量级锁

这四种锁是指锁的状态，专门针对Synchronized的，在jdk1.6以前Synchronized的默认实现都是重量级锁，即只要多线程访问同步资源时，都会阻塞进行等待，直到线程被唤醒，在jdk1.6之后，Synchronized的锁机制进行了优化，针对资源访问的激烈程度，对锁的操作细节进行了划分。

按照竞争的激烈程度来升序排序是：无锁》偏向锁》轻量级锁》重量级锁，按照性能高低来升序排序 是：重量级锁》轻量级锁》偏向锁》无锁。

在使用Synchronized对同步资源进行控制时，会按照以下流程进行锁升级的操作

![](https://telegraph-image-aed.pages.dev/file/fbec3ea10ba62661515d6.png)

其实现是通过对访问资源的对象头进行操作，相当于记录了一个状态

### Java对象头
synchronized是悲观锁，在操作同步资源之前需要给同步资源先加锁，这把锁就是存在Java对象头里的，而Java对象头又是什么呢？

我们以Hotspot虚拟机为例，Hotspot的对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。

Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。

Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

Monitor
Monitor可以理解为一个同步工具或一种同步机制，通常被描述为一个对象。每一个Java对象就有一把看不见的锁，称为内部锁或者Monitor锁。

Monitor是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联，同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。

现在话题回到synchronized，synchronized通过Monitor来实现线程同步，Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的线程同步。

如同我们在自旋锁中提到的“阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态转换需要耗费处理器时间。如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长”。这种方式就是synchronized最初实现同步的方式，这就是JDK 6之前synchronized效率低的原因。这种依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”，JDK 6中为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。

所以目前锁一共有4种状态，级别从低到高依次是：无锁、偏向锁、轻量级锁和重量级锁。锁状态只能升级不能降级。

通过上面的介绍，我们对synchronized的加锁机制以及相关知识有了一个了解，那么下面我们给出四种锁状态对应的的Mark Word内容，然后再分别讲解四种锁状态的思路以及特点：

![](https://telegraph-image-aed.pages.dev/file/1a7a8ca042c6c99cc6d1a.png)

## 公平锁和非公平锁

### 公平锁
在访问同步资源时，出现需要等待情况，当锁资源释放时，需要按照申请访问资源的顺序排队获取锁，比如在ReentrantLock中的公平锁FairSync实现中，会在尝试获取锁资源前，先检查当前线程是否位于等待队列的第一位，如果是第一位才允许获取锁。hasQueuedPredecessors就是检测当前线程是否是排在队列的第一位。
```
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
### 非公平锁
在访问同步资源时，出现需要等待情况，当锁资源释放时，无需严格按照申请访问资源的顺序排队，也就是说允许插队，如果一个线程在尝试获取锁的时候，恰好能直接获取到锁（比较走运），那就能直接访问同步资源，而无需考虑等待队列中是否还有线程等待，也就是说对于等待队列中线程以及新加入的线程拥有同等的概率获取到锁，这也是非公平的体现（先来的不一定先拿到锁）。

在Java中，在ReentrantLock中的公平锁NonfairSync实现中, 就是按照此思想设计的，源码如下,可以看到他与FairSync的实现的区别就是少了hasQueuedPredecessors方法的判断。
```
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

## 可重入锁和不可重入锁

可重入锁是指在线程获取到锁的情况下，在线程操作的其他节点再次申请获取锁，也能获取成功的情况，Java中的锁都是可重入锁，也就是说对于同一个线程的操作，如果分多次获取同一把锁，都是可以获取成功的。

## 共享锁和独占锁

### 共享锁

独占锁和共享锁同样是一种概念。我们先介绍一下具体的概念，然后通过ReentrantLock和ReentrantReadWriteLock的源码来介绍独享锁和共享锁。

独占锁也叫排他锁，是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。JDK中的synchronized和JUC中Lock的实现类就是互斥锁。

共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

独占锁与共享锁也是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享

下面是ReentrantReadWriteLock中对于独占锁和共享锁的实现，分别对应了其写锁和读锁的实现，现在分别看下写锁和读锁在获取锁的方法实现

写锁

```
final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
写锁在加锁时会按照如下流程操作
1. 获取当前state的值
2. 判断当前独占锁的数量，判断当前独占锁的所有者是否为当前线程
3. 判断独占锁是否超过最大锁定数量
4. 尝试通过CAS设置state的值
5. 设置当前线程为独占锁的所有者

读锁
```
final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```
读锁在加锁时会按照如下流程操作
1. 判断是否有非当前线程的独占锁，有的话则加锁失败
2. 判断共享锁的持有数是否超过限制
3. 尝试通过CAS操作设置state的值加锁成功
4. 如果是一个加锁的，则设置当前线程为第一个reader
5. 如果第一个reader是自己，则增加统计
6. 否则通过HoldCounter计数

上述的加锁过程中有几个点需要关注
1. 获取独占锁或共享锁时，如果已有独占锁且非本线程，直接加锁失败
2. 获取共享锁时，如果有其他共享锁存在，则state计数增加，允许共同访问。
3. 独占锁和共享锁都是通过CAS操作来控制加锁的，只不过独占锁只允许0和1，而共享锁允许0-n,每次加锁都是计数增加。

## 总结

本文系统地介绍了Java中锁的种类及其特性，涵盖了乐观锁和悲观锁、自旋锁和自适应自旋锁、无锁、偏向锁、轻量级锁、重量级锁、公平锁和非公平锁、可重入锁和不可重入锁、共享锁和独占锁等内容。文章通过对各种锁的定义、实现原理以及源码分析，深入剖析了每种锁的适用场景和使用方法。同时，还探讨了Java对象头的结构以及锁的状态转换过程。通过本文的阐述，读者可以全面了解Java中锁的机制，从而更好地应用于实际的多线程编程中。

Java中的锁种类繁多，适合在不同的场景下使用，除了上述描述的这些锁之外，Java中还有多种机制保证线程安全，比如voliate关键字、Atom原子类，以及线程安全的集合类等等，这些都需要我们在日常变成中在具体的场景下逐步掌握。

欢迎关注我的公众号“**毕知必会**”，原创技术文章第一时间推送。

<center>
    <img src="https://telegraph-image-aed.pages.dev/file/74ebf1fb389a9ab62228a.jpg" style="width: 100px;">
</center>
