闭锁可以初始化一个数量，所有在该闭锁上调用await方法的线程都会阻塞直到该闭锁上的countDown方法被调用指定次数，闭锁只能使用一次，使用例子：[闭锁-CountDownLatch](../../并发/闭锁-CountDownLatch.md)

实现原理是利用ReentrantLock，每当调用cyclicBarrier.await()方法时先获取独占锁，以此来线程安全的维护栅栏状态，每当一个线程调用cyclicBarrier.await()方法后都会将栅栏的count - 1，表示需要等待的线程数量减1，当某个线程发现需要等待的数量等于0时调用从ReentrantLock对象创建的Condition对象的signalAll方法唤醒所有Condition链表上被阻塞线程并重置count，之后该线程释放独占锁继续运行。被唤醒的线程都会被添加到AQS的Sync队列中尝试获取锁，每次只有一个线程能获取独占锁，在获取独占锁之后线程从之前阻塞的位置继续执行并释放独占锁，这样其他线程也陆续获取锁并释放，实现所有线程都继续运行。如果线程调用cyclicBarrier.await()方法时发现将等待的数量减1后不为0，则调用Condition对象的await方法阻塞，Condition对象的await方法会释放线程的独占锁，将其放到Condition对象的等待队列并park线程。CyclicBarrier的主要逻辑是doAwait方法，下面是源码分析

```java
package dhf.utils;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;

/*
闭锁可以初始化一个数量，所有在该闭锁上调用await方法的线程都会阻塞直到该闭锁上的countDown方法被调用指定次数，闭锁只能使用一次
 */
public class CountDownLatch {
    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        // 闭锁在状态为0的情况下放过所有线程，否则阻塞所有线程
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        // 当前状态 - 1
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                // 如果c已经等于0返回false，以防止doReleaseShared方法被多次调用（看AQS的releaseShared方法）
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    /**
     * Constructs a {@code CountDownLatch} initialized with the given count.
     *
     * @param count the number of times {@link #countDown} must be invoked
     *        before threads can pass through {@link #await}
     * @throws IllegalArgumentException if {@code count} is negative
     */
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    /**
     * Causes the current thread to wait until the latch has counted down to
     * zero, unless the thread is {@linkplain Thread#interrupt interrupted}.
     *
     * <p>If the current count is zero then this method returns immediately.
     *
     * <p>If the current count is greater than zero then the current
     * thread becomes disabled for thread scheduling purposes and lies
     * dormant until one of two things happen:
     * <ul>
     * <li>The count reaches zero due to invocations of the
     * {@link #countDown} method; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * @throws InterruptedException if the current thread is interrupted
     *         while waiting
     */
    // 尝试获取锁，即等待闭锁的数量到0
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * Causes the current thread to wait until the latch has counted down to
     * zero, unless the thread is {@linkplain Thread#interrupt interrupted},
     * or the specified waiting time elapses.
     *
     * <p>If the current count is zero then this method returns immediately
     * with the value {@code true}.
     *
     * <p>If the current count is greater than zero then the current
     * thread becomes disabled for thread scheduling purposes and lies
     * dormant until one of three things happen:
     * <ul>
     * <li>The count reaches zero due to invocations of the
     * {@link #countDown} method; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread; or
     * <li>The specified waiting time elapses.
     * </ul>
     *
     * <p>If the count reaches zero then the method returns with the
     * value {@code true}.
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * <p>If the specified waiting time elapses then the value {@code false}
     * is returned.  If the time is less than or equal to zero, the method
     * will not wait at all.
     *
     * @param timeout the maximum time to wait
     * @param unit the time unit of the {@code timeout} argument
     * @return {@code true} if the count reached zero and {@code false}
     *         if the waiting time elapsed before the count reached zero
     * @throws InterruptedException if the current thread is interrupted
     *         while waiting
     */
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * Decrements the count of the latch, releasing all waiting threads if
     * the count reaches zero.
     *
     * <p>If the current count is greater than zero then it is decremented.
     * If the new count is zero then all waiting threads are re-enabled for
     * thread scheduling purposes.
     *
     * <p>If the current count equals zero then nothing happens.
     */
    // 调用tryReleaseShared使得闭锁状态 - 1，如果cas状态到0后唤醒头结点的后继，被唤醒的后继会继续执行doAcquireSharedInterruptibly方法，
    // 该方法在线程获取到锁后会调用setHeadAndPropagate方法继续唤醒后面的线程，这也是AQS里共享锁的作用
    public void countDown() {
        sync.releaseShared(1);
    }

    /**
     * Returns the current count.
     *
     * <p>This method is typically used for debugging and testing purposes.
     *
     * @return the current count
     */
    public long getCount() {
        return sync.getCount();
    }

    /**
     * Returns a string identifying this latch, as well as its state.
     * The state, in brackets, includes the String {@code "Count ="}
     * followed by the current count.
     *
     * @return a string identifying this latch, as well as its state
     */
    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```