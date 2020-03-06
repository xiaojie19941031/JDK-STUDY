# CountDownLatch
定义

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并发工具类,允许一个或多个线程等待其它线程完成操作,通过计数器实现,初始值为需要执行多少次countDown操作,每当执行一次countDown计数器值-1,当计数器值为0是表示需要等待的线程全部执行完成,此时在闭锁上等待的线程可以执行

内部实现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过AQS实现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sync:继承自AQS

应用场景

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如游戏加载画面需要等到玩家全部加载完才能展现或者等待某一段时间如果玩家还未加载完再展现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如考试,考生可以提交交卷但是监考老师必须等到考生全部交完试卷

构造方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CountDownLatch(int count):传入一个int类型数字代表需要等待多少线程

核心方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;await():调用此方法线程会被挂起,直到计数器为0是才会继续执行

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、判断当前内部的计算器是否为0,为0返回1否则返回-1

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、AQS根据返回的值如果小于0代表需要进行阻塞,将其添加入等待队列中进行自旋操作

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// AQS中的方法
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {

    if (Thread.interrupted()){
        throw new InterruptedException();
    }
    // 此方法交由子类自行实现
    if (tryAcquireShared(arg) < 0){
        doAcquireSharedInterruptibly(arg);
    }
}

// Sync判断线程是否可以执行,判断当前的计数器的值是否为0,为0执行不为0不执行
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;await(long timeout,TimeUnit unit):调用此方法线程会被挂起,直到计数器为0或者等待一定时间后计数器还不为0继续执行

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;countDown():将计数器进行-1操作

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取当前内部计数器值,如果计数器值为0直接返回false代表失败,否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、计数器值自减并通过CAS改变当前内存中的计数器值,更改成功判断是否计数器为0为0需要唤醒线程否则继续阻塞
```
public void countDown() {
    sync.releaseShared(1);
}

// AQS中的方法
public final boolean releaseShared(int arg) {

    // 此方法交由子类自行实现
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
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
```
不足

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CountDownLatch是一次性的,计数器的值只能在构造方法中初始化一次,之后没有任何机制再次对其设置值,当CountDownLatch使用完毕后,它不能再次被使用
