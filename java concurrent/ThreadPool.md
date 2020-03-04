# ThreadPoolExecutor[线程池]

Worker:内部存放执行任务的线程以及任务[真正执行任务]

线程池状态控制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用AtomicInteger变量将工作线程数量与运行状态压缩在一起[其中高3位为线程池的运行状态,低29位为当前工作线程数]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运行状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RUNNING:运行状态,可以处理新任务并执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SHUTDOWN:关闭状态,不接受新任务但是执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;STOP:停止状态,不接受新任务也不处理队列中的任务并且会打断当前运行中任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TIDYING:整理状态,所有任务结束并且此时工作线程为0会进入结束状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TERMINATED:结束状态

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
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;corePoolSize:核心线程数,线程池的基本大小

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;maximumPoolSize:线程池最大数量,线程池允许创建的最大线程数[如果是无界队列,此参数意义不大]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;keepAliveTime:线程空闲时最大存活时间

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unit:keepAliveTime时间类型

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;BlockingQueue:用于保存等待执行的任务的阻塞队列,一般可以使用下面几种

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ArrayBlockingQueue:基于数组实现的有界阻塞队列,FIFO(先进先出)对元素进行排序

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LinkedBlockingQueue:一个基于链表实现的阻塞队列,不设置容量则为无界队列,FIFO(先进先出)对元素进行排序,吞吐量高

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SynchrousQueue:一个不存储元素的阻塞队列,每个插入元素都必须等到另一个线程调用移出操作,否则插入操作会一直阻塞[相反也是],吞吐量高

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PriorityBlockingQueue:优先级阻塞队列

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;threadFactory:用于设置创建线程的工厂

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RejectedExecutionHandler:饱和策略,当线程池中的线程数超过线程池最大数量执行的拒绝策略

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AbortPolicy:直接抛出异常

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CallerRunsPolicy:直接调用所在线程执行任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DiscardOldestPolicy:丢弃队列中最近的一个任务,并执行当前任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;DisacrdPolicy:不处理直接丢弃

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;自定义策略:实现RejectedExecutionHandler





