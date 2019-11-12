ConcurrentSkipListMap是线程安全的有序map，底层采用的是“跳表数据结构”，跳表的原理和实现可以看[SkipList-跳表](../../../算法/数据结构/SkipList-跳表.md)
实现原理简单来说就是利用跳表的元素有序性，在查找结点时找到前驱结点并判断前驱的后一个结点是不是要找的，插入时也是找到前驱结点，之后在前驱后面插入新结点，在插入结点时利用随机数判断是否需要添加该结点的Index（1/4的概率），如果需要则再根据随机数的二进制中1的个数（最低位的1不算）判断该结点的Index最高到第几层（最多32层），如果计算结果超过了当前最高层数则新增一层，否则在计算出来的每一层都新建一个Index。
下面是源码分析，主要逻辑是doGet、doPut、doRemove：

```java
package dhf;
import java.io.Serializable;
import java.util.AbstractCollection;
import java.util.AbstractMap;
import java.util.AbstractSet;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.NavigableMap;
import java.util.NavigableSet;
import java.util.NoSuchElementException;
import java.util.Set;
import java.util.SortedMap;
import java.util.SortedSet;
import java.util.Spliterator;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.BiFunction;
import java.util.function.Consumer;
import java.util.function.BiConsumer;
import java.util.function.Function;

/**
 * A scalable concurrent {@link ConcurrentNavigableMap} implementation.
 * The map is sorted according to the {@linkplain Comparable natural
 * ordering} of its keys, or by a {@link Comparator} provided at map
 * creation time, depending on which constructor is used.
 *
 * <p>This class implements a concurrent variant of <a
 * href="http://en.wikipedia.org/wiki/Skip_list" target="_top">SkipLists</a>
 * providing expected average <i>log(n)</i> time cost for the
 * {@code containsKey}, {@code get}, {@code put} and
 * {@code remove} operations and their variants.  Insertion, removal,
 * update, and access operations safely execute concurrently by
 * multiple threads.
 *
 * <p>Iterators and spliterators are
 * <a href="package-summary.html#Weakly"><i>weakly consistent</i></a>.
 *
 * <p>Ascending key ordered views and their iterators are faster than
 * descending ones.
 *
 * <p>All {@code Map.Entry} pairs returned by methods in this class
 * and its views represent snapshots of mappings at the time they were
 * produced. They do <em>not</em> support the {@code Entry.setValue}
 * method. (Note however that it is possible to change mappings in the
 * associated map using {@code put}, {@code putIfAbsent}, or
 * {@code replace}, depending on exactly which effect you need.)
 *
 * <p>Beware that, unlike in most collections, the {@code size}
 * method is <em>not</em> a constant-time operation. Because of the
 * asynchronous nature of these maps, determining the current number
 * of elements requires a traversal of the elements, and so may report
 * inaccurate results if this collection is modified during traversal.
 * Additionally, the bulk operations {@code putAll}, {@code equals},
 * {@code toArray}, {@code containsValue}, and {@code clear} are
 * <em>not</em> guaranteed to be performed atomically. For example, an
 * iterator operating concurrently with a {@code putAll} operation
 * might view only some of the added elements.
 *
 * <p>This class and its views and iterators implement all of the
 * <em>optional</em> methods of the {@link Map} and {@link Iterator}
 * interfaces. Like most other concurrent collections, this class does
 * <em>not</em> permit the use of {@code null} keys or values because some
 * null return values cannot be reliably distinguished from the absence of
 * elements.
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/../technotes/guides/collections/index.html">
 * Java Collections Framework</a>.
 *
 * @author Doug Lea
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 * @since 1.6
 */
public class ConcurrentSkipListMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentNavigableMap<K,V>, Cloneable, Serializable {
    /*
     * This class implements a tree-like two-dimensionally linked skip
     * list in which the index levels are represented in separate
     * nodes from the base nodes holding data.  There are two reasons
     * for taking this approach instead of the usual array-based
     * structure: 1) Array based implementations seem to encounter
     * more complexity and overhead 2) We can use cheaper algorithms
     * for the heavily-traversed index lists than can be used for the
     * base lists.  Here's a picture of some of the basics for a
     * possible list with 2 levels of index:
     *
     * Head nodes          Index nodes
     * +-+    right        +-+                      +-+
     * |2|---------------->| |--------------------->| |->null
     * +-+                 +-+                      +-+
     *  | down              |                        |
     *  v                   v                        v
     * +-+            +-+  +-+       +-+            +-+       +-+
     * |1|----------->| |->| |------>| |----------->| |------>| |->null
     * +-+            +-+  +-+       +-+            +-+       +-+
     *  v              |    |         |              |         |
     * Nodes  next     v    v         v              v         v
     * +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+
     * | |->|A|->|B|->|C|->|D|->|E|->|F|->|G|->|H|->|I|->|J|->|K|->null
     * +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+
     *
     * The base lists use a variant of the HM linked ordered set
     * algorithm. See Tim Harris, "A pragmatic implementation of
     * non-blocking linked lists"
     * http://www.cl.cam.ac.uk/~tlh20/publications.html and Maged
     * Michael "High Performance Dynamic Lock-Free Hash Tables and
     * List-Based Sets"
     * http://www.research.ibm.com/people/m/michael/pubs.htm.  The
     * basic idea in these lists is to mark the "next" pointers of
     * deleted nodes when deleting to avoid conflicts with concurrent
     * insertions, and when traversing to keep track of triples
     * (predecessor, node, successor) in order to detect when and how
     * to unlink these deleted nodes.
     *
     * Rather than using mark-bits to mark list deletions (which can
     * be slow and space-intensive using AtomicMarkedReference), nodes
     * use direct CAS'able next pointers.  On deletion, instead of
     * marking a pointer, they splice in another node that can be
     * thought of as standing for a marked pointer (indicating this by
     * using otherwise impossible field values).  Using plain nodes
     * acts roughly like "boxed" implementations of marked pointers,
     * but uses new nodes only when nodes are deleted, not for every
     * link.  This requires less space and supports faster
     * traversal. Even if marked references were better supported by
     * JVMs, traversal using this technique might still be faster
     * because any search need only read ahead one more node than
     * otherwise required (to check for trailing marker) rather than
     * unmasking mark bits or whatever on each read.
     *
     * This approach maintains the essential property needed in the HM
     * algorithm of changing the next-pointer of a deleted node so
     * that any other CAS of it will fail, but implements the idea by
     * changing the pointer to point to a different node, not by
     * marking it.  While it would be possible to further squeeze
     * space by defining marker nodes not to have key/value fields, it
     * isn't worth the extra type-testing overhead.  The deletion
     * markers are rarely encountered during traversal and are
     * normally quickly garbage collected. (Note that this technique
     * would not work well in systems without garbage collection.)
     *
     * In addition to using deletion markers, the lists also use
     * nullness of value fields to indicate deletion, in a style
     * similar to typical lazy-deletion schemes.  If a node's value is
     * null, then it is considered logically deleted and ignored even
     * though it is still reachable. This maintains proper control of
     * concurrent replace vs delete operations -- an attempted replace
     * must fail if a delete beat it by nulling field, and a delete
     * must return the last non-null value held in the field. (Note:
     * Null, rather than some special marker, is used for value fields
     * here because it just so happens to mesh with the Map API
     * requirement that method get returns null if there is no
     * mapping, which allows nodes to remain concurrently readable
     * even when deleted. Using any other marker value here would be
     * messy at best.)
     *
     * Here's the sequence of events for a deletion of node n with
     * predecessor b and successor f, initially:
     *
     *        +------+       +------+      +------+
     *   ...  |   b  |------>|   n  |----->|   f  | ...
     *        +------+       +------+      +------+
     *
     * 1. CAS n's value field from non-null to null.
     *    From this point on, no public operations encountering
     *    the node consider this mapping to exist. However, other
     *    ongoing insertions and deletions might still modify
     *    n's next pointer.
     *
     * 2. CAS n's next pointer to point to a new marker node.
     *    From this point on, no other nodes can be appended to n.
     *    which avoids deletion errors in CAS-based linked lists.
     *
     *        +------+       +------+      +------+       +------+
     *   ...  |   b  |------>|   n  |----->|marker|------>|   f  | ...
     *        +------+       +------+      +------+       +------+
     *
     * 3. CAS b's next pointer over both n and its marker.
     *    From this point on, no new traversals will encounter n,
     *    and it can eventually be GCed.
     *        +------+                                    +------+
     *   ...  |   b  |----------------------------------->|   f  | ...
     *        +------+                                    +------+
     *
     * A failure at step 1 leads to simple retry due to a lost race
     * with another operation. Steps 2-3 can fail because some other
     * thread noticed during a traversal a node with null value and
     * helped out by marking and/or unlinking.  This helping-out
     * ensures that no thread can become stuck waiting for progress of
     * the deleting thread.  The use of marker nodes slightly
     * complicates helping-out code because traversals must track
     * consistent reads of up to four nodes (b, n, marker, f), not
     * just (b, n, f), although the next field of a marker is
     * immutable, and once a next field is CAS'ed to point to a
     * marker, it never again changes, so this requires less care.
     *
     * Skip lists add indexing to this scheme, so that the base-level
     * traversals start close to the locations being found, inserted
     * or deleted -- usually base level traversals only traverse a few
     * nodes. This doesn't change the basic algorithm except for the
     * need to make sure base traversals start at predecessors (here,
     * b) that are not (structurally) deleted, otherwise retrying
     * after processing the deletion.
     *
     * Index levels are maintained as lists with volatile next fields,
     * using CAS to link and unlink.  Races are allowed in index-list
     * operations that can (rarely) fail to link in a new index node
     * or delete one. (We can't do this of course for data nodes.)
     * However, even when this happens, the index lists remain sorted,
     * so correctly serve as indices.  This can impact performance,
     * but since skip lists are probabilistic anyway, the net result
     * is that under contention, the effective "p" value may be lower
     * than its nominal value. And race windows are kept small enough
     * that in practice these failures are rare, even under a lot of
     * contention.
     *
     * The fact that retries (for both base and index lists) are
     * relatively cheap due to indexing allows some minor
     * simplifications of retry logic. Traversal restarts are
     * performed after most "helping-out" CASes. This isn't always
     * strictly necessary, but the implicit backoffs tend to help
     * reduce other downstream failed CAS's enough to outweigh restart
     * cost.  This worsens the worst case, but seems to improve even
     * highly contended cases.
     *
     * Unlike most skip-list implementations, index insertion and
     * deletion here require a separate traversal pass occurring after
     * the base-level action, to add or remove index nodes.  This adds
     * to single-threaded overhead, but improves contended
     * multithreaded performance by narrowing interference windows,
     * and allows deletion to ensure that all index nodes will be made
     * unreachable upon return from a public remove operation, thus
     * avoiding unwanted garbage retention. This is more important
     * here than in some other data structures because we cannot null
     * out node fields referencing user keys since they might still be
     * read by other ongoing traversals.
     *
     * Indexing uses skip list parameters that maintain good search
     * performance while using sparser-than-usual indices: The
     * hardwired parameters k=1, p=0.5 (see method doPut) mean
     * that about one-quarter of the nodes have indices. Of those that
     * do, half have one level, a quarter have two, and so on (see
     * Pugh's Skip List Cookbook, sec 3.4).  The expected total space
     * requirement for a map is slightly less than for the current
     * implementation of java.util.TreeMap.
     *
     * Changing the level of the index (i.e, the height of the
     * tree-like structure) also uses CAS. The head index has initial
     * level/height of one. Creation of an index with height greater
     * than the current level adds a level to the head index by
     * CAS'ing on a new top-most head. To maintain good performance
     * after a lot of removals, deletion methods heuristically try to
     * reduce the height if the topmost levels appear to be empty.
     * This may encounter races in which it possible (but rare) to
     * reduce and "lose" a level just as it is about to contain an
     * index (that will then never be encountered). This does no
     * structural harm, and in practice appears to be a better option
     * than allowing unrestrained growth of levels.
     *
     * The code for all this is more verbose than you'd like. Most
     * operations entail locating an element (or position to insert an
     * element). The code to do this can't be nicely factored out
     * because subsequent uses require a snapshot of predecessor
     * and/or successor and/or value fields which can't be returned
     * all at once, at least not without creating yet another object
     * to hold them -- creating such little objects is an especially
     * bad idea for basic internal search operations because it adds
     * to GC overhead.  (This is one of the few times I've wished Java
     * had macros.) Instead, some traversal code is interleaved within
     * insertion and removal operations.  The control logic to handle
     * all the retry conditions is sometimes twisty. Most search is
     * broken into 2 parts. findPredecessor() searches index nodes
     * only, returning a base-level predecessor of the key. findNode()
     * finishes out the base-level search. Even with this factoring,
     * there is a fair amount of near-duplication of code to handle
     * variants.
     *
     * To produce random values without interference across threads,
     * we use within-JDK thread local random support (via the
     * "secondary seed", to avoid interference with user-level
     * ThreadLocalRandom.)
     *
     * A previous version of this class wrapped non-comparable keys
     * with their comparators to emulate Comparables when using
     * comparators vs Comparables.  However, JVMs now appear to better
     * handle infusing comparator-vs-comparable choice into search
     * loops. Static method cpr(comparator, x, y) is used for all
     * comparisons, which works well as long as the comparator
     * argument is set up outside of loops (thus sometimes passed as
     * an argument to internal methods) to avoid field re-reads.
     *
     * For explanation of algorithms sharing at least a couple of
     * features with this one, see Mikhail Fomitchev's thesis
     * (http://www.cs.yorku.ca/~mikhail/), Keir Fraser's thesis
     * (http://www.cl.cam.ac.uk/users/kaf24/), and Hakan Sundell's
     * thesis (http://www.cs.chalmers.se/~phs/).
     *
     * Given the use of tree-like index nodes, you might wonder why
     * this doesn't use some kind of search tree instead, which would
     * support somewhat faster search operations. The reason is that
     * there are no known efficient lock-free insertion and deletion
     * algorithms for search trees. The immutability of the "down"
     * links of index nodes (as opposed to mutable "left" fields in
     * true trees) makes this tractable using only CAS operations.
     *
     * Notation guide for local variables
     * Node:         b, n, f    for  predecessor, node, successor
     * Index:        q, r, d    for index node, right, down.
     *               t          for another index node
     * Head:         h
     * Levels:       j
     * Keys:         k, key
     * Values:       v, value
     * Comparisons:  c
     */

    private static final long serialVersionUID = -8627078645895051609L;

    /**
     * Special value used to identify base-level header
     */
    private static final Object BASE_HEADER = new Object();

    /**
     * The topmost head index of the skiplist.
     */
    private transient volatile HeadIndex<K,V> head;

    /**
     * The comparator used to maintain order in this map, or null if
     * using natural ordering.  (Non-private to simplify access in
     * nested classes.)
     * @serial
     */
    final Comparator<? super K> comparator;

    /** Lazily initialized key set */
    private transient KeySet<K> keySet;
    /** Lazily initialized entry set */
    private transient EntrySet<K,V> entrySet;
    /** Lazily initialized values collection */
    private transient Values<V> values;
    /** Lazily initialized descending key set */
    private transient ConcurrentNavigableMap<K,V> descendingMap;

    /**
     * Initializes or resets state. Needed by constructors, clone,
     * clear, readObject. and ConcurrentSkipListSet.clone.
     * (Note that comparator must be separately initialized.)
     */
    private void initialize() {
        keySet = null;
        entrySet = null;
        values = null;
        descendingMap = null;
        // 初始化最底层
        head = new HeadIndex<K,V>(new Node<K,V>(null, BASE_HEADER, null),
                                  null, null, 1);
    }

    /**
     * compareAndSet head node
     */
    private boolean casHead(HeadIndex<K,V> cmp, HeadIndex<K,V> val) {
        return UNSAFE.compareAndSwapObject(this, headOffset, cmp, val);
    }

    /* ---------------- Nodes -------------- */

    /**
     * Nodes hold keys and values, and are singly linked in sorted
     * order, possibly with some intervening marker nodes. The list is
     * headed by a dummy node accessible as head.node. The value field
     * is declared only as Object because it takes special non-V
     * values for marker and header nodes.
     */
    static final class Node<K,V> {
        final K key;
        volatile Object value;
        volatile Node<K,V> next;

        /**
         * Creates a new regular node.
         */
        Node(K key, Object value, Node<K,V> next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }

        /**
         * Creates a new marker node. A marker is distinguished by
         * having its value field point to itself.  Marker nodes also
         * have null keys, a fact that is exploited in a few places,
         * but this doesn't distinguish markers from the base-level
         * header node (head.node), which also has a null key.
         */
        Node(Node<K,V> next) {
            this.key = null;
            this.value = this;
            this.next = next;
        }

        /**
         * compareAndSet value field
         */
        boolean casValue(Object cmp, Object val) {
            return UNSAFE.compareAndSwapObject(this, valueOffset, cmp, val);
        }

        /**
         * compareAndSet next field
         */
        boolean casNext(Node<K,V> cmp, Node<K,V> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        /**
         * Returns true if this node is a marker. This method isn't
         * actually called in any current code checking for markers
         * because callers will have already read value field and need
         * to use that read (not another done here) and so directly
         * test if value points to node.
         *
         * @return true if this node is a marker node
         */
        boolean isMarker() {
            return value == this;
        }

        /**
         * Returns true if this node is the header of base-level list.
         * @return true if this node is header node
         */
        boolean isBaseHeader() {
            return value == BASE_HEADER;
        }

        /**
         * Tries to append a deletion marker to this node.
         * @param f the assumed current successor of this node
         * @return true if successful
         */
        // 给准备删除的结点添加标记，作用可以看doRemove方法
        boolean appendMarker(Node<K,V> f) {
            return casNext(f, new Node<K,V>(f));
        }

        /**
         * Helps out a deletion by appending marker or unlinking from
         * predecessor. This is called during traversals when value
         * field seen to be null.
         * @param b predecessor
         * @param f successor
         */
        void helpDelete(Node<K,V> b, Node<K,V> f) {
            /*
             * Rechecking links and then doing only one of the
             * help-out stages per call tends to minimize CAS
             * interference among helping threads.
             */
            if (f == next && this == b.next) {
                // f == null即表示mark结点还没有被添加，即doRemove方法里的情况1，并且n刚好是链表的最后一个结点，导致n本来的next == null
                // 而f.value != f表示n不是最后一个结点，情况1的appendMarker执行失败使得n的next还是一个普通结点，符合这两个条件的任意一个都再次mark，mark
                // 之后不再执行其他操作，下次结点n再次被发现的时候就会被删除了，而casNext即使失败也没关系，其他线程发现n的时候还是会再次执行这一逻辑
                if (f == null || f.value != f) // not already marked
                    casNext(f, new Node<K,V>(f));
                else
                    // 如果已经mark则直接更新b.next为f.next即可，f就是mark结点，f.next指向的是n原先的next
                    b.casNext(this, f.next);
            }
        }

        /**
         * Returns value if this node contains a valid key-value pair,
         * else null.
         * @return this node's value if it isn't a marker or header or
         * is deleted, else null
         */
        V getValidValue() {
            Object v = value;
            if (v == this || v == BASE_HEADER)
                return null;
            @SuppressWarnings("unchecked") V vv = (V)v;
            return vv;
        }

        /**
         * Creates and returns a new SimpleImmutableEntry holding current
         * mapping if this node holds a valid value, else null.
         * @return new entry or null
         */
        AbstractMap.SimpleImmutableEntry<K,V> createSnapshot() {
            Object v = value;
            if (v == null || v == this || v == BASE_HEADER)
                return null;
            @SuppressWarnings("unchecked") V vv = (V)v;
            return new AbstractMap.SimpleImmutableEntry<K,V>(key, vv);
        }

        // UNSAFE mechanics

        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("value"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    /* ---------------- Indexing -------------- */

    /**
     * Index nodes represent the levels of the skip list.  Note that
     * even though both Nodes and Indexes have forward-pointing
     * fields, they have different types and are handled in different
     * ways, that can't nicely be captured by placing field in a
     * shared abstract class.
     */
    static class Index<K,V> {
        final Node<K,V> node;
        final Index<K,V> down;
        volatile Index<K,V> right;

        /**
         * Creates index node with given values.
         */
        Index(Node<K,V> node, Index<K,V> down, Index<K,V> right) {
            this.node = node;
            this.down = down;
            this.right = right;
        }

        /**
         * compareAndSet right field
         */
        final boolean casRight(Index<K,V> cmp, Index<K,V> val) {
            return UNSAFE.compareAndSwapObject(this, rightOffset, cmp, val);
        }

        /**
         * Returns true if the node this indexes has been deleted.
         * @return true if indexed node is known to be deleted
         */
        final boolean indexesDeletedNode() {
            return node.value == null;
        }

        /**
         * Tries to CAS newSucc as successor.  To minimize races with
         * unlink that may lose this index node, if the node being
         * indexed is known to be deleted, it doesn't try to link in.
         * @param succ the expected current successor
         * @param newSucc the new successor
         * @return true if successful
         */
        // 在当前Index和其后继之间插入一个结点
        final boolean link(Index<K,V> succ, Index<K,V> newSucc) {
            Node<K,V> n = node;
            newSucc.right = succ;
            // 在更新right之前判断当前Index是否被删除了，这一步并不是必须的，因为即使n.value == null执行casRight(succ, newSucc)也没啥
            // 问题，而且并发环境下，前面的判断n.value != null满足了也不能保证casRight(succ, newSucc)执行时还是满足的，这里之所以这么
            // 判断是因为当出现n.value == null时该方法执行失败，则调用link方法的函数会重新循环，并调用findPredecessor清理Index
            return n.value != null && casRight(succ, newSucc);
        }

        /**
         * Tries to CAS right field to skip over apparent successor
         * succ.  Fails (forcing a retraversal by caller) if this node
         * is known to be deleted.
         * @param succ the expected current successor
         * @return true if successful
         */
        final boolean unlink(Index<K,V> succ) {
            return node.value != null && casRight(succ, succ.right);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long rightOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Index.class;
                rightOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("right"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    /* ---------------- Head nodes -------------- */

    /**
     * Nodes heading each level keep track of their level.
     */
    static final class HeadIndex<K,V> extends Index<K,V> {
        final int level;
        HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
            super(node, down, right);
            this.level = level;
        }
    }

    /* ---------------- Comparison utilities -------------- */

    /**
     * Compares using comparator or natural ordering if null.
     * Called only by methods that have performed required type checks.
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    static final int cpr(Comparator c, Object x, Object y) {
        return (c != null) ? c.compare(x, y) : ((Comparable)x).compareTo(y);
    }

    /* ---------------- Traversal -------------- */

    /**
     * Returns a base-level node with key strictly less than given key,
     * or the base-level header if there is no such node.  Also
     * unlinks indexes to deleted nodes found along the way.  Callers
     * rely on this side-effect of clearing indices to deleted nodes.
     * @param key the key
     * @return a predecessor of key
     */
    // 返回指定key的前驱结点
    private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        for (;;) {
            // Index表示某一层，每一层也是个单链表，这里从最顶层开始
            for (Index<K,V> q = head, r = q.right, d;;) {
                if (r != null) {
                    Node<K,V> n = r.node;
                    K k = n.key;
                    // 如果结点的value是空的则更新q.right为r.right，即从单链表中删除该Index
                    if (n.value == null) {
                        if (!q.unlink(r))
                            break;           // restart
                        // 如果更新成功则继续从下一个结点开始
                        r = q.right;         // reread r
                        continue;
                    }
                    // 如果寻找的key大于当前结点的key则再往由前进一个结点
                    if (cpr(cmp, key, k) > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
                // 执行到这说明r要么指向当前层最后一个结点（因为r == null），要么指向key比要找的key小或相等的结点（而且是最大的那个）
                // 这里判断是不是到了最底层，如果是则返回r的前一个结点q，r的前驱也是key的前驱，否则再向下一层
                if ((d = q.down) == null)
                    return q.node;
                // 更新q和r重新循环
                q = d;
                r = d.right;
            }
        }
    }

    /**
     * Returns node holding key or null if no such, clearing out any
     * deleted nodes seen along the way.  Repeatedly traverses at
     * base-level looking for key starting at predecessor returned
     * from findPredecessor, processing base-level deletions as
     * encountered. Some callers rely on this side-effect of clearing
     * deleted nodes.
     *
     * Restarts occur, at traversal step centered on node n, if:
     *
     *   (1) After reading n's next field, n is no longer assumed
     *       predecessor b's current successor, which means that
     *       we don't have a consistent 3-node snapshot and so cannot
     *       unlink any subsequent deleted nodes encountered.
     *
     *   (2) n's value field is null, indicating n is deleted, in
     *       which case we help out an ongoing structural deletion
     *       before retrying.  Even though there are cases where such
     *       unlinking doesn't require restart, they aren't sorted out
     *       here because doing so would not usually outweigh cost of
     *       restarting.
     *
     *   (3) n is a marker or n's predecessor's value field is null,
     *       indicating (among other possibilities) that
     *       findPredecessor returned a deleted node. We can't unlink
     *       the node because we don't know its predecessor, so rely
     *       on another call to findPredecessor to notice and return
     *       some earlier predecessor, which it will do. This check is
     *       only strictly needed at beginning of loop, (and the
     *       b.value check isn't strictly needed at all) but is done
     *       each iteration to help avoid contention with other
     *       threads by callers that will fail to be able to change
     *       links, and so will retry anyway.
     *
     * The traversal loops in doPut, doRemove, and findNear all
     * include the same three kinds of checks. And specialized
     * versions appear in findFirst, and findLast and their
     * variants. They can't easily share code because each uses the
     * reads of fields held in locals occurring in the orders they
     * were performed.
     *
     * @param key the key
     * @return node holding key, or null if no such
     */
    // 该方法在很多地方都会被调用，但是doRemove方法中最具代表意义，只要说明doRemove方法里的情况就行了，其他方法也是一样的
    private Node<K,V> findNode(Object key) {
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            /*
             根据doRemove里的描述，findPredecessor会删除被删除结点的Index，并且返回如下情况的b
             对于doRemove的情况1
             b -> n -> f
             对于doRemove的情况2
             b -> n -> mark -> f
              */
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                if (n != b.next)                // inconsistent read
                    break;
                // n的value在doRemove方法中被设置成了null
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)  // b is deleted
                    break;
                if ((c = cpr(cmp, key, n.key)) == 0)
                    return n;
                if (c < 0)
                    break outer;
                b = n;
                n = f;
            }
        }
        return null;
    }

    /**
     * Gets value for key. Almost the same as findNode, but returns
     * the found value (to avoid retries during re-reads)
     *
     * @param key the key
     * @return the value, or null if absent
     */
    private V doGet(Object key) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            // 获取前驱，在ConcurrentSkipListMap为空的情况下返回的是BASE_HEADER
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 如果n为空说明前驱的next是null，说明key对应的结点不存在
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                // 检查是否被其他线程修改过
                if (n != b.next)                // inconsistent read
                    break;
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)  // b is deleted
                    break;
                // 判断n是否就是要找的结点
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }
                if (c < 0)
                    break outer;
                // 执行到这说明key大于前驱的后面一个结点的key（在没有其他线程干扰的情况下这种情况是不可能的），此时往后移动一个结点
                b = n;
                n = f;
            }
        }
        return null;
    }

    /* ---------------- Insertion -------------- */

    /**
     * Main insertion method.  Adds element if not present, or
     * replaces value if present and onlyIfAbsent is false.
     * @param key the key
     * @param value the value that must be associated with key
     * @param onlyIfAbsent if should not insert if already present
     * @return the old value, or null if newly inserted
     */
    private V doPut(K key, V value, boolean onlyIfAbsent) {
        Node<K,V> z;             // added node
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                if (n != null) {
                    Object v; int c;
                    Node<K,V> f = n.next;
                    // 由于是并发环境，所以这里判断是否出现了不一致
                    if (n != b.next)               // inconsistent read
                        break;
                    // 如果n的value为空则说明该结点被删除了
                    if ((v = n.value) == null) {   // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    if (b.value == null || v == n) // b is deleted
                        break;
                    // 如果前驱的后继的key小于要找的key则向后移一个结点再来一次
                    if ((c = cpr(cmp, key, n.key)) > 0) {
                        b = n;
                        n = f;
                        continue;
                    }
                    // 如果相等则说明n就是要找的结点，这里覆盖原来的值
                    if (c == 0) {
                        if (onlyIfAbsent || n.casValue(v, value)) {
                            @SuppressWarnings("unchecked") V vv = (V)v;
                            return vv;
                        }
                        // 如果更新失败则再次循环
                        break; // restart if lost race to replace value
                    }
                    // else c < 0; fall through
                }

                // 如果前驱的next为null或找到了正确的前驱则直接新建一个结点并设置为b的next为新结点的next
                z = new Node<K,V>(key, value, n);
                // 如更新失败则重新循环
                if (!b.casNext(n, z))
                    break;         // restart if lost race to append to b
                // 更新成功则结束当前方法
                break outer;
            }
        }

        int rnd = ThreadLocalRandom.nextSecondarySeed();
        // 用随机数判断是否需要将结点提升到基础层以上，最高位和最低位都为0的情况下才提升，即1/4的概率
        // 如果不需要判断则直接返回null，因为新建的node已经在上面插入到了最底层数据结点层的正确位置了
        if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
            int level = 1, max;
            // 计算rnd二进制位中1的个数
            while (((rnd >>>= 1) & 1) != 0)
                ++level;
            Index<K,V> idx = null;
            HeadIndex<K,V> h = head;
            // 如果计算出来的level没有找过当前最高level则直接新建level个Index
            if (level <= (max = h.level)) {
                for (int i = 1; i <= level; ++i)
                    // z为刚才创建的结点，除了HeadIndex，所有Index的node都指向普通结点（即最底层的数据结点）
                    // 每一层的down都是level - 1层的Index
                    idx = new Index<K,V>(z, idx, null);
            }
            else { // try to grow by one level
                // 如果level超过了当前的最高level，则新增一层
                level = max + 1; // hold in array and later pick the one to use
                // 层数是从1开始计算的，所以这里新建Index的数组大小为level + 1
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                // 和上面一样，初始化level个Index
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                for (;;) {
                    h = head;
                    int oldLevel = h.level;
                    // 如果level <= oldLevel说明有其他线程添加了一层，这里就重新开始循环
                    if (level <= oldLevel) // lost race to add level
                        break;
                    HeadIndex<K,V> newh = h;
                    // HeadIndex的node是共享的，都是BASE_HEADER
                    Node<K,V> oldbase = h.node;
                    // 正常情况下都是level = oldLevel+1，作用这里相当于创建一个HeadIndex，并将down指向原先的最高层HeadIndex
                    // right指向刚创建Index数组中的最后一个
                    for (int j = oldLevel+1; j <= level; ++j)
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    // 更新新的HeadIndex为head
                    if (casHead(h, newh)) {
                        h = newh;
                        // 下面splice后面的循环是为了将新建的Index添加到对应的层，这里更新level为旧的level，因为最高的一层已经创建并执行了right了，
                        // 所以不需要再添加了，idx也更新为现在第二高的Index
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
            // 下面将新建的Index插入到对应的层
            // find insertion points and splice in
            splice: for (int insertionLevel = level;;) {
                int j = h.level;
                // 从最高层开始遍历
                for (Index<K,V> q = h, r = q.right, t = idx;;) {
                    if (q == null || t == null)
                        break splice;
                    // r为当前遍历的Index的右边的Index，如果不为空则在下面判断r的node的key是否大于新建的结点的key
                    if (r != null) {
                        Node<K,V> n = r.node;
                        // compare before deletion check avoids needing recheck
                        int c = cpr(cmp, key, n.key);
                        if (n.value == null) {
                            if (!q.unlink(r))
                                break;
                            r = q.right;
                            continue;
                        }
                        // 向右前进一位
                        if (c > 0) {
                            q = r;
                            r = r.right;
                            continue;
                        }
                    }

                    // j为当前遍历的层数，insertionLevel为新建的当前新建结点的Index中的最高层数
                    // 遍历到这时q指向当前Index应该插入的位置之前的位置，r指向Index应该插入的位置之前的位置，r可能为null
                    if (j == insertionLevel) {
                        // 如果cas不成功或q已经被删除了则重新循环一遍
                        if (!q.link(r, t))
                            break; // restart
                        // 如果结点被删除了则调用findNode清空Index和node
                        if (t.node.value == null) {
                            findNode(key);
                            break splice;
                        }
                        // 更新insertionLevel为下一个需要插入的层数，Index最底层为1，这里判断是否到了最底层
                        if (--insertionLevel == 0)
                            break splice;
                    }

                    // 更新j为下一个遍历的层数，j的初始值是最高层的层数，而insertionLevel的初始化在level小于等于原先
                    // 最高层数情况下是通过rnd计算出来的level，此时j肯定大于等于insertionLevel，而在计算出来的level大于
                    // 原先最高层数情况下insertionLevel等于新增一层HeadIndex后的第二高层，即insertionLevel = j - 1
                    // 而在这个if之前还有个判断是j == insertionLevel，满足的情况下会执行--insertionLevel，所以这里的
                    //--j >= insertionLevel是一直成立的，而j < level时更新t为下一层的Index即可（j = level已经在上面的if中处理过了）
                    if (--j >= insertionLevel && j < level)
                        t = t.down;
                    // 如果j >= level说明还没有到应该插入Inde下的层，则向下一层重新循环
                    q = q.down;
                    r = q.right;
                }
            }
        }
        return null;
    }

    /* ---------------- Deletion -------------- */

    /**
     * Main deletion method. Locates node, nulls value, appends a
     * deletion marker, unlinks predecessor, removes associated index
     * nodes, and possibly reduces head index level.
     *
     * Index nodes are cleared out simply by calling findPredecessor.
     * which unlinks indexes to deleted nodes found along path to key,
     * which will include the indexes to this node.  This is done
     * unconditionally. We can't check beforehand whether there are
     * index nodes because it might be the case that some or all
     * indexes hadn't been inserted yet for this node during initial
     * search for it, and we'd like to ensure lack of garbage
     * retention, so must call to be sure.
     *
     * @param key the key
     * @param value if non-null, the value that must be
     * associated with key
     * @return the node, or null if not found
     */
    final V doRemove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 如果n为空说明没有需要删除的key对应的结点
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                // 判断其他线程是否修改过b.next
                if (n != b.next)                    // inconsistent read
                    break;
                if ((v = n.value) == null) {        // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                // 如果b被删除了则再次循环，循环时调用的findPredecessor方法会清理被删除的Index
                if (b.value == null || v == n)      // b is deleted
                    break;
                // 如果key < n.key说明不存在想要删除的结点（因为b是key的前驱，所以n.key >= key，如果不等于的话则说明不存在key这个结点），直接返回
                if ((c = cpr(cmp, key, n.key)) < 0)
                    break outer;
                // 如果c > 0说明有其他线程更新了b.next，这里向前前进一个结点再次循环
                if (c > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                // 在传入的value不为空的情况下判断key对应的value是否等于value，等于才删除
                if (value != null && !value.equals(v))
                    break outer;
                // 更新结点的value为null，表示删除了
                if (!n.casValue(v, null))
                    break;
                /* casNext操作更新b的next为n的next，即相当于删除了n结点，但是在删除n结点之前有个appendMarker操作，这个操作的目的是防止
                 误删了其他线程刚刚插入在n结点后面的结点，查看appendMarker方法可以发现，该方法是更新n的next为new Node(f)，而new Node(f)
                 相当于是个标记结点，如果没有标记结点，直接执行casNext，可能出现的情况如下，b、n、f的关系和当前代码的关系一样，现在需要删除
                 n结点，只需要更新b.next为f即可
                 更新前：
                 b -> n -> f
                 更新后：
                      n
                      |
                      v
                 b -> f
                 但是在更新前另一个线程调用doPut方法插入一个结点到n的后面，即在更新前发现要插入的结点的前驱是n，则会执行n.casNext(f, new Node(d))
                 无论这个n.casNext发生在上面的b.casNext之前还是之后，只要在b.casNext前doPut方法定位到了n结点，那么这两个casNext方法都能执行成功，
                 因为一个更新的是b.next，一个更新的是n.next，互不冲突，这样会导致的情况是，在两个casNext执行结束后，情况如下：
                      n
                      |
                      v
                      d
                      |
                      v
                 b -> f
                 这回导致新插入的结点d也被删除了，所以需要调用appendMarker方法避免这种情况，避免的原理是：
                 还是刚才的背景，不同的是像下面if一样，在执行b.casNext之前执行appendMarker，执行appendMarker后更新n的next为mark，情况如下：
                      n
                      |
                      v
                     mark
                      |
                      v
                 b -> f
                 假设第二个线程还是已经定位到了前驱n，将要执行n.casNext(f, new Node(d))，那么可能出现的情况是：
                 1. n.casNext(f, new Node(d))发生在下面appendMarker之前，则appendMarker方法调用失败，因为n的next变成了d，所以if条件满足，执行findNode方法
                 2. n.casNext(f, new Node(d))发生在下面appendMarker之后，b.casNext(n, f)之前，即成功添加了mark结点，但是更新b的next失败了，
                 此时n.casNext(f, new Node(d))将执行失败因为n.next变成了mark，则doPut方法会break内层循环并从新开始循环，不出意外的话这次的循环的前驱将是b，再次casNext就可以了
                 情况2发生时如下：
                 b -> n-> mark -> f
                 对于情况2，还是会调用findNode方法
                 3. n.casNext(f, new Node(d))发生在下面appendMarker之后，b.casNext(n, f)之后，和情况2一样，doPut方法break重新循环就可以

                 所以接下来要着重看findNode方法
                 */
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    findNode(key);                  // retry via findNode
                else {
                    // 上面设置了n的value==null，而findPredecessor方法会检查每个Inde的node的value是否为null，如果是则删除该Index，
                    // 所以在上述情况3发生时直接通过findPredecessor方法清除Index即可
                    findPredecessor(key, cmp);      // clean index
                    // 如果最高层只有HeadIndex了则尝试删除一层
                    if (head.right == null)
                        tryReduceLevel();
                }
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
        return null;
    }

    /**
     * Possibly reduce head level if it has no nodes.  This method can
     * (rarely) make mistakes, in which case levels can disappear even
     * though they are about to contain index nodes. This impacts
     * performance, not correctness.  To minimize mistakes as well as
     * to reduce hysteresis, the level is reduced by one only if the
     * topmost three levels look empty. Also, if the removed level
     * looks non-empty after CAS, we try to change it back quick
     * before anyone notices our mistake! (This trick works pretty
     * well because this method will practically never make mistakes
     * unless current thread stalls immediately before first CAS, in
     * which case it is very unlikely to stall again immediately
     * afterwards, so will recover.)
     *
     * We put up with all this rather than just let levels grow
     * because otherwise, even a small map that has undergone a large
     * number of insertions and removals will have a lot of levels,
     * slowing down access more than would an occasional unwanted
     * reduction.
     */
    private void tryReduceLevel() {
        HeadIndex<K,V> h = head;
        HeadIndex<K,V> d;
        HeadIndex<K,V> e;
        // 只有在超过3层的情况下才清除
        if (h.level > 3 &&
            (d = (HeadIndex<K,V>)h.down) != null &&
            (e = (HeadIndex<K,V>)d.down) != null &&
            e.right == null && // 连续3层是空的才删除
            d.right == null &&
            h.right == null &&
            casHead(h, d) && // try to set // 更新d为新的head，即删除一层
            h.right != null) // recheck // 在执行casHead前可能会在最高层插入结点，这里判断原先的最高层的next是否为null，如果不是则还原head
            casHead(d, h);   // try to backout
    }

    /* ---------------- Finding and removing first element -------------- */

    /**
     * Specialized variant of findNode to get first valid node.
     * @return first node or null if empty
     */
    final Node<K,V> findFirst() {
        for (Node<K,V> b, n;;) {
            // head的node为BASE_HEADER，next为最小的结点
            if ((n = (b = head.node).next) == null)
                return null;
            // 如果结点没有被删除则返回结点
            if (n.value != null)
                return n;
            // 否则删除该结点并重新开始循环
            n.helpDelete(b, n.next);
        }
    }

    /**
     * Removes first entry; returns its snapshot.
     * @return null if empty, else snapshot of first entry
     */
    private Map.Entry<K,V> doRemoveFirstEntry() {
        for (Node<K,V> b, n;;) {
            // 如果已经被删除了则直接返回
            if ((n = (b = head.node).next) == null)
                return null;
            Node<K,V> f = n.next;
            // 检查是否被其他线程修改了
            if (n != b.next)
                continue;
            Object v = n.value;
            // 如果n已经被删除了则调用helpDelete并再次循环
            if (v == null) {
                n.helpDelete(b, f);
                continue;
            }
            // 删除n结点
            if (!n.casValue(v, null))
                continue;
            // 和doRemove一样的逻辑，区别是这里调用的是findFirst，没必要调用findPredecessor查找前驱了，前驱就是BASE_HERDER
            if (!n.appendMarker(f) || !b.casNext(n, f))
                findFirst(); // retry
            clearIndexToFirst();
            @SuppressWarnings("unchecked") V vv = (V)v;
            return new AbstractMap.SimpleImmutableEntry<K,V>(n.key, vv);
        }
    }

    /**
     * Clears out index nodes associated with deleted first entry.
     */
    private void clearIndexToFirst() {
        for (;;) {
            for (Index<K,V> q = head;;) {
                Index<K,V> r = q.right;
                // 如果r将被删除的Index不为空并且其node已经被删除，则调用unlink从单链表中删除Index，如果失败了则重新循环
                if (r != null && r.indexesDeletedNode() && !q.unlink(r))
                    break;
                // 或者r == null或者r的结点没有被删除或者unlink执行成功，则判断是否到了最后一层
                if ((q = q.down) == null) {
                    // 如果最高层为空则尝试删除删除一层Index
                    if (head.right == null)
                        tryReduceLevel();
                    return;
                }
            }
        }
    }

    /**
     * Removes last entry; returns its snapshot.
     * Specialized variant of doRemove.
     * @return null if empty, else snapshot of last entry
     */
    private Map.Entry<K,V> doRemoveLastEntry() {
        for (;;) {
            Node<K,V> b = findPredecessorOfLast();
            Node<K,V> n = b.next;
            if (n == null) {
                if (b.isBaseHeader())               // empty
                    return null;
                else
                    continue; // all b's successors are deleted; retry
            }
            for (;;) {
                Node<K,V> f = n.next;
                if (n != b.next)                    // inconsistent read
                    break;
                Object v = n.value;
                if (v == null) {                    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)      // b is deleted
                    break;
                if (f != null) {
                    b = n;
                    n = f;
                    continue;
                }
                if (!n.casValue(v, null))
                    break;
                K key = n.key;
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    findNode(key);                  // retry via findNode
                else {                              // clean index
                    findPredecessor(key, comparator);
                    if (head.right == null)
                        tryReduceLevel();
                }
                @SuppressWarnings("unchecked") V vv = (V)v;
                return new AbstractMap.SimpleImmutableEntry<K,V>(key, vv);
            }
        }
    }

    /* ---------------- Finding and removing last element -------------- */

    /**
     * Specialized version of find to get last valid node.
     * @return last node or null if empty
     */
    final Node<K,V> findLast() {
        /*
         * findPredecessor can't be used to traverse index level
         * because this doesn't use comparisons.  So traversals of
         * both levels are folded together.
         */
        Index<K,V> q = head;
        for (;;) {
            Index<K,V> d, r;
            if ((r = q.right) != null) {
                if (r.indexesDeletedNode()) {
                    q.unlink(r);
                    q = head; // restart
                }
                else
                    q = r;
            } else if ((d = q.down) != null) {
                q = d;
            } else {
                for (Node<K,V> b = q.node, n = b.next;;) {
                    if (n == null)
                        return b.isBaseHeader() ? null : b;
                    Node<K,V> f = n.next;            // inconsistent read
                    if (n != b.next)
                        break;
                    Object v = n.value;
                    if (v == null) {                 // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    if (b.value == null || v == n)      // b is deleted
                        break;
                    b = n;
                    n = f;
                }
                q = head; // restart
            }
        }
    }

    /**
     * Specialized variant of findPredecessor to get predecessor of last
     * valid node.  Needed when removing the last entry.  It is possible
     * that all successors of returned node will have been deleted upon
     * return, in which case this method can be retried.
     * @return likely predecessor of last node
     */
    private Node<K,V> findPredecessorOfLast() {
        for (;;) {
            for (Index<K,V> q = head;;) {
                Index<K,V> d, r;
                if ((r = q.right) != null) {
                    // 如果r的node已经被删除则删除r并重新循环
                    if (r.indexesDeletedNode()) {
                        q.unlink(r);
                        break;    // must restart
                    }
                    // proceed as far across as possible without overshooting
                    // 如果r的node的下一个结点不为空则当前Index向下走一个，否则r的结点就是最后一个结点，此时q指向r的前驱
                    if (r.node.next != null) {
                        q = r;
                        continue;
                    }
                }
                // 到这里说明r已经指向了当前Index层的最后一个结点，这里向下一层并重新循环
                if ((d = q.down) != null)
                    q = d;
                else // 循环直到最后一层为止
                    return q.node;
            }
        }
    }

    /* ---------------- Relational operations -------------- */

    // Control values OR'ed as arguments to findNear

    private static final int EQ = 1;
    private static final int LT = 2;
    private static final int GT = 0; // Actually checked as !LT

    /**
     * Utility for ceiling, floor, lower, higher methods.
     * @param key the key
     * @param rel the relation -- OR'ed combination of EQ, LT, GT
     * @return nearest node fitting relation, or null if no such
     */
    final Node<K,V> findNear(K key, int rel, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException();
        for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v;
                // n == null的情况下，如果需要的是大于或等于key的（(rel & LT) == 0表示想要的结点不小于key）或b是BASE_HEADER则返回null，否则返回b
                if (n == null)
                    return ((rel & LT) == 0 || b.isBaseHeader()) ? null : b;
                Node<K,V> f = n.next;
                if (n != b.next)                  // inconsistent read
                    break;
                if ((v = n.value) == null) {      // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)      // b is deleted
                    break;
                int c = cpr(cmp, key, n.key);
                // 如果想要的是相等的结点并且n的key刚好等于key，或者key < n.key并且想要的结点不小于key（大于或等于都返回n，因为是near而不是精确查找的）则返回n即可
                if ((c == 0 && (rel & EQ) != 0) ||
                    (c <  0 && (rel & LT) == 0))
                    return n;
                // 如果key <= n.key并且想要的是key小于返回的结点则返回b（因为上次循环已经判断过b和b的前驱，当时没有返回就说明在上次循环c > 0，即b的key是小于指定的key的）
                if ( c <= 0 && (rel & LT) != 0)
                    return b.isBaseHeader() ? null : b;
                b = n;
                n = f;
            }
        }
    }

    /**
     * Returns SimpleImmutableEntry for results of findNear.
     * @param key the key
     * @param rel the relation -- OR'ed combination of EQ, LT, GT
     * @return Entry fitting relation, or null if no such
     */
    final AbstractMap.SimpleImmutableEntry<K,V> getNear(K key, int rel) {
        Comparator<? super K> cmp = comparator;
        for (;;) {
            Node<K,V> n = findNear(key, rel, cmp);
            if (n == null)
                return null;
            AbstractMap.SimpleImmutableEntry<K,V> e = n.createSnapshot();
            if (e != null)
                return e;
        }
    }

    /* ---------------- Constructors -------------- */

    /**
     * Constructs a new, empty map, sorted according to the
     * {@linkplain Comparable natural ordering} of the keys.
     */
    public ConcurrentSkipListMap() {
        this.comparator = null;
        initialize();
    }

    /**
     * Constructs a new, empty map, sorted according to the specified
     * comparator.
     *
     * @param comparator the comparator that will be used to order this map.
     *        If {@code null}, the {@linkplain Comparable natural
     *        ordering} of the keys will be used.
     */
    public ConcurrentSkipListMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
        initialize();
    }

    /**
     * Constructs a new map containing the same mappings as the given map,
     * sorted according to the {@linkplain Comparable natural ordering} of
     * the keys.
     *
     * @param  m the map whose mappings are to be placed in this map
     * @throws ClassCastException if the keys in {@code m} are not
     *         {@link Comparable}, or are not mutually comparable
     * @throws NullPointerException if the specified map or any of its keys
     *         or values are null
     */
    public ConcurrentSkipListMap(Map<? extends K, ? extends V> m) {
        this.comparator = null;
        initialize();
        putAll(m);
    }

    /**
     * Constructs a new map containing the same mappings and using the
     * same ordering as the specified sorted map.
     *
     * @param m the sorted map whose mappings are to be placed in this
     *        map, and whose comparator is to be used to sort this map
     * @throws NullPointerException if the specified sorted map or any of
     *         its keys or values are null
     */
    public ConcurrentSkipListMap(SortedMap<K, ? extends V> m) {
        this.comparator = m.comparator();
        initialize();
        buildFromSorted(m);
    }

    /**
     * Returns a shallow copy of this {@code ConcurrentSkipListMap}
     * instance. (The keys and values themselves are not cloned.)
     *
     * @return a shallow copy of this map
     */
    public ConcurrentSkipListMap<K,V> clone() {
        try {
            @SuppressWarnings("unchecked")
            ConcurrentSkipListMap<K,V> clone =
                (ConcurrentSkipListMap<K,V>) super.clone();
            clone.initialize();
            clone.buildFromSorted(this);
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new InternalError();
        }
    }

    /**
     * Streamlined bulk insertion to initialize from elements of
     * given sorted map.  Call only from constructor or clone
     * method.
     */
    private void buildFromSorted(SortedMap<K, ? extends V> map) {
        if (map == null)
            throw new NullPointerException();

        HeadIndex<K,V> h = head;
        Node<K,V> basepred = h.node;

        // Track the current rightmost node at each level. Uses an
        // ArrayList to avoid committing to initial or maximum level.
        ArrayList<Index<K,V>> preds = new ArrayList<Index<K,V>>();

        // initialize
        for (int i = 0; i <= h.level; ++i)
            preds.add(null);
        Index<K,V> q = h;
        for (int i = h.level; i > 0; --i) {
            preds.set(i, q);
            q = q.down;
        }

        Iterator<? extends Map.Entry<? extends K, ? extends V>> it =
            map.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<? extends K, ? extends V> e = it.next();
            int rnd = ThreadLocalRandom.current().nextInt();
            int j = 0;
            if ((rnd & 0x80000001) == 0) {
                do {
                    ++j;
                } while (((rnd >>>= 1) & 1) != 0);
                if (j > h.level) j = h.level + 1;
            }
            K k = e.getKey();
            V v = e.getValue();
            if (k == null || v == null)
                throw new NullPointerException();
            Node<K,V> z = new Node<K,V>(k, v, null);
            basepred.next = z;
            basepred = z;
            if (j > 0) {
                Index<K,V> idx = null;
                for (int i = 1; i <= j; ++i) {
                    idx = new Index<K,V>(z, idx, null);
                    if (i > h.level)
                        h = new HeadIndex<K,V>(h.node, h, idx, i);

                    if (i < preds.size()) {
                        preds.get(i).right = idx;
                        preds.set(i, idx);
                    } else
                        preds.add(idx);
                }
            }
        }
        head = h;
    }

    /* ---------------- Serialization -------------- */

    /**
     * Saves this map to a stream (that is, serializes it).
     *
     * @param s the stream
     * @throws java.io.IOException if an I/O error occurs
     * @serialData The key (Object) and value (Object) for each
     * key-value mapping represented by the map, followed by
     * {@code null}. The key-value mappings are emitted in key-order
     * (as determined by the Comparator, or by the keys' natural
     * ordering if no Comparator).
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out the Comparator and any hidden stuff
        s.defaultWriteObject();

        // Write out keys and values (alternating)
        for (Node<K,V> n = findFirst(); n != null; n = n.next) {
            V v = n.getValidValue();
            if (v != null) {
                s.writeObject(n.key);
                s.writeObject(v);
            }
        }
        s.writeObject(null);
    }

    /**
     * Reconstitutes this map from a stream (that is, deserializes it).
     * @param s the stream
     * @throws ClassNotFoundException if the class of a serialized object
     *         could not be found
     * @throws java.io.IOException if an I/O error occurs
     */
    @SuppressWarnings("unchecked")
    private void readObject(final java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in the Comparator and any hidden stuff
        s.defaultReadObject();
        // Reset transients
        initialize();

        /*
         * This is nearly identical to buildFromSorted, but is
         * distinct because readObject calls can't be nicely adapted
         * as the kind of iterator needed by buildFromSorted. (They
         * can be, but doing so requires type cheats and/or creation
         * of adaptor classes.) It is simpler to just adapt the code.
         */

        HeadIndex<K,V> h = head;
        Node<K,V> basepred = h.node;
        ArrayList<Index<K,V>> preds = new ArrayList<Index<K,V>>();
        for (int i = 0; i <= h.level; ++i)
            preds.add(null);
        Index<K,V> q = h;
        for (int i = h.level; i > 0; --i) {
            preds.set(i, q);
            q = q.down;
        }

        for (;;) {
            Object k = s.readObject();
            if (k == null)
                break;
            Object v = s.readObject();
            if (v == null)
                throw new NullPointerException();
            K key = (K) k;
            V val = (V) v;
            int rnd = ThreadLocalRandom.current().nextInt();
            int j = 0;
            if ((rnd & 0x80000001) == 0) {
                do {
                    ++j;
                } while (((rnd >>>= 1) & 1) != 0);
                if (j > h.level) j = h.level + 1;
            }
            Node<K,V> z = new Node<K,V>(key, val, null);
            basepred.next = z;
            basepred = z;
            if (j > 0) {
                Index<K,V> idx = null;
                for (int i = 1; i <= j; ++i) {
                    idx = new Index<K,V>(z, idx, null);
                    if (i > h.level)
                        h = new HeadIndex<K,V>(h.node, h, idx, i);

                    if (i < preds.size()) {
                        preds.get(i).right = idx;
                        preds.set(i, idx);
                    } else
                        preds.add(idx);
                }
            }
        }
        head = h;
    }

    /* ------ Map API methods ------ */

    /**
     * Returns {@code true} if this map contains a mapping for the specified
     * key.
     *
     * @param key key whose presence in this map is to be tested
     * @return {@code true} if this map contains a mapping for the specified key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     */
    public boolean containsKey(Object key) {
        return doGet(key) != null;
    }

    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key} compares
     * equal to {@code k} according to the map's ordering, then this
     * method returns {@code v}; otherwise it returns {@code null}.
     * (There can be at most one such mapping.)
     *
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        return doGet(key);
    }

    /**
     * Returns the value to which the specified key is mapped,
     * or the given defaultValue if this map contains no mapping for the key.
     *
     * @param key the key
     * @param defaultValue the value to return if this map contains
     * no mapping for the given key
     * @return the mapping for the key, if present; else the defaultValue
     * @throws NullPointerException if the specified key is null
     * @since 1.8
     */
    public V getOrDefault(Object key, V defaultValue) {
        V v;
        return (v = doGet(key)) == null ? defaultValue : v;
    }

    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with the specified key, or
     *         {@code null} if there was no mapping for the key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key or value is null
     */
    public V put(K key, V value) {
        if (value == null)
            throw new NullPointerException();
        return doPut(key, value, false);
    }

    /**
     * Removes the mapping for the specified key from this map if present.
     *
     * @param  key key for which mapping should be removed
     * @return the previous value associated with the specified key, or
     *         {@code null} if there was no mapping for the key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     */
    public V remove(Object key) {
        return doRemove(key, null);
    }

    /**
     * Returns {@code true} if this map maps one or more keys to the
     * specified value.  This operation requires time linear in the
     * map size. Additionally, it is possible for the map to change
     * during execution of this method, in which case the returned
     * result may be inaccurate.
     *
     * @param value value whose presence in this map is to be tested
     * @return {@code true} if a mapping to {@code value} exists;
     *         {@code false} otherwise
     * @throws NullPointerException if the specified value is null
     */
    public boolean containsValue(Object value) {
        if (value == null)
            throw new NullPointerException();
        for (Node<K,V> n = findFirst(); n != null; n = n.next) {
            V v = n.getValidValue();
            if (v != null && value.equals(v))
                return true;
        }
        return false;
    }

    /**
     * Returns the number of key-value mappings in this map.  If this map
     * contains more than {@code Integer.MAX_VALUE} elements, it
     * returns {@code Integer.MAX_VALUE}.
     *
     * <p>Beware that, unlike in most collections, this method is
     * <em>NOT</em> a constant-time operation. Because of the
     * asynchronous nature of these maps, determining the current
     * number of elements requires traversing them all to count them.
     * Additionally, it is possible for the size to change during
     * execution of this method, in which case the returned result
     * will be inaccurate. Thus, this method is typically not very
     * useful in concurrent applications.
     *
     * @return the number of elements in this map
     */
    public int size() {
        long count = 0;
        for (Node<K,V> n = findFirst(); n != null; n = n.next) {
            if (n.getValidValue() != null)
                ++count;
        }
        return (count >= Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int) count;
    }

    /**
     * Returns {@code true} if this map contains no key-value mappings.
     * @return {@code true} if this map contains no key-value mappings
     */
    public boolean isEmpty() {
        return findFirst() == null;
    }

    /**
     * Removes all of the mappings from this map.
     */
    public void clear() {
        for (;;) {
            Node<K,V> b, n;
            HeadIndex<K,V> h = head, d = (HeadIndex<K,V>)h.down;
            if (d != null)
                casHead(h, d);            // remove levels
            else if ((b = h.node) != null && (n = b.next) != null) {
                Node<K,V> f = n.next;     // remove values
                if (n == b.next) {
                    Object v = n.value;
                    if (v == null)
                        n.helpDelete(b, f);
                    else if (n.casValue(v, null) && n.appendMarker(f))
                        b.casNext(n, f);
                }
            }
            else
                break;
        }
    }

    /**
     * If the specified key is not already associated with a value,
     * attempts to compute its value using the given mapping function
     * and enters it into this map unless {@code null}.  The function
     * is <em>NOT</em> guaranteed to be applied once atomically only
     * if the value is not present.
     *
     * @param key key with which the specified value is to be associated
     * @param mappingFunction the function to compute a value
     * @return the current (existing or computed) value associated with
     *         the specified key, or null if the computed value is null
     * @throws NullPointerException if the specified key is null
     *         or the mappingFunction is null
     * @since 1.8
     */
    public V computeIfAbsent(K key,
                             Function<? super K, ? extends V> mappingFunction) {
        if (key == null || mappingFunction == null)
            throw new NullPointerException();
        V v, p, r;
        if ((v = doGet(key)) == null &&
            (r = mappingFunction.apply(key)) != null)
            v = (p = doPut(key, r, true)) == null ? r : p;
        return v;
    }

    /**
     * If the value for the specified key is present, attempts to
     * compute a new mapping given the key and its current mapped
     * value. The function is <em>NOT</em> guaranteed to be applied
     * once atomically.
     *
     * @param key key with which a value may be associated
     * @param remappingFunction the function to compute a value
     * @return the new value associated with the specified key, or null if none
     * @throws NullPointerException if the specified key is null
     *         or the remappingFunction is null
     * @since 1.8
     */
    public V computeIfPresent(K key,
                              BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (key == null || remappingFunction == null)
            throw new NullPointerException();
        Node<K,V> n; Object v;
        while ((n = findNode(key)) != null) {
            if ((v = n.value) != null) {
                @SuppressWarnings("unchecked") V vv = (V) v;
                V r = remappingFunction.apply(key, vv);
                if (r != null) {
                    if (n.casValue(vv, r))
                        return r;
                }
                else if (doRemove(key, vv) != null)
                    break;
            }
        }
        return null;
    }

    /**
     * Attempts to compute a mapping for the specified key and its
     * current mapped value (or {@code null} if there is no current
     * mapping). The function is <em>NOT</em> guaranteed to be applied
     * once atomically.
     *
     * @param key key with which the specified value is to be associated
     * @param remappingFunction the function to compute a value
     * @return the new value associated with the specified key, or null if none
     * @throws NullPointerException if the specified key is null
     *         or the remappingFunction is null
     * @since 1.8
     */
    public V compute(K key,
                     BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (key == null || remappingFunction == null)
            throw new NullPointerException();
        for (;;) {
            Node<K,V> n; Object v; V r;
            if ((n = findNode(key)) == null) {
                if ((r = remappingFunction.apply(key, null)) == null)
                    break;
                if (doPut(key, r, true) == null)
                    return r;
            }
            else if ((v = n.value) != null) {
                @SuppressWarnings("unchecked") V vv = (V) v;
                if ((r = remappingFunction.apply(key, vv)) != null) {
                    if (n.casValue(vv, r))
                        return r;
                }
                else if (doRemove(key, vv) != null)
                    break;
            }
        }
        return null;
    }

    /**
     * If the specified key is not already associated with a value,
     * associates it with the given value.  Otherwise, replaces the
     * value with the results of the given remapping function, or
     * removes if {@code null}. The function is <em>NOT</em>
     * guaranteed to be applied once atomically.
     *
     * @param key key with which the specified value is to be associated
     * @param value the value to use if absent
     * @param remappingFunction the function to recompute a value if present
     * @return the new value associated with the specified key, or null if none
     * @throws NullPointerException if the specified key or value is null
     *         or the remappingFunction is null
     * @since 1.8
     */
    public V merge(K key, V value,
                   BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        if (key == null || value == null || remappingFunction == null)
            throw new NullPointerException();
        for (;;) {
            Node<K,V> n; Object v; V r;
            if ((n = findNode(key)) == null) {
                if (doPut(key, value, true) == null)
                    return value;
            }
            else if ((v = n.value) != null) {
                @SuppressWarnings("unchecked") V vv = (V) v;
                // 计算value和cas是分开的，所以merge方法不保证原子性
                if ((r = remappingFunction.apply(vv, value)) != null) {
                    if (n.casValue(vv, r))
                        return r;
                }
                else if (doRemove(key, vv) != null)
                    return null;
            }
        }
    }

    /* ---------------- View methods -------------- */

    /*
     * Note: Lazy initialization works for views because view classes
     * are stateless/immutable so it doesn't matter wrt correctness if
     * more than one is created (which will only rarely happen).  Even
     * so, the following idiom conservatively ensures that the method
     * returns the one it created if it does so, not one created by
     * another racing thread.
     */

    /**
     * Returns a {@link NavigableSet} view of the keys contained in this map.
     *
     * <p>The set's iterator returns the keys in ascending order.
     * The set's spliterator additionally reports {@link Spliterator#CONCURRENT},
     * {@link Spliterator#NONNULL}, {@link Spliterator#SORTED} and
     * {@link Spliterator#ORDERED}, with an encounter order that is ascending
     * key order.  The spliterator's comparator (see
     * {@link java.util.Spliterator#getComparator()}) is {@code null} if
     * the map's comparator (see {@link #comparator()}) is {@code null}.
     * Otherwise, the spliterator's comparator is the same as or imposes the
     * same total ordering as the map's comparator.
     *
     * <p>The set is backed by the map, so changes to the map are
     * reflected in the set, and vice-versa.  The set supports element
     * removal, which removes the corresponding mapping from the map,
     * via the {@code Iterator.remove}, {@code Set.remove},
     * {@code removeAll}, {@code retainAll}, and {@code clear}
     * operations.  It does not support the {@code add} or {@code addAll}
     * operations.
     *
     * <p>The view's iterators and spliterators are
     * <a href="package-summary.html#Weakly"><i>weakly consistent</i></a>.
     *
     * <p>This method is equivalent to method {@code navigableKeySet}.
     *
     * @return a navigable set view of the keys in this map
     */
    public NavigableSet<K> keySet() {
        KeySet<K> ks = keySet;
        return (ks != null) ? ks : (keySet = new KeySet<K>(this));
    }

    public NavigableSet<K> navigableKeySet() {
        KeySet<K> ks = keySet;
        return (ks != null) ? ks : (keySet = new KeySet<K>(this));
    }

    /**
     * Returns a {@link Collection} view of the values contained in this map.
     * <p>The collection's iterator returns the values in ascending order
     * of the corresponding keys. The collections's spliterator additionally
     * reports {@link Spliterator#CONCURRENT}, {@link Spliterator#NONNULL} and
     * {@link Spliterator#ORDERED}, with an encounter order that is ascending
     * order of the corresponding keys.
     *
     * <p>The collection is backed by the map, so changes to the map are
     * reflected in the collection, and vice-versa.  The collection
     * supports element removal, which removes the corresponding
     * mapping from the map, via the {@code Iterator.remove},
     * {@code Collection.remove}, {@code removeAll},
     * {@code retainAll} and {@code clear} operations.  It does not
     * support the {@code add} or {@code addAll} operations.
     *
     * <p>The view's iterators and spliterators are
     * <a href="package-summary.html#Weakly"><i>weakly consistent</i></a>.
     */
    public Collection<V> values() {
        Values<V> vs = values;
        return (vs != null) ? vs : (values = new Values<V>(this));
    }

    /**
     * Returns a {@link Set} view of the mappings contained in this map.
     *
     * <p>The set's iterator returns the entries in ascending key order.  The
     * set's spliterator additionally reports {@link Spliterator#CONCURRENT},
     * {@link Spliterator#NONNULL}, {@link Spliterator#SORTED} and
     * {@link Spliterator#ORDERED}, with an encounter order that is ascending
     * key order.
     *
     * <p>The set is backed by the map, so changes to the map are
     * reflected in the set, and vice-versa.  The set supports element
     * removal, which removes the corresponding mapping from the map,
     * via the {@code Iterator.remove}, {@code Set.remove},
     * {@code removeAll}, {@code retainAll} and {@code clear}
     * operations.  It does not support the {@code add} or
     * {@code addAll} operations.
     *
     * <p>The view's iterators and spliterators are
     * <a href="package-summary.html#Weakly"><i>weakly consistent</i></a>.
     *
     * <p>The {@code Map.Entry} elements traversed by the {@code iterator}
     * or {@code spliterator} do <em>not</em> support the {@code setValue}
     * operation.
     *
     * @return a set view of the mappings contained in this map,
     *         sorted in ascending key order
     */
    public Set<Map.Entry<K,V>> entrySet() {
        EntrySet<K,V> es = entrySet;
        return (es != null) ? es : (entrySet = new EntrySet<K,V>(this));
    }

    public ConcurrentNavigableMap<K,V> descendingMap() {
        ConcurrentNavigableMap<K,V> dm = descendingMap;
        return (dm != null) ? dm : (descendingMap = new SubMap<K,V>
                                    (this, null, false, null, false, true));
    }

    public NavigableSet<K> descendingKeySet() {
        return descendingMap().navigableKeySet();
    }

    /* ---------------- AbstractMap Overrides -------------- */

    /**
     * Compares the specified object with this map for equality.
     * Returns {@code true} if the given object is also a map and the
     * two maps represent the same mappings.  More formally, two maps
     * {@code m1} and {@code m2} represent the same mappings if
     * {@code m1.entrySet().equals(m2.entrySet())}.  This
     * operation may return misleading results if either map is
     * concurrently modified during execution of this method.
     *
     * @param o object to be compared for equality with this map
     * @return {@code true} if the specified object is equal to this map
     */
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Map))
            return false;
        Map<?,?> m = (Map<?,?>) o;
        try {
            for (Map.Entry<K,V> e : this.entrySet())
                if (! e.getValue().equals(m.get(e.getKey())))
                    return false;
            for (Map.Entry<?,?> e : m.entrySet()) {
                Object k = e.getKey();
                Object v = e.getValue();
                if (k == null || v == null || !v.equals(get(k)))
                    return false;
            }
            return true;
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }
    }

    /* ------ ConcurrentMap API methods ------ */

    /**
     * {@inheritDoc}
     *
     * @return the previous value associated with the specified key,
     *         or {@code null} if there was no mapping for the key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key or value is null
     */
    public V putIfAbsent(K key, V value) {
        if (value == null)
            throw new NullPointerException();
        return doPut(key, value, true);
    }

    /**
     * {@inheritDoc}
     *
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key is null
     */
    public boolean remove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        return value != null && doRemove(key, value) != null;
    }

    /**
     * {@inheritDoc}
     *
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if any of the arguments are null
     */
    public boolean replace(K key, V oldValue, V newValue) {
        if (key == null || oldValue == null || newValue == null)
            throw new NullPointerException();
        for (;;) {
            Node<K,V> n; Object v;
            if ((n = findNode(key)) == null)
                return false;
            if ((v = n.value) != null) {
                if (!oldValue.equals(v))
                    return false;
                if (n.casValue(v, newValue))
                    return true;
            }
        }
    }

    /**
     * {@inheritDoc}
     *
     * @return the previous value associated with the specified key,
     *         or {@code null} if there was no mapping for the key
     * @throws ClassCastException if the specified key cannot be compared
     *         with the keys currently in the map
     * @throws NullPointerException if the specified key or value is null
     */
    public V replace(K key, V value) {
        if (key == null || value == null)
            throw new NullPointerException();
        for (;;) {
            Node<K,V> n; Object v;
            if ((n = findNode(key)) == null)
                return null;
            if ((v = n.value) != null && n.casValue(v, value)) {
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
    }

    /* ------ SortedMap API methods ------ */

    public Comparator<? super K> comparator() {
        return comparator;
    }

    /**
     * @throws NoSuchElementException {@inheritDoc}
     */
    public K firstKey() {
        Node<K,V> n = findFirst();
        if (n == null)
            throw new NoSuchElementException();
        return n.key;
    }

    /**
     * @throws NoSuchElementException {@inheritDoc}
     */
    public K lastKey() {
        Node<K,V> n = findLast();
        if (n == null)
            throw new NoSuchElementException();
        return n.key;
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} or {@code toKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> subMap(K fromKey,
                                              boolean fromInclusive,
                                              K toKey,
                                              boolean toInclusive) {
        if (fromKey == null || toKey == null)
            throw new NullPointerException();
        return new SubMap<K,V>
            (this, fromKey, fromInclusive, toKey, toInclusive, false);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code toKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> headMap(K toKey,
                                               boolean inclusive) {
        if (toKey == null)
            throw new NullPointerException();
        return new SubMap<K,V>
            (this, null, false, toKey, inclusive, false);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> tailMap(K fromKey,
                                               boolean inclusive) {
        if (fromKey == null)
            throw new NullPointerException();
        return new SubMap<K,V>
            (this, fromKey, inclusive, null, false, false);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} or {@code toKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> subMap(K fromKey, K toKey) {
        return subMap(fromKey, true, toKey, false);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code toKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> headMap(K toKey) {
        return headMap(toKey, false);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if {@code fromKey} is null
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public ConcurrentNavigableMap<K,V> tailMap(K fromKey) {
        return tailMap(fromKey, true);
    }

    /* ---------------- Relational operations -------------- */

    /**
     * Returns a key-value mapping associated with the greatest key
     * strictly less than the given key, or {@code null} if there is
     * no such key. The returned entry does <em>not</em> support the
     * {@code Entry.setValue} method.
     *
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public Map.Entry<K,V> lowerEntry(K key) {
        return getNear(key, LT);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public K lowerKey(K key) {
        Node<K,V> n = findNear(key, LT, comparator);
        return (n == null) ? null : n.key;
    }

    /**
     * Returns a key-value mapping associated with the greatest key
     * less than or equal to the given key, or {@code null} if there
     * is no such key. The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     *
     * @param key the key
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public Map.Entry<K,V> floorEntry(K key) {
        return getNear(key, LT|EQ);
    }

    /**
     * @param key the key
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public K floorKey(K key) {
        Node<K,V> n = findNear(key, LT|EQ, comparator);
        return (n == null) ? null : n.key;
    }

    /**
     * Returns a key-value mapping associated with the least key
     * greater than or equal to the given key, or {@code null} if
     * there is no such entry. The returned entry does <em>not</em>
     * support the {@code Entry.setValue} method.
     *
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public Map.Entry<K,V> ceilingEntry(K key) {
        return getNear(key, GT|EQ);
    }

    /**
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public K ceilingKey(K key) {
        Node<K,V> n = findNear(key, GT|EQ, comparator);
        return (n == null) ? null : n.key;
    }

    /**
     * Returns a key-value mapping associated with the least key
     * strictly greater than the given key, or {@code null} if there
     * is no such key. The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     *
     * @param key the key
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public Map.Entry<K,V> higherEntry(K key) {
        return getNear(key, GT);
    }

    /**
     * @param key the key
     * @throws ClassCastException {@inheritDoc}
     * @throws NullPointerException if the specified key is null
     */
    public K higherKey(K key) {
        Node<K,V> n = findNear(key, GT, comparator);
        return (n == null) ? null : n.key;
    }

    /**
     * Returns a key-value mapping associated with the least
     * key in this map, or {@code null} if the map is empty.
     * The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     */
    public Map.Entry<K,V> firstEntry() {
        for (;;) {
            Node<K,V> n = findFirst();
            if (n == null)
                return null;
            AbstractMap.SimpleImmutableEntry<K,V> e = n.createSnapshot();
            if (e != null)
                return e;
        }
    }

    /**
     * Returns a key-value mapping associated with the greatest
     * key in this map, or {@code null} if the map is empty.
     * The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     */
    public Map.Entry<K,V> lastEntry() {
        for (;;) {
            Node<K,V> n = findLast();
            if (n == null)
                return null;
            AbstractMap.SimpleImmutableEntry<K,V> e = n.createSnapshot();
            if (e != null)
                return e;
        }
    }

    /**
     * Removes and returns a key-value mapping associated with
     * the least key in this map, or {@code null} if the map is empty.
     * The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     */
    public Map.Entry<K,V> pollFirstEntry() {
        return doRemoveFirstEntry();
    }

    /**
     * Removes and returns a key-value mapping associated with
     * the greatest key in this map, or {@code null} if the map is empty.
     * The returned entry does <em>not</em> support
     * the {@code Entry.setValue} method.
     */
    public Map.Entry<K,V> pollLastEntry() {
        return doRemoveLastEntry();
    }


    /* ---------------- Iterators -------------- */

    /**
     * Base of iterator classes:
     */
    abstract class Iter<T> implements Iterator<T> {
        /** the last node returned by next() */
        Node<K,V> lastReturned;
        /** the next node to return from next(); */
        Node<K,V> next;
        /** Cache of next value field to maintain weak consistency */
        V nextValue;

        /** Initializes ascending iterator for entire range. */
        Iter() {
            while ((next = findFirst()) != null) {
                Object x = next.value;
                if (x != null && x != next) {
                    @SuppressWarnings("unchecked") V vv = (V)x;
                    nextValue = vv;
                    break;
                }
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        /** Advances next to higher entry. */
        final void advance() {
            if (next == null)
                throw new NoSuchElementException();
            lastReturned = next;
            while ((next = next.next) != null) {
                Object x = next.value;
                if (x != null && x != next) {
                    @SuppressWarnings("unchecked") V vv = (V)x;
                    nextValue = vv;
                    break;
                }
            }
        }

        public void remove() {
            Node<K,V> l = lastReturned;
            if (l == null)
                throw new IllegalStateException();
            // It would not be worth all of the overhead to directly
            // unlink from here. Using remove is fast enough.
            ConcurrentSkipListMap.this.remove(l.key);
            lastReturned = null;
        }

    }

    final class ValueIterator extends Iter<V> {
        public V next() {
            V v = nextValue;
            advance();
            return v;
        }
    }

    final class KeyIterator extends Iter<K> {
        public K next() {
            Node<K,V> n = next;
            advance();
            return n.key;
        }
    }

    final class EntryIterator extends Iter<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            Node<K,V> n = next;
            V v = nextValue;
            advance();
            return new AbstractMap.SimpleImmutableEntry<K,V>(n.key, v);
        }
    }

    // Factory methods for iterators needed by ConcurrentSkipListSet etc

    Iterator<K> keyIterator() {
        return new KeyIterator();
    }

    Iterator<V> valueIterator() {
        return new ValueIterator();
    }

    Iterator<Map.Entry<K,V>> entryIterator() {
        return new EntryIterator();
    }

    /* ---------------- View Classes -------------- */

    /*
     * View classes are static, delegating to a ConcurrentNavigableMap
     * to allow use by SubMaps, which outweighs the ugliness of
     * needing type-tests for Iterator methods.
     */

    static final <E> List<E> toList(Collection<E> c) {
        // Using size() here would be a pessimization.
        ArrayList<E> list = new ArrayList<E>();
        for (E e : c)
            list.add(e);
        return list;
    }

    static final class KeySet<E>
            extends AbstractSet<E> implements NavigableSet<E> {
        final ConcurrentNavigableMap<E,?> m;
        KeySet(ConcurrentNavigableMap<E,?> map) { m = map; }
        public int size() { return m.size(); }
        public boolean isEmpty() { return m.isEmpty(); }
        public boolean contains(Object o) { return m.containsKey(o); }
        public boolean remove(Object o) { return m.remove(o) != null; }
        public void clear() { m.clear(); }
        public E lower(E e) { return m.lowerKey(e); }
        public E floor(E e) { return m.floorKey(e); }
        public E ceiling(E e) { return m.ceilingKey(e); }
        public E higher(E e) { return m.higherKey(e); }
        public Comparator<? super E> comparator() { return m.comparator(); }
        public E first() { return m.firstKey(); }
        public E last() { return m.lastKey(); }
        public E pollFirst() {
            Map.Entry<E,?> e = m.pollFirstEntry();
            return (e == null) ? null : e.getKey();
        }
        public E pollLast() {
            Map.Entry<E,?> e = m.pollLastEntry();
            return (e == null) ? null : e.getKey();
        }
        @SuppressWarnings("unchecked")
        public Iterator<E> iterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<E,Object>)m).keyIterator();
            else
                return ((ConcurrentSkipListMap.SubMap<E,Object>)m).keyIterator();
        }
        public boolean equals(Object o) {
            if (o == this)
                return true;
            if (!(o instanceof Set))
                return false;
            Collection<?> c = (Collection<?>) o;
            try {
                return containsAll(c) && c.containsAll(this);
            } catch (ClassCastException unused) {
                return false;
            } catch (NullPointerException unused) {
                return false;
            }
        }
        public Object[] toArray()     { return toList(this).toArray();  }
        public <T> T[] toArray(T[] a) { return toList(this).toArray(a); }
        public Iterator<E> descendingIterator() {
            return descendingSet().iterator();
        }
        public NavigableSet<E> subSet(E fromElement,
                                      boolean fromInclusive,
                                      E toElement,
                                      boolean toInclusive) {
            return new KeySet<E>(m.subMap(fromElement, fromInclusive,
                                          toElement,   toInclusive));
        }
        public NavigableSet<E> headSet(E toElement, boolean inclusive) {
            return new KeySet<E>(m.headMap(toElement, inclusive));
        }
        public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
            return new KeySet<E>(m.tailMap(fromElement, inclusive));
        }
        public NavigableSet<E> subSet(E fromElement, E toElement) {
            return subSet(fromElement, true, toElement, false);
        }
        public NavigableSet<E> headSet(E toElement) {
            return headSet(toElement, false);
        }
        public NavigableSet<E> tailSet(E fromElement) {
            return tailSet(fromElement, true);
        }
        public NavigableSet<E> descendingSet() {
            return new KeySet<E>(m.descendingMap());
        }
        @SuppressWarnings("unchecked")
        public Spliterator<E> spliterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<E,?>)m).keySpliterator();
            else
                return (Spliterator<E>)((SubMap<E,?>)m).keyIterator();
        }
    }

    static final class Values<E> extends AbstractCollection<E> {
        final ConcurrentNavigableMap<?, E> m;
        Values(ConcurrentNavigableMap<?, E> map) {
            m = map;
        }
        @SuppressWarnings("unchecked")
        public Iterator<E> iterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<?,E>)m).valueIterator();
            else
                return ((SubMap<?,E>)m).valueIterator();
        }
        public boolean isEmpty() {
            return m.isEmpty();
        }
        public int size() {
            return m.size();
        }
        public boolean contains(Object o) {
            return m.containsValue(o);
        }
        public void clear() {
            m.clear();
        }
        public Object[] toArray()     { return toList(this).toArray();  }
        public <T> T[] toArray(T[] a) { return toList(this).toArray(a); }
        @SuppressWarnings("unchecked")
        public Spliterator<E> spliterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<?,E>)m).valueSpliterator();
            else
                return (Spliterator<E>)((SubMap<?,E>)m).valueIterator();
        }
    }

    static final class EntrySet<K1,V1> extends AbstractSet<Map.Entry<K1,V1>> {
        final ConcurrentNavigableMap<K1, V1> m;
        EntrySet(ConcurrentNavigableMap<K1, V1> map) {
            m = map;
        }
        @SuppressWarnings("unchecked")
        public Iterator<Map.Entry<K1,V1>> iterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<K1,V1>)m).entryIterator();
            else
                return ((SubMap<K1,V1>)m).entryIterator();
        }

        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            V1 v = m.get(e.getKey());
            return v != null && v.equals(e.getValue());
        }
        public boolean remove(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            return m.remove(e.getKey(),
                            e.getValue());
        }
        public boolean isEmpty() {
            return m.isEmpty();
        }
        public int size() {
            return m.size();
        }
        public void clear() {
            m.clear();
        }
        public boolean equals(Object o) {
            if (o == this)
                return true;
            if (!(o instanceof Set))
                return false;
            Collection<?> c = (Collection<?>) o;
            try {
                return containsAll(c) && c.containsAll(this);
            } catch (ClassCastException unused) {
                return false;
            } catch (NullPointerException unused) {
                return false;
            }
        }
        public Object[] toArray()     { return toList(this).toArray();  }
        public <T> T[] toArray(T[] a) { return toList(this).toArray(a); }
        @SuppressWarnings("unchecked")
        public Spliterator<Map.Entry<K1,V1>> spliterator() {
            if (m instanceof ConcurrentSkipListMap)
                return ((ConcurrentSkipListMap<K1,V1>)m).entrySpliterator();
            else
                return (Spliterator<Map.Entry<K1,V1>>)
                    ((SubMap<K1,V1>)m).entryIterator();
        }
    }

    /**
     * Submaps returned by {@link ConcurrentSkipListMap} submap operations
     * represent a subrange of mappings of their underlying
     * maps. Instances of this class support all methods of their
     * underlying maps, differing in that mappings outside their range are
     * ignored, and attempts to add mappings outside their ranges result
     * in {@link IllegalArgumentException}.  Instances of this class are
     * constructed only using the {@code subMap}, {@code headMap}, and
     * {@code tailMap} methods of their underlying maps.
     *
     * @serial include
     */
    static final class SubMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentNavigableMap<K,V>, Cloneable, Serializable {
        private static final long serialVersionUID = -7647078645895051609L;

        /** Underlying map */
        private final ConcurrentSkipListMap<K,V> m;
        /** lower bound key, or null if from start */
        private final K lo;
        /** upper bound key, or null if to end */
        private final K hi;
        /** inclusion flag for lo */
        private final boolean loInclusive;
        /** inclusion flag for hi */
        private final boolean hiInclusive;
        /** direction */
        private final boolean isDescending;

        // Lazily initialized view holders
        private transient KeySet<K> keySetView;
        private transient Set<Map.Entry<K,V>> entrySetView;
        private transient Collection<V> valuesView;

        /**
         * Creates a new submap, initializing all fields.
         */
        SubMap(ConcurrentSkipListMap<K,V> map,
               K fromKey, boolean fromInclusive,
               K toKey, boolean toInclusive,
               boolean isDescending) {
            Comparator<? super K> cmp = map.comparator;
            if (fromKey != null && toKey != null &&
                cpr(cmp, fromKey, toKey) > 0)
                throw new IllegalArgumentException("inconsistent range");
            this.m = map;
            this.lo = fromKey;
            this.hi = toKey;
            this.loInclusive = fromInclusive;
            this.hiInclusive = toInclusive;
            this.isDescending = isDescending;
        }

        /* ----------------  Utilities -------------- */

        boolean tooLow(Object key, Comparator<? super K> cmp) {
            int c;
            return (lo != null && ((c = cpr(cmp, key, lo)) < 0 ||
                                   (c == 0 && !loInclusive)));
        }

        boolean tooHigh(Object key, Comparator<? super K> cmp) {
            int c;
            return (hi != null && ((c = cpr(cmp, key, hi)) > 0 ||
                                   (c == 0 && !hiInclusive)));
        }

        boolean inBounds(Object key, Comparator<? super K> cmp) {
            return !tooLow(key, cmp) && !tooHigh(key, cmp);
        }

        void checkKeyBounds(K key, Comparator<? super K> cmp) {
            if (key == null)
                throw new NullPointerException();
            if (!inBounds(key, cmp))
                throw new IllegalArgumentException("key out of range");
        }

        /**
         * Returns true if node key is less than upper bound of range.
         */
        boolean isBeforeEnd(ConcurrentSkipListMap.Node<K,V> n,
                            Comparator<? super K> cmp) {
            if (n == null)
                return false;
            if (hi == null)
                return true;
            K k = n.key;
            if (k == null) // pass by markers and headers
                return true;
            int c = cpr(cmp, k, hi);
            if (c > 0 || (c == 0 && !hiInclusive))
                return false;
            return true;
        }

        /**
         * Returns lowest node. This node might not be in range, so
         * most usages need to check bounds.
         */
        ConcurrentSkipListMap.Node<K,V> loNode(Comparator<? super K> cmp) {
            if (lo == null)
                return m.findFirst();
            else if (loInclusive)
                return m.findNear(lo, GT|EQ, cmp);
            else
                return m.findNear(lo, GT, cmp);
        }

        /**
         * Returns highest node. This node might not be in range, so
         * most usages need to check bounds.
         */
        ConcurrentSkipListMap.Node<K,V> hiNode(Comparator<? super K> cmp) {
            if (hi == null)
                return m.findLast();
            else if (hiInclusive)
                return m.findNear(hi, LT|EQ, cmp);
            else
                return m.findNear(hi, LT, cmp);
        }

        /**
         * Returns lowest absolute key (ignoring directonality).
         */
        K lowestKey() {
            Comparator<? super K> cmp = m.comparator;
            ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
            if (isBeforeEnd(n, cmp))
                return n.key;
            else
                throw new NoSuchElementException();
        }

        /**
         * Returns highest absolute key (ignoring directonality).
         */
        K highestKey() {
            Comparator<? super K> cmp = m.comparator;
            ConcurrentSkipListMap.Node<K,V> n = hiNode(cmp);
            if (n != null) {
                K last = n.key;
                if (inBounds(last, cmp))
                    return last;
            }
            throw new NoSuchElementException();
        }

        Map.Entry<K,V> lowestEntry() {
            Comparator<? super K> cmp = m.comparator;
            for (;;) {
                ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
                if (!isBeforeEnd(n, cmp))
                    return null;
                Map.Entry<K,V> e = n.createSnapshot();
                if (e != null)
                    return e;
            }
        }

        Map.Entry<K,V> highestEntry() {
            Comparator<? super K> cmp = m.comparator;
            for (;;) {
                ConcurrentSkipListMap.Node<K,V> n = hiNode(cmp);
                if (n == null || !inBounds(n.key, cmp))
                    return null;
                Map.Entry<K,V> e = n.createSnapshot();
                if (e != null)
                    return e;
            }
        }

        Map.Entry<K,V> removeLowest() {
            Comparator<? super K> cmp = m.comparator;
            for (;;) {
                Node<K,V> n = loNode(cmp);
                if (n == null)
                    return null;
                K k = n.key;
                if (!inBounds(k, cmp))
                    return null;
                V v = m.doRemove(k, null);
                if (v != null)
                    return new AbstractMap.SimpleImmutableEntry<K,V>(k, v);
            }
        }

        Map.Entry<K,V> removeHighest() {
            Comparator<? super K> cmp = m.comparator;
            for (;;) {
                Node<K,V> n = hiNode(cmp);
                if (n == null)
                    return null;
                K k = n.key;
                if (!inBounds(k, cmp))
                    return null;
                V v = m.doRemove(k, null);
                if (v != null)
                    return new AbstractMap.SimpleImmutableEntry<K,V>(k, v);
            }
        }

        /**
         * Submap version of ConcurrentSkipListMap.getNearEntry
         */
        Map.Entry<K,V> getNearEntry(K key, int rel) {
            Comparator<? super K> cmp = m.comparator;
            if (isDescending) { // adjust relation for direction
                if ((rel & LT) == 0)
                    rel |= LT;
                else
                    rel &= ~LT;
            }
            if (tooLow(key, cmp))
                return ((rel & LT) != 0) ? null : lowestEntry();
            if (tooHigh(key, cmp))
                return ((rel & LT) != 0) ? highestEntry() : null;
            for (;;) {
                Node<K,V> n = m.findNear(key, rel, cmp);
                if (n == null || !inBounds(n.key, cmp))
                    return null;
                K k = n.key;
                V v = n.getValidValue();
                if (v != null)
                    return new AbstractMap.SimpleImmutableEntry<K,V>(k, v);
            }
        }

        // Almost the same as getNearEntry, except for keys
        K getNearKey(K key, int rel) {
            Comparator<? super K> cmp = m.comparator;
            if (isDescending) { // adjust relation for direction
                if ((rel & LT) == 0)
                    rel |= LT;
                else
                    rel &= ~LT;
            }
            if (tooLow(key, cmp)) {
                if ((rel & LT) == 0) {
                    ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
                    if (isBeforeEnd(n, cmp))
                        return n.key;
                }
                return null;
            }
            if (tooHigh(key, cmp)) {
                if ((rel & LT) != 0) {
                    ConcurrentSkipListMap.Node<K,V> n = hiNode(cmp);
                    if (n != null) {
                        K last = n.key;
                        if (inBounds(last, cmp))
                            return last;
                    }
                }
                return null;
            }
            for (;;) {
                Node<K,V> n = m.findNear(key, rel, cmp);
                if (n == null || !inBounds(n.key, cmp))
                    return null;
                K k = n.key;
                V v = n.getValidValue();
                if (v != null)
                    return k;
            }
        }

        /* ----------------  Map API methods -------------- */

        public boolean containsKey(Object key) {
            if (key == null) throw new NullPointerException();
            return inBounds(key, m.comparator) && m.containsKey(key);
        }

        public V get(Object key) {
            if (key == null) throw new NullPointerException();
            return (!inBounds(key, m.comparator)) ? null : m.get(key);
        }

        public V put(K key, V value) {
            checkKeyBounds(key, m.comparator);
            return m.put(key, value);
        }

        public V remove(Object key) {
            return (!inBounds(key, m.comparator)) ? null : m.remove(key);
        }

        public int size() {
            Comparator<? super K> cmp = m.comparator;
            long count = 0;
            for (ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
                 isBeforeEnd(n, cmp);
                 n = n.next) {
                if (n.getValidValue() != null)
                    ++count;
            }
            return count >= Integer.MAX_VALUE ? Integer.MAX_VALUE : (int)count;
        }

        public boolean isEmpty() {
            Comparator<? super K> cmp = m.comparator;
            return !isBeforeEnd(loNode(cmp), cmp);
        }

        public boolean containsValue(Object value) {
            if (value == null)
                throw new NullPointerException();
            Comparator<? super K> cmp = m.comparator;
            for (ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
                 isBeforeEnd(n, cmp);
                 n = n.next) {
                V v = n.getValidValue();
                if (v != null && value.equals(v))
                    return true;
            }
            return false;
        }

        public void clear() {
            Comparator<? super K> cmp = m.comparator;
            for (ConcurrentSkipListMap.Node<K,V> n = loNode(cmp);
                 isBeforeEnd(n, cmp);
                 n = n.next) {
                if (n.getValidValue() != null)
                    m.remove(n.key);
            }
        }

        /* ----------------  ConcurrentMap API methods -------------- */

        public V putIfAbsent(K key, V value) {
            checkKeyBounds(key, m.comparator);
            return m.putIfAbsent(key, value);
        }

        public boolean remove(Object key, Object value) {
            return inBounds(key, m.comparator) && m.remove(key, value);
        }

        public boolean replace(K key, V oldValue, V newValue) {
            checkKeyBounds(key, m.comparator);
            return m.replace(key, oldValue, newValue);
        }

        public V replace(K key, V value) {
            checkKeyBounds(key, m.comparator);
            return m.replace(key, value);
        }

        /* ----------------  SortedMap API methods -------------- */

        public Comparator<? super K> comparator() {
            Comparator<? super K> cmp = m.comparator();
            if (isDescending)
                return Collections.reverseOrder(cmp);
            else
                return cmp;
        }

        /**
         * Utility to create submaps, where given bounds override
         * unbounded(null) ones and/or are checked against bounded ones.
         */
        SubMap<K,V> newSubMap(K fromKey, boolean fromInclusive,
                              K toKey, boolean toInclusive) {
            Comparator<? super K> cmp = m.comparator;
            if (isDescending) { // flip senses
                K tk = fromKey;
                fromKey = toKey;
                toKey = tk;
                boolean ti = fromInclusive;
                fromInclusive = toInclusive;
                toInclusive = ti;
            }
            if (lo != null) {
                if (fromKey == null) {
                    fromKey = lo;
                    fromInclusive = loInclusive;
                }
                else {
                    int c = cpr(cmp, fromKey, lo);
                    if (c < 0 || (c == 0 && !loInclusive && fromInclusive))
                        throw new IllegalArgumentException("key out of range");
                }
            }
            if (hi != null) {
                if (toKey == null) {
                    toKey = hi;
                    toInclusive = hiInclusive;
                }
                else {
                    int c = cpr(cmp, toKey, hi);
                    if (c > 0 || (c == 0 && !hiInclusive && toInclusive))
                        throw new IllegalArgumentException("key out of range");
                }
            }
            return new SubMap<K,V>(m, fromKey, fromInclusive,
                                   toKey, toInclusive, isDescending);
        }

        public SubMap<K,V> subMap(K fromKey, boolean fromInclusive,
                                  K toKey, boolean toInclusive) {
            if (fromKey == null || toKey == null)
                throw new NullPointerException();
            return newSubMap(fromKey, fromInclusive, toKey, toInclusive);
        }

        public SubMap<K,V> headMap(K toKey, boolean inclusive) {
            if (toKey == null)
                throw new NullPointerException();
            return newSubMap(null, false, toKey, inclusive);
        }

        public SubMap<K,V> tailMap(K fromKey, boolean inclusive) {
            if (fromKey == null)
                throw new NullPointerException();
            return newSubMap(fromKey, inclusive, null, false);
        }

        public SubMap<K,V> subMap(K fromKey, K toKey) {
            return subMap(fromKey, true, toKey, false);
        }

        public SubMap<K,V> headMap(K toKey) {
            return headMap(toKey, false);
        }

        public SubMap<K,V> tailMap(K fromKey) {
            return tailMap(fromKey, true);
        }

        public SubMap<K,V> descendingMap() {
            return new SubMap<K,V>(m, lo, loInclusive,
                                   hi, hiInclusive, !isDescending);
        }

        /* ----------------  Relational methods -------------- */

        public Map.Entry<K,V> ceilingEntry(K key) {
            return getNearEntry(key, GT|EQ);
        }

        public K ceilingKey(K key) {
            return getNearKey(key, GT|EQ);
        }

        public Map.Entry<K,V> lowerEntry(K key) {
            return getNearEntry(key, LT);
        }

        public K lowerKey(K key) {
            return getNearKey(key, LT);
        }

        public Map.Entry<K,V> floorEntry(K key) {
            return getNearEntry(key, LT|EQ);
        }

        public K floorKey(K key) {
            return getNearKey(key, LT|EQ);
        }

        public Map.Entry<K,V> higherEntry(K key) {
            return getNearEntry(key, GT);
        }

        public K higherKey(K key) {
            return getNearKey(key, GT);
        }

        public K firstKey() {
            return isDescending ? highestKey() : lowestKey();
        }

        public K lastKey() {
            return isDescending ? lowestKey() : highestKey();
        }

        public Map.Entry<K,V> firstEntry() {
            return isDescending ? highestEntry() : lowestEntry();
        }

        public Map.Entry<K,V> lastEntry() {
            return isDescending ? lowestEntry() : highestEntry();
        }

        public Map.Entry<K,V> pollFirstEntry() {
            return isDescending ? removeHighest() : removeLowest();
        }

        public Map.Entry<K,V> pollLastEntry() {
            return isDescending ? removeLowest() : removeHighest();
        }

        /* ---------------- Submap Views -------------- */

        public NavigableSet<K> keySet() {
            KeySet<K> ks = keySetView;
            return (ks != null) ? ks : (keySetView = new KeySet<K>(this));
        }

        public NavigableSet<K> navigableKeySet() {
            KeySet<K> ks = keySetView;
            return (ks != null) ? ks : (keySetView = new KeySet<K>(this));
        }

        public Collection<V> values() {
            Collection<V> vs = valuesView;
            return (vs != null) ? vs : (valuesView = new Values<V>(this));
        }

        public Set<Map.Entry<K,V>> entrySet() {
            Set<Map.Entry<K,V>> es = entrySetView;
            return (es != null) ? es : (entrySetView = new EntrySet<K,V>(this));
        }

        public NavigableSet<K> descendingKeySet() {
            return descendingMap().navigableKeySet();
        }

        Iterator<K> keyIterator() {
            return new SubMapKeyIterator();
        }

        Iterator<V> valueIterator() {
            return new SubMapValueIterator();
        }

        Iterator<Map.Entry<K,V>> entryIterator() {
            return new SubMapEntryIterator();
        }

        /**
         * Variant of main Iter class to traverse through submaps.
         * Also serves as back-up Spliterator for views
         */
        abstract class SubMapIter<T> implements Iterator<T>, Spliterator<T> {
            /** the last node returned by next() */
            Node<K,V> lastReturned;
            /** the next node to return from next(); */
            Node<K,V> next;
            /** Cache of next value field to maintain weak consistency */
            V nextValue;

            SubMapIter() {
                Comparator<? super K> cmp = m.comparator;
                for (;;) {
                    next = isDescending ? hiNode(cmp) : loNode(cmp);
                    if (next == null)
                        break;
                    Object x = next.value;
                    if (x != null && x != next) {
                        if (! inBounds(next.key, cmp))
                            next = null;
                        else {
                            @SuppressWarnings("unchecked") V vv = (V)x;
                            nextValue = vv;
                        }
                        break;
                    }
                }
            }

            public final boolean hasNext() {
                return next != null;
            }

            final void advance() {
                if (next == null)
                    throw new NoSuchElementException();
                lastReturned = next;
                if (isDescending)
                    descend();
                else
                    ascend();
            }

            private void ascend() {
                Comparator<? super K> cmp = m.comparator;
                for (;;) {
                    next = next.next;
                    if (next == null)
                        break;
                    Object x = next.value;
                    if (x != null && x != next) {
                        if (tooHigh(next.key, cmp))
                            next = null;
                        else {
                            @SuppressWarnings("unchecked") V vv = (V)x;
                            nextValue = vv;
                        }
                        break;
                    }
                }
            }

            private void descend() {
                Comparator<? super K> cmp = m.comparator;
                for (;;) {
                    next = m.findNear(lastReturned.key, LT, cmp);
                    if (next == null)
                        break;
                    Object x = next.value;
                    if (x != null && x != next) {
                        if (tooLow(next.key, cmp))
                            next = null;
                        else {
                            @SuppressWarnings("unchecked") V vv = (V)x;
                            nextValue = vv;
                        }
                        break;
                    }
                }
            }

            public void remove() {
                Node<K,V> l = lastReturned;
                if (l == null)
                    throw new IllegalStateException();
                m.remove(l.key);
                lastReturned = null;
            }

            public Spliterator<T> trySplit() {
                return null;
            }

            public boolean tryAdvance(Consumer<? super T> action) {
                if (hasNext()) {
                    action.accept(next());
                    return true;
                }
                return false;
            }

            public void forEachRemaining(Consumer<? super T> action) {
                while (hasNext())
                    action.accept(next());
            }

            public long estimateSize() {
                return Long.MAX_VALUE;
            }

        }

        final class SubMapValueIterator extends SubMapIter<V> {
            public V next() {
                V v = nextValue;
                advance();
                return v;
            }
            public int characteristics() {
                return 0;
            }
        }

        final class SubMapKeyIterator extends SubMapIter<K> {
            public K next() {
                Node<K,V> n = next;
                advance();
                return n.key;
            }
            public int characteristics() {
                return Spliterator.DISTINCT | Spliterator.ORDERED |
                    Spliterator.SORTED;
            }
            public final Comparator<? super K> getComparator() {
                return SubMap.this.comparator();
            }
        }

        final class SubMapEntryIterator extends SubMapIter<Map.Entry<K,V>> {
            public Map.Entry<K,V> next() {
                Node<K,V> n = next;
                V v = nextValue;
                advance();
                return new AbstractMap.SimpleImmutableEntry<K,V>(n.key, v);
            }
            public int characteristics() {
                return Spliterator.DISTINCT;
            }
        }
    }

    // default Map method overrides

    public void forEach(BiConsumer<? super K, ? super V> action) {
        if (action == null) throw new NullPointerException();
        V v;
        for (Node<K,V> n = findFirst(); n != null; n = n.next) {
            if ((v = n.getValidValue()) != null)
                action.accept(n.key, v);
        }
    }

    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        if (function == null) throw new NullPointerException();
        V v;
        for (Node<K,V> n = findFirst(); n != null; n = n.next) {
            while ((v = n.getValidValue()) != null) {
                V r = function.apply(n.key, v);
                if (r == null) throw new NullPointerException();
                if (n.casValue(v, r))
                    break;
            }
        }
    }

    /**
     * Base class providing common structure for Spliterators.
     * (Although not all that much common functionality; as usual for
     * view classes, details annoyingly vary in key, value, and entry
     * subclasses in ways that are not worth abstracting out for
     * internal classes.)
     *
     * The basic split strategy is to recursively descend from top
     * level, row by row, descending to next row when either split
     * off, or the end of row is encountered. Control of the number of
     * splits relies on some statistical estimation: The expected
     * remaining number of elements of a skip list when advancing
     * either across or down decreases by about 25%. To make this
     * observation useful, we need to know initial size, which we
     * don't. But we can just use Integer.MAX_VALUE so that we
     * don't prematurely zero out while splitting.
     */
    abstract static class CSLMSpliterator<K,V> {
        final Comparator<? super K> comparator;
        final K fence;     // exclusive upper bound for keys, or null if to end
        Index<K,V> row;    // the level to split out
        Node<K,V> current; // current traversal node; initialize at origin
        int est;           // pseudo-size estimate
        CSLMSpliterator(Comparator<? super K> comparator, Index<K,V> row,
                        Node<K,V> origin, K fence, int est) {
            this.comparator = comparator; this.row = row;
            this.current = origin; this.fence = fence; this.est = est;
        }

        public final long estimateSize() { return (long)est; }
    }

    static final class KeySpliterator<K,V> extends CSLMSpliterator<K,V>
        implements Spliterator<K> {
        KeySpliterator(Comparator<? super K> comparator, Index<K,V> row,
                       Node<K,V> origin, K fence, int est) {
            super(comparator, row, origin, fence, est);
        }

        public Spliterator<K> trySplit() {
            Node<K,V> e; K ek;
            Comparator<? super K> cmp = comparator;
            K f = fence;
            if ((e = current) != null && (ek = e.key) != null) {
                for (Index<K,V> q = row; q != null; q = row = q.down) {
                    Index<K,V> s; Node<K,V> b, n; K sk;
                    if ((s = q.right) != null && (b = s.node) != null &&
                        (n = b.next) != null && n.value != null &&
                        (sk = n.key) != null && cpr(cmp, sk, ek) > 0 &&
                        (f == null || cpr(cmp, sk, f) < 0)) {
                        current = n;
                        Index<K,V> r = q.down;
                        row = (s.right != null) ? s : s.down;
                        est -= est >>> 2;
                        return new KeySpliterator<K,V>(cmp, r, e, sk, est);
                    }
                }
            }
            return null;
        }

        public void forEachRemaining(Consumer<? super K> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            current = null;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0)
                    break;
                if ((v = e.value) != null && v != e)
                    action.accept(k);
            }
        }

        public boolean tryAdvance(Consumer<? super K> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0) {
                    e = null;
                    break;
                }
                if ((v = e.value) != null && v != e) {
                    current = e.next;
                    action.accept(k);
                    return true;
                }
            }
            current = e;
            return false;
        }

        public int characteristics() {
            return Spliterator.DISTINCT | Spliterator.SORTED |
                Spliterator.ORDERED | Spliterator.CONCURRENT |
                Spliterator.NONNULL;
        }

        public final Comparator<? super K> getComparator() {
            return comparator;
        }
    }
    // factory method for KeySpliterator
    final KeySpliterator<K,V> keySpliterator() {
        Comparator<? super K> cmp = comparator;
        for (;;) { // ensure h corresponds to origin p
            HeadIndex<K,V> h; Node<K,V> p;
            Node<K,V> b = (h = head).node;
            if ((p = b.next) == null || p.value != null)
                return new KeySpliterator<K,V>(cmp, h, p, null, (p == null) ?
                                               0 : Integer.MAX_VALUE);
            p.helpDelete(b, p.next);
        }
    }

    static final class ValueSpliterator<K,V> extends CSLMSpliterator<K,V>
        implements Spliterator<V> {
        ValueSpliterator(Comparator<? super K> comparator, Index<K,V> row,
                       Node<K,V> origin, K fence, int est) {
            super(comparator, row, origin, fence, est);
        }

        public Spliterator<V> trySplit() {
            Node<K,V> e; K ek;
            Comparator<? super K> cmp = comparator;
            K f = fence;
            if ((e = current) != null && (ek = e.key) != null) {
                for (Index<K,V> q = row; q != null; q = row = q.down) {
                    Index<K,V> s; Node<K,V> b, n; K sk;
                    if ((s = q.right) != null && (b = s.node) != null &&
                        (n = b.next) != null && n.value != null &&
                        (sk = n.key) != null && cpr(cmp, sk, ek) > 0 &&
                        (f == null || cpr(cmp, sk, f) < 0)) {
                        current = n;
                        Index<K,V> r = q.down;
                        row = (s.right != null) ? s : s.down;
                        est -= est >>> 2;
                        return new ValueSpliterator<K,V>(cmp, r, e, sk, est);
                    }
                }
            }
            return null;
        }

        public void forEachRemaining(Consumer<? super V> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            current = null;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0)
                    break;
                if ((v = e.value) != null && v != e) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    action.accept(vv);
                }
            }
        }

        public boolean tryAdvance(Consumer<? super V> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0) {
                    e = null;
                    break;
                }
                if ((v = e.value) != null && v != e) {
                    current = e.next;
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    action.accept(vv);
                    return true;
                }
            }
            current = e;
            return false;
        }

        public int characteristics() {
            return Spliterator.CONCURRENT | Spliterator.ORDERED |
                Spliterator.NONNULL;
        }
    }

    // Almost the same as keySpliterator()
    final ValueSpliterator<K,V> valueSpliterator() {
        Comparator<? super K> cmp = comparator;
        for (;;) {
            HeadIndex<K,V> h; Node<K,V> p;
            Node<K,V> b = (h = head).node;
            if ((p = b.next) == null || p.value != null)
                return new ValueSpliterator<K,V>(cmp, h, p, null, (p == null) ?
                                                 0 : Integer.MAX_VALUE);
            p.helpDelete(b, p.next);
        }
    }

    static final class EntrySpliterator<K,V> extends CSLMSpliterator<K,V>
        implements Spliterator<Map.Entry<K,V>> {
        EntrySpliterator(Comparator<? super K> comparator, Index<K,V> row,
                         Node<K,V> origin, K fence, int est) {
            super(comparator, row, origin, fence, est);
        }

        public Spliterator<Map.Entry<K,V>> trySplit() {
            Node<K,V> e; K ek;
            Comparator<? super K> cmp = comparator;
            K f = fence;
            if ((e = current) != null && (ek = e.key) != null) {
                for (Index<K,V> q = row; q != null; q = row = q.down) {
                    Index<K,V> s; Node<K,V> b, n; K sk;
                    if ((s = q.right) != null && (b = s.node) != null &&
                        (n = b.next) != null && n.value != null &&
                        (sk = n.key) != null && cpr(cmp, sk, ek) > 0 &&
                        (f == null || cpr(cmp, sk, f) < 0)) {
                        current = n;
                        Index<K,V> r = q.down;
                        row = (s.right != null) ? s : s.down;
                        est -= est >>> 2;
                        return new EntrySpliterator<K,V>(cmp, r, e, sk, est);
                    }
                }
            }
            return null;
        }

        public void forEachRemaining(Consumer<? super Map.Entry<K,V>> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            current = null;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0)
                    break;
                if ((v = e.value) != null && v != e) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    action.accept
                        (new AbstractMap.SimpleImmutableEntry<K,V>(k, vv));
                }
            }
        }

        public boolean tryAdvance(Consumer<? super Map.Entry<K,V>> action) {
            if (action == null) throw new NullPointerException();
            Comparator<? super K> cmp = comparator;
            K f = fence;
            Node<K,V> e = current;
            for (; e != null; e = e.next) {
                K k; Object v;
                if ((k = e.key) != null && f != null && cpr(cmp, f, k) <= 0) {
                    e = null;
                    break;
                }
                if ((v = e.value) != null && v != e) {
                    current = e.next;
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    action.accept
                        (new AbstractMap.SimpleImmutableEntry<K,V>(k, vv));
                    return true;
                }
            }
            current = e;
            return false;
        }

        public int characteristics() {
            return Spliterator.DISTINCT | Spliterator.SORTED |
                Spliterator.ORDERED | Spliterator.CONCURRENT |
                Spliterator.NONNULL;
        }

        public final Comparator<Map.Entry<K,V>> getComparator() {
            // Adapt or create a key-based comparator
            if (comparator != null) {
                return Map.Entry.comparingByKey(comparator);
            }
            else {
                return (Comparator<Map.Entry<K,V>> & Serializable) (e1, e2) -> {
                    @SuppressWarnings("unchecked")
                    Comparable<? super K> k1 = (Comparable<? super K>) e1.getKey();
                    return k1.compareTo(e2.getKey());
                };
            }
        }
    }

    // Almost the same as keySpliterator()
    final EntrySpliterator<K,V> entrySpliterator() {
        Comparator<? super K> cmp = comparator;
        for (;;) { // almost same as key version
            HeadIndex<K,V> h; Node<K,V> p;
            Node<K,V> b = (h = head).node;
            if ((p = b.next) == null || p.value != null)
                return new EntrySpliterator<K,V>(cmp, h, p, null, (p == null) ?
                                                 0 : Integer.MAX_VALUE);
            p.helpDelete(b, p.next);
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long headOffset;
    private static final long SECONDARY;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentSkipListMap.class;
            headOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("head"));
            Class<?> tk = Thread.class;
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));

        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```