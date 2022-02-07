# Condition源码分析
## 使用
首先我们通过一个例子来看下condition的使用，用法和wait非常相似，必须先要获取到锁。
```java
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notEmpty = lock.newCondition();
    final Condition notFull = lock.newCondition();
    final Object[] items = new Object[100];

    int putptr, takeptr, count;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```
## lock.newCondition()
主要是创建了一个ConditionObject
```java
public class ConditionObject implements Condition, java.io.Serializable {
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
}
```
内部也使用Node创建了一个队列（条件队列），主要用到的属性有：
```java
/** waitStatus value to indicate thread is waiting on condition */
static final int CONDITION = -2;

Node nextWaiter;
```
## condition.await()
使线程等待，知道被唤醒或者发送**中断**(抛出异常)。
线程在阻塞后，返回该方法之前，必须重新获取与该条件相关联的锁。
首先要明确的是，**在调用await方法之前，线程肯定是获得锁的**。
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //添加到条件队列的尾部
    Node node = addConditionWaiter();
    //释放当前锁，并返回当前重入数
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //判断节点是否在同步队列中，如果不在，则让其在条件队列阻塞着
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        //此处是被唤醒了或者是中断了；中断则跳出循环。
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //线程去抢锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
