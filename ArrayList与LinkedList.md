# ArrayList

底层存储结构是：

```
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
// Android-note: Also accessed from java.util.Collections
transient Object[] elementData; // non-private to simplify nested class access
```

因此，对索引访问方式效率高，对增删效率低。



## add操作

```
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
		// 1.递增modCount变量；2.判断存储空间是否够，不够，则通过grow函数扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```



```java
/**
 * Increases the capacity to ensure that it can hold at least the
 * number of elements specified by the minimum capacity argument.
 *
 * @param minCapacity the desired minimum capacity
 */
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

计算出新的存储空间大小后，再通过Arrays的copyof函数拷贝复制过去。因此，扩容效率偏低。如果能提前知道大概的存储数据，可以直接在初始化时，直接申请这么大的，避免中途扩容。



## get操作

```
/**
 * Returns the element at the specified position in this list.
 *
 * @param  index index of the element to return
 * @return the element at the specified position in this list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index];
}
```

get操作相对简单，只对上届进行了判断。如果index是负数，则应该会异常。





# LinkedList

底层结构为链表，所以对于随机访问效率较低；但对增删效率较高。

该数据结构提供了很多方法以便于操作。

## 表头表尾插入

```
/**
 * Inserts the specified element at the beginning of this list.
 *
 * @param e the element to add
 */
public void addFirst(E e) {
    linkFirst(e);
}

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
public void addLast(E e) {
    linkLast(e);
}
```

实际上调用：

```
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

/**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```



## 随机插入操作：

```dart
/** * Inserts the specified element at the specified position in this list. 
     * Shifts the element currently at that position (if any) and any 
     * subsequent elements to the right (adds one to their indices). 
     * * @param index index at which the specified element is to be inserted 
     * @param element element to be inserted 
     * @throws IndexOutOfBoundsException {@inheritDoc} 
*/
public void add(int index, E element) {    
  checkPositionIndex(index);    
  if (index == size)        
    linkLast(element);    
  else        
    linkBefore(element, node(index));
}
```

先检查index的合法性；

判断index是否处于表尾，若是，则表尾插入；

否则，找到插入位置后，再插入；

```dart
/** * Inserts element e before non-null Node succ. */
void linkBefore(E e, Node<E> succ) {    
  // assert succ != null;    
  final Node<E> pred = succ.prev;    
  final Node<E> newNode = new Node<>(pred, e, succ);    
  succ.prev = newNode;    
  if (pred == null)        
    first = newNode;    
  else        
    pred.next = newNode;    
  size++;    
  modCount++;
}
```



查找位置，通过node函数查找到特定节点，然后在此节点前插入：

```csharp
/** * Returns the (non-null) Node at the specified element index. */
Node<E> node(int index) {    
  // assert isElementIndex(index);    
  if (index < (size >> 1)) {        
    Node<E> x = first;        
    for (int i = 0; i < index; i++)            
      x = x.next;        
    return x;    
  } else {        
    Node<E> x = last;        
    for (int i = size - 1; i > index; i--)            
      x = x.prev;        
    return x;    
  }
}
```

查找的复杂度为O（n），插入复杂度为O（1）



## 删除元素

```
/**
 * Removes the first occurrence of the specified element from this list,
 * if it is present.  If this list does not contain the element, it is
 * unchanged.  More formally, removes the element with the lowest index
 * {@code i} such that
 * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
 * (if such an element exists).  Returns {@code true} if this list
 * contained the specified element (or equivalently, if this list
 * changed as a result of the call).
 *
 * @param o element to be removed from this list, if present
 * @return {@code true} if this list contained the specified element
 */
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```



## 获取头部元素

```
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
```

等价于：

```
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
 }
```





```
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

等价于：

```
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```

