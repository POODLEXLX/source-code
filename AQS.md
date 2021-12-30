# AbstractQueuedSynchronizer源码解析

## 基本字段介绍

AQS，又叫作队列同步器，是构建众多同步组件的基础框架。依赖于一个原子变量来表示同步状态，通过**模板方法**，各子类继承并实现它提供的抽象方法来管理同步状态。

### 同步状态
使用volatile来修饰，默认为0，标识锁空闲；state>0表示锁被持有，可以大于1，表示同一线程可以多次获取锁，即**可重入**

**线程通过修改state>0**，修改成功即表示获取锁。
```java
private volatile int state;

```
AQS内部改变该状态通过使用Unsafe类来实现的，使用CAS乐观锁来更新状态。避免使用synchronized悲观锁
> java中不能直接访问操作系统底层，Unsafe提供了硬件级别的原子访问，通过compareAndSwapXXX比较对象偏移量内存位置上的值和期望值，来判断是否更新的。
```java
//获取state在内存中的偏移量
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long stateOffset;

static {
    try {
        stateOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("state"));
                    ...
    } catch (Exception ex) { throw new Error(ex); }
}
//利用CAS来更新state值，若当前state的值 == expect，则更新state = update；否则false 
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
## lock.lock()
本文以ReentrantLock为例来分析。ReentrantLock是一个独占锁，内部主要通过Sync类来实现公平锁和非公平锁的，默认非公平。Sync又继承了AbstractQueuedSynchronizer，所以底层是通过AQS来实现的。
`abstract static class Sync extends AbstractQueuedSynchronizer `
```java
public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```
我们先看非公平锁（NonfairSync）的lock方法。方法很好理解：先通过CAS设置同步状态的值，设置成功，则将当前锁的持有者设为自己；否则则执行acquire()方法。
AbstractOwnableSynchronizer主要是标明当前锁的持有者是哪个线程，主要用来实现独占功能的。
```java
final void lock() {
    //cas更新state值，上面分析过
    if (compareAndSetState(0, 1))
        //设置当前线程持有锁   
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
### acquire
用来获取独占锁，且忽略中断。
1. 调用tryAcquire，来尝试再次通过CAS获取锁
2. 获锁失败后将线程包装成Node节点加入到**同步队列**中
3. 阻塞线程直到线程**中断或者被唤醒**
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### tryAcquire
此方法还是比较容易理解的，区分公平锁和非公平锁实现，主要看非公平的
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    //先看当前锁是否空闲，若空闲则CAS设置state，成功则抢到锁
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //若当前锁非空闲，则判断占锁的线程是否为当前线程。是则将state+1（重入锁）
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //此时锁被当前线程占用，不会出现并发情况。所以直接设置state即可，无需通过CAS    
        setState(nextc);
        return true;
    }
    return false;
}
```
讲完获锁成功这部分的逻辑后，接下来我们来看下，获锁失败是如何处理的。

### 同步队列
AQS中主要通过一个同步双向队列来完成线程获取资源的排队工作。当线程获锁失败时，会将该线程加入到同步队列中。**线程的控制信息被保持在其上一个节点中**。

我们这边主要讨论 watiStatus = -1、0、1这3种情况。
```java
static final class Node {
	//标记是否为共享还是独占模式
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

	/**
	等待状态： waitStatus默认为0(初始化状态)
    CANCELLED-1(后面也会出现waitState>0)：节点为取消状态，即节点对应的线程中断或者超时，需要从队列中移除。且后续不再参与获锁活动
    SIGNAL--1：主要用来标明后继节点为等待状态。节点在队列中等待，必要条件为：其前继节点为SIGNAL(具体在shouldParkAfterFailedAcquire)。当前节点释放锁或者被取消时，会唤醒后继节点(具体在cancelAcquire和release)
    CONDITION--2：Condition中使用，节点在等待队列中等待被唤醒
    PROPAGATE--3：共享模式下才会使用
	**/
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus;
	//前继节点
    volatile Node prev;
	//后继节点
    volatile Node next;
	//当前节点对应的线程
    volatile Thread thread;
} 	
```
### addWaiter
同步队列中
```java
主要是新建一个node并把它加入到队尾。入参标识释放为共享锁,独占为null。
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    //如果尾结点不为空，将当前节点设置为尾结点
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
队列的