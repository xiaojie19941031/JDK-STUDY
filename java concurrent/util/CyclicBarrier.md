# CyclicBarrier
定义

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并发工具类,让一组线程达到一个屏障(同步点)时被阻塞,直到最后一个线程也到达屏障时屏障会打开,此时阻塞在此屏障上的线程可以继续运行[线程之间相互等待]

内部通过ReentrantLock和Condition配合实现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lock:全局锁,执行操作需要获取锁

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;trip:条件队列,阻塞同步点的线程,当所有线程执行到同步点再唤醒

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;barrierCommand:方法,指定此方法当线程全部到达同步点时优先执行此方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;count:指定需要到达同步点的线程数量

应用场景

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;多线程计算数据,最后进行数据合并[相关结果合并的操作都可以使用]

构造方法

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CyclicBarrier(int parties, Runnable barrierAction):通过指定需要达到同步点的线程数以及方法构造

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CyclicBarrier(int parties):通过指定需要达到同步点的线程数构造

await():阻塞线程,如果此时线程全部到达同步点则唤醒阻塞的线程继续执行
```
public int await() throws InterruptedException, BrokenBarrierException {
    
    try {
        // 不带超时机制执行等待
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

private int dowait(boolean timed, long nanos) throws InterruptedException, 
                                                     BrokenBarrierException,
                                                     TimeoutException {
        
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        final Generation g = generation;
        // 理解为状态机
        if (g.broken){
            throw new BrokenBarrierException();
        }
        // 线程被打断,唤醒阻塞的线程防止这些线程一直被阻塞
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        // 自减
        int index = --count;
        // 代表当前所有线程到达同步点
        if (index == 0) {
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // 优先执行传入的runnable方法
                if (command != null){
                    command.run();
                }
                ranAction = true;
                // 唤醒阻塞中的其它线程
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction){
                    breakBarrier();
                }
            }
        }
        // 无限循环直至线程都到达同步点或者被打断超时等
        for (;;) {
            try {
                // 不带超时间直接等待
                if (!timed){
                    trip.await();
                // 设置了超时时间
                } else if (nanos > 0L){
                    nanos = trip.awaitNanos(nanos);
                }
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }
            if (g.broken){
                throw new BrokenBarrierException();
            }
            if (g != generation){
                return index;
            }
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}

private void nextGeneration() {

    // 唤醒阻塞在trip上的线程
    trip.signalAll();
    // 值重新塞回
    count = parties;
    generation = new Generation();
}
```
await(long timeout, TimeUnit unit):在await的基础上增加了超时机制,如果此时线程全部到达同步点则唤醒阻塞的线程继续执行,如果超时会抛出超时异常并且唤醒其它线程

reset():重置CyclicBarrier
```
public void reset() {

    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 将当前的Cyclibarrier销毁
        breakBarrier();
        // 重置,方法上面有
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```

CyclicBarrier与CountDownLatch对比

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CountDownLatch计数器只能使用一次[内部未提供任何可以重置count的操作],CyclicBarrier带有重置的reset操作并且当唤醒线程时也会进行重新的值分配因此可以复用