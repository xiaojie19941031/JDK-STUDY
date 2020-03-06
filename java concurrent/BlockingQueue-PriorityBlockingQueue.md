# PriorityBlockingQueue[优先级队列]
基于二叉堆实现的优先级无界阻塞队列,默认情况下采用升序,也可以通过Comparator比较接口进行比较[只保存实现了Comparable接口的元素],内部使用了可重入锁以及Condition[可以理解为条件队列,等待/通知模式]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;private static final int DEFAULT_INITIAL_CAPACITY = 11;默认容量为11

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lock:全局锁,出队和入队都需要使用

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;notEmpty:当队列中元素为空时用于阻塞出队线程

入队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e):往队列中添加元素,如果此时队列已满进行扩容操作再添加进元素