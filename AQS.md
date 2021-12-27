# AbstractQueuedSynchronizer源码解析

## 基本字段介绍

AQS，又叫作队列同步器，是构建众多同步组件的基础框架。依赖于一个原子变量来表示同步状态，通过**模板方法**，各子类继承并实现它提供的抽象方法来管理同步状态。

### 同步状态
使用volatile来修饰，默认为0，标识锁空闲；state>0表示锁被持有，可以大于1，表示同一线程可以多次获取锁，即**可重入**
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
//利用CAS来更新state值
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
### 同步队列
```java
static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

  
    volatile int waitStatus;

    volatile Node prev;

    volatile Node next;

    volatile Thread thread;


    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
} 	
```
