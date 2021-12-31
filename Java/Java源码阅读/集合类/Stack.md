Stack继承Vector，所以底层也是数组，并且支持多线程，所有实现的方法都是直接调用的Vector方法的实现。值得一提的是Vector删除元素是通过``System.arraycopy(elementData, index + 1, elementData, index, j)``实现的，所以删除的效率很低，另外删除后Vector容量是不变的，所以Vector的数组只是变大不会变小。

```java
package dhf;

import java.util.EmptyStackException;
import java.util.Vector;

public class Stack<E> extends Vector<E> {
    public Stack() {
    }

    public E push(E item) {
        addElement(item);

        return item;
    }

    public synchronized E pop() {
        E       obj;
        int     len = size();

        obj = peek();
        removeElementAt(len - 1);

        return obj;
    }

    public synchronized E peek() {
        int     len = size();

        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);
    }

    public boolean empty() {
        return size() == 0;
    }

    public synchronized int search(Object o) {
        int i = lastIndexOf(o);

        if (i >= 0) {
            return size() - i;
        }
        return -1;
    }

    private static final long serialVersionUID = 1224463164541339165L;
}
```