
> [JUC锁之AQS详解](https://github.com/ProgrammerGoGo/document/blob/main/Java%E5%9F%BA%E7%A1%80/JUC%E9%94%81%E4%B9%8BAQS%E8%AF%A6%E8%A7%A3.md)

# ReentrantLock

## ReentrantLock的lock()方法

```java
private final Sync sync;

// ReentrantLock默认使用非公平锁
public ReentrantLock() {
        sync = new NonfairSync();
}

// 通过构造方法的入参控制使用的锁
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
}

public void lock() {
        sync.lock();
}
```

## ReentrantLock的内部类 - Sync类

Sync类继承了AQS，并且有两个类实现：  
* 公平锁：FairSync
* 非公平锁：NonfairSync

```java
// 非公平锁
final void lock() {
    // 直接cas尝试将state从0改到1，修改成功就拿到了锁资源
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    // 如果失败，就执行acquire方法，走后续流程
    else
        acquire(1);
}
// 公平锁
final void lock() {
    // 执行acquire方法，走后续流程
    acquire(1);
}
```

公平锁和非公平锁中的acquire()方法相同，都是执行父类`AbstractQueuedSynchronizer`中用`final`修饰的acquire()方法

```java
// AbstractQueuedSynchronizer类中的acquire()方法
public final void acquire(int arg) {
        // tryAcquire：尝试拿到锁
        // addWaiter：没拿到锁，准备排队
        // acquireQueued：挂起线程和后续被唤继续锁资源的逻辑
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```

## ReentrantLock内部类NonfairSync（非公平锁）的tryacquire()方法

AbstractQueuedSynchronizer抽象类中的tryacquire()方法

```java
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
}
```

由于`AbstractQueuedSynchronizer`抽象类中的tryacquire()方法的直接抛出了异常，所以必须在 AQS 实现类中重写 tryacquire() 方法。

ReentrantLock内部类NonfairSync的tryacquire()方法实现：

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程对象
    final Thread current = Thread.currentThread();
    // 获取state值，调用的是父类AQS中的方法
    int c = getState();
    // 锁没有被某个线程所持有（这里的 “某个线程” 也可能是当前线程），可以获取当前锁
    if (c == 0) {
        // 因为当前是非公平锁的tryacquire()方法，所以这里 再次 通过cas操作抢锁（将state从0改为1）
        // 注意：这里之所以说是 “再次抢锁” ，是因为在调用方法前已经通过cas抢了一次锁（调用非公平锁 NonfairSync.lock() 方法的一开始就会先抢一次锁），只是没抢到才走到了当前方法
        if (compareAndSetState(0, acquires)) {
            // 成功获取到锁，将当前线程设置为互斥锁的持有者，即将当前线程的引用赋值给 AbstractOwnableSynchronizer(AQS的父类) 类的 exclusiveOwnerThread 属性
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 锁已经被某个线程所持有，并且这个持有锁的线程就是“当前线程”（ReentrantLock是可重入锁）
    else if (current == getExclusiveOwnerThread()) {
        // 计算重入的次数
        int nextc = c + acquires;
        // 如果重入次数小于0，说明重入次数已经大于int的最大值了，直接抛异常。（Integer.MAX_VALUE + 1 即 0111 1111 1111 1111 1111 1111 1111 1111 + 1 为负数）
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 锁已经被某个线程所持有，并且这个持有锁的线程不是“当前线程”，则获取锁失败
    return false;
}
```

## ReentrantLock内部类FairSync（公平锁）的tryacquire()方法

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 锁没有被某个线程所持有（这里的 “某个线程” 也可能是当前线程），可以获取当前锁
    if (c == 0) {
        // hasQueuedPredecessors：锁资源没有被某个线程持有，需要再次判断有没有线程正在排队
        //        1、没有线程排队：直接占用锁
        //        2、有线程在排队：查看第一排队的线程是否为当前线程，如果是，直接占用锁，如果不是则排队
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## AbstractQueuedSynchronizer类中的addWaiter()方法

公平锁 **FairSync** 和非公平锁 **NonfairSync** 的`tryacquire()`方法返回false，即没有获取到锁时，就需要执行后续的`addWaiter()`方法了。

`addWaiter()`方法的实现在 `AbstractQueuedSynchronizer` 类中。

















