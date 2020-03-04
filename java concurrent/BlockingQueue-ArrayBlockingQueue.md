# ArrayBlockingQueue[分为公平(FIFO)和非公平模式]
基于数组实现的有界阻塞队列,内部定义了一把锁并根据同一把锁定义了两个Condition[可以理解为等待队列]用于线程之间的协作[等待/通知模式]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lock:用于往队列中添加或者取出元素时需要获取[全局锁,入队和出队都需要此锁,分为公平锁和非公平锁]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;notEmpty:阻塞取元素的take线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;notFull:阻塞添加元素的put线程

入队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e):将指定元素添加入队列尾部,如果队列满了直接返回false

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取全局锁lock,如果此时队列是满状态直接false,否则执行下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、将元素添加进队列中,并且唤醒阻塞在notEmpty上的出队操作[因为此时队列中肯定存在元素,此时如果有出队线程阻塞直接唤醒]
```
public boolean offer(E e) {
        
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获取全局锁
    lock.lock();
    try {
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
    // 循环数组
    if (++putIndex == items.length){
        putIndex = 0;
    }
    count++;
    // 唤醒阻塞在notEmpty上的出队操作
    notEmpty.signal();
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;put(E e):原理和offer差不多,在offer的基础上添加了阻塞操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e,long timeout,TimeUnit unit):原理和offer差不多,添加了阻塞和超时机制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;add(E e):原理跟offer差不多,当队列满了直接抛异常

出队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll():从队列中获取元素,如果此时队列为空直接返回null

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取全局锁,如果当前队列为空直接返回null,否则执行下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、从队列中获取元素,同时唤醒阻塞在notFull上的入队线程
```
public E poll() {
        
    // 获取全局锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
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
    // 循环数组
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
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;take():原理跟poll差不多,当队列为空时会阻塞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll(long timeout, TimeUnit unit):原理跟poll差不多,添加了阻塞和超时操作

设计缺点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;入队和出队使用同一把锁,这样出队和入队之间存在资源竞争吞吐量就会明显下降