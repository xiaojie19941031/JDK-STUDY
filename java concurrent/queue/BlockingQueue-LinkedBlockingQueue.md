# LinkedBlockingQueue
基于链表实现的阻塞队列,在未指定具体容量的情况下,此队列为无界队列[实际使用中最好指定容量避免造成内存泄露],内部使用了可重入锁以及Condition[可以理解为条件队列,等待/通知模式]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;putLock:入队时获取 -------->	notFull:代表队列里元素未存满可以进行入队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;takeLock:出队时获取 -------->	notEmpty:代表队列里存在元素可以去进行移出操作

入队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;put(E e):将指定元素添加入队列尾部,如果队列满了直接阻塞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取putLock,判断当前队列是否已满,如果已满则阻塞当前的入队线程,否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、将元素添加进队列,如果此时队列未满,唤醒阻塞在notFull上的入队线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、如果添加元素前队列是空的[代表可能有出队线程阻塞在notEmpty上],添加完元素后唤醒阻塞在notEmpty上的出队线程[添加后可以有元素被获取]

```
public void put(E e) throws InterruptedException {

    if (e == null){
        throw new NullPointerException();
    }
    int c = -1;
    Node<E> node = new Node<E>(e);
    // 获取putLock
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 可以响应中断
    putLock.lockInterruptibly();
    try {
        // 当队列满时,入队线程阻塞在notFull上
        while (count.get() == capacity) {
            notFull.await();
        }
        // 入队操作
        enqueue(node);
        c = count.getAndIncrement();
        // 如果此时添加完成队列还是未满的,唤醒阻塞在notFull上的入队线程
        if (c + 1 < capacity){
            notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    // 如果添加前队列是空的[代表有出队队列阻塞在notEmpty上],唤醒阻塞在notEmpty上的出队线程
    if (c == 0){
        signalNotEmpty();
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e):和put操作相似,只是当队列满时不阻塞直接返回false

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;offer(E e,long timeout,TimeUnit unit):在offer(E e)基础上添加了超时操作,当队列满时阻塞timeout时间,如果超过timeout没被唤醒线程直接自动取消

出队操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;take():从队列中获取头节点进行返回,如果队列为空阻塞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取takeLock,判断当前队列中是否存在元素,如果为空阻塞当前的出队线程,否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、出队操作,判断此时的队列中是否还存在元素存在则唤醒阻塞在notEmpty上的出队线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、如果出队操作前队列已满,出队后唤醒阻塞在notFull上的入队线程[出队前满,出队后则队列可以进行添加操作]

```
public E take() throws InterruptedException {
        
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    // 获取takeLock
    final ReentrantLock takeLock = this.takeLock;
    // 可以响应中断
    takeLock.lockInterruptibly();
    try {
        // 如果队列为空,阻塞当前的出队线程
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 出队
        x = dequeue();
        c = count.getAndDecrement();
        // 如果出队前的队列元素比1大[表示这次出队后还有元素],唤醒阻塞在notEmpty上的出队队列
        if (c > 1){
            notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    // 如果出队操作前队列是满的[表示当前队列是满的,可能有入队线程阻塞在notFull上],出队后唤醒阻塞在notFull上的线程
    if (c == capacity){
        signalNotFull();
    }
    return x;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll():和take操作相似,只是当队列为空时不阻塞直接返回null

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;poll(long timeout,TimeUnit unit):在poll()基础上添加了超时操作,当队列为空时阻塞timeout,如果超过timeout没被唤醒线程直接自动取消

设计优点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;内部通过两把锁分别控制出队和入队,入队和出队不用同时争夺一把锁,这样可以大大的提高吞吐量