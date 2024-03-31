## 什么是重排序

大家可以先看下下面这段代码，思考以下输出的结果
```
    private static int x = 0, y = 0;
    private static int a = 0, b =0;

    public static void main(String[] args) throws InterruptedException {
        x = 0; y = 0;
        a = 0; b = 0;
        Thread one = new Thread(new Runnable() {
            public void run() {
                //由于线程one先启动，下面这句话让它等一等线程two. 读着可根据自己电脑的实际性能适当调整等待时间.
                shortWait(100000);
                a = 1;
                x = b;
            }
        });

        Thread other = new Thread(new Runnable() {
            public void run() {
                b = 1;
                y = a;
            }
        });
        one.start();other.start();
        one.join();other.join();
        String result = " (" + x + "," + y + "）";
        System.out.println(result);
    }

    public static void shortWait(long interval){
        long start = System.nanoTime();
        long end;
        do{
            end = System.nanoTime();
        }while(start + interval >= end);
    }
```
大致解释下这段代码，
1. 现在有x、y、a、b四个int类型的变量，初始值都为0
2. 现在有两个线程，线程A执行的逻辑是将变量【a】赋值为1,将变量【x】赋值为b,线程B执行的逻辑是将变量【b】赋值为1，将变量【y】赋值为a。
3. 启动这两个线程，打印最终x和y的值

那么，最终会输出什么内容呢是（0，1）、（1，0）、（1，1）还是（0，0）呢，答案是都有可能，对于上面这段代码中，我们不难看出影响输出结果的最关键因素就是这些指令的执行顺序，
首先，我们对于Java中多线程执行过程得有个基本的了解，多线程的执行调度是由操作系统决定的，同一时刻，单个CPU只能执行一个线程，其他的线程只能处于等待状态，也就是我们常说的并发任务，其实是根据操作系统的时间片分配进行上下文切换调度执行的，
也就是说对于上述代码而言，A、B两个线程的执行过程可能是交替执行的，也可能是A线程执行完了再执行B的，也可能是B线程执行完了再执行A的。但执行的过程会满足as-if-serial语义

## as-if-serial语义
As-if-serial语义的意思是，所有的动作(Action)都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。Java编译器、运行时和处理器都会保证单线程下的as-if-serial语义。 比如，为了保证这一语义，重排序不会发生在有数据依赖的操作之中。

int a = 1;
int b = 2;
int c = a + b;
将上面的代码编译成Java字节码或生成机器指令，可视为展开成了以下几步动作（实际可能会省略或添加某些步骤）。

对a赋值1
对b赋值2
取a的值
取b的值
将取到两个值相加后存入c
在上面5个动作中，动作1可能会和动作2、4重排序，动作2可能会和动作1、3重排序，动作3可能会和动作2、4重排序，动作4可能会和1、3重排序。但动作1和动作3、5不能重排序。动作2和动作4、5不能重排序。因为它们之间存在数据依赖关系，一旦重排，as-if-serial语义便无法保证。

为保证as-if-serial语义，Java异常处理机制也会为重排序做一些特殊处理。例如在下面的代码中，y = 0 / 0可能会被重排序在x = 2之前执行，为了保证最终不致于输出x = 1的错误结果，JIT在重排序时会在catch语句中插入错误代偿代码，将x赋值为2，将程序恢复到发生异常时应有的状态。这种做法的确将异常捕捉的逻辑变得复杂了，但是JIT的优化的原则是，尽力优化正常运行下的代码逻辑，哪怕以catch块逻辑变得复杂为代价，毕竟，进入catch块内是一种“异常”情况的表现

