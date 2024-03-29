# HashMap一记

[TOC]



## 1.容量大小

初始化时，默认容量为16；最大容量为：1<<30。

可以人为设置大小，比如设置14，会通过计算后得出一个2的幂次方的一个数，具体算法如下：

```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

为什么一定要是2的幂次方，后面会说。

其中位运算符号 >>> 与 >> 的区别是：前者表示 不带符号位右移，后者表示 带符号位右移。这对于正数是无影响的，但对负数是不一样的结果。



## 2.负载因子

默认是0.75。

这个值表示hashmap可以装载数据的饱和度。越大表示保存的数据越多，与此冲突的概率就会越高；越低，则空间利用率就降低，会影响内存使用率。所以，这是时间与空间的tradeoff。

当容量为16、负载因子为0.75时，存储的元素达到个数size=16*0.75=12时，会触发扩容机制。

举个例子：有1000个数据要存储，在默认加载因子下，直接申请多大的空间合适？

答案是：2048



## 3.TREEIFY_THRESHOLD=8

自JDK1.8后，HashMap采用了数组+链表+红黑树的方式来实现。当链表的长度大于8时，会将链表转换为红黑树来存储数据。

![image-20200126112400255](/Users/dazhou/Library/Application Support/typora-user-images/image-20200126112400255.jpg)

HashMap的基本存储结构如上图所示。

## 4.UNTREEIFY_THRESHOLD=6

当红黑树的节点数小于6时，会再转换成链表式。



## 5.MIN_TREEIFY_CAPACITY=64

即使是在某个table节点下，链表长度大于了8，但整个table的长度还在64以内时，就不会对此table节点后的链表进行红黑树化，而是通过扩大table的整个长度来实现扩容，具体是通过resize()函数来实现。







## 6.扩容为什么是2的幂次方

数据要存储的位置，是通过hash计算以后得到的值value后，与table表的长度进行求余操作后最终得到的位置index。由于求余操作相对耗时，如果能通过位运算得到index，则效率会更高。

在table的length为2的幂次方时，value%length == value & （length-1），因此只要length满足2的幂次方，则求index时就可以通过位运算来实现。

length-1的值最高位后都是1111，比如16-1=15，为0x1111，这样与key位运算时，计算出相同的index的几率会降低，这样就会降低碰撞几率，使数据分布更均匀。从而查询时效率就会更高。

在已经确定通过&操作求index的前提下，如果某位为0，比如length为15时，length-1=0x1110，这意味着第0位对应的数据，无论与什么数据进行&操作，都不可能是1，这样就会导致0001、0011、0101、1001、1011、0111、1101，这几个位置永远不可能存放数据，这会导致空间的浪费，会增加碰撞几率。



## 7.`HashMap`和`HashTable`的区别

1）容器整体结构：

- `HashMap`的`key`和`value`都允许为`null`，`HashMap`遇到`key`为`null`的时候，调用`putForNullKey`方法进行处理，而对`value`没有处理。
- `Hashtable`的`key`和`value`都不允许为`null`。`Hashtable`遇到`null`，直接返回`NullPointerException`。

2） 容量设定与扩容机制：

- `HashMap`默认初始化容量为 16，并且容器容量一定是2的n次方，扩容时，是以原容量 2倍 的方式 进行扩容。
- `Hashtable`默认初始化容量为 11，扩容时，是以原容量 2倍 再加 1的方式进行扩容。即`int newCapacity = (oldCapacity << 1) + 1;`。

3） 散列分布方式（计算存储位置）：

- `HashMap`是先将`key`键的`hashCode`经过扰动函数扰动后得到`hash`值，然后再利用 `hash & (length - 1)`的方式代替取模，得到元素的存储位置。
- `Hashtable`则是除留余数法进行计算存储位置的（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算），`int index = (hash & 0x7FFFFFFF) % tab.length;`。
- 由于`HashMap`的容器容量一定是2的n次方，所以能使用`hash & (length - 1)`的方式代替取模的方式计算元素的位置提高运算效率，但`Hashtable`的容器容量不一定是2的n次方，所以不能使用此运算方式代替。

4）线程安全（最重要）：

- `HashMap` 不是线程安全，如果想线程安全，可以通过调用`synchronizedMap(Map m)`使其线程安全。但是使用时的运行效率会下降，所以建议使用`ConcurrentHashMap`容器以此达到线程安全。
- `Hashtable`则是线程安全的，每个操作方法前都有`synchronized`修饰使其同步，但运行效率也不高，所以还是建议使用`ConcurrentHashMap`容器以此达到线程安全。

因此，`Hashtable`是一个遗留容器，如果我们不需要线程同步，则建议使用`HashMap`，如果需要线程同步，则建议使用`ConcurrentHashMap`。



## 8.key值选择

- 可变对象：指创建后自身状态能改变的对象。换句话说，可变对象是该对象在创建后它的哈希值可能被改变。
- 我们在使用`HashMap`时，最好选择不可变对象作为`key`。例如`String`，`Integer`等不可变类型作为`key`是非常明智的。
- 如果`key`对象是可变的，那么`key`的哈希值就可能改变。在`HashMap`中可变对象作为Key会造成数据丢失。因为我们再进行`hash & (length - 1)`取模运算计算位置查找对应元素时，位置可能已经发生改变，导致数据丢失。



## 9.put操作

- 

1. 判断哈希表`Node[] table`是否为空或者`null`，是则执行`resize()`方法进行扩容。
2. 根据插入的键值`key`的`hash`值，通过`(n - 1) & hash`当前元素的`hash`值 & `hash`表长度 - 1（实际就是 `hash`值 % `hash`表长度） 计算出存储位置`table[i]`。如果存储位置没有元素存放，则将新增结点存储在此位置`table[i]`。
3. 如果存储位置已经有键值对元素存在，则判断该位置元素的`hash`值和`key`值是否和当前操作元素一致，一致则证明是修改`value`操作，覆盖`value`即可。
4. 当前存储位置即有元素，又不和当前操作元素一致，则证明此位置`table[i]`已经发生了hash冲突，则通过判断头结点是否是`treeNode`，如果是`treeNode`则证明此位置的结构是红黑树，已红黑树的方式新增结点。
5. 如果不是红黑树，则证明是单链表，将新增结点插入至链表的最后位置，随后判断当前链表长度是否 大于等于 8，是则将当前存储位置的链表转化为红黑树。遍历过程中如果发现`key`已经存在，则直接覆盖`value`。
6. 插入成功后，判断当前存储键值对的数量 大于 阈值`threshold` 是则扩容。



## 10.get操作

1. 先调用 `hash(key)`方法计算出 `key` 的 `hash`值
2. 根据查找的键值`key`的`hash`值，通过`(n - 1) & hash`当前元素的`hash`值  & `hash`表长度 - 1（实际就是 `hash`值 % `hash`表长度） 计算出存储位置`table[i]`，判断存储位置是否有元素存在 。

- 如果存储位置有元素存放，则首先比较头结点元素，如果头结点的`key`的`hash`值 和 要获取的`key`的`hash`值相等，并且 头结点的`key`本身 和要获取的 `key` 相等，则返回该位置的头结点。
- 如果存储位置没有元素存放，则返回`null`。

1. 如果存储位置有元素存放，但是头结点元素不是要查找的元素，则需要遍历该位置进行查找。
2. 先判断头结点是否是`treeNode`，如果是`treeNode`则证明此位置的结构是红黑树，以红色树的方式遍历查找该结点，没有则返回`null`。
3. 如果不是红黑树，则证明是单链表。遍历单链表，逐一比较链表结点，链表结点的`key`的`hash`值 和 要获取的`key`的`hash`值相等，并且 链表结点的`key`本身 和要获取的 `key` 相等，则返回该结点，遍历结束仍未找到对应`key`的结点，则返回`null`。



## 11.remove操作

1. 先调用 `hash(key)`方法计算出 `key` 的 `hash`值
2. 根据查找的键值`key`的`hash`值，通过`(n - 1) & hash`当前元素的`hash`值  & `hash`表长度 - 1（实际就是 `hash`值 % `hash`表长度） 计算出存储位置`table[i]`，判断存储位置是否有元素存在 。

- 如果存储位置有元素存放，则首先比较头结点元素，如果头结点的`key`的`hash`值 和 要获取的`key`的`hash`值相等，并且 头结点的`key`本身 和要获取的 `key` 相等，则该位置的头结点即为要删除的结点，记录此结点至变量`node`中。
- 如果存储位置没有元素存放，则没有找到对应要删除的结点，则返回`null`。

1. 如果存储位置有元素存放，但是头结点元素不是要删除的元素，则需要遍历该位置进行查找。
2. 先判断头结点是否是`treeNode`，如果是`treeNode`则证明此位置的结构是红黑树，以红色树的方式遍历查找并删除该结点，没有则返回`null`。
3. 如果不是红黑树，则证明是单链表。遍历单链表，逐一比较链表结点，链表结点的`key`的`hash`值 和 要获取的`key`的`hash`值相等，并且 链表结点的`key`本身 和要获取的 `key` 相等，则此为要删除的结点，记录此结点至变量`node`中，遍历结束仍未找到对应`key`的结点，则返回`null`。
4. 如果找到要删除的结点`node`，则判断是否需要比较`value`也是否一致，如果`value`值一致或者不需要比较`value`值，则执行删除结点操作，删除操作根据不同的情况与结构进行不同的处理。

- 如果当前结点是树结点，则证明当前位置的链表已变成红黑树结构，通过红黑树结点的方式删除对应结点。
- 如果不是红黑树，则证明是单链表。如果要删除的是头结点，则当前存储位置`table[i]`的头结点指向删除结点的下一个结点。
- 如果要删除的结点不是头结点，则将要删除的结点的后继结点`node.next`赋值给要删除结点的前驱结点的`next`域，即`p.next = node.next;`。

7. `HashMap`当前存储键值对的数量 - 1，并返回删除结点。





## 总结

1. `HashMap`是基于`Map`接口实现的一种键-值对``的存储结构，允许`null`值，同时非有序，非同步(即线程不安全)。`HashMap`的底层实现是数组 + 链表 + 红黑树（JDK1.8增加了红黑树部分）。
2. `HashMap`定位元素位置是通过键`key`经过扰动函数扰动后得到`hash`值，然后再通过`hash & (length - 1)`代替取模的方式进行元素定位的。
3. `HashMap`是使用链地址法解决`hash`冲突的，当有冲突元素放进来时，会将此元素插入至此位置链表的最后一位，形成单链表。当存在位置的链表长度 大于等于 8 时，`HashMap`会将链表 转变为 红黑树，以此提高查找效率。
4. `HashMap`的容量是2的n次方，有利于提高计算元素存放位置时的效率，也降低了`hash`冲突的几率。因此，我们使用`HashMap`存储大量数据的时候，最好先预先指定容器的大小为2的n次方，即使我们不指定为2的n次方，`HashMap`也会把容器的大小设置成最接近设置数的2的n次方，如，设置`HashMap`的大小为 7 ，则`HashMap`会将容器大小设置成最接近7的一个2的n次方数，此值为 8 。
5. `HashMap`的负载因子表示哈希表空间的使用程度（或者说是哈希表空间的利用率）。当负载因子越大，则`HashMap`的装载程度就越高。也就是能容纳更多的元素，元素多了，发生`hash`碰撞的几率就会加大，从而链表就会拉长，此时的查询效率就会降低。当负载因子越小，则链表中的数据量就越稀疏，此时会对空间造成浪费，但是此时查询效率高。
6. `HashMap`不是线程安全的，`Hashtable`则是线程安全的。但`Hashtable`是一个遗留容器，如果我们不需要线程同步，则建议使用`HashMap`，如果需要线程同步，则建议使用`ConcurrentHashMap`。
7. 在多线程下操作`HashMap`，由于存在扩容机制，当`HashMap`调用`resize()`进行自动扩容时，可能会导致死循环的发生。
8. 我们在使用`HashMap`时，最好选择不可变对象作为`key`。例如`String`，`Integer`等不可变类型作为`key`是非常明智的。



## 参考链接

https://www.jianshu.com/p/32f67f9e71b5









