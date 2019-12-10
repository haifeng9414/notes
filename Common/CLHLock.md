CLH指的是该算法的三位作者：Craig、Landin和Hagersten名字首字母的缩写。

CLH Lock实现了先进先出的非重入锁，JDK8的AbstractQueuedSynchronizer也参考了该算法的思想，算法简单实现：

```java
class ClhSpinLock {
    // 每个线程有一个prev引用
    private final ThreadLocal<Node> prev = new ThreadLocal<>();
    // 每个线程有一个node引用
    private final ThreadLocal<Node> node = ThreadLocal.withInitial(Node::new);
    // 初始情况下队列只有一个节点
    private final AtomicReference<Node> tail = new AtomicReference<>(new Node());

    public ClhSpinLock() {
    }

    public void lock() {
        final Node node = this.node.get();
        node.locked = true;
        // 用AtomicReference保证了多线程情况下各个线程的node按CAS的成功顺序组成队列元素，
        // tail一直指向最后一个CAS成功的线程node，getAndSet返回值是上一次CAS成功的线程node
        Node pred = this.tail.getAndSet(node);
        /*
        每个线程的prev指向上一次CAS成功的线程的node，所以多个线程竞争锁之后，将存在如下结构：

                            t1线程获得了锁  t2、t3、t4都有个prev引用指向前一个node，并自旋等待
                                |           |
                                v           v
        tail引用的初始node <-- t1.node <-- t2.node <-- t3.node <-- t4.node

         t1.node获得了锁，其locked属性为true，t2自旋等待t1的locked为false
         */
        this.prev.set(pred);
        while (pred.locked) {// 进入自旋等待
        }
    }

    public void unlock() {
        final Node node = this.node.get();
        // 释放锁，让后继停止自旋
        node.locked = false;
        // 设置更新node为前驱，这样当前线程的node和prev都指向了之前的pred，也就是当前线程不再有指
        // 向更新之前的node的引用，更新之前的node只有当前线程的后继线程的prev指向它，所有线程释放锁
        // 的时候都会执行这一个操作，这么做的好处是，观察系统初始状态可以发现，tail初始情况下有一个node，
        // 每个线程加锁时也都会创建一个自己的node，多个线程在引用链上等待时，tail总是指向最后一个node，
        // 如果每个线程都执行下面这一行语句，最后tail指向的node只有tail这一个引用，而每个线程的node和prev
        // 引用都指向自己的node，而prev指针在再次lock时会重新计算，node对象本身在不加锁的状态下属性是一样
        // 的，无所谓线程指向哪个node，从而实现了线程能重复lock/unlock
        this.node.set(this.prev.get());
    }

    private static class Node {
        private volatile boolean locked;
    }

    public static void main(String[] args) throws InterruptedException {
        final ClhSpinLock lock = new ClhSpinLock();
        lock.lock();

        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                lock.lock();
                System.out.println(Thread.currentThread().getId() + " acquired the lock!");
                lock.unlock();
            }).start();
            Thread.sleep(100);
        }

        System.out.println("main thread unlock!");
        lock.unlock();
    }
}
```