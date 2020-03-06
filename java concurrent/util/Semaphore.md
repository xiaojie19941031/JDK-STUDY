# Semaphore
定义

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并发工具类,用于控制同时访问特定资源的线程数量,通过协调各个线程以保证合理的使用公共资源

内部实现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;基于AQS共享锁实现

应用场景[举例]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;日常生活中的停车场为例子,比如停车场支持10个车位[太小了],如果一次性来了8辆车,看门人可以全部放进去,因为车库可以停的下,如果此时又来了4辆车,看门人一看8辆车都没走,此时车库只够停两辆,就会放其中的两辆进去另外的两辆在外等待,此时车库有一辆车走了看门人就再放一辆车进来

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个例子可以看出车库可以看成一个共享资源,车可以看成线程,看门人则看成Semaphore,而看门人的职责就是保证最多10辆车停在车库

构造方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Semaphore(int permits):指定最多能同时访问特定资源的线程数,同时指定为非公平模式
```
public Semaphore(int permits) {
    
    // 非公平模式
    sync = new NonfairSync(permits);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Semaphore(int permits, boolean fair):指定最多能同时访问特定资源的线程数,同时根据fair值指定为公平模式还是非公平模式
```
public Semaphore(int permits, boolean fair) {

    // 根据fair参数指定为公平锁还是非公平锁模式
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
核心方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;acquire():获取许可证,如果没有会一直阻塞到获取或者被中断

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、判断当前线程是否被打断,被打断则抛出InterruptedException异常否则执行下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、非公平模式下获取当前许可证数量减去需要的许可证数量,如果剩余许可证小于0或者CAS操作成功的更改了当前内存中可用许可证返回,否则无限循环[公平模式下会多一步操作,先判断是否有线程等待时间比当前线程长,有直接返回获取共享锁失败]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、判断是否获取到了共享锁,获取到了线程直接运行,否则加入等待队列进入自旋状态去获取同步锁,获取到了执行否则一直阻塞
```
public void acquire() throws InterruptedException {

    // 可中断获取许可证[共享模式]
    sync.acquireSharedInterruptibly(1);
}

// AQS中的方法
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {

    // 线程被中断抛异常
    if (Thread.interrupted()){
        throw new InterruptedException();
    }
    // 此方法由继承类实现[小于0代表获取共享锁失败]
    if (tryAcquireShared(arg) < 0){
        // 处理未获取到同步状态的线程
        doAcquireSharedInterruptibly(arg);
    }
}

// 内部非公平锁实现
protected int tryAcquireShared(int acquires) {
        
    return nonfairTryAcquireShared(acquires);
}

// 非公平锁下实现
final int nonfairTryAcquireShared(int acquires) {
           
    // CAS操作基本套路无限循环 
    for (;;) {
        // 获取当前还有的许可证
        int available = getState();
        // 获取剩余的许可证
        int remaining = available - acquires;
        // 小于0代表许可证不够,或者CAS操作改变当前可获取的许可证数量成功
        if (remaining < 0 || compareAndSetState(available, remaining)){
            return remaining;
        }
    }
}

// 公平模式下获取锁
protected int tryAcquireShared(int acquires) {

    for (;;) {
        // AQS方法查看是否有线程比当前线程等待时间长
        if (hasQueuedPredecessors()){
            return -1;
        }
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 || compareAndSetState(available, remaining)){
            return remaining;
        }
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;acquireUninterruptibly():获取许可证,如果没有会一直阻塞[实现上和acquire差不多,只是不再响应中断]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;acquire(int permits):获取指定permits个许可证,如果没有会一直阻塞到获取或者被中断

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;acquireUninterruptibly(int permits):获取指定permits,如果没有会一直阻塞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tryAcquire():尝试去获取共享锁,获取到了返回true,未获取到直接返回false,不阻塞线程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tryAcquire(long timeout, TimeUnit unit):在指定时间内去获取共享锁,获取到了返回true,否在会在超时时间内自旋获取共享锁,超过超时间返回false

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tryAcquire(int permits):尝试去获取permits个许可证,如果获取到了直接返回true,未获取到返回false不阻塞

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tryAcquire(int permits, long timeout, TimeUnit unit):在指定时间内去获取permits个许可证,获取到了返回true,否在会在超时时间内自旋获取共享锁,超过超时间返回false

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;release():释放许可证

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;原理

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、获取当前许可证数并加上释放后的许可证数与当前对比,如果小代表数据出问题了抛异常否则进入下一步

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、通过CAS方式更改内存中的许可证数,更改成功代表释放共享锁,释放由AQS实现
```
public void release() {
    sync.releaseShared(1);
}

// AQS中的方法
public final boolean releaseShared(int arg) {
    
    // 判断释放需要释放共享锁
    if (tryReleaseShared(arg)) {
        // 释放共享锁
        doReleaseShared();
        return true;
    }
    return false;
}

// 由子类自行实现
protected final boolean tryReleaseShared(int releases) {

    for (;;) {
        // 获取当前的许可证数
        int current = getState();
        // 这是释放后的
        int next = current + releases;
        // 释放后比当前小抛异常
        if (next < current){
            throw new Error("Maximum permit count exceeded");
        }
        // 通过CAS改变,成功就返回true,不然一直自旋
        if (compareAndSetState(current, next)){
            return true;
        }
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;release(int permits):释放固定permits数量的共享锁

优点

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;内部通过大量的自旋CAS操作,相比使用锁效率上会提高