# ForkJoinPool是什么？ 

> `ForkJoinPool` 类是 Java 平台中用于实现任务并行执行的关键类之一。它提供了一种高效的并行计算方式，特别适用于递归任务的分解和执行。

## 应用场景

1. **并行计算**：
   - `ForkJoinPool` 提供了一种方便的方式来执行大规模的计算任务，并充分利用多核处理器的性能优势。通过将大任务分解成小任务，并通过工作窃取算法实现任务的并行执行，可以提高计算效率。

2. **递归任务处理**：
   - `ForkJoinPool` 特别适用于递归式的任务分解和执行。它可以将一个大任务递归地分解成许多小任务，并通过工作窃取算法动态地将这些小任务分配给工作线程执行。

3. **并行流操作**：
   - Java 8 引入了 Stream API，用于对集合进行函数式编程风格的操作。`ForkJoinPool` 通常用于执行并行流操作中的并行计算部分，例如对流中的元素进行过滤、映射、聚合等操作。

4. **高性能任务执行**：
   - `ForkJoinPool` 提供了一种高性能的任务执行机制，通过对任务进行动态调度和线程池管理，可以有效地利用系统资源，并在多核处理器上实现任务的并行执行。

总的来说，`ForkJoinPool` 类在 Java 中具有广泛的应用场景，特别适用于大规模的并行计算任务和递归式的任务处理。它通过工作窃取算法和任务分割合并机制，提供了一种高效的并行计算方式，可以显著提高计算效率和性能。


## 并行计算

并行计算是指在多个处理单元（如多核处理器、多处理器系统、分布式系统等）上同时执行计算任务的一种计算方式。其核心原理是将一个大任务分解成多个小任务，并通过同时执行这些小任务来加速整体计算过程。

并行计算的核心原理是将大任务分解成多个小任务，并通过同时执行这些小任务来提高计算效率。它在处理大规模数据和复杂计算任务时具有重要的意义，可以有效地利用系统资源，提高计算速度和性能。

## ForkJoinPool的特点

1. **任务分解**：
   - `ForkJoinPool` 通过将一个大任务分解成多个小任务来实现并行计算。这种分解方式通常是递归的，即将一个任务分解成若干子任务，每个子任务又可以继续分解成更小的子任务，直到任务足够小或无法再分解为止。

2. **工作窃取**：
   - `ForkJoinPool` 使用工作窃取（work-stealing）算法来实现任务的分配和执行。在工作窃取算法中，每个工作线程都有自己的工作队列，当一个线程执行完自己队列中的任务后，它会去其他线程的队列中偷取任务来执行。这种方式可以减少线程之间的竞争，提高并行性能。

3. **任务合并**：
   - `ForkJoinPool` 中的任务通常是可以合并的。当一个线程执行一个任务时，它可能会将该任务的子任务分配给其他线程执行，并等待子任务执行完成后将结果合并。这种任务合并机制可以减少线程之间的同步和通信开销，提高并行计算效率。

4. **线程池管理**：
   - `ForkJoinPool` 负责管理线程池中的线程，包括线程的创建、销毁和重用。它会根据需要动态地调整线程的数量，以适应当前的工作负载。

5. **递归任务处理**：
   - `ForkJoinPool` 特别适用于递归式的任务处理。通过工作窃取算法和任务分解合并机制，可以实现递归任务的高效并行执行。

总的来说，`ForkJoinPool` 类提供了一种高效的并行计算方式，通过工作窃取算法和任务分解合并机制，可以充分利用多核处理器的性能优势，并实现任务的高效并行执行。它在处理大规模数据和递归式任务时具有重要的应用价值。

## 工作窃取算法

工作窃取算法的基本思想是将任务队列分配给每个线程，并通过线程间的协作来完成任务的执行。每个线程都有自己的任务队列，当一个线程执行完自己队列中的任务后，它会去其他线程的队列中偷取（即 "窃取"）任务来执行。

以下是工作窃取算法的基本原理：

1. **本地队列**：
   - 每个线程都有一个自己的本地任务队列，用于存放待执行的任务。这些任务通常是由当前线程创建或分配的。

2. **窃取任务**：
   - 当一个线程执行完自己队列中的任务后，它会去其他线程的队列中偷取任务来执行。通常情况下，线程会选择窃取其他线程队列中的末尾位置（或近似末尾位置）的任务，因为这些任务最可能是最近添加的，即最新的任务。

3. **动态调整**：
   - 工作窃取算法具有一定的自适应性，它会根据当前系统的负载情况动态地调整任务的分配策略。例如，当一个线程的本地队列为空时，它会去窃取其他线程的任务来执行；当一个线程的本地队列过长时，它可能会主动将部分任务分配给其他线程。

工作窃取算法的优点是能够减少线程之间的竞争和同步开销，提高并行性能。它利用了多核处理器的特性，使得任务可以在多个处理单元上并行执行，从而提高了系统的整体吞吐量和响应速度。

## ForkJoinPool工作原理

### 类层次结构
![](https://telegraph-image-aed.pages.dev/file/f7350520eaa476ce75483.jpg)
可以看到`ForkJoinPool`是继承自接口`ExecutorService`的实现类，该类的另一个重要实现是`ThreadPoolExecutor`,`ForkJoinPool`实现了接口中的submit、invokeAll等关键方法
![](https://telegraph-image-aed.pages.dev/file/8eab696e968a4db2d64c7.jpg)

![](https://telegraph-image-aed.pages.dev/file/dca4739d5abe51475886c.jpg)

### 核心设计
1.  ForkJoinPool的任务会被内部存储了一个WorkQueue数组，提交给ForkJoinPool的任务会被分配到指定的WorkQueue上执行
2. 每个WorkQueue内部维护了一个ForkJoinTask数组用来存储待执行的任务，以及一个独立的ForkJoinWorkerThread用来真正执行任务
![](https://telegraph-image-aed.pages.dev/file/24e84dc79cec82d701036.png)

### 执行过程

#### 提交任务
以submit方法为例，我们看下`ForkJoinPool`是如何处理提交的任务的，真正处理的方法是`externalPush`，如下所示，
1. 判断工作队列数组是否为空
2. 通过按位与得到当前任务分配的工作队列
3. 尝试通过CAS锁定该队列
4. 判断工作队列中的任务数组是否为空
5. 释放锁

```
final void externalPush(ForkJoinTask<?> task) {
        WorkQueue[] ws; WorkQueue q; int m;
        int r = ThreadLocalRandom.getProbe();
        int rs = runState;
        if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&  // 判断工作队列数组是否为空
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 && // 通过按位与得到当前任务分配的工作队列
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {  // 尝试通过CAS锁定该队列
            ForkJoinTask<?>[] a; int am, n, s;
            if ((a = q.array) != null &&   // 判断工作队列中的任务数组是否为空
                (am = a.length - 1) > (n = (s = q.top) - q.base)) { // 尝试添加任务
                int j = ((am & s) << ASHIFT) + ABASE;
                U.putOrderedObject(a, j, task);
                U.putOrderedInt(q, QTOP, s + 1);
                U.putIntVolatile(q, QLOCK, 0);
                if (n <= 1)  // 如果就只有一个任务
                    signalWork(ws, q); // 调用单个任务执行
                return;
            }
            U.compareAndSwapInt(q, QLOCK, 1, 0); // 释放锁
        }
        externalSubmit(task); // 提交任务
    }
```
#### 执行任务
ForkJoinPool执行任务的过程如下
1. 判断工作队列是否需要扩容
2. 阻塞调用对应WorkQueue的runTask方法
3. 如果无法获取到执行资源，就等待任务调用
```
final void runWorker(WorkQueue w) {
        w.growArray();                   // allocate queue
        int seed = w.hint;               // initially holds randomization hint
        int r = (seed == 0) ? 1 : seed;  // avoid 0 for xorShift
        for (ForkJoinTask<?> t;;) {
            if ((t = scan(w, r)) != null)
                w.runTask(t);
            else if (!awaitWork(w, r))
                break;
            r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
        }
    }
```
WorkQueue的rukTask
1. 标记当前状态为忙碌
2. 调用当前任务执行
3. 继续执行当前队列中等待执行的任务
4. 尝试从其他WorkQueue中窃取任务执行
```
final void runTask(ForkJoinTask<?> task) {
            if (task != null) {
                scanState &= ~SCANNING; // mark as busy
                (currentSteal = task).doExec();
                U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
                execLocalTasks();
                ForkJoinWorkerThread thread = owner;
                if (++nsteals < 0)      // collect on overflow
                    transferStealCount(pool);
                scanState |= SCANNING;
                if (thread != null)
                    thread.afterTopLevelExec();
            }
        }
        
final void execLocalTasks() {
            int b = base, m, s;
            ForkJoinTask<?>[] a = array;
            if (b - (s = top - 1) <= 0 && a != null &&
                (m = a.length - 1) >= 0) {
                if ((config & FIFO_QUEUE) == 0) {
                    for (ForkJoinTask<?> t;;) {
                        if ((t = (ForkJoinTask<?>)U.getAndSetObject
                             (a, ((m & s) << ASHIFT) + ABASE, null)) == null)
                            break;
                        U.putOrderedInt(this, QTOP, s);
                        t.doExec();
                        if (base - (s = top - 1) > 0)
                            break;
                    }
                }
                else
                    pollAndExecAll();
            }
        }
```
窃取任务的实现
```
final class WorkQueue {

    // 省略其他代码...

    // 偷取任务的方法
    final ForkJoinTask<?> pollAt(int b) {
        ForkJoinTask<?>[] a; int i; ForkJoinTask<?> t;
        if ((a = array) != null && (i = a.length - 1) >= 0 &&
            (t = a[b & i]) != null && t != a[b & (i >>>= 1)]) {
            if (t instanceof CountedCompleter) // 偷取CountedCompleter任务
                ((CountedCompleter<?>)t).propagateCompletion();
            return t;
        }
        return null;
    }

    // 从其他队列中偷取任务
    final ForkJoinTask<?> poll() {
        WorkQueue[] ws; int m;
        int r = ThreadLocalRandom.current().nextInt();
        if ((ws = pool.workQueues) != null && (m = ws.length - 1) >= 0 &&
            (ws = ws[m & r & SQMASK]) != null) {
            int j = r & SMASK; // 随机选择一个队列进行偷取
            WorkQueue w; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
            if ((w = ws[j]) != null && w.base - w.top < 0 &&
                (a = w.array) != null) { // 如果队列不为空，且还有任务未执行
                for (int b = r;;) { // 循环偷取任务
                    int n = a.length;
                    if (n >= NCPU || (t = w.pollAt(b & (n - 1))) == null) // 如果当前队列中没有可偷取的任务
                        break;
                    if (w.base == (b + 1)) // 如果队列被其他线程修改
                        return null;
                    if (UNSAFE.compareAndSwapObject(a, ((n - 1) & b) << ASHIFT, t, null))
                        return t;
                    if (n <= 1 || w.currentSteal != b)
                        break;
                }
            }
        }
        return null;
    }

    // 省略其他代码...
}

```

### 应用示例
如下所示是一个计算数组求和的任务，分别通过ForkJoinPool和ThreadPoolExecutor实现
#### ThreadPoolExecutor实现
```
import org.springframework.util.StopWatch;

import java.util.concurrent.*;

class MyTask implements Callable<Integer> {
    private int[] array;
    private int start;
    private int end;

    public MyTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    // 重写 call 方法来实现任务的具体逻辑
    @Override
    public Integer call() throws Exception {
        int sum = 0;
        for (int i = start; i < end; i++) {
            sum += array[i];
        }
        return sum;
    }
}

public class ThreadPoolExecutorTest {

    // 实现 Callable 接口来表示可并行执行的任务


    public static void main(String[] args) {
        int[] array = new int[10000000]; // 假设有一个大数组
        StopWatch stopWatch = new StopWatch();
        MyTask myTask = new MyTask(array, 0, array.length);
        ExecutorService executor = new ThreadPoolExecutor(4, 16, 0, TimeUnit.SECONDS, new ArrayBlockingQueue<>(1000));

        stopWatch.start("ThreadPoolExecutor");
        Future<Integer> future = executor.submit(myTask);
        try {
            future.get();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
        stopWatch.stop();
        System.out.println(stopWatch.prettyPrint());
    }
}
```
#### ForkJoinPool实现
```
import org.springframework.util.StopWatch;

import java.util.concurrent.*;

// 继承 RecursiveTask 类来表示可分解的任务
class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10000; // 阈值，控制任务的分解
    private int[] array;
    private int start;
    private int end;

    public SumTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    // 重写 compute 方法来实现任务的具体逻辑
    @Override
    protected Long compute() {
        // 如果任务足够小，则直接计算任务结果
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // 否则，将任务分解成子任务
            int mid = (start + end) / 2;
            SumTask leftTask = new SumTask(array, start, mid);
            SumTask rightTask = new SumTask(array, mid, end);

            // 并行执行子任务
            leftTask.fork();
            long rightResult = rightTask.compute();

            // 等待左子任务执行完毕，并获取其结果
            long leftResult = leftTask.join();

            // 返回子任务结果的总和
            return leftResult + rightResult;
        }
    }


}


public class ForkJoinPoolTest {
    public static void main(String[] args) {
        int[] array = new int[10000000]; // 假设有一个大数组

        // 初始化数组
        for (int i = 0; i < array.length; i++) {
            array[i] = i;
        }

        // 创建 ForkJoinPool
        ForkJoinPool forkJoinPool = new ForkJoinPool();

        // 创建任务
        SumTask task = new SumTask(array, 0, array.length);

        StopWatch stopWatch = new StopWatch();
        stopWatch.start("ForkJoinPool");



        // 提交任务到 ForkJoinPool，并等待执行结果
        long result = forkJoinPool.invoke(task);
        stopWatch.stop();

        System.out.println(stopWatch.prettyPrint());
    }
}

```
### 总结
ForkJoinPool通过内部维护WorkQueue数组的方式，将ForkJoinTask任务分配到指定的工作队列中执行，通过将`鸡蛋分散在多个篮子`这种分治思想提升线程的工作效率，并且针对线程执行过程中出现的空闲情况，设计出了工作窃取的算法，促使工作线程尽可能的饱和执行，在线程安全方面，通过Unsafe的CAS操作配合原子类，保证了多线程场景下的竞争安全问题。

欢迎关注我的公众号“**毕知必会**”，原创技术文章第一时间推送。

<center>
    <img src="https://telegraph-image-aed.pages.dev/file/74ebf1fb389a9ab62228a.jpg" style="width: 100px;">
</center>
