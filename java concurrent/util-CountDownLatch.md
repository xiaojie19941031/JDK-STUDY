# CountDownLatch
并发工具类,允许一个或多个线程等待其它线程完成操作,通过计数器实现,初始值为需要等待多少次countDown操作,每当一个countDown完成后计数器值-1,当计数器值为0是表示需要等待的线程全部执行完成,此时在闭锁上等待的线程可以执行,实现依赖AQS

构造方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;传入一个int类型数字代表需要等待多少线程,内部生成Sync[Sync继承自AQS]

```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

await():调用此方法线程会被挂起,直到计数器为0是才会继续执行[这里只看判断条件,真正的阻塞流程是通过AQS实现的]
```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// Sync判断线程是否可以执行,判断当前的计数器的值是否为0,为0执行不为0不执行
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// 线程阻塞的方法,AQS实现[在AQS中理解]
doAcquireSharedInterruptibly(arg);
```
await(long timeout,TimeUnit unit):调用此方法线程会被挂起,直到计数器为0或者等待一定时间后计数器还不为0继续执行
```
public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

// Sync判断线程是否可以执行,判断当前的计数器的值是否为0,为0执行不为0不执行
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// 线程等待以及阻塞的方法[AQS中理解]
doAcquireSharedNanos(arg, nanosTimeout)
```
countDown():将计数器进行-1操作
```
public void countDown() {
    sync.releaseShared(1);
}

protected boolean tryReleaseShared(int releases) {

    for (;;) {
        // 获取计数器值
        int c = getState();
        // 计数器为0直接返回false
        if (c == 0){
            return false;
        }
        // 值进行减一操作
        int nextc = c-1;
        // 通过CAS操作将计数器值与内存中进行比较如果没被其它线程改变直接将值进行替换比较
        if (compareAndSetState(c, nextc)){
            return nextc == 0;
        }
    }
}

// 真正的唤醒线程操作[AQS实现]
doReleaseShared();
```
不足

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CountDownLatch是一次性的,计数器的值只能在构造方法中初始化一次,之后没有任何机制再次对其设置值,当CountDownLatch使用完毕后,它不能再次被使用