对于上述代码，执行的顺序可能是
1. a = 1;x = b;b = 1;y = a;  输出的结果就是 (0,1)
2. b = 1;y = a;a = 1;x = b;  输出的结果就是 (1,0)
3. a = 1;b = 1;x = b;y = a;  输出的结果就是 (1,1)
看到这里，大家可能会疑问，这不是只有三种输出结果么，怎么还会出现（0，0）这种场景呢，说到这里就不得不说下内存访问重排序与内存可见性，下面是我测试的代码
```
public class Test {
    private static int x = 0, y = 0;
    private static int a = 0, b =0;
    

    public static void main(String[] args) throws InterruptedException {
        int i = 0;
        for(;;) {
            i++;
            x = 0; y = 0;
            a = 0; b = 0;
            Thread one = new Thread(new Runnable() {
                public void run() {
                    //由于线程one先启动，下面这句话让它等一等线程two. 读着可根据自己电脑的实际性能适当调整等待时间.
                    shortWait(100000);
                    a = 1;
                    x = b;
                }
            });

            Thread other = new Thread(new Runnable() {
                public void run() {
                    b = 1;
                    y = a;
                }
            });
            one.start();other.start();
            one.join();other.join();
            String result = "第" + i + "次 (" + x + "," + y + "）";
            if(x == 0 && y == 0) {
                System.err.println(result);
                break;
            } else {
                System.out.println(result);
            }
        }
    }


    public static void shortWait(long interval){
        long start = System.nanoTime();
        long end;
        do{
            end = System.nanoTime();
        }while(start + interval >= end);
    }
}
```
输出的结果如下
![](https://telegraph-image-aed.pages.dev/file/6782c517e97d1045498ed.jpg)



## 内存可见性

在现代的计算机体系中，由于存储介质的限制，为了提升内存的访问速度，通过会通过缓存机制来提升访问效率，如下图所示
在这种模型下会存在一个现象，即缓存中的数据与主内存的数据并不是实时同步的，各CPU（或CPU核心）间缓存的数据也不是实时同步的。这导致在同一个时间点，
各CPU所看到同一内存地址的数据的值可能是不一致的。从程序的视角来看，就是在同一个时间点，各个线程所看到的共享变量的值可能是不一致的。

![](https://telegraph-image-aed.pages.dev/file/942e529950590c2ac9c12.png)

对于多线程执行过程，对内存数据进行赋值操作时，会有以下几个步骤
1. 尝试从线程独有的缓存中获取到数据
2. 如果缓存中不存在，则从主内存获取并存入缓存（正常在线程初始化时就完成了这步操作）
3. 修改数据，刷新本地缓存
4. 刷新数据至主内存

因为存在主内存和线程缓存的区分，因此诞生了可见性的概念，对于线程A和线程B，都从主内存中获取了a和b的值，
那当他们各自对a和b进行操作时，以a = 1;x = b;b = 1;y = a;这种执行顺序，内存中可能出现以下这种情况

![](https://telegraph-image-aed.pages.dev/file/86f434465633f860ad79e.png)

当线程A执行完a = 1;x = b;时，线程B看到的a的值本来应该是1，而实际却是0，这就是多线程操作下可见性的问题，对于线程A和线程B，执行过程中使用同一个对象时，互相之间感知不到对象的变化,
这种问题会导致文章开头提到的代码执行结果会输出（0，0）

## happendbefore原则

Java的目标是成为一门平台无关性的语言，即Write once, run anywhere. 但是不同硬件环境下指令重排序的规则不尽相同。例如，x86下运行正常的Java程序在IA64下就可能得到非预期的运行结果。为此，JSR-1337制定了Java内存模型(Java Memory Model, JMM)，旨在提供一个统一的可参考的规范，屏蔽平台差异性。从Java 5开始，Java内存模型成为Java语言规范的一部分。

根据Java内存模型中的规定，可以总结出以下几条happens-before规则8。Happens-before的前后两个操作不会被重排序且后者对前者的内存可见。

程序次序法则：线程中的每个动作A都happens-before于该线程中的每一个动作B，其中，在程序中，所有的动作B都能出现在A之后。
监视器锁法则：对一个监视器锁的解锁 happens-before于每一个后续对同一监视器锁的加锁。
volatile变量法则：对volatile域的写入操作happens-before于每一个后续对同一个域的读写操作。
线程启动法则：在一个线程里，对Thread.start的调用会happens-before于每个启动线程的动作。
线程终结法则：线程中的任何动作都happens-before于其他线程检测到这个线程已经终结、或者从Thread.join调用中成功返回，或Thread.isAlive返回false。
中断法则：一个线程调用另一个线程的interrupt happens-before于被中断的线程发现中断。
终结法则：一个对象的构造函数的结束happens-before于这个对象finalizer的开始。
传递性：如果A happens-before于B，且B happens-before于C，则A happens-before于C
Happens-before关系只是对Java内存模型的一种近似性的描述，它并不够严谨，但便于日常程序开发参考使用，关于更严谨的Java内存模型的定义和描述，请阅读JSR-133原文或Java语言规范章节17.4。

### Java内存模型

除此之外，Java内存模型对volatile和final的语义做了扩展。对volatile语义的扩展保证了volatile变量在一些情况下不会重排序，volatile的64位变量double和long的读取和赋值操作都是原子的。对final语义的扩展保证一个对象的构建方法结束前，所有final成员变量都必须完成初始化（的前提是没有this引用溢出）。

Java内存模型关于重排序的规定，总结后如下表所示：
![](https://telegraph-image-aed.pages.dev/file/7ca0280703acae70ee0d1.jpg)
表中“第二项操作”的含义是指，第一项操作之后的所有指定操作。如，普通读不能与其之后的所有volatile写重排序。另外，JMM也规定了上述volatile和同步块的规则尽适用于存在多线程访问的情景。例如，若编译器（这里的编译器也包括JIT，下同）证明了一个volatile变量只能被单线程访问，那么就可能会把它做为普通变量来处理。

留白的单元格代表允许在不违反Java基本语义的情况下重排序。例如，编译器不会对对同一内存地址的读和写操作重排序，但是允许对不同地址的读和写操作重排序。

除此之外，为了保证final的新增语义。JSR-133对于final变量的重排序也做了限制。

构建方法内部的final成员变量的存储，并且，假如final成员变量本身是一个引用的话，这个final成员变量可以引用到的一切存储操作，都不能与构建方法外的将当期构建对象赋值于多线程共享变量的存储操作重排序。例如对于如下语句：
x.finalField = v; … ;构建方法边界sharedRef = x; v.afield = 1; x.finalField = v; … ; 构建方法边界sharedRef = x;

这两条语句中，构建方法边界前后的指令都不能重排序。

初始读取共享对象与初始读取该共享对象的final成员变量之间不能重排序。例如对于如下语句：
x = sharedRef; … ; i = x.finalField;

前后两句语句之间不会发生重排序。由于这两句语句有数据依赖关系，编译器本身就不会对它们重排序，但确实有一些处理器会对这种情况重排序，因此特别制定了这一规则。

## 内存屏障

在Java中，内存屏障（Memory Barrier）是一种用于控制内存可见性和指令重排序的特殊指令或者机制。它们确保了在多线程环境下对共享变量的操作能够按照预期顺序进行，从而避免了由于指令重排序或者缓存一致性等因素导致的数据不一致性问题。

内存屏障可以分为两类：读屏障（Read Barrier）和写屏障（Write Barrier）。

1. 读屏障（Read Barrier）：保证在读取操作之前，所有在读取操作之前的内存访问操作都已经完成。它确保了读取的值是最新的，并且避免了乱序读取导致的数据不一致问题。

2. 写屏障（Write Barrier）：保证在写入操作之后，所有在写入操作之前的内存访问操作都已经完成。它确保了写入的值对于其他线程是可见的，并且避免了乱序写入导致的数据不一致问题。

Java中的内存屏障主要通过synchronized关键字、volatile关键字、以及java.util.concurrent包中的锁、原子类等方式来实现。这些机制在底层都包含了内存屏障的操作，保证了多线程环境下的内存可见性和指令重排序的正确性。

应用场景包括但不限于：

1. **双重检查锁定（Double-Checked Locking）**：在单例模式中，通过双重检查锁定可以减少synchronized的开销，但需要使用volatile关键字来保证多线程环境下的正确性。

2. **发布-订阅模式中的事件通知**：通过volatile关键字来保证事件发布者发布的事件对所有订阅者都是可见的。

3. **并发容器**：Java中的并发容器（如ConcurrentHashMap、ConcurrentLinkedQueue等）都使用了内存屏障来确保线程安全和可见性。

4. **多线程同步**：使用synchronized关键字、Lock接口等同步机制时，内部都包含了内存屏障，保证了临界区内外的内存可见性和指令重排序的正确性。

总的来说，内存屏障在Java中扮演着重要的角色，保证了多线程环境下程序的正确性和可靠性。

## 多线程三大特性

在多线程编程中，有三大重要特性：

1. **原子性（Atomicity）**：原子性指的是操作不可被中断，要么全部执行成功，要么全部不执行，不存在执行了一半的情况。在多线程环境中，如果某个操作是原子的，那么在任何时刻只有一个线程能够执行该操作，不会被其他线程干扰。Java中的原子性通常通过synchronized关键字、volatile关键字、以及java.util.concurrent.atomic包中的原子类来实现。

2. **可见性（Visibility）**：可见性指的是当一个线程对共享变量的修改能够被其他线程立即观察到。在多线程环境中，由于线程间的缓存不一致或者指令重排序等原因，某个线程对共享变量的修改并不一定会立即被其他线程看到，这就导致了可见性问题。为了解决可见性问题，Java中提供了volatile关键字和内存屏障（Memory Barrier）机制来确保共享变量的可见性。

3. **有序性（Ordering）**：有序性指的是程序执行的顺序与代码编写的顺序一致。在多线程环境中，由于指令重排序的存在，有可能导致代码执行的顺序与编写的顺序不一致，从而产生错误的结果。Java中通过内存屏障（Memory Barrier）机制来确保代码执行的顺序与编写的顺序一致，保证了程序的正确性。

这三个特性是多线程编程中非常重要的概念，了解并正确处理它们有助于编写高效、正确、可靠的多线程程序。

## 应用场景

通过上述文章的分析，我们了解了重排序的定义，以及Java中提供一些能力帮助我们避免出现这种问题，那回到我们这个代码本身，应该如何保证执行结果符合我们预期呢
，按照单线程环境下，运行的结果，应该有序的，结果应该是（1，0）或者（0，1），那我们试着用synchronized和volatile分别保证了有序性和可见性试试看
```
private static volatile int x = 0, y = 0;
private static volatile int a = 0, b =0;
```

```
Thread one = new Thread(new Runnable() {
                public void run() {
                    //由于线程one先启动，下面这句话让它等一等线程two. 读着可根据自己电脑的实际性能适当调整等待时间.
                    shortWait(100000);
                    synchronized (this) {
                        a = 1;
                        x = b;
                    }
                    
                }
            });

            Thread other = new Thread(new Runnable() {
                public void run() {
                    synchronized (this) {
                        b = 1;
                        y = a;
                    }
                    
                }
            });
```

输出结果如下
![](https://telegraph-image-aed.pages.dev/file/8b712b8ad7680a37bdba5.jpg)

其实，在我们实际使用过程中一般都会组合起来使用，`synchronized` 和 `volatile` 关键字在 Java 中分别保障了不同的特性：

1. **synchronized**：
   - **原子性（Atomicity）**：`synchronized` 关键字可以确保被它修饰的代码块或方法在同一时刻只能被一个线程执行，因此可以保证原子性。即使在方法内部有多个操作，也会作为一个原子操作来执行。
   - **可见性（Visibility）**：进入 synchronized 块时，线程会将本地内存中的共享变量无效化，强制从主内存中重新获取最新的共享变量值。当一个线程释放锁时，会将对共享变量的修改刷新到主内存中，这样其他线程就可以看到最新的值，因此也保证了可见性。
   - **有序性（Ordering）**：`synchronized` 保证了代码块的执行是按照顺序进行的，即一个线程在执行完一个 synchronized 代码块后，其他线程才能进入相同的 synchronized 代码块。

2. **volatile**：
   - **可见性（Visibility）**：`volatile` 关键字保证了被它修饰的变量对所有线程的可见性。当一个线程修改了 volatile 变量的值，这个新值会立即被其他线程看到，因此保证了可见性。
   - **有序性（Ordering）**：`volatile` 关键字可以保证被它修饰的变量的读写操作是按照顺序进行的，即对于每个 volatile 变量的读操作都发生在后续的写操作之前。

总的来说，`synchronized` 保障了原子性、可见性和有序性，而 `volatile` 仅保障了可见性和部分有序性。


## 总结

总结起来，重排序是指在计算机程序执行过程中，由于编译器、处理器或者运行时系统的优化，指令的执行顺序与代码编写顺序不一致的现象。为了保证程序的正确性和可靠性，Java提供了Java内存模型（Java Memory Model，JMM）以及相关机制来控制重排序，主要包括以下几点：

1. **Java内存模型（JMM）**：JMM定义了Java程序中多线程之间的内存访问规则，确保了多线程环境下的可见性、原子性和有序性。

2. **可见性、原子性和有序性**：这三个特性是多线程编程中的关键概念。可见性指的是一个线程对共享变量的修改能被其他线程立即观察到；原子性指的是操作不可被中断，要么全部执行成功，要么全部不执行；有序性指的是程序执行的顺序与代码编写的顺序一致。

3. **内存屏障（Memory Barrier）**：内存屏障是一种用于控制内存可见性和指令重排序的特殊指令或者机制。Java中的内存屏障通过synchronized关键字、volatile关键字以及java.util.concurrent包中的锁、原子类等来实现，保证了多线程环境下程序的正确性和可靠性。

4. **happens-before规则**：Java内存模型定义了happens-before规则，确保了在多线程环境下的一致性。这些规则包括程序次序法则、监视器锁法则、volatile变量法则等，保证了线程之间的顺序一致性。

综上所述，重排序是多线程编程中需要注意的一个重要问题，Java提供了丰富的机制来解决这个问题，保证了程序在多线程环境下的正确性和可靠性。