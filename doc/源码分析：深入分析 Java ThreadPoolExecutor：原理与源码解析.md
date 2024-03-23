# ThreadPoolExecutor是什么？ 

> `ThreadPoolExecutor` 是 Java 中用于管理线程池的类之一。它提供了一个灵活的线程池实现，可以根据需要自动管理线程的创建、销毁和重用，以及控制并发任务的执行。

## 应用场景

1. **线程生命周期管理**：
   - `ThreadPoolExecutor` 负责管理线程的生命周期，包括线程的创建、销毁和重用。通过线程池，可以避免频繁地创建和销毁线程，减少资源消耗和系统开销。

2. **任务调度与执行**：
   - `ThreadPoolExecutor` 负责调度和执行提交给线程池的任务。可以将任务提交给线程池执行，并根据需要调整线程池的大小，以适应不同的工作负载和系统资源。

3. **并发控制**：
   - `ThreadPoolExecutor` 提供了一种并发控制的机制，可以限制线程池中同时执行的任务数量，以避免系统资源耗尽或者任务堆积导致的性能问题。

4. **性能优化**：
   - 通过合理地配置线程池参数，可以优化任务的执行效率和系统的资源利用率，提高程序的性能和响应速度。

## ThreadPoolExecutor工作原理

### 类层次结构
![](https://telegraph-image-aed.pages.dev/file/11394e855e6e40b7d985a.jpg)
可以看到`ThreadPoolExecutor`是继承自接口`ExecutorService`的实现类，`ThreadPoolExecutor`的上层抽象接口实现了submit、invoke等方法，在ThreadPoolExecutor中实现了模板方法中的execute方法
![](https://telegraph-image-aed.pages.dev/file/129910d7e98bdb1bc96df.jpg)

### 执行过程
1.  获取当前工作的线程运行状态控制
2.  比较运行中的线程数和初始化时的核心线程数
3.  如果小于核心线程数，则创建新的线程来处理任务
4.  如果等待队列未满，则加入等待队列
5.  否则判断是否超过了核心线程数，未超过则新增线程来处理任务
6.  否则执行拒绝策略
![](https://telegraph-image-aed.pages.dev/file/a357963bc21b8b5eb3f4c.png)
```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```
### 核心参数
1. 核心线程数：真正执行任务的线程数，一般设置为CPU核数+1，如果是IO密集型可以适当增加
2. 最大线程数：允许创建的最大线程数，一般设置等于核心线程数，因为超过了核心线程数，再创建新的线程，只会产生新的上下文切换和资源竞争的开销，并不能提升线程池处理任务的速度
3. 等待超时时间： 设定一个线程的执行超时时间，超过这个时间还未执行完，则会释放当前线程资源
4. 等待队列： 用于留存等待处理的任务

### ThreadPoolExecutor 创建方式
1.  通过ThreadPoolExecutor给定的构造方法
2. 通过Executors提供的静态方法创建，Executors目前提供了newFixedThreadPool、newWorkStealingPool、newCachedThreadPool：用来处理短期任务，在60秒内会重用已创建的线程执行任务

建议通过ThreadPoolExecutor的构造方法手动创建

### Executors创建的线程池

1. newFixedThreadPool：固定线程数的线程池，使用LinkedBlockingQueue作为无界的等待队列，可能出现因任务等待过多导致OOM
2. newWorkStealingPool：通过ForkJoinPool实现的线程池，具备工作窃取的能力（1.8以后开始提供）
3. newCachedThreadPool：用来处理短期任务，在60秒内会重用已创建的线程执行任务
4. newSingleThreadScheduledExecutor：用来延迟调度的线程池


### 注意点
1. 最大线程数和核心线程数并不是越大越好，需要根据服务器配置以及任务类型合理配置
2. 设定一个合理的等待超时时间，有助于快速处理任务
3. 等待队列需要考虑OOM相关问题，例如不要用无界队列，避免因创建过多任务导致OOM问题



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

### 总结
总的来说，ThreadPoolExecutor 是一个功能强大、灵活可配置的线程池实现，被广泛应用于 Java 并发编程中，用于管理线程的生命周期、调度任务的执行、优化系统的性能和资源利用率

欢迎关注我的公众号“**毕知必会**”，原创技术文章第一时间推送。

<center>
    <img src="https://telegraph-image-aed.pages.dev/file/74ebf1fb389a9ab62228a.jpg" style="width: 100px;">
</center>
