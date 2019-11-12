线程安全的，但是是弱一致的（体现在get、clear方法和迭代器的实现），实现逻辑是，维护一个Segment数组，该数组的大小在ConcurrentHashMap创建完后久不变了，Segment类继承自ReentrantLock，同时Segment类和HashMap实现又有点像，内部有一个HashEntry数组（默认大小是2），HashEntry是个单链表结点，ConcurrentHashMap在put时先获取key对应的Segment的锁，再用HashMap类似的操作逻辑操作Segment，把数据放到Segment的HashEntry中，即分段锁，如果多线程环境下多个线程执行的key在不同的Segment，则分别获取自己对应的key的锁，互不影响（除非调用了size等需要锁住所有Segment的方法，size也不总是获取所有的Segment锁，可以看size的实现）

注意点：
get、clear方法没有加锁，直接遍历所有的Segment，所以是弱一致的
isEmpty方法中没有加锁，采用的是循环一次判断是否存在非空Segment，如果都为空则记住modCount，再循环一次，如果再次为空并且两次modCount相等，则返回true
size方法也是循环多次（默认两次）计算modCount是否相等，不想等并超出循环次数则获取Segment锁并再次计算size
containsValue的思路和size是一样的

```java
package dhf;

import java.io.IOException;
import java.io.Serializable;
import java.util.*;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.locks.ReentrantLock;

public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
    /**
     * The default initial capacity for this table,
     * used when not otherwise specified in a constructor.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    /*
     * The basic strategy is to subdivide the table among Segments,
     * each of which itself is a concurrently readable hash table. To
     * reduce footprint, all but one segments are constructed only
     * when first needed (see ensureSegment). To maintain visibility
     * in the presence of lazy construction, accesses to segments as
     * well as elements of segment's table must use volatile access,
     * which is done via Unsafe within methods segmentAt etc
     * below. These provide the functionality of AtomicReferenceArrays
     * but reduce the levels of indirection. Additionally,
     * volatile-writes of table elements and entry "next" fields
     * within locked operations use the cheaper "lazySet" forms of
     * writes (via putOrderedObject) because these writes are always
     * followed by lock releases that maintain sequential consistency
     * of table updates.
     *
     * Historical note: The previous version of this class relied
     * heavily on "final" fields, which avoided some volatile reads at
     * the expense of a large initial footprint. Some remnants of
     * that design (including forced construction of segment 0) exist
     * to ensure serialization compatibility.
     */

    /* ---------------- Constants -------------- */
    /**
     * The default load factor for this table, used when not
     * otherwise specified in a constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    /**
     * The default concurrency level for this table, used when not
     * otherwise specified in a constructor.
     */
    // 默认并发级别
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    /**
     * The maximum capacity, used if a higher value is implicitly
     * specified by either of the constructors with arguments. MUST
     * be a power of two <= 1<<30 to ensure that entries are indexable
     * using ints.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
    /**
     * The minimum capacity for per-segment tables. Must be a power
     * of two, at least two to avoid immediate resizing on next use
     * after lazy construction.
     */
    static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
    /**
     * The maximum number of segments to allow; used to bound
     * constructor arguments. Must be power of two less than 1 << 24.
     */
    static final int MAX_SEGMENTS = 1 << 16; // slightly conservative
    /**
     * Number of unsynchronized retries in size and containsValue
     * methods before resorting to locking. This is used to avoid
     * unbounded retries if tables undergo continuous modification
     * which would make it impossible to obtain an accurate result.
     */
    static final int RETRIES_BEFORE_LOCK = 2;
    private static final long serialVersionUID = 7249069246763182397L;

    /* ---------------- Fields -------------- */
    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long SBASE;
    private static final int SSHIFT;
    private static final long TBASE;
    private static final int TSHIFT;
    private static final long HASHSEED_OFFSET;

    static {
        int ss, ts;
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class tc = HashEntry[].class;
            Class sc = Segment[].class;
            // arrayBaseOffset能获得数组在内存中第一个元素的地址，个人理解为获取指定数组类元素开始的位置，即数组的第一个元素的位置，此时
            // tc还是个class而不是对象，说明对于一个特定的数组类，其创建出来的数组的元素基础偏移地址是相等的。
            // arrayIndexScale能获得数组中元素的增量地址
            TBASE = UNSAFE.arrayBaseOffset(tc);
            SBASE = UNSAFE.arrayBaseOffset(sc);
            ts = UNSAFE.arrayIndexScale(tc);
            ss = UNSAFE.arrayIndexScale(sc);
            HASHSEED_OFFSET = UNSAFE.objectFieldOffset(
                    ConcurrentHashMap.class.getDeclaredField("hashSeed"));
        } catch (Exception e) {
            throw new Error(e);
        }
        //arrayIndexScale返回的是2的n次方，这里检查是否真的是2的n次方
        if ((ss & (ss - 1)) != 0 || (ts & (ts - 1)) != 0)
            throw new Error("data type scale not a power of two");
        // Integer.numberOfLeadingZeros方法获取指定参数左边0的个数，如1左边有31个0，2左边有30个0
        // 下面相当于获取ss最高位1的位置，从0开始计算
        SSHIFT = 31 - Integer.numberOfLeadingZeros(ss);
        TSHIFT = 31 - Integer.numberOfLeadingZeros(ts);
    }

    /**
     * Mask value for indexing into segments. The upper bits of a
     * key's hash code are used to choose the segment.
     */
    //默认参数的情况下为15也就是1111B
    final int segmentMask;
    /**
     * Shift value for indexing within segments.
     */
    //默认参数的情况下为28
    final int segmentShift;
    /**
     * The segments, each of which is a specialized hash table.
     */
    final Segment<K, V>[] segments;
    /**
     * A randomizing value associated with this instance that is applied to
     * hash code of keys to make hash collisions harder to find.
     */
    private transient final int hashSeed = randomHashSeed(this);
    transient Set<K> keySet;
    transient Set<Map.Entry<K, V>> entrySet;
    transient Collection<V> values;

    /**
     * Creates a new, empty map with the specified initial
     * capacity, load factor and concurrency level.
     *
     * @param initialCapacity  the initial capacity. The implementation
     *                         performs internal sizing to accommodate this many elements.
     * @param loadFactor       the load factor threshold, used to control resizing.
     *                         Resizing may be performed when the average number of elements per
     *                         bin exceeds this threshold.
     * @param concurrencyLevel the estimated number of concurrently
     *                         updating threads. The implementation performs internal sizing
     *                         to try to accommodate this many threads.
     * @throws IllegalArgumentException if the initial capacity is
     *                                  negative or the load factor or concurrencyLevel are
     *                                  nonpositive.
     */
    @SuppressWarnings("unchecked")
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        //concurrencyLevel默认为16，所以ssize默认为16，sshift默认为4
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        //默认为28
        this.segmentShift = 32 - sshift;
        //默认为15
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //容量除以segment的个数等于每个segment中链表的个数
        int c = initialCapacity / ssize;
        //由于除法结果转int是向下取整，所以判断数量是否够，不够则每个segment多一个链表，保证总的链表个数大于等于initialCapacity
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        // 如果cap小于c则扩大cap
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        //只创建一个segment，后面的用延迟初始化方式
        Segment<K, V> s0 =
                new Segment<K, V>(loadFactor, (int) (cap * loadFactor),
                        (HashEntry<K, V>[]) new HashEntry[cap]);
        //创建segment数组
        Segment<K, V>[] ss = (Segment<K, V>[]) new Segment[ssize];
        // 设置segment数组的第一个元素为s0，这里用putOrderedObject设置数组第一个元素主要是为了效率吧，putOrderedObject不保证对其他线程立即可见，
        // 但是this.segments是final的，所以构造函数结束后this.segments会被正确赋值
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }

    /**
     * Creates a new, empty map with the specified initial capacity
     * and load factor and with the default concurrencyLevel (16).
     *
     * @param initialCapacity The implementation performs internal
     *                        sizing to accommodate this many elements.
     * @param loadFactor      the load factor threshold, used to control resizing.
     *                        Resizing may be performed when the average number of elements per
     *                        bin exceeds this threshold.
     * @throws IllegalArgumentException if the initial capacity of
     *                                  elements is negative or the load factor is nonpositive
     * @since 1.6
     */
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, DEFAULT_CONCURRENCY_LEVEL);
    }

// Hash-based segment and entry accesses

    /**
     * Creates a new, empty map with the specified initial capacity,
     * and with default load factor (0.75) and concurrencyLevel (16).
     *
     * @param initialCapacity the initial capacity. The implementation
     *                        performs internal sizing to accommodate this many elements.
     * @throws IllegalArgumentException if the initial capacity of
     *                                  elements is negative.
     */
    public ConcurrentHashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
    }

    /**
     * Creates a new, empty map with a default initial capacity (16),
     * load factor (0.75) and concurrencyLevel (16).
     */
    public ConcurrentHashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
    }

    /* ---------------- Public operations -------------- */

    /**
     * Creates a new map with the same mappings as the given map.
     * The map is created with a capacity of 1.5 times the number
     * of mappings in the given map or 16 (whichever is greater),
     * and a default load factor (0.75) and concurrencyLevel (16).
     *
     * @param m the map
     */
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                DEFAULT_INITIAL_CAPACITY),
                DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
        putAll(m);
    }

    private static int randomHashSeed(ConcurrentHashMap instance) {
        if (sun.misc.VM.isBooted() && Holder.ALTERNATIVE_HASHING) {
            return sun.misc.Hashing.randomHashSeed(instance);
        }

        return 0;
    }

    /**
     * Gets the ith element of given table (if nonnull) with volatile
     * read semantics. Note: This is manually integrated into a few
     * performance-sensitive methods to reduce call overhead.
     */
    // 获取tab的第i项
    // 获取的原理是TSHIFT在静态代码块中计算出来的HashEntry[].class的偏移量的最高位1的位置，如HashEntry[].class的数组偏移量是4，则TSHIFT == 2
    // 而下面就是利用TSHIFT计算数组第i项的偏移量，TBASE是在静态代码块中计算出来的HashEntry[].class的基础偏移量。加起来组成第i项元素的偏移量
    @SuppressWarnings("unchecked")
    static final <K, V> HashEntry<K, V> entryAt(HashEntry<K, V>[] tab, int i) {
        return (tab == null) ? null :
                (HashEntry<K, V>) UNSAFE.getObjectVolatile
                        (tab, ((long) i << TSHIFT) + TBASE);
    }

    /**
     * Sets the ith element of given table, with volatile write
     * semantics. (See above about use of putOrderedObject.)
     */
    // 设置tab的第i项
    // 原理和entryAt是一样的
    static final <K, V> void setEntryAt(HashEntry<K, V>[] tab, int i,
                                        HashEntry<K, V> e) {
        // 这里没有用putObjectVolatile是因为该方法被调用的地方都是获取到了锁的地方，JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，
        // 从而保证这些变更对其他线程是可见的
        UNSAFE.putOrderedObject(tab, ((long) i << TSHIFT) + TBASE, e);
    }

    /**
     * Gets the jth element of given segment array (if nonnull) with
     * volatile element access semantics via Unsafe. (The null check
     * can trigger harmlessly only during deserialization.) Note:
     * because each element of segments array is set only once (using
     * fully ordered writes), some performance-sensitive methods rely
     * on this method only as a recheck upon null reads.
     */
    @SuppressWarnings("unchecked")
    // 获取ss数组的第j项
    // 空检查是在反序列化的时候可能会触发
    static final <K, V> Segment<K, V> segmentAt(Segment<K, V>[] ss, int j) {
        long u = (j << SSHIFT) + SBASE;
        return ss == null ? null :
                (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u);
    }

    /**
     * Gets the table entry for the given segment and hash
     */
    // 获取seg的table的第(tab.length - 1) & h项，即根据指定hash返回seg的数组的指定元素
    @SuppressWarnings("unchecked")
    static final <K, V> HashEntry<K, V> entryForHash(Segment<K, V> seg, int h) {
        HashEntry<K, V>[] tab;
        return (seg == null || (tab = seg.table) == null) ? null :
                (HashEntry<K, V>) UNSAFE.getObjectVolatile
                        (tab, ((long) (((tab.length - 1) & h)) << TSHIFT) + TBASE);
    }

    /**
     * Applies a supplemental hash function to a given hashCode, which
     * defends against poor quality hash functions. This is critical
     * because ConcurrentHashMap uses power-of-two length hash tables,
     * that otherwise encounter collisions for hashCodes that do not
     * differ in lower or upper bits.
     */
    private int hash(Object k) {
        int h = hashSeed;

        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h << 15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h << 3);
        h ^= (h >>> 6);
        h += (h << 2) + (h << 14);
        return h ^ (h >>> 16);
    }

    /**
     * Returns the segment for the given index, creating it and
     * recording in segment table (via CAS) if not already present.
     *
     * @param k the index
     * @return the segment
     */
    @SuppressWarnings("unchecked")
    // 获取第k项Segment，由于构造函数只初始化了Segment数组的第一个Segment，其他的都是延迟创建的，所以这里用于确保想要的Segment被创建了
    // 该方法会在多线程环境下运行，但是又没有获取锁，所以用了一些特殊的方法确保了线程安全
    private Segment<K, V> ensureSegment(int k) {
        final Segment<K, V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K, V> seg;
        // 判断第k项Segment是否为null，不为null直接返回现有的
        if ((seg = (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u)) == null) {
            // 使用第一个segment作为模板
            Segment<K, V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int) (cap * lf);
            HashEntry<K, V>[] tab = (HashEntry<K, V>[]) new HashEntry[cap];
            // 由于是多线程环境，所以这里再次检查是否为null，同时用getObjectVolatile确保获取操作的原子性
            if ((seg = (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u))
                    == null) { // recheck
                Segment<K, V> s = new Segment<K, V>(lf, threshold, tab);
                // 该方法的任意时刻都有可能存在其他线程已经创建好了第k项的Segment，所以这里再次判断是否为null
                // 这里用while是因为在执行循环体内的compareAndSwapObject返回之前第k项的Segment也可能被其他线程创建了，所以在循环体内
                // 的if用CAS进行操作，同时循环体内的if可能为true即break出去了，也可能为false，此时再次循环
                while ((seg = (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u))
                        == null) {
                    // public native boolean compareAndSwapObject(Object obj, long offset, Object expect, Object update)
                    // 比较obj的offset处的对象与expect是否相等，相等则更新成update并返回true，否则返回false
                    // compareAndSwapObject是原子操作，如果在操作时发现ss的偏移为u处的segment为null，则seg更新成s并break后返回，
                    // 否则再次执行while循环，重新获取一次seg，此时seg是其他线程创建的Segment不为null，直接返回
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }

    /**
     * Get the segment for the given hash
     */
    @SuppressWarnings("unchecked")
    // 根据指定hash获取Segment数组的某个元素，hash公式：((h >>> segmentShift) & segmentMask)
    private Segment<K, V> segmentForHash(int h) {
        // 根据hash的高位获取应该在segment数组中的位置，假设segment数组长16则h >>> segmentShift == h >>> 28，获取高4位，并且此时segmentMask等于15，SSHIFT为segment数组的增量地址，SBASE
        // 为数组第一个元素的偏移地址，如果h >>> segmentShift) & segmentMask = 5，则((h >>> segmentShift) & segmentMask) << SSHIFT就是数组第5个元素的偏移地址，加上SBASE则能获得第5个元素的地址
        // 通过getObjectVolatile方法获取该地址对应的对象
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        return (Segment<K, V>) UNSAFE.getObjectVolatile(segments, u);
    }

    /**
     * Returns <tt>true</tt> if this map contains no key-value mappings.
     *
     * @return <tt>true</tt> if this map contains no key-value mappings
     */
    public boolean isEmpty() {
        /*
         * Sum per-segment modCounts to avoid mis-reporting when
         * elements are concurrently added and removed in one segment
         * while checking another, in which case the table was never
         * actually empty at any point. (The sum ensures accuracy up
         * through at least 1<<31 per-segment modifications before
         * recheck.) Methods size() and containsValue() use similar
         * constructions for stability checks.
         */
        long sum = 0L;
        // 下面有两个循环，不论哪个循环，只要发现某个seg的count不为0直接返回false即可，如果等于0则观察两次的modCount是否一致也就是循环过程中map结果是否变化，
        // 如果变化了则肯定不为空因为modCount不论是put还是remove都是递增的，在第一次循环过程中如果检查完某个Segment后该Segment新增了一个元素（只能是新增，因为之前计算过count为0）
        // 此时Segment的modCount就递增了，在第二次循环再次检查一次所有的Segment，此时可以发现Segment的count不为0，返回false即可，
        // 发现这个Segment的count又是0（说明在新增之后又删除了），此时modCount又新增了，最后sum计算结果肯定不为0，直接返回false
        final Segment<K, V>[] segments = this.segments;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K, V> seg = segmentAt(segments, j);
            if (seg != null) {
                if (seg.count != 0)
                    return false;
                sum += seg.modCount;
            }
        }
        if (sum != 0L) { // recheck unless no modifications
            for (int j = 0; j < segments.length; ++j) {
                Segment<K, V> seg = segmentAt(segments, j);
                if (seg != null) {
                    if (seg.count != 0)
                        return false;
                    sum -= seg.modCount;
                }
            }
            if (sum != 0L)
                return false;
        }
        return true;
    }

    /**
     * Returns the number of key-value mappings in this map. If the
     * map contains more than <tt>Integer.MAX_VALUE</tt> elements, returns
     * <tt>Integer.MAX_VALUE</tt>.
     *
     * @return the number of key-value mappings in this map
     */
    // 总的思想就是遍历map至少两次，每次遍历时记录总的modCount，并在结束时与上次的modCount比较，
    // 如果相等说明两次遍历中map结构没有改变则返回结果，否则再遍历一次，当遍历的次数达到了RETRIES_BEFORE_LOCK则获取所有的segment的锁再遍历并返回结果
    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K, V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits // 标记计算过程中的结果是否超过了整数上限，如果超过了直接返回Integer.MAX_VALUE
        long sum; // sum of modCounts
        long last = 0L; // previous sum
        // 至少重试2次，所以从-1开始
        int retries = -1; // first iteration isn't retry
        try {
            for (; ; ) {
                // 在尝试多次后sum都不等于last的将所有的segment加锁，以此保证sum == last从而获取到size
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        // 这里确保所有的Segment都创建出来了，这样才能获取到所有的Segment的锁，个人认为目的是如果不获取为空的Segment的锁，
                        // 则其他线程可能在执行size方法期间创建Segment并添加元素，此时size的结果可能就错了
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L; // 保存modCounts的总和
                size = 0; // 保存所有Segment的元素个数总和
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K, V> seg = segmentAt(segments, j);
                    // 由于构造函数中Segment数组是延迟创建其他元素的，所以可能为null
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                // 判断当前的sum是否等于上次的sum，如果是说明这两次遍历过程中整个map没有修改操作，则返回size，注意无论是put还是remove，
                // modCount都是增加的，所以只要在sum != last说明map肯定是变化了，则需要重新开始遍历
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            //解锁
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }

    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}. (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    // get方法没有获取锁，直接找指定的Segment和Segment的table的指定HashEntry并开始遍历，所以get方法不能获取到遍历过程中插入的数据
    // 即ConcurrentHashMap是弱一致的
    public V get(Object key) {
        Segment<K, V> s; // manually integrate access methods to reduce overhead
        HashEntry<K, V>[] tab;
        // 如果key为null，这里将会抛出NullPointerException
        int h = hash(key);
        // 获取key对应的Segment的地址
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K, V>) UNSAFE.getObjectVolatile(segments, u)) != null &&
                (tab = s.table) != null) {
            // 首先根据hash获取Segment的table的指定HashEntry，HashEntry是单链表的头结点，遍历单链表寻找key即可
            for (HashEntry<K, V> e = (HashEntry<K, V>) UNSAFE.getObjectVolatile
                            (tab, ((long) (((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }

    /**
     * Tests if the specified object is a key in this table.
     *
     * @param key possible key
     * @return <tt>true</tt> if and only if the specified object
     * is a key in this table, as determined by the
     * <tt>equals</tt> method; <tt>false</tt> otherwise.
     * @throws NullPointerException if the specified key is null
     */
    @SuppressWarnings("unchecked")
    // 和get方法没区别
    public boolean containsKey(Object key) {
        Segment<K, V> s; // same as get() except no need for volatile value read
        HashEntry<K, V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K, V>) UNSAFE.getObjectVolatile(segments, u)) != null &&
                (tab = s.table) != null) {
            for (HashEntry<K, V> e = (HashEntry<K, V>) UNSAFE.getObjectVolatile
                    (tab, ((long) (((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return true;
            }
        }
        return false;
    }

    /**
     * Returns <tt>true</tt> if this map maps one or more keys to the
     * specified value. Note: This method requires a full internal
     * traversal of the hash table, and so is much slower than
     * method <tt>containsKey</tt>.
     *
     * @param value value whose presence in this map is to be tested
     * @return <tt>true</tt> if this map maps one or more keys to the
     * specified value
     * @throws NullPointerException if the specified value is null
     */
    // 判断是否包含指定的value，由于是查询value，所以不能定位到某个Segment，需要扫描整个Map
    // 下面的思路和size方法思路一样，如果找到了直接返回即可，如果没找到则判断这次计算的modCount总和和上次的是否相等，相等就跳出循环
    public boolean containsValue(Object value) {
        // Same idea as size()
        if (value == null)
            throw new NullPointerException();
        final Segment<K, V>[] segments = this.segments;
        boolean found = false;
        long last = 0;
        int retries = -1;
        try {
            //当break outer后循环就结束了，直接执行finally了，这也是一种退出循环的方式
            outer:
            for (; ; ) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                long hashSum = 0L;
                int sum = 0;
                for (int j = 0; j < segments.length; ++j) {
                    HashEntry<K, V>[] tab;
                    Segment<K, V> seg = segmentAt(segments, j);
                    if (seg != null && (tab = seg.table) != null) {
                        for (int i = 0; i < tab.length; i++) {
                            HashEntry<K, V> e;
                            for (e = entryAt(tab, i); e != null; e = e.next) {
                                V v = e.value;
                                if (v != null && value.equals(v)) {
                                    found = true;
                                    break outer;
                                }
                            }
                        }
                        sum += seg.modCount;
                    }
                }
                // sum和last第一次循环可能都是0，为了保证至少循环两次，这里额外判断一下retries是否大于0
                if (retries > 0 && sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return found;
    }

    /**
     * Legacy method testing if some key maps into the specified value
     * in this table. This method is identical in functionality to
     * {@link #containsValue}, and exists solely to ensure
     * full compatibility with class {@link java.util.Hashtable},
     * which supported this method prior to introduction of the
     * Java Collections framework.
     *
     * @param value a value to search for
     * @return <tt>true</tt> if and only if some key maps to the
     * <tt>value</tt> argument in this table as
     * determined by the <tt>equals</tt> method;
     * <tt>false</tt> otherwise
     * @throws NullPointerException if the specified value is null
     */
    // contains方法查询的是value而不是key
    public boolean contains(Object value) {
        return containsValue(value);
    }

    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p> The value can be retrieved by calling the <tt>get</tt> method
     * with a key that is equal to the original key.
     *
     * @param key   key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     * <tt>null</tt> if there was no mapping for <tt>key</tt>
     * @throws NullPointerException if the specified key or value is null
     */
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K, V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        // 找到key对应的segment，这里没有用getObjectVolatile是因为一旦segment被初始化后，就不会创建新的segment替代它了，所以这里如果返回的不是null则直接使用，如果是null
        // 在ensureSegment中用了getObjectVolatile保证了能获取到正确的segment，并且ensureSegment里也已经保证了不会重复创建Segment，所以这里不需要getObjectVolatile了
        if ((s = (Segment<K, V>) UNSAFE.getObject // nonvolatile; recheck
                (segments, (j << SSHIFT) + SBASE)) == null) // in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }

    /**
     * {@inheritDoc}
     *
     * @return the previous value associated with the specified key,
     * or <tt>null</tt> if there was no mapping for the key
     * @throws NullPointerException if the specified key or value is null
     */
    @SuppressWarnings("unchecked")
    // 和put方法一样，区别只是传入Segment的put方法的onlyIfAbsent参数
    public V putIfAbsent(K key, V value) {
        Segment<K, V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K, V>) UNSAFE.getObject
                (segments, (j << SSHIFT) + SBASE)) == null)
            s = ensureSegment(j);
        return s.put(key, hash, value, true);
    }

    /**
     * Copies all of the mappings from the specified map to this one.
     * These mappings replace any mappings that this map had for any of the
     * keys currently in the specified map.
     *
     * @param m mappings to be stored in this map
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
            put(e.getKey(), e.getValue());
    }

    /**
     * Removes the key (and its corresponding value) from this map.
     * This method does nothing if the key is not in the map.
     *
     * @param key the key that needs to be removed
     * @return the previous value associated with <tt>key</tt>, or
     * <tt>null</tt> if there was no mapping for <tt>key</tt>
     * @throws NullPointerException if the specified key is null
     */
    // 下面都是调用Segment里的方法，没什么好说的
    public V remove(Object key) {
        int hash = hash(key);
        Segment<K, V> s = segmentForHash(hash);
        return s == null ? null : s.remove(key, hash, null);
    }

    /**
     * {@inheritDoc}
     *
     * @throws NullPointerException if the specified key is null
     */
    public boolean remove(Object key, Object value) {
        int hash = hash(key);
        Segment<K, V> s;
        return value != null && (s = segmentForHash(hash)) != null &&
                s.remove(key, hash, value) != null;
    }

    /**
     * {@inheritDoc}
     *
     * @throws NullPointerException if any of the arguments are null
     */
    public boolean replace(K key, V oldValue, V newValue) {
        int hash = hash(key);
        if (oldValue == null || newValue == null)
            throw new NullPointerException();
        Segment<K, V> s = segmentForHash(hash);
        return s != null && s.replace(key, hash, oldValue, newValue);
    }

    /**
     * {@inheritDoc}
     *
     * @return the previous value associated with the specified key,
     * or <tt>null</tt> if there was no mapping for the key
     * @throws NullPointerException if the specified key or value is null
     */
    public V replace(K key, V value) {
        int hash = hash(key);
        if (value == null)
            throw new NullPointerException();
        Segment<K, V> s = segmentForHash(hash);
        return s == null ? null : s.replace(key, hash, value);
    }

    /**
     * Removes all of the mappings from this map.
     */
     // clear方法没有加全局锁，遍历所有的segment并调用clear方法，所以可能已经执行过clear方法的segment中
     // 又会添加新的数据，所以clear方法执行完后不能保证map为空
    public void clear() {
        final Segment<K, V>[] segments = this.segments;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K, V> s = segmentAt(segments, j);
            if (s != null)
                s.clear();
        }
    }

    /**
     * Returns a {@link Set} view of the keys contained in this map.
     * The set is backed by the map, so changes to the map are
     * reflected in the set, and vice-versa. The set supports element
     * removal, which removes the corresponding mapping from this map,
     * via the <tt>Iterator.remove</tt>, <tt>Set.remove</tt>,
     * <tt>removeAll</tt>, <tt>retainAll</tt>, and <tt>clear</tt>
     * operations. It does not support the <tt>add</tt> or
     * <tt>addAll</tt> operations.
     *
     * <p>The view's <tt>iterator</tt> is a "weakly consistent" iterator
     * that will never throw {@link ConcurrentModificationException},
     * and guarantees to traverse elements as they existed upon
     * construction of the iterator, and may (but is not guaranteed to)
     * reflect any modifications subsequent to construction.
     */
    public Set<K> keySet() {
        Set<K> ks = keySet;
        return (ks != null) ? ks : (keySet = new KeySet());
    }

    /* ---------------- Iterator Support -------------- */

    /**
     * Returns a {@link Collection} view of the values contained in this map.
     * The collection is backed by the map, so changes to the map are
     * reflected in the collection, and vice-versa. The collection
     * supports element removal, which removes the corresponding
     * mapping from this map, via the <tt>Iterator.remove</tt>,
     * <tt>Collection.remove</tt>, <tt>removeAll</tt>,
     * <tt>retainAll</tt>, and <tt>clear</tt> operations. It does not
     * support the <tt>add</tt> or <tt>addAll</tt> operations.
     *
     * <p>The view's <tt>iterator</tt> is a "weakly consistent" iterator
     * that will never throw {@link ConcurrentModificationException},
     * and guarantees to traverse elements as they existed upon
     * construction of the iterator, and may (but is not guaranteed to)
     * reflect any modifications subsequent to construction.
     */
    public Collection<V> values() {
        Collection<V> vs = values;
        return (vs != null) ? vs : (values = new Values());
    }

    /**
     * Returns a {@link Set} view of the mappings contained in this map.
     * The set is backed by the map, so changes to the map are
     * reflected in the set, and vice-versa. The set supports element
     * removal, which removes the corresponding mapping from the map,
     * via the <tt>Iterator.remove</tt>, <tt>Set.remove</tt>,
     * <tt>removeAll</tt>, <tt>retainAll</tt>, and <tt>clear</tt>
     * operations. It does not support the <tt>add</tt> or
     * <tt>addAll</tt> operations.
     *
     * <p>The view's <tt>iterator</tt> is a "weakly consistent" iterator
     * that will never throw {@link ConcurrentModificationException},
     * and guarantees to traverse elements as they existed upon
     * construction of the iterator, and may (but is not guaranteed to)
     * reflect any modifications subsequent to construction.
     */
    public Set<Map.Entry<K, V>> entrySet() {
        Set<Map.Entry<K, V>> es = entrySet;
        return (es != null) ? es : (entrySet = new EntrySet());
    }

    /**
     * Returns an enumeration of the keys in this table.
     *
     * @return an enumeration of the keys in this table
     * @see #keySet()
     */
    public Enumeration<K> keys() {
        return new KeyIterator();
    }

    /**
     * Returns an enumeration of the values in this table.
     *
     * @return an enumeration of the values in this table
     * @see #values()
     */
    public Enumeration<V> elements() {
        return new ValueIterator();
    }

    /**
     * Save the state of the <tt>ConcurrentHashMap</tt> instance to a
     * stream (i.e., serialize it).
     *
     * @param s the stream
     * @serialData the key (Object) and value (Object)
     * for each key-value mapping, followed by a null pair.
     * The key-value mappings are emitted in no particular order.
     */
    private void writeObject(java.io.ObjectOutputStream s) throws IOException {
// force all segments for serialization compatibility
        for (int k = 0; k < segments.length; ++k)
            ensureSegment(k);
        s.defaultWriteObject();

        final Segment<K, V>[] segments = this.segments;
        for (int k = 0; k < segments.length; ++k) {
            Segment<K, V> seg = segmentAt(segments, k);
            seg.lock();
            try {
                HashEntry<K, V>[] tab = seg.table;
                for (int i = 0; i < tab.length; ++i) {
                    HashEntry<K, V> e;
                    for (e = entryAt(tab, i); e != null; e = e.next) {
                        s.writeObject(e.key);
                        s.writeObject(e.value);
                    }
                }
            } finally {
                seg.unlock();
            }
        }
        s.writeObject(null);
        s.writeObject(null);
    }

    /**
     * Reconstitute the <tt>ConcurrentHashMap</tt> instance from a
     * stream (i.e., deserialize it).
     *
     * @param s the stream
     */
    @SuppressWarnings("unchecked")
    private void readObject(java.io.ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        s.defaultReadObject();

        // set hashMask
        UNSAFE.putIntVolatile(this, HASHSEED_OFFSET, randomHashSeed(this));

        // Re-initialize segments to be minimally sized, and let grow.
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        final Segment<K, V>[] segments = this.segments;
        for (int k = 0; k < segments.length; ++k) {
            Segment<K, V> seg = segments[k];
            if (seg != null) {
                seg.threshold = (int) (cap * seg.loadFactor);
                seg.table = (HashEntry<K, V>[]) new HashEntry[cap];
            }
        }

        // Read the keys and values, and put the mappings in the table
        for (; ; ) {
            K key = (K) s.readObject();
            V value = (V) s.readObject();
            if (key == null)
                break;
            put(key, value);
        }
    }

    /**
     * holds values which can't be initialized until after VM is booted.
     */
    private static class Holder {

        /**
         * Enable alternative hashing of String keys?
         *
         * <p>Unlike the other hash map implementations we do not implement a
         * threshold for regulating whether alternative hashing is used for
         * String keys. Alternative hashing is either enabled for all instances
         * or disabled for all instances.
         */
        static final boolean ALTERNATIVE_HASHING;

        static {
            // Use the "threshold" system property even though our threshold
            // behaviour is "ON" or "OFF".
            String altThreshold = java.security.AccessController.doPrivileged(
                    new sun.security.action.GetPropertyAction(
                            "jdk.map.althashing.threshold"));

            int threshold;
            try {
                threshold = (null != altThreshold)
                        ? Integer.parseInt(altThreshold)
                        : Integer.MAX_VALUE;

                // disable alternative hashing if -1
                if (threshold == -1) {
                    threshold = Integer.MAX_VALUE;
                }

                if (threshold < 0) {
                    throw new IllegalArgumentException("value must be positive integer.");
                }
            } catch (IllegalArgumentException failed) {
                throw new Error("Illegal value for 'jdk.map.althashing.threshold'", failed);
            }
            ALTERNATIVE_HASHING = threshold <= MAXIMUM_CAPACITY;
        }
    }

    /**
     * ConcurrentHashMap list entry. Note that this is never exported
     * out as a user-visible Map.Entry.
     */
    // Segment的table的元素，同时也是个单链表结点
    static final class HashEntry<K, V> {
        // Unsafe mechanics
        static final sun.misc.Unsafe UNSAFE;
        // 保存HashEntry类的next属性的偏移值，便于使用UNSAFE设置
        static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class k = HashEntry.class;
                nextOffset = UNSAFE.objectFieldOffset
                        (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }

        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K, V> next;

        HashEntry(int hash, K key, V value, HashEntry<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        /**
         * Sets next field with volatile write semantics. (See above
         * about use of putOrderedObject.)
         */
        final void setNext(HashEntry<K, V> n) {
            // 设置当前HashEntry的next属性为n，putOrderedObject不保证对其他线程立即可见，除非是在获取到锁的情况下调用的
            // 调用这个方法的地方都是获取到锁的地方，所以不需要用putObjectVolatile来保证可见性，以提高效率
            UNSAFE.putOrderedObject(this, nextOffset, n);
        }
    }

    /* ---------------- Serialization Support -------------- */

    /**
     * Segments are specialized versions of hash tables. This
     * subclasses from ReentrantLock opportunistically, just to
     * simplify some locking and avoid separate construction.
     */
    // Segment继承了ReentrantLock对象，Segment和JDK7的HashMap是一个原理
    static final class Segment<K, V> extends ReentrantLock implements Serializable {
        /*
         * Segments maintain a table of entry lists that are always
         * kept in a consistent state, so can be read (via volatile
         * reads of segments and tables) without locking. This
         * requires replicating nodes when necessary during table
         * resizing, so the old lists can be traversed by readers
         * still using old version of table.
         *
         * This class defines only mutative methods requiring locking.
         * Except as noted, the methods of this class perform the
         * per-segment versions of ConcurrentHashMap methods. (Other
         * methods are integrated directly into ConcurrentHashMap
         * methods.) These mutative methods use a form of controlled
         * spinning on contention via methods scanAndLock and
         * scanAndLockForPut. These intersperse tryLocks with
         * traversals to locate nodes. The main benefit is to absorb
         * cache misses (which are very common for hash tables) while
         * obtaining locks so that traversal is faster once
         * acquired. We do not actually use the found nodes since they
         * must be re-acquired under lock anyway to ensure sequential
         * consistency of updates (and in any case may be undetectably
         * stale), but they will normally be much faster to re-locate.
         * Also, scanAndLockForPut speculatively creates a fresh node
         * to use in put if no node is found.
         */

        /**
         * The maximum number of times to tryLock in a prescan before
         * possibly blocking on acquire in preparation for a locked
         * segment operation. On multiprocessors, using a bounded
         * number of retries maintains cache acquired while locating
         * nodes.
         */
        // 最大可重试次数，如果是多处理器则为64
        static final int MAX_SCAN_RETRIES =
                Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
        private static final long serialVersionUID = 2249069246763182397L;
        /**
         * The load factor for the hash table. Even though this value
         * is same for all segments, it is replicated to avoid needing
         * links to outer object.
         *
         * @serial
         */
        final float loadFactor;
        /**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K, V>[] table;
        /**
         * The number of elements. Accessed only either within locks
         * or among other volatile reads that maintain visibility.
         */
        transient int count;
        /**
         * The total number of mutative operations in this segment.
         * Even though this may overflows 32 bits, it provides
         * sufficient accuracy for stability checks in CHM isEmpty()
         * and size() methods. Accessed only either within locks or
         * among other volatile reads that maintain visibility.
         */
        transient int modCount;
        /**
         * The table is rehashed when its size exceeds this threshold.
         * (The value of this field is always <tt>(int)(capacity *
         * loadFactor)</tt>.)
         */
        transient int threshold;

        Segment(float lf, int threshold, HashEntry<K, V>[] tab) {
            this.loadFactor = lf;
            this.threshold = threshold;
            this.table = tab;
        }

        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            // tryLock方法尝试获取锁，不论是否获取到立即返回，如果能获取到锁则node为null，否则将会在scanAndLockForPut中循环直到获取到锁并返回node(如果key对应的node已存在则null，否则新建一个并返回)
            // 这是ConcurrentHashMap做的一个优化，scanAndLockForPut在等待锁的过程中创建node（如果需要的话），注意，下面的tryLock如果直接获取到了锁，node就是null，
            // 下面的for循环会检查key是否存在，如果存在则更新value并break出来，否则会在else里创建node
            HashEntry<K, V> node = tryLock() ? null :
                    // 循环并tryLock，并遍历链表以提高缓存命中率
                    scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                // 下面的代码都是线程安全的，已经获取到锁了
                HashEntry<K, V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K, V> first = entryAt(tab, index);
                for (HashEntry<K, V> e = first; ; ) {
                    // 首先遍历链表查看是否存在key，scanAndLockForPut方法在key不存在的情况下会创建一个HashEntry并赋值给node，有一种情况是
                    // 在scanAndLockForPut创建HashEntry后获取到锁之前其他线程put了同一个key，之后scanAndLockForPut获取到锁返回了新建的
                    // HashEntry，那么下面的代码块是可以发现其他线程put的key的，此时就不用管新建的HashEntry了，就当key已存在即可
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                                (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            // onlyIfAbsent为true表示如果key已存在则不更新，为false则更新
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    // 如果不存在key或者链表是空的则创建一个node
                    else {
                        // 如果scanAndLockForPut已经创建好了一个node，直接拿来用
                        if (node != null)
                            // 新建的HashEntry放到链表头
                            node.setNext(first);
                        else
                            node = new HashEntry<K, V>(hash, key, value, first);
                        // count为该segment的总key-value对个数
                        int c = count + 1;
                        // 如果超过阈值则扩容
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            // 扩容，并传入新的node，扩容后node已经放到数组了，所有不用执行setEntryAt方法
                            rehash(node);
                        else
                            // 将HashEntry数组中index处设置为node
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

        /**
         * Doubles size of table and repacks entries, also adding the
         * given node to new table
         */
        @SuppressWarnings("unchecked")
        // 扩容，node是刚创建的结点，该方法只在put时会被调用，调用时已经获取到锁了
        private void rehash(HashEntry<K, V> node) {
            /*
             * Reclassify nodes in each list to new table. Because we
             * are using power-of-two expansion, the elements from
             * each bin must either stay at same index, or move with a
             * power of two offset. We eliminate unnecessary node
             * creation by catching cases where old nodes can be
             * reused because their next fields won't change.
             * Statistically, at the default threshold, only about
             * one-sixth of them need cloning when a table
             * doubles. The nodes they replace will be garbage
             * collectable as soon as they are no longer referenced by
             * any reader thread that may be in the midst of
             * concurrently traversing table. Entry accesses use plain
             * array indexing because they are followed by volatile
             * table write.
             */
            HashEntry<K, V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            // 每次扩大两倍
            int newCapacity = oldCapacity << 1;
            threshold = (int) (newCapacity * loadFactor);
            HashEntry<K, V>[] newTable =
                    (HashEntry<K, V>[]) new HashEntry[newCapacity];
            int sizeMask = newCapacity - 1;
            // 遍历原来的table，所有的HashEntry重新计算一次位置
            for (int i = 0; i < oldCapacity; i++) {
                HashEntry<K, V> e = oldTable[i];
                if (e != null) {
                    HashEntry<K, V> next = e.next;
                    // 获取当前HashEntry在新table中的位置
                    int idx = e.hash & sizeMask;
                    // 如果next为null说明链表只有一个结点，直接设置即可
                    if (next == null) // Single node on list
                        newTable[idx] = e;
                    else { // Reuse consecutive sequence at same slot
                        // 假设扩容前某HashEntry对应到Segment中数组的index为i，数组的容量为capacity，那么扩容后该HashEntry对应到新数组中的index只可能为i或者i+capacity，因为capacity都是2的幂
                        // 在求index时计算e.hash & sizeMask时sizeMask都是capcacity - 1也就是二进制若干个连续的1，而扩容也是原来的容量乘以2所以sizeMask只是比原来多一个1，这样当求e.hash & sizeMask时
                        // 假设sizeMask原来是4个1，e.hash & sizeMask相等于求e.hash的低4位对应的十进制，而扩容后sizeMask为5个1，e.hash & sizeMask计算时e.hash低位的第5为要么为0要么为1，如果为0则新的
                        // e.hash & sizeMask计算结果和原来一样，如果为1则e.hash & sizeMask的结果相对于原来的计算结果加了10000B(对于假设的sizeMask原来是4个1来说)，而10000B就是原来的capacity
                        // 所以扩容某个链表时这个链表上的HashEntry要么位置保持不变要么变成index + capacity，只有这两种结果，所以可以进行优化，下面的第一个for循环遍历链表，如果lastIdx和当前遍历到的结点的位置即k
                        // 不相等则更新lastIdx和lastRun，否则不更新，这样直到遍历完链表，lastRun及其之后的所有结点的位置肯定是一样的，此时把这条原来单链表的子链表作为新链表赋值给数组的lastIdx，然后再进行第二次遍历，
                        // 遍历到的结点重新计算一次位置，第二次遍历只需要遍历到lastRun即可，因为lastRun及其之后的结点都已经放到了正确的位置
                        HashEntry<K, V> lastRun = e;
                        int lastIdx = idx;
                        for (HashEntry<K, V> last = next;
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;
                                lastRun = last;
                            }
                        }
                        newTable[lastIdx] = lastRun;
                        // 复制剩下的结点，这些剩下的结点可能位于index也可能位于index + capacity
                        // Clone remaining nodes
                        for (HashEntry<K, V> p = e; p != lastRun; p = p.next) {
                            V v = p.value;
                            int h = p.hash;
                            int k = h & sizeMask;
                            HashEntry<K, V> n = newTable[k];
                            newTable[k] = new HashEntry<K, V>(h, p.key, v, n);
                        }
                    }
                }
            }
            // node为新建的HashEntry，看put方法，这里把新建的node放到数组中
            int nodeIndex = node.hash & sizeMask; // add the new node
            node.setNext(newTable[nodeIndex]);
            newTable[nodeIndex] = node;
            table = newTable;
        }

        /**
         * Scans for a node containing given key while trying to
         * acquire lock, creating and returning one if not found. Upon
         * return, guarantees that lock is held. UNlike in most
         * methods, calls to method equals are not screened: Since
         * traversal speed doesn't matter, we might as well help warm
         * up the associated code and accesses as well.
         *
         * @return a new node if key not found, else null
         */
        // 循环并tryLock，遍历链表以提高cpu缓存命中率，相当于数据预热，同时判断key是否已经存在，如果不存在则创建一个以提高获取到锁之后
        // put方法的执行效率，如果key已存在则在获取锁后返回null
        private HashEntry<K, V> scanAndLockForPut(K key, int hash, V value) {
            // 获取hash对应的HashEntry
            HashEntry<K, V> first = entryForHash(this, hash);
            HashEntry<K, V> e = first;
            HashEntry<K, V> node = null;
            int retries = -1; // negative while locating node
            // 循环尝试获取锁
            while (!tryLock()) {
                HashEntry<K, V> f; // to recheck first below
                // retries在第一次循环的时候是-1，所以会进入下面的代码块，第一次进入下面的代码块时e是当前Segment的某个链表的头结点，
                // 如果为null则说明链表是空的，直接创建一个HashEntry并赋值给node，再设置retries = 0，或e不为空时判断e的key是否是
                // 正在寻找的key，如果是则说明key已存在，直接设置retries = 0，否则设置e为e.next并再次循环，直到后面的node中找到了
                // key或者没找到导致e == null，也会设置retries = 0，在retries = 0后第一个代码块将不再执行
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K, V>(hash, key, value, null);
                        retries = 0;
                        // 这里在找到key之后只是设置retries等于0，所以在key已存在的情况下node是不会被设置的，即scanAndLockForPut
                        // 方法在key存在的情况下，获取到锁后返回null
                    } else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                // 如果tryLock尝试获取锁的次数到达了上限则使用lock方法阻塞的获取锁
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                // (retries & 1) == 0表示retries偶数时执行，下面在retries为偶数时再获取一次链表头，并判断和之前的链表头是否相等，如果不想等则说明
                // 链表发生了变化（可能新增了一个结点，或者删除了头结点），此时重设头结点，并重新开始尝试获取锁
                else if ((retries & 1) == 0 &&
                        (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }

        /**
         * Scans for a node containing the given key while trying to
         * acquire lock for a remove or replace operation. Upon
         * return, guarantees that lock is held. Note that we must
         * lock even if the key is not found, to ensure sequential
         * consistency of updates.
         */
        //循环并tryLock，并遍历链表以提高缓存命中率，和scanAndLockForPut方法差不多，少了创建HashEntry的过程
        private void scanAndLock(Object key, int hash) {
            // similar to but simpler than scanAndLockForPut
            HashEntry<K, V> first = entryForHash(this, hash);
            HashEntry<K, V> e = first;
            int retries = -1;
            while (!tryLock()) {
                HashEntry<K, V> f;
                if (retries < 0) {
                    if (e == null || key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                } else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                } else if ((retries & 1) == 0 &&
                        (f = entryForHash(this, hash)) != first) {
                    e = first = f;
                    retries = -1;
                }
            }
        }

        /**
         * Remove; match on key only if value null, else match both.
         */
        //当key对应的value等于参数value时或参数value为null时remove
        final V remove(Object key, int hash, Object value) {
            if (!tryLock())
                //循环并tryLock，并遍历链表以提高缓存命中率
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K, V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K, V> e = entryAt(tab, index);
                HashEntry<K, V> pred = null;
                while (e != null) {
                    K k;
                    HashEntry<K, V> next = e.next;
                    if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                        V v = e.value;
                        if (value == null || value == v || value.equals(v)) {
                            // 移除对应的node
                            if (pred == null)
                                // 如果是链表的头结点则设置next为新的头结点
                                setEntryAt(tab, index, next);
                            else
                                // 否则更新前面一个结点的next
                                pred.setNext(next);
                            ++modCount;
                            --count;
                            oldValue = v;
                        }
                        break;
                    }
                    pred = e;
                    e = next;
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

        //如果key对应的value等于oldValue时replace
        final boolean replace(K key, int hash, V oldValue, V newValue) {
            if (!tryLock())
                scanAndLock(key, hash);
            boolean replaced = false;
            try {
                HashEntry<K, V> e;
                for (e = entryForHash(this, hash); e != null; e = e.next) {
                    K k;
                    if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                        if (oldValue.equals(e.value)) {
                            e.value = newValue;
                            ++modCount;
                            replaced = true;
                        }
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return replaced;
        }

        final V replace(K key, int hash, V value) {
            if (!tryLock())
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K, V> e;
                for (e = entryForHash(this, hash); e != null; e = e.next) {
                    K k;
                    if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        e.value = value;
                        ++modCount;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

        final void clear() {
            lock();
            try {
                HashEntry<K, V>[] tab = table;
                for (int i = 0; i < tab.length; i++)
                    setEntryAt(tab, i, null);
                ++modCount;
                count = 0;
            } finally {
                unlock();
            }
        }
    }

    abstract class HashIterator {
        int nextSegmentIndex;
        int nextTableIndex;
        HashEntry<K, V>[] currentTable;
        HashEntry<K, V> nextEntry;
        HashEntry<K, V> lastReturned;

        HashIterator() {
            // 从segments.length - 1往0过滤，因为ConcurrentHashMap的Segment个数只会增不会减，为了防止在遍历的时候Segment数量增加
            // 导致可能一直遍历下去，所以直接从segments.length - 1开始遍历，即在HashIterator创建后需要遍历的Segment数量就确定了
            nextSegmentIndex = segments.length - 1;
            nextTableIndex = -1;
            advance();
        }

        /**
         * Set nextEntry to first node of next non-empty table
         * (in backwards order, to simplify checks).
         */
        // 找到不为null的segment中不为null的链表，该方法会多次被调用，功能实际上就是遍历map找到下一个非null的那个链表
        final void advance() {
            for (; ; ) {
                if (nextTableIndex >= 0) {
                    if ((nextEntry = entryAt(currentTable,
                            nextTableIndex--)) != null)
                        break;
                } else if (nextSegmentIndex >= 0) {
                    Segment<K, V> seg = segmentAt(segments, nextSegmentIndex--);
                    if (seg != null && (currentTable = seg.table) != null)
                        nextTableIndex = currentTable.length - 1;
                } else
                    break;
            }
        }

        // 注意该方法没有ConcurrentModificationException异常了，假设当前的nextEntry被其他线程从map中remove了也没事，因为即使从map中删除了这个node，该node在内存中也是存在的，nextEntry还指向的是它，而且
        // 该node的next也是和删除前一样的，所以不影响遍历，不过这样会导致执行map.remove后删除了nextEntry对应的结点再调用iterator.next()仍然遍历的是删除掉的那个结点，这个需要注意
        final HashEntry<K, V> nextEntry() {
            HashEntry<K, V> e = nextEntry;
            if (e == null)
                // 当没有next元素时抛出异常
                throw new NoSuchElementException();
            lastReturned = e; // cannot assign until after null check
            // 获取链表下一个元素，如果下一个为null则找下一个链表
            if ((nextEntry = e.next) == null)
                advance();
            return e;
        }

        public final boolean hasNext() {
            return nextEntry != null;
        }

        public final boolean hasMoreElements() {
            return nextEntry != null;
        }

        public final void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            ConcurrentHashMap.this.remove(lastReturned.key);
            lastReturned = null;
        }
    }

    final class KeyIterator
            extends HashIterator
            implements Iterator<K>, Enumeration<K> {
        public final K next() {
            return super.nextEntry().key;
        }

        public final K nextElement() {
            return super.nextEntry().key;
        }
    }

    final class ValueIterator
            extends HashIterator
            implements Iterator<V>, Enumeration<V> {
        public final V next() {
            return super.nextEntry().value;
        }

        public final V nextElement() {
            return super.nextEntry().value;
        }
    }

    /**
     * Custom Entry class used by EntryIterator.next(), that relays
     * setValue changes to the underlying map.
     */
    // 用于表示可写的HashEntry，set方法实际上就是调用put，这个类的目的是使EntryIterator返回的Entry能够调用setValue
    final class WriteThroughEntry
            extends AbstractMap.SimpleEntry<K, V> {
        WriteThroughEntry(K k, V v) {
            super(k, v);
        }

        /**
         * Set our entry's value and write through to the map. The
         * value to return is somewhat arbitrary here. Since a
         * WriteThroughEntry does not necessarily track asynchronous
         * changes, the most recent "previous" value could be
         * different from what we return (or could even have been
         * removed in which case the put will re-establish). We do not
         * and cannot guarantee more.
         */
        public V setValue(V value) {
            if (value == null) throw new NullPointerException();
            V v = super.setValue(value);
            ConcurrentHashMap.this.put(getKey(), value);
            return v;
        }
    }

    final class EntryIterator
            extends HashIterator
            implements Iterator<Entry<K, V>> {
        public Map.Entry<K, V> next() {
            HashEntry<K, V> e = super.nextEntry();
            return new WriteThroughEntry(e.key, e.value);
        }
    }

    final class KeySet extends AbstractSet<K> {
        public Iterator<K> iterator() {
            return new KeyIterator();
        }

        public int size() {
            return ConcurrentHashMap.this.size();
        }

        public boolean isEmpty() {
            return ConcurrentHashMap.this.isEmpty();
        }

        public boolean contains(Object o) {
            return ConcurrentHashMap.this.containsKey(o);
        }

        public boolean remove(Object o) {
            return ConcurrentHashMap.this.remove(o) != null;
        }

        public void clear() {
            ConcurrentHashMap.this.clear();
        }
    }

    final class Values extends AbstractCollection<V> {
        public Iterator<V> iterator() {
            return new ValueIterator();
        }

        public int size() {
            return ConcurrentHashMap.this.size();
        }

        public boolean isEmpty() {
            return ConcurrentHashMap.this.isEmpty();
        }

        public boolean contains(Object o) {
            return ConcurrentHashMap.this.containsValue(o);
        }

        public void clear() {
            ConcurrentHashMap.this.clear();
        }
    }

    final class EntrySet extends AbstractSet<Map.Entry<K, V>> {
        public Iterator<Map.Entry<K, V>> iterator() {
            return new EntryIterator();
        }

        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
            V v = ConcurrentHashMap.this.get(e.getKey());
            return v != null && v.equals(e.getValue());
        }

        public boolean remove(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
            return ConcurrentHashMap.this.remove(e.getKey(), e.getValue());
        }

        public int size() {
            return ConcurrentHashMap.this.size();
        }

        public boolean isEmpty() {
            return ConcurrentHashMap.this.isEmpty();
        }

        public void clear() {
            ConcurrentHashMap.this.clear();
        }
    }

}
```

