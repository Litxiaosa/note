`ReentrantLock`是一个可重入的排他锁，他的加锁过程是通过`cas`操作完成的。`ReentrantLock`有三个内部类，分别是`Sync` , 	`FairSync`,`NofairSync`。


为了便于后面加锁流程的理解，先对`AbstractQueuedSynchronizer`的几个重要的属性进行简单说明（代码省略了很多注释）。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
  
  
    static final class Node {
        
        static final Node SHARED = new Node();

        /**
         * 该属性记录当前节点的状态
         */
        volatile int waitStatus;

        /**
         * 指向前置节点
         */
        volatile Node prev;

        /**
         * 指向后继节点
         */
        volatile Node next;

    		/**
    		 * 节点所代表的线程
     		 */
        volatile Thread thread;
    }
  
    /**
     *  等待队列的头结点
     */
  	private transient volatile Node head;

    /**
     * 等待队列的尾结点
     */
    private transient volatile Node tail;

    /**
     * 同步状态，多个线程进行锁竞争时，其实就是通过cas操作将 state 的值从预期值变为想要更新的值
     * cas操作就是先比较在更新(即在更新之前先获取state的值，在真正进行更新的时候先将之前获取到state的值与现在
     * 的state的值进行比较，如果相等则更新否则就更新失败)。
     */
    private volatile int state;
```





先看一下ReentrantLock的***非公平锁***加锁的流程图



<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811134619296.png" alt="image-20200811134619296"/>



**下面按照流程图结合代码来说明：**

```java
/**
 * Performs lock.  Try immediate barge, backing up to normal
 * acquire on failure.
 *
 * 执行lock()方法，首先通过cas操作去修改state的值,尝试由0变为1，
 * 如果失败则执行acquire(1)
 */
final void lock() {
    /**
     * 因为是非公平锁，所以不管等待队列里面是否有等待的线程, 直接通过cas操作 尝试将 state的值从0 更新为 1
     * 这里这个 1 还表示获取锁的次数。调用lock()方法,是线程第一次竞争锁，当获取成功,state的值为1，
     * 该线程在没有释放锁的情况下又调用了lock()方法, 则改state的值变为2 以此类推。
     * 这也说明ReentrantLock是可重入锁。
     * 这里compareAndSetState方法里面调用的是unsafe的compareAndSwapInt方法。
     */
   if (compareAndSetState(0, 1))sdsd
		 /*
		  * 若修改成功 这将持有锁的线程 置为当前线程
		  */
     setExclusiveOwnerThread(Thread.currentThread());
   else
     /**
      * 否者执行 acquire方法。注意公平锁里是没有上面if的操作的，直接执行acquire方法
      */
     acquire(1);
}
```





根据代码可以看到非公平锁在上锁时会先直接竞争锁，如果竞争成功则方法执行结束 ，进入同步代码块(即调用lock方法的线程的后面的代码)，也就是流程途中最上面的部分：

<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811143006887.png" alt="image-20200811143006887" style="zoom:67%;" />





**若竞争失败则执行acquire(1)方法：**

```java
/**
 * Acquires in exclusive mode, ignoring interrupts.  Implemented
 * by invoking at least once {@link #tryAcquire},
 * returning on success.  Otherwise the thread is queued, possibly
 * repeatedly blocking and unblocking, invoking {@link
 * #tryAcquire} until success.  This method can be used
 * to implement method {@link Lock#lock}.
 *
 * 以独占的方式获取锁，忽略中断。返回成功时至少调用一次 {@link #tryAcquire}，
 * 若tryAcquire失败，当前线程入队，会一直调用{@link #tryAcquire}直到成功获取到锁
 * 该方法通常被{@link Lock#lock}调用。
 */
public final void acquire(int arg) {
  /**
   * 当尝试获取锁失败的时候则将当前线程保存到一个节点里以独占的模式添加到等待
   */
  if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```





由上代码可知，`acquire`方法先`tryAcquire` ，当`tryAcquire`失败时才执行入队操作即`addWaiter`方法。下面贴出`tryAcquire`方法的代码:

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
 }
```





**继续看 `nonfairTryAcquire`方法：**

```java
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 *
 * 非公平的tryLock实现。
 */
final boolean nonfairTryAcquire(int acquires) {
  //获取到当前线程
  final Thread current = Thread.currentThread();
  
  //先获取state的值，记录他的初始状态
  int c = getState();
  
 //如果 c == 0 则说明在上一步获取state时没有线程竞争到锁，可以去竞争锁(即通过cas操作将state 由 0 变为 1）
  if (c == 0) {
    //过cas操作将state 由 0 变为 1, 该操作保证只有一个线程可以操作成功
    if (compareAndSetState(0, acquires)) {
      //如果成功 则state的值已经变为 1, 将当前线程置为拥有锁的线程,这里的代码只有一个线程可以执行到
      setExclusiveOwnerThread(current);
      //返回true 该线程的加锁流程执行完毕
      return true;
    }
  }
  
 // 如果 c != 0 然后判断 当前线程是否是拥有锁的线程
  else if (current == getExclusiveOwnerThread()) {
    /**
     * 如果是的话 则将 将state的值加上方法的参数(acquires)，
     * 这里是处理同一个线程在没有释放锁的情况下多次获取锁的过程(即锁的重入)，
     * state记录的就是重入的次数
     */
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    //这里设置state的值的时候没有用cas操作，因为能执行到这块代码,的只能是是拥有锁的线程,不存在竞争所以没有必要使用cas操作
    setState(nextc);
    return true;
  }
  //当state的值不为0而且当前线程也不是拥有锁的线程则返回false
  return false;
}
```



从`nonfairTryAcquire`方法可知 两点：

- 非公平锁去获取锁的时候是不管等待队列的，最新竞争的线程很有可能比这它前面的线程先获取锁。

- `tryAcquire`方法里面处理了锁重入的逻辑，`state`的值记录了重入的次数。



**整个tryAcquire的执行流程对应如下图(从执行tryAcquire()开始)：**

![image-20200811143218219](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811143218219.png)





从流程图可以轻易的发现，这个方法没有自旋的情况，要么成功要么失败。

若`tryAcquire`失败则执行入队操作`addWaiter(Node.EXCLUSIVE)`（可以看上面`acquire`方法）。



**下面是addWaiter(Node.EXCLUSIVE)代码：**

```java
 /**
	* Creates and enqueues node for current thread and given mode.
	* 以当前线程和指定的模式创建节点并且入队
	*
	* @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
	* @return the new node
	*/
private Node addWaiter(Node mode) {
  Node node = new Node(Thread.currentThread(), mode);
  /**
   * Try the fast path of enq; backup to full enq on failure
   * 先尝试快速将节点插入队尾。若果失败则执行完整的入队方法。
   * 问题 1、为什么有这个先尝试入队的操作？
   * 这里先尝试将当前node加入队尾，是因为这段代码的执行成功的概率是很高的，
   * 所以不用每次都创建循环，这样jvm不需要创建循环，提高效率
   */
  Node pred = tail;
  if (pred != null) {
    node.prev = pred;
    // 通过cas操作确定当前线程是否可以更新队尾的next指针指向当前的node
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  /**
   * 如果上面的代码没有走到return则进入enq方法, 这里需要注意的是 并没有return enq(node)
   * 问题 2、为什么没有return enq(node);
   * 因为enq(node)方法返回的是队尾的前置节点
   */
  enq(node);
  return node;
}
```



**下面看enq(node)方法**

```java
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * 在队列里插入节点，如果队列为空先初始化
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
  // 用死循环来保证当前节点一定能插入到队尾
  for (;;) {
    //获取当前尾结点, 此时可能有其他线程也执行到这一步
    Node t = tail;
    //如果尾结点为空说明队列还没有初始化 先初始化,此时可能有其他线程也执行到这一步
    if (t == null) { // Must initialize
      /**
       * 这里通过cas操作初始化队列的head,此时可能有其他线程也执行到这一步,
       * 所以使用cas 操作进行队列初始化，保证队列只初始化一次。
       * 假如当先线程初始化失败，则会又进入循环最开部分
       * 在判断就不会为空了,还要注意的，并不是把当前的node作为head，而是新创建的node
       */
      if (compareAndSetHead(new Node()))
        // 这里tail = head tail就不为空
        tail = head;
    } else {
      // 这里可能会有多个未加入队尾的node指向队列的尾结点，但是不影响
      node.prev = t;
      /**
       * 通过cas操作将node加入到队尾，代码compareAndSetTail(t, node)实际的操作是将锁的 队尾偏移量
       * 从原来指向t 变为指向 node 。只执行完这句话 在java层面这个入队操作
       * 还没完，还需要执行 t.next = node; 即下面的代码。因为等待队列是一个双向队列。
       */
      if (compareAndSetTail(t, node)) {
        // 这句代码执行玩才算入队操作完成
        t.next = node;
        // 注意这里并没有返回 node 而是node前置节点，也就是旧的队尾的节点。

        return t;
      }
    }
  }
}
```



**对应流程图的部分为：**

![image-20200811143425748](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811143425748.png)



`addWaiter`方法会有一个小小的自旋，那就是入队操作。当节点入队成功后会执行 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`，`acquireQueued`方法通过循环（自旋）和`LockSupport.park()`完成线程在`lock()`方法上的等待，下面看`acquieQueued`方法的代码：

```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 * 队列里的线程会不停的以独占模式去获取锁(即cas操作返回成功)。
 * condition wait与acquire都会调用这个方法。
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    //记录线程中断标志
    boolean interrupted = false;
    //开始自循环
    for (;;) {
      //获取节点的前置节点，当前节点的前置节点为空时,会抛出空指针异常
      final Node p = node.predecessor();
      //当前置节点为头结点 并且 获取到了锁( 成功修改了锁的 state )
      if (p == head && tryAcquire(arg)) {
        /**
         * 将当前节点设置为头结点
         * 具体的做法是将 node
         *      head = node
         *      node.thread = null;
         *      node.prev = null
         * 这里当前线程已经出队了，node作为新的head节点已经不保存线程的信息了。
         */
        setHead(node);
        p.next = null; // help GC
        failed = false;
        /**
         * 返回中断标志，这里需要注意并没有返回true，因为本方法能执行完
         * 就说明该线程已经成功获取锁。返回中断标志是为了让acquire方法响应中断。
         */
        return interrupted;
      }
      /**
       * 若当前节点的前置节点不是head节点，或者竞争锁失败 进入shouldParkAfterFailedAcquire方法，
       * 该方法见名知意获取锁失败时是否要park线程
       */
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    /**
     * 当程序正常执行的话，是一定 执行过 failed = false;
     * 在退出循环然后执行 finally 代码块，此时 failed == false, cancelAcquire(node)是执行不到的，
     * 当for循环里遇到异常时没有走到 failed = false; 就退出循环才会执行到cancelAcquire(node)
     */
    if (failed)
      cancelAcquire(node);
  }
}
```



在`acquireQueued`方法里，正常情况下只有当 `p == head && tryAcquire(arg) `的时候才能跳出循环。若没有执行

```java
if (shouldParkAfterFailedAcquire(p, node) &&parkAndCheckInterrupt())
     interrupted = true;
```



这块代码，那么当前节点只能等到他的前辈们一个一个的出队了才能轮到他。在等待的这段时间，当前线程也没有闲着，会一直循环的问cup我是不是可以出队了。这样似乎也能完成加锁的过程。如果线程少还行，cup可能会有耐心和那个时间，但是如果在高并发的情况下，cup就必须咋百忙中去回答你。对应术语上来说就是太占用cpu资源了。所以 `shouldParkAfterFailedAcquire`方法就是处理这种情况的，下面看代码：



```java
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 *
 * 检查更新失败acquire的节点的status。如果线程需要阻塞则返回true。
 * acquire的循环主要有他来控制(比如替换前置节点或者前置节点的waiteStatus)。
 * 要求pred = node.prev（指的是这个方法的参数的关系）
 *
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  /**
   * 获取前置节点的waitSatue 下面是 waitStatue的几个值所代表的的意思
   * SIGNAL：    -1, 该节点的后继节点可以park()
   * CANCELLED:   1 ，该节点已经被取消，如果某个节点的前置节点的waitStatue则跳过
   * CONDITION:  -1, 该节点在条件队列里才会出现这个值，一般在等待队列是不会出现的
   * 0:          不是上述的任何一个。
   */
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    /*
     * This node has already set status asking a release
     * to signal it, so it can safely park.
     * 该线程可以执行LockSupport.park()方法。
     * 就是意味着该线程已经知道不该我执行呢，在队列里面等这前面的节点叫你吧
     * 通过LockSupport.unpark()方法唤醒
     */
    return true;
  if (ws > 0) {
    /*
     * Predecessor was cancelled. Skip over predecessors and
     * indicate retry.
     * 当ws > 0 时 说明该节点的前置节点已经被取消了但是还没有出队，
     * 这个前置节点已经不能叫醒你了(这个前置节点里的线程已经死了不能执行唤醒操作了)，
     * 所以该节点需要向前找一个可以叫醒自己的节点，他会一直问向前问直到前面的某个节点
     * 可以叫醒自己。对应waitStatus <= 0
     */
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    /*
     * waitStatus must be 0 or PROPAGATE.  Indicate that we
     * need a signal, but don't park yet.  Caller will need to
     * retry to make sure it cannot acquire before parking.
     * 当前置节点的waitStatus 为 0 或者 PROPAGATE 时 说明我们需要一个信号，
     * 但是还不需要park呢，这时候我们需要等一轮 看看前置节点是否释放锁，
     * 这时候先把前置节点的waitStatuus设置为Node.SIGNAL 如果没有获取到锁
     * 那么就可以park等着了
     */
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```



**shouldParkAfterFailedAcquire做了三件事**

- 若`pred.waitStatue = -1` , 则返回`true`,进入`park`阻塞状态
- 若`pred.waitStatue = 1` ,剔除`pred`，并且向前找到，并且将自己连接到一个非取消的线程节点上 ，返回`false`
- 若`pred.waitStatue = 0  or pred.waitStatue = -2`，则将`pred.waitStatue` 置为 -1 ，返回`false`

当shouldParkAfterFailedAcquire返回false时会进入下一次循环，

当shouldParkAfterFailedAcquire返回true时，调用parkAndCheckInterrupt方法进入阻塞状态：

```java
/**
 * Convenience method to park and then check if interrupted
 * 让当前线程阻塞，返回中断状态
 * @return {@code true} if interrupted
 */
private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  return Thread.interrupted();
}
```



**LockSupport.park(this)的实现如下**

```java
public static void park(Object blocker) {
  Thread t = Thread.currentThread();
  /**
   * setBlocker的调用了UNSAFE.putObject(t, parkBlockerOffset, arg);
   * 该方法是将当前线程的 parkBlock偏移量置为blocker
   */
  setBlocker(t, blocker);
  // 进入阻塞状态 等待被唤醒 等着其他线程执行 LockSupport.unpark(t)
  UNSAFE.park(false, 0L);
  // 当被唤醒后先把parkBlock偏移量置为null,因为已经不被阻塞了所以需要移除blocker
  setBlocker(t, null);
}
```



当线程从阻塞状态被唤醒后会返回线程的 中断状态 return Thread.interrupted()。对应流程图的最后部分：



![image-20200811150042959](https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811150042959.png)



到这里在正常的情况下(没有异常的情况下)整个上锁流程算是走完了。 



**稍微总结一下**：从lock.lock()开始:

>  1：通过cas操作获取锁若成功，方法执行结束，否则进入第2步

> 2:  若state == 0 尝试获取锁，成功则方法结束，否则进入第3步，

> 若（state != 0 && 当先线程 == 获取锁的线程） 则 state++ 方法结 束，否则进入第3步

> 3:  将线程包装成一个node 进入到等待队列

> 4:  若 前置节点 == head && tryAcquire(1) == true 方法结束 返回中断状态，否则进入第5步。

> 5:  若pred.waitStatue = -1 , 则返回true, 进入park阻塞状态。

> 若pred.waitStatue = 1 ,剔除pred，并且向前找到，并且将自己连接到一个非取消的线程节点上 ，返回false，

> 若pred.waitStatue = 0 or pred.waitStatue = -2，则将pred.waitStatue 置为 -1 ，返回false. 

> 第五步执行完毕后返回第四步。