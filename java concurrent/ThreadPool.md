# ThreadPoolExecutor[线程池]

<font size=5>Worker:内部存放执行任务的线程以及任务[真正执行任务]</font>

线程池状态控制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用AtomicInteger变量将工作线程数量与运行状态压缩在一起[其中高3位为线程池的运行状态,低29位为当前工作线程数]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运行状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RUNNING:运行状态,可以处理新任务并执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SHUTDOWN:关闭状态,不接受新任务但是执行队列中的任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;STOP:停止状态,不接受新任务也不处理队列中的任务并且会打断当前运行中任务

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TIDYING:整理状态,所有任务结束并且此时工作线程为0会进入结束状态

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TERMINATED:结束状态





