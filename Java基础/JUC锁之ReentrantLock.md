
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
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```
