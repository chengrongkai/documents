## 什么是AQS
AQS（AbstractQueuedSynchronizer）是Java中用于构建同步器的抽象基类。它提供了一种实现同步器的框架，使得开发者可以基于它构建各种类型的同步器，比如锁、信号量、倒计数器等。AQS主要通过一个FIFO（先进先出）的等待队列来管理线程的排队和等待，以实现对共享资源的控制和同步。

## 为什么要用AQS

大家可以思考一个问题，在Java中，当某个方法或者对象需要被多个线程同时操作时，应当如何保证线程安全，使我们的操作符合预期，也就是和单线程下操作的结果一致。例如下面这个例子,在这种操作下，可能出现预期结果不等于2000的情况

```
public class UnsafeCounter {
    private int count = 0;

    public void increment() {
        count++; // 非原子性操作
    }

    public void decrement() {
        count--; // 非原子性操作
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) {
        UnsafeCounter counter = new UnsafeCounter();

        // 创建并启动多个线程对计数器进行增加操作
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        thread1.start();
        thread2.start();

        // 等待两个线程执行完成
        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 输出最终计数器的值
        System.out.println("Final count: " + counter.getCount());
    }
}

```
那应该怎么做呢，答案是加锁，例如使用synchronized对方法加锁，当然下面这段代码也不能完全保证线程安全，还需要辅以voliate之类的方式保证可见性。
```
public class UnsafeCounter {
    private int count = 0;

    public synchronized void increment() {
        count++; // 非原子性操作
    }

    public synchronized void decrement() {
        count--; // 非原子性操作
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) {
        UnsafeCounter counter = new UnsafeCounter();

        // 创建并启动多个线程对计数器进行增加操作
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter.increment();
            }
        });

        thread1.start();
        thread2.start();

        // 等待两个线程执行完成
        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


        // 输出最终计数器的值
        System.out.println("Final count: " + counter.getCount());
    }
}

```
到这里是否就结束了呢，答案显然是否定的，synchronized是重量级锁，会以来系统底层唤醒阻塞的线程，严重影响系统的执行效率，虽然在jdk1.6之后进行了优化，但还是需要其他的加锁方式替代。这时AQS的作用就得以体现了

## ASQ的设计思想

AQS是基于乐观锁的思想设计，对于需要访问同步资源的线程，尝试获取锁失败后，会将其加入到一个等待队列中，通过自旋的方式尝试获取锁，因为线程没有发生上下文的切换，所以在性能上是要优于synchronized的，其中两个点需要特别注意下

1. 通过AQS尝试加锁时，加锁的对象是啥？是我们要访问的同步资源么，NO，AQS加锁的对象其实是内部维护的一个state变量，这一点与synchronized不一样，synchronized加锁的对象就是访问的同步资源
2. 通过AQS加锁时，实际上是通过CAS操作尝试改变state的值，AQS提供了独占和共享两种模式，独占模式下state只存在0和1两个值，共享模式下state会存在0-n个值，具体实现取决于子类
3. AQS的代码采用了模板方法的设计模式，只定义了加锁的流程，对于CAS操作等关键步骤予以实现，对于独占锁、共享锁等实际操作加锁的行为交由子类来实现

如下图所示，AQS加锁的本质

