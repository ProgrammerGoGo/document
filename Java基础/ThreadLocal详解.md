
> https://blog.csdn.net/f641385712/article/details/104573489  
> [Java 并发 - ThreadLocal详解](https://pdai.tech/md/java/thread/java-thread-x-threadlocal.html)  
> [黑马程序员-ThreadLocal](https://www.bilibili.com/video/BV1N741127FH?p=5&spm_id_from=pageDriver&vd_source=d039f8798e1b7db3c7fad9ee7b012612)


# ThreadLocal和synchronized的区别？？？
相同：ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。

不同：Synchronized同步机制采用了“以时间换空间”的方式，仅提供一份变量，让不同的线程排队访问；而ThreadLocal采用了“以空间换时间”的方式，每一个线程都提供了一份变量，因此可以同时访问而互不影响。以时间换空间: 即枷锁方式，某个区域代码或变量只有一份节省了内存，但是会形成很多线程等待现象，因此浪费了时间而节省了空间。以空间换时间: 为每一个线程提供一份变量，多开销一些内存，但是呢线程不用等待，可以一起执行而相互之间没有影响。

虽然ThreadLocal和Synchonized都用于解决多线程并发访问，ThreadLocal与synchronized还是有本质的区别。

synchronized是利用锁的机制，使变量或代码块在某一时该只能被一个线程访问。而ThreadLocal为每一个线程都提供了变量的副本，使得每个线程在某一时间访问到的并不是同一个对象，这样就隔离了多个线程对数据的数据共享。而Synchronized却正好相反，它用于在多个线程间通信时能够获得数据共享。

**即Synchronized用于线程间的数据共享，而ThreadLocal则用于线程间的数据隔离。**

# Spring事务中ThreadLocal的使用？？？

