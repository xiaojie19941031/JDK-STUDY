# ThreadPoolExecutor

Worker:内部存放执行任务的线程以及任务[真正执行任务]

线程池状态控制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用AtomicInteger变量将工作线程数量与运行状态压缩在一起[其中高3位为线程池的运行状态,低29位为当前工作线程数]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运行状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RUNNING:运行状态,可以处理新任务并执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SHUTDOWN:关闭状态,不接受新任务但是执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;STOP:停止状态,不接受新任务也不处理队列中的任务并且会打断当前运行中任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TIDYING:整理状态,所有任务结束并且此时工作线程为0会进入结束状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TERMINATED:结束状态

构造方法

```
    new ThreadPoolExecutor(
                int corePoolSize,
                int maximumPoolSize,
                long keepAliveTime,
                TimeUnit unit,
                BlockingQueue<Runnable> workQueue,
                ThreadFactory threadFactory,
                RejectedExecutionHandler handler),这是参数最全的构造方式也是推荐使用的构造方式,因此必须熟悉每一个参数的含义
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;corePoolSize:核心线程数,线程池的基本大小

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;maximumPoolSize:线程池最大数量,线程池允许创建的最大线程数[如果是无界队列,此参数意义不大]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;keepAliveTime:线程空闲时最大存活时间

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unit:keepAliveTime时间类型

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;BlockingQueue:用于保存等待执行的任务的阻塞队列,一般可以使用下面几种

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ArrayBlockingQueue:基于数组实现的有界阻塞队列,FIFO(先进先出)对元素进行排序

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LinkedBlockingQueue:一个基于链表实现的阻塞队列,不设置容量则为无界队列,FIFO(先进先出)对元素进行排序,吞吐量高

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SynchrousQueue:一个不存储元素的阻塞队列,每个插入元素都必须等到另一个线程调用移出操作,否则插入操作会一直阻塞[相反也是],吞吐量高

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PriorityBlockingQueue:优先级阻塞队列

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还有很多拓展的队列,在JCTools中可以学习

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;threadFactory:用于设置创建线程的工厂

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RejectedExecutionHandler:饱和策略,当线程池中的线程数超过线程池最大数量执行的拒绝策略

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AbortPolicy:直接抛出异常

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CallerRunsPolicy:直接调用所在线程执行任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DiscardOldestPolicy:丢弃队列中最近的一个任务,并执行当前任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DisacrdPolicy:不处理直接丢弃

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;自定义策略:实现RejectedExecutionHandler

往线程池中提交任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;execute(Runnable command):向线程池提交任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;工作流程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、判断当前线程池中的工作线程数与corePoolSize的大小,如果比corePoolSize小,会创建一个新的线程用于执行任务,否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、判断当前线程池中的任务队列是否已塞满,如果没有塞满直接将任务扔进队列中,否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、判断当前线程池中的工作线程数是否达到了maximumPoolSize,未达到创建非核心线程执行任务,否则根据饱和策略处理

```
public void execute(Runnable command) {
    if (command == null){
        throw new NullPointerException();
    }
    // 这个值包含了线程池的状态以及当前线程池中工作线程的个数
    int c = ctl.get();
    // 如果工作线程数 < 核心线程数
    if (workerCountOf(c) < corePoolSize) {
        // 新建一个工作线程来运行任务,如果成功了,则直接返回[此过程会获取全局锁会消耗性能,因此线程池有个优化点,可以预热线程池]
        if (addWorker(command, true)){
            return;
        }
        c = ctl.get();
    }
    // RUNNING状态,则尝试把任务添加到任务队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //如果线程池已经不是RUNNING状态了,把任务从队列中移除,并执行拒绝任务策略
        if (!isRunning(recheck) && remove(command)){
            reject(command);
        }
        // 如果工作线程数为0,就添加一个新的工作线程,因为旧线程可能已经被回收了,所以工作线程数可能为0
        else if (workerCountOf(recheck) == 0){
            addWorker(null, false);
        }
    }
    // 创建非核心线程来执行任务
    else if (!addWorker(command, false)){
        reject(command);
    }
}
```
添加新的线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addWorker(Runnable command,boolean core):根据core值判断创建的是核心线程还是非核心线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;addWorker的工作可分为两个部分

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、原子操作,判断是否可以创建worker,通过自旋、CAS等操作判断继续创建还是返回false,自旋周期一般很短[比锁效率高]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、创建Worker启动线程执行任务
```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 无限循环
    for (;;) {
        // 这个值包含了线程池的状态以及当前线程池中工作线程的个数
        int c = ctl.get();
        // 获取线程池的状态值
        int rs = runStateOf(c);
        if (rs >= SHUTDOWN && !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty())){
            return false;
        }
        for (;;) {
            // 获取工作线程的数量
            int wc = workerCountOf(c);
            // 代表线程已经超标了
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize)){
                return false;
            }
            // 如果原子性增加工作线程数(+1)成功,代表创建线程成功
            if (compareAndIncrementWorkerCount(c)){
                break retry;
            }
            c = ctl.get();
            if (runStateOf(c) != rs){
                continue retry;
            }
        }
    }
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 创建Worker
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 获取全局锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    // 线程正在运行
                    if (t.isAlive()){
                        throw new IllegalThreadStateException();
                    }
                    // 把worker加入到工作线程Set里面
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize){
                        largestPoolSize = s;
                    }
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 开始执行任务
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (!workerStarted){
            addWorkerFailed(w);
        }
    }
    return workerStarted;
}
```
线程真正运行的地方

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;runWorker(Worker w):执行任务
```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 可以响应中断
    w.unlock(); 
    boolean completedAbruptly = true;
    try {
        // 任务不为null,或者队列中还存在任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // Stop状态下直接中断
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted()){
                wt.interrupt()
            }
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 开始正式运行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行完成后的钩子方法
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
线程池带来的好处

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、降低资源消耗,通过重复利用已创建的线程降低线程创建和销毁造成的消耗

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、提高响应速度,当任务到达时不用等待线程创建就能执行任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、提高线程的可管理性,线程是稀缺资源,如果无限制的创建不仅会消耗系统资源还会降低系统的稳定性

线程池类型

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、FixedThreadPool:固定线程数线程池,特点核心线程池与最大线程池大小一样并且线程不会被回收,没有一个任务就会创建一个线程去执行直到到达最大线程数

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、CachedThreadPool:缓存线程池,特点核心线程数为0,当有任务到来时会创建一个线程执行任务,如果存在空闲线程则不会创建,空闲线程超过60s会自动回收

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、SingleThreadExecutor:只存在一个线程的线程池,特点核心线程池与最大线程池大小都为1并且线程不会被回收[保证了task按照指定顺序被执行]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4、ScheduledThreadPoolExecutor:定时线程池,特点可以定时让线程执行任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5、WorkStealingPool:窃取式、抢占式线程池,特点通过线程并行实现,此线程不能保证任务的顺序执行