![](https://telegraph-image-aed.pages.dev/file/b23076315b91bc70fd1e2.jpg)

上图只是简化的操作，AQS内部维护的实际上是一个双端队列，既有前驱也有后驱，在每一个Node节点里，都维护等待状态、线程信息等
```
 static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;

        static final int PROPAGATE = -3;
        volatile int waitStatus;

        volatile Node prev;

        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;

        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

下面我们看下AQS具体是怎样实现加锁逻辑的
1. 找到当前节点的前驱，如果前驱是头部节点，
2. 并且尝试加锁成功（独占锁），则将当前节点设置为同步节点
3. 如果加锁失败后，需要检测线程中断状态，则检查
4. 如果自旋加锁失败了，则取消获取锁（其实就是取消节点排队）
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
上面是独占锁的加锁逻辑，共享锁的加锁也大同小异,需要关注的是共享锁里调用的方法是tryAcquireShared，而独占锁里调用的方法是tryAcquire
```
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
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
打开AQS的源码，我们可以看到tryAcquireShared和tryAcquire这两个方法都是没有具体实现的，这里其实就是模板方法设计模式的体现

![](https://telegraph-image-aed.pages.dev/file/baaf6fa772e7e001dd36a.jpg)

## 模板方法设计模式
模板方法模式是一种行为设计模式，它定义了一个算法的骨架，将算法中的一些步骤延迟到子类中实现。模板方法使得子类可以在不改变算法结构的情况下重新定义算法中的某些步骤。
模板方法模式通常由以下几个部分组成：

抽象类（Abstract Class）：抽象类定义了一个模板方法，该方法中包含了算法的骨架，其中的一些步骤是具体实现的，而另一些步骤则留给子类来实现。抽象类中可能还包含一些普通方法，它们可以被模板方法或子类直接调用。

具体子类（Concrete Class）：具体子类实现了抽象类中定义的抽象方法，从而提供了算法中特定步骤的具体实现。

优点
通过模板方法模式，可以将算法的骨架和具体实现分离，使得算法的结构更清晰，易于维护和扩展。
子类可以灵活地重新定义算法中的特定步骤，而无需改变算法的结构。

### 示例代码
```
// 抽象类定义制作饮料的模板方法
abstract class BeverageMaker {
    // 模板方法，定义了制作饮料的算法骨架
    public final void makeBeverage() {
        boilWater();
        brew();
        pourInCup();
        addCondiments();
    }

    // 抽象方法，子类必须实现的步骤
    protected abstract void brew();
    protected abstract void addCondiments();

    // 公共方法，被模板方法直接调用
    private void boilWater() {
        System.out.println("Boiling water");
    }

    private void pourInCup() {
        System.out.println("Pouring into cup");
    }
}

// 具体子类实现制作咖啡的具体步骤
class CoffeeMaker extends BeverageMaker {
    protected void brew() {
        System.out.println("Brewing coffee");
    }

    protected void addCondiments() {
        System.out.println("Adding sugar and milk");
    }
}

// 具体子类实现制作茶的具体步骤
class TeaMaker extends BeverageMaker {
    protected void brew() {
        System.out.println("Steeping tea");
    }

    protected void addCondiments() {
        System.out.println("Adding lemon");
    }
}

// 客户端代码
public class TemplateMethodPatternDemo {
    public static void main(String[] args) {
        System.out.println("Making coffee:");
        BeverageMaker coffeeMaker = new CoffeeMaker();
        coffeeMaker.makeBeverage();

        System.out.println("\nMaking tea:");
        BeverageMaker teaMaker = new TeaMaker();
        teaMaker.makeBeverage();
    }
}

```

### AQS的典型实现

AQS的典型实现有ReentrantLock、Semaphore等，比如在ReentrantLock中就实现了tryAcquire方法，提供了独占锁的实现逻辑，而在Semaphore中就实现了tryAcquireShared方法，提供了共享锁的实现逻辑

独占锁
```
 protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }

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

共享锁

```
protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

通过对源码分析，可以看到对于独占锁的实现时通过CAS直接操作state变量，state变量只能存在0和1两个值，如果CAS操作失败则代表加锁失败，而对于共享锁的实现，则是限定了一个state的available范围，这个范围由具体子类控制，允许多个线程操作CAS多次，只要state的值不等于0，就表示可以进行加锁
，对比图例如下
![](https://telegraph-image-aed.pages.dev/file/578e89d033740a240c0bb.png)
![](https://telegraph-image-aed.pages.dev/file/3fadc93cc4f7912b51ec0.png)

共享锁和独占锁在实际场景下都很常见，AQS巧妙的通过state来控制允许访问资源的线程数量实现了。

## 应用场景

下面分别是用ReentrantReadWriteLock应用的AQS的示例，ReentrantReadWriteLock中的读锁就是共享锁的实现，而写锁则是独占锁的是西安
```
import java.util.concurrent.locks.ReentrantReadWriteLock;

class SharedResource {
    private String data = ""; // 共享资源

    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    // 读操作
    public String readData() {
        lock.readLock().lock(); // 获取读锁
        try {
            return data;
        } finally {
            lock.readLock().unlock(); // 释放读锁
        }
    }

    // 写操作
    public void writeData(String newData) {
        lock.writeLock().lock(); // 获取写锁
        try {
            data = newData;
        } finally {
            lock.writeLock().unlock(); // 释放写锁
        }
    }
}

public class ReentrantReadWriteLockExample {
    public static void main(String[] args) {
        SharedResource sharedResource = new SharedResource();

        // 多个线程读取共享资源
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                System.out.println("Read data: " + sharedResource.readData());
            }).start();
        }

        // 写入共享资源
        new Thread(() -> {
            sharedResource.writeData("New data");
            System.out.println("Write data");
        }).start();
    }
}

```

## 总结

AQS（AbstractQueuedSynchronizer）是Java中用于构建同步器的抽象基类。它提供了一种实现同步器的框架，使得开发者可以基于它构建各种类型的同步器，比如锁、信号量、倒计数器等。AQS主要通过一个FIFO（先进先出）的等待队列来管理线程的排队和等待，以实现对共享资源的控制和同步。

为了确保多线程环境下对共享资源的操作是线程安全的，需要使用同步机制。传统的同步机制如`synchronized`关键字存在性能问题，因此需要更高效的替代方案。AQS就是为了解决这一问题而设计的。

AQS的设计思想是基于乐观锁的，它允许线程在没有获得锁时自旋等待，避免了线程切换带来的开销。AQS通过模板方法设计模式实现了同步器的框架，定义了加锁的流程，并将关键步骤留给子类来实现，例如独占锁和共享锁的获取逻辑。

在实际应用中，AQS被广泛应用于Java的并发框架中，比如`ReentrantLock`、`Semaphore`等。这些框架利用AQS提供的同步机制来实现线程安全的并发控制，提高了程序的性能和可维护性。

欢迎关注我的公众号“**毕知必会**”，原创技术文章第一时间推送。

<center>
    <img src="https://telegraph-image-aed.pages.dev/file/74ebf1fb389a9ab62228a.jpg" style="width: 100px;">
</center>
