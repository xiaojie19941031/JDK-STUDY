# ArrayBlockingQueue
基于数组实现的有界阻塞队列,内部使用可重入锁以及Condition[可以理解为条件队列用于线程之间的协作实现等待/通知模式]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lock:全局锁,入队出队时需要获取[分为公平锁和非公平锁,公平锁下遵循FIFO先进先出原则]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;notEmpty:阻塞取元素take线程条件队列

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;notFull:阻塞添加元素put线程条件队列

入队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e):将指定元素添加入队列尾部,如果队列满了直接返回false

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取全局锁lock,如果此时队列是满状态直接false,否则执行下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、将元素添加进队列中,并且唤醒阻塞在notEmpty上的出队操作[此时队列中存在元素,直接唤醒阻塞在notEmpty的出队线程]
```
public boolean offer(E e) {
        
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获取全局锁[此锁不能响应中断]
    lock.lock();
    try {
        // 队列满直接返回false
        if (count == items.length){
            return false;
        } else {
            // 入队
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
        			
    final Object[] items = this.items;
    items[putIndex] = x;
    // 循环数组,表示当下标到最大时从头开始
    if (++putIndex == items.length){
        putIndex = 0;
    }
    count++;
    // 唤醒阻塞在notEmpty上的出队操作
    notEmpty.signal();
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;put(E e):在offer的基础上添加了阻塞操作[当队列满时通过notFull.await()阻塞后面的入队线程]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e,long timeout,TimeUnit unit):在offer的基础上添加了阻塞和超时机制[当队列满时通过notFull.await(long nanosTimeout)阻塞等待nanosTimeout,超过后线程自动取消]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;add(E e):同样不阻塞,只不过此方法如果队列满了会直接抛出异常

出队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll():从队列中获取元素,如果此时队列为空直接返回null

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取全局锁,如果当前队列为空直接返回null,否则执行下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、从队列中获取元素,同时唤醒阻塞在notFull上的入队线程[此时队列中有元素被取出,队列可以添加元素,直接唤醒阻塞在notFull上的入队线程]
```
public E poll() {
        
    // 获取全局锁
    final ReentrantLock lock = this.lock;
    // 此锁不响应中断
    lock.lock();
    try {
        // 如果队列为空直接返回null
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
        
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    // 循环数组,当下标到达最大值直接变为0
    if (++takeIndex == items.length){
        takeIndex = 0;
    }
    count--;
    if (itrs != null){
        itrs.elementDequeued();
    }
    // 唤醒阻塞在notFull上的入队线程
    notFull.signal();
    return x;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;take():在poll的基础上添加了阻塞功能[当队列为空时会通过调用notEmpty.await()进行阻塞]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll(long timeout, TimeUnit unit):在poll的基础上添加了阻塞和超时操作[当队列为空时会通过notEmpty(long timeout)阻塞timeout时间,超过timeout线程自动取消]

设计缺点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;内部只使用了一把全局锁,入队和出队都必须去争夺这把锁[资源竞争激励,吞吐量叶大大下降]