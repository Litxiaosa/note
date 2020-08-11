<img src="https://gitee.com/gean_807/typora/raw/typora/uPic/image-20200811153811679.png" alt="image-20200811153811679" style="zoom:67%;" />



解锁流程从调用`lock.unlock()`开始，`lock.unlock()`方法调用的是`sync.release(1)`方法，`sync.release(1)`的代码如下：

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    //当 tryRelease返回true 即 c == 0 时, 需要唤醒头结点的后继节点
    Node h = head;
    /**
     * 当waitStatus == -1 时说明后继节点正在park中需要unpark
     * 当waitStatus == 0时，后继节点是不会park的会在竞争一轮锁
     */
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```



`release(int arg) `方法是`sync`的父类`AbstractQueuedSynchronizer`实现的，其调用了`tryRelease(arg)`，`tryRelease(arg)`跟`tryAquire`一样，在AQS里面 没有具体实现，需要子类实现，下面贴出`ReentrantLock`的`tryRelease(arg)` 的代码：

```java
protected final boolean tryRelease(int releases) {
  // 获取本次释放锁后state的值 执行unlock时一般为 release的值为1
  int c = getState() - releases;
  //若当前线程不是拥有锁的线程，则抛出异常，只有拥有锁的线程才可以释放锁
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  boolean free = false;
  /**
   * 若本次解锁后 c 的值为0 说明该线程已经完全释放了锁
   * 若c不为0，则是在多次重入的情况下
   */
  if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
  }
  /**
   * 当c == 0时，这一步执行完，该线程就真正释放了锁，其他线程就可以拥有锁了,
   * 因为还没有执行unpark方法,还没有唤醒该线程的后继节点的线程，
   * 所以在非公平锁的情况下，未入队的线程比这等待队列里的线程更容易竞争到锁
   */
  setState(c);
  return free;
}
```



`tryRelease`后，(若锁的状态为0（即`state` == 0）&& 等待队列不为空 && `head.waitStatus != 0`) 就可以唤醒`head`的后继节点。关于`waitStatues !=0 `这个条件，这里做一下简要说明: `waitStatus ==0` 时，改节点的后继节点还没有`park`所以不需要`unpark`。下面看一下`unparkSuccessor(h)`的代码：

```java
/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
  /*
   * If status is negative (i.e., possibly needing signal) try
   * to clear in anticipation of signalling.  It is OK if this
   * fails or if status is changed by waiting thread.
   *
   */
  int ws = node.waitStatus;
  if (ws < 0)
    /**
     * 尝试将ws改为 0 ，即使失败也无所谓。 
     * 这里使用cas的方式修改是因为他的前置节点也可能修改他的waitStatus
     * 将ws更新为0的理由是让唤醒的线程可以多一轮竞争。提高竞争率
     */
    compareAndSetWaitStatus(node, ws, 0);
  /*
   * Thread to unpark is held in successor, which is normally
   * just the next node.  But if cancelled or apparently null,
   * traverse backwards from tail to find the actual
   * non-cancelled successor.
   */
  Node s = node.next;
  //当头结点的后继节点为空或者已经被取消时
  if (s == null || s.waitStatus > 0) {
    s = null;
    //从队尾开始向前遍历知道找到一个活着的可唤醒的线程
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  if (s != null)
    LockSupport.unpark(s.thread);
}
```



`unparkSuccessor`是AQS的一个私有方法，只能该类内部调用。当调用完`LockSupport.unpark(s.thread)`方法后，唤醒在`park`状态的线程`s.thread`。唤醒不等于拥有锁，因为`s.thread`被唤醒后需要通过`tryAcquire()` 方法去竞争，如果竞争失败则在竞争一次如果两次都没有竞争则继续阻塞，等着获取的线程再次唤醒自己。