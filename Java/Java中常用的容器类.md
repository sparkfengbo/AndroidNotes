**文章内容**

- [1.容器总体结构](#容器总体结构)
- [2.Map](#Map)
  - [2.1 HashMap](#HashMap)
  - [2.2 Hashtable](#Hashtable)
  - [2.3 LinkedHashMap](#LinkedHashMap)
  - [2.4 TreeMap](#TreeMap)
- [3.Collection](#Collection)
  - [3.1 List](#List)
      - [ArrayList](#ArrayList)
      - [LinkedList](#LinkedList)
      - [Vector](#Vector)
  - [3.2 Set](#Set)
      - [HashSet](#HashSet)
      - [TreeSet](#TreeSet)
      - [LinkedHashSet](#LinkedHashSet)
  - [3.3 Queue与Deque](#Queue与Deque)
      - [ArrayDeque](#ArrayDeque)
      - [PriorityQueue](#PriorityQueue)
- [4.同步集合](#tongbu)     
   - [4.1 CopyOnWriteArrayList](#CopyOnWriteArrayList)
   - [4.2 CopyOnWriteArrayset](#CopyOnWriteArrayset)
   - [4.3 ArrayBlockkingQueue](#ArrayBlockkingQueue)
   - [4.4 LinkedBlockingQueue](#LinkedBlockingQueue)
- [5.散列码产生规则](#sanlie) 
    

## <a name="1.容器总体结构">容器总体结构</a>

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/%E5%AE%B9%E5%99%A8%E7%BB%93%E6%9E%84.jpeg?raw=true)

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/%E5%AE%B9%E5%99%A8%E7%AE%80%E5%8C%96%E5%9B%BE.png?raw=true)

```
public interface Collection<E> extends Iterable<E> {}

public interface Map<K, V> {
	...
	
	Collection<V> values();	
	...
}

public interface List<E> extends Collection<E> {}

public abstract class AbstractCollection<E> implements Collection<E> {}

public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {}

```


## <a name="Map">2.Map</a>

### <a name="HashMap">2.1 HashMap</a>

|关  注  点| 结      论|
| --------  | :----: |
| HashMap是否允许空  | Key和Value都允许为空,允许<null, null>的键值对   |
| HashMap是否允许重复数据  | Key重复会覆盖、Value允许重复   |
| HashMap是否有序  | 无序，特别说明这个无序指的是遍历HashMap的时候，得到的元素的顺序基本不可能是put的顺序   |
| HashMap是否线程安全  | 非线程安全   |

#### 2.1.1 HashMap的存储结构

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/JavaBCXS/hashmap.jpeg?raw=true)

#### 2.1.2 两个重要的参数 Capacity 和 Load Factor

简单的说，Capacity就是buckets的数目，容量都是2的幂。Load factor就是buckets填满程度的最大比例。如果对迭代性能要求很高的话不要把capacity设置过大，也不要把load factor设置过小。当bucket填充的数目（即hashmap中元素的个数）大于capacity*load factor时就需要调整buckets的数目为当前的2倍。

#### 2.1.3 put方法的实现

put函数大致的思路为：

- 对key的hashCode()做hash(调用了hash方法)，然后再计算index;
- 如果没碰撞直接放到bucket里；
- 如果碰撞了，以链表的形式存在buckets后；
- 如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树(Java 8中做的优化,但是在Android API25 中还没有看到这样的优化)；
- 如果节点已经存在就替换old value(保证key的唯一性)
- 如果bucket满了(超过load factor*current capacity)，就要resize。


#### 2.1.4 get方法的实现

在理解了put之后，get就很简单了。大致思路如下：

- bucket里的第一个节点，直接命中；
- 如果有冲突，则通过key.equals(k)去查找对应的entry
    - 若为树，则在树中通过key.equals(k)查找，O(logn)；
    - 若为链表，则在链表中通过key.equals(k)查找，O(n)。


#### 2.1.5 hash方法的实现

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

```

#### 2.1.6 resize

当put时，如果发现目前的bucket占用程度已经超过了Load Factor所希望的比例，那么就会发生resize。在resize的过程，简单的说就是把bucket扩充为2倍，之后重新计算index，把节点再放到新的bucket中。


#### 2.1.7 红黑树和链表的实现

我们在树中确实存储了比链表更多的数据。

根据继承原则，内部表中可以包含Node（链表）或者TreeNode（红黑树）。Oracle决定根据下面的规则来使用这两种数据结构：

- 对于内部表中的指定索引（桶），如果node的数目多于8个（TREEIFY_THRESHOLD = 8），那么链表就会被转换成红黑树。

- 对于内部表中的指定索引（桶），如果node的数目小于6个（UNTREEIFY_THRESHOLD = 6），那么红黑树就会被转换成链表。


**参考:**

- [Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)(主要参考这一篇，写的很清晰)
- [深入分析hashmap](https://blog.csdn.net/lianhuazy167/article/details/66967698)
- [Java HashMap工作原理](http://www.importnew.com/16599.html)


### <a name="Hashtable">2.2 Hashtable</a>


|关  注  点| 结      论|
| --------  | :----: |
| Hashtable是否允许空  | key和value都不允许null  |
| Hashtable是否允许重复数据  | key不能重复，value允许   |
| Hashtable是否有序  | 不保证有序   |
| Hashtable是否线程安全  | 线程安全   |

**Hashtable和HashMap区别**

- 第一，继承不同。
   `public class Hashtable extends Dictionary implements Map`
  
   `public class HashMap  extends AbstractMap implements Map`
　
- 第二，Hashtable中的方法是同步的(通过synchronized实现)，而HashMap中的方法在缺省情况下是非同步的。在多线程并发的环境下，可以直接使用Hashtable，但是要使用HashMap的话就要自己增加同步处理了。

- 第三，Hashtable中，key和value都不允许出现null值。在HashMap中，null可以作为键，这样的键只有一个；可以有一个或多个键所对应的值为null。

	当get()方法返回null值时，既可以表示HashMap中没有该键，也可以表示该键所对应的值为null。因此，**在HashMap中不能由get()方法来判断HashMap中是否存在某个键，**而应该用containsKey()方法来判断。
- 第四，两个遍历方式的内部实现上不同。Hashtable，HashMap都使用了Iterator。而由于历史原因，Hashtable还使用了Enumeration的方式。并且HashMap的迭代器是[fail-fast机制](https://baike.baidu.com/item/fail-fast/16329854?fr=aladdin)，Hashtable的Enumeration迭代器不是fail-fast的
- 第五，哈希值的使用不同，Hashtable直接使用对象的hashCode。而HashMap重新计算hash值。
- 第六，Hashtable和HashMap它们两个内部实现方式的数组的初始大小和扩容的方式。Hashtable中hash数组默认大小是11，增加的方式是 old*2+1。HashMap中hash数组的默认大小是16，而且一定是2的指数。

**参考**

- [Hashtable 的实现原理](https://www.beibq.cn/book/sjsa262-9221)
- [HashMap和Hashtable的区别](http://www.importnew.com/7010.html)

### <a name="LinkedHashMap">2.3 LinkedHashMap</a> 

|关  注  点| 结      论|
| --------  | :----: |
| LinkedHashMap是否允许空  | Key和Value都允许空  |
| LinkedHashMap是否允许重复数据  | Key重复会覆盖、Value允许重复   |
| LinkedHashMap是否有序  | 有序   |
| LinkedHashMap是否线程安全  | 非线程安全 |

Entry增加了两个变量，after和befor用来维护双向链表

**LruCache使用LinkedHashMap作为缓存**

对节点进行访问后会调用afterNodeAccess方法，更新列表，将最近访问的元素放在最后，afterNodeAccess 会调用afterNodeInsertion方法，在afterNodeInsertion方法中会调用removeNode方法，而在LruCache中对重写removeNode方法，当size超过LruCache的容量的时候就会删除。(不同的JDK和Android API中实现的方式有所不同)


**参考**

- [Java LinkedHashMap工作原理及实现](https://yikun.github.io/2015/04/02/Java-LinkedHashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [LinkedHashMap 与 LRUcache](https://www.beibq.cn/book/sjsa262-9227)
- [图解集合6：LinkedHashMap](https://www.cnblogs.com/xrq730/p/5052323.html)

### <a name="TreeMap">2.4 TreeMap</a>

|关  注  点| 结      论|
| --------  | :----: |
| TreeMap是否允许空  | Key不能为null，Value允许空  |
| TreeMap是否允许重复数据  | Key重复会覆盖、Value允许重复   |
| TreeMap是否有序  | 不能保证按插入的顺序有序，而是根据Key值进行排序   |
| TreeMap是否线程安全  | 非线程安全 |

之前已经学习过HashMap和LinkedHashMap了，HashMap不保证数据有序，LinkedHashMap保证数据可以保持插入顺序，而**如果我们希望Map可以保持key的大小顺序的时候，我们就需要利用TreeMap了**。

另外需要参考[红黑树的笔记](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E7%AE%97%E6%B3%95/%E7%BA%A2%E9%BB%91%E6%A0%91.md)


**参考文章**

- [Java TreeMap工作原理及实现](http://yikun.github.io/2015/04/06/Java-TreeMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [通过分析 JDK 源代码研究 TreeMap 红黑树算法实现](https://www.ibm.com/developerworks/cn/java/j-lo-tree/index.html)

### <a name="WeakHashMap">2.5 WeakHashMap</a>

WeakHashMap是一种改进的HashMap，它对key实行“弱引用”.

```
private static class Entry<K, V> extends WeakReference<Object> implements java.util.Map.Entry<K, V>
```


- [Java 中的 WeakHashMap](http://www.importnew.com/27358.html)
- [Java中的四大引用](https://github.com/sparkfengbo/AndroidNotes/blob/master/Java/Java%E4%B8%AD%E7%9A%84%E5%9B%9B%E5%A4%A7%E5%BC%95%E7%94%A8.md)


## <a name="Collection">3.Collection</a>


**不要将Collection误认为Collections**

- 1.`java.util.Collection` 是一个集合接口。它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java 类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式。 
- 2.`java.util.Collections` 是一个包装类。它包含有各种有关集合操作的静态多态方法。此类不能实例化，就像一个工具类，服务于Java的Collection框架。

### <a name="List">3.1 List</a>

#### <a name="ArrayList">3.1.1 ArrayList</a>


|关  注  点| 结      论|
| --------  | :----: |
| ArrayList是否允许空  | 允许   |
| ArrayList是否允许重复数据  | 允许   |
| ArrayList是否有序  | 有序   |
| ArrayList是否线程安全  | 非线程安全   |

（是否有序，有序的意思是读取数据的顺序和存放数据的顺序是否一致）

**ArrayList比较适合顺序添加.随机访问的场景。**


**增加元素进行扩容**

内部维护了一个int类型的size和Object[]数组，默认大小是10，如果add之后大小不够的话会调用`ensureCapacityInternal `方法进行动态扩容，扩容后的大小是原大小的1.5倍，并调用Arrays.copyOf进行一次数组的复制。(查阅源码发现默认情况下ArrayList的最大长度是Integer.MAX_VALUE - 8，当超过这个数值时会扩展到Integer.MAX_VALUE，如果超过这个限制会抛OOM)

**删除元素**

不论按下标删除还是按元素删除，总的来说做了两件事

- 1.把指定元素后面位置的所有元素，利用System.arraycopy方法整体向前移动一个位置

- 2.最后一个位置的元素指定为null，这样让gc可以去回收它


**优点**

- 1.ArrayList底层以数组实现，是一种随机访问模式，再加上它实现了RandomAccess接口，因此查找也就是get的时候非常快

- 2.ArrayList在顺序添加一个元素的时候非常方便，只是往数组里面添加了一个元素而已

**缺点**

- 1.删除元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能

- 2.插入元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能


- [图解集合1：ArrayList](http://www.cnblogs.com/xrq730/p/4989451.html)
- [Java ArrayList工作原理及实现](https://yikun.github.io/2015/04/04/Java-ArrayList%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)


#### <a name="LinkedList">3.1.2 LinkedList</a>


|关  注  点| 结      论|
| --------  | :----: |
| LinkedList是否允许空  | 允许   |
| LinkedList是否允许重复数据  | 允许   |
| LinkedList是否有序  | 有序   |
| LinkedList是否线程安全  | 非线程安全   |


使用双向链表实现，set和get方法时间复杂度是O(n/2),同时实现了Deque接口，可以将LinkedList作为双端队列使用

```
private static class Entry<E> {
    E element;
    Entry<E> next;
    Entry<E> previous;
    ...
}
```

set和get的时间复杂度为什么是O(n/2)?

```
public E get(int index) {
    return entry(index).element;
}

private Entry<E> entry(int index) {
     if (index < 0 || index >= size)
         throw new IndexOutOfBoundsException("Index: "+index+
                                             ", Size: "+size);
     Entry<E> e = header;
     if (index < (size >> 1)) {
         for (int i = 0; i <= index; i++)
             e = e.next;
     } else {
         for (int i = size; i > index; i--)
             e = e.previous;
     }
     return e;
}
```

**与ArrayList的对比**

- 1、顺序插入速度ArrayList会比较快
- 2、因为LinkedList里面不仅维护了待插入的元素，还维护了Entry的前置Entry和后继Entry，如果一个LinkedList中的Entry非常多，那么LinkedList将比ArrayList更耗费一些内存
- 3、使用各自遍历效率最高的方式，ArrayList的遍历效率会比LinkedList的遍历效率高一些。因为使用普通for比for each快一些。
- 4、有些说法认为LinkedList做插入和删除更快，这种说法其实是不准确的：
   
   -（1）LinkedList做插入、删除的时候，慢在寻址，快在只需要改变前后Entry的引用地址
   -（2）ArrayList做插入、删除的时候，慢在数组元素的批量copy，快在寻址

- [图解集合2：LinkedList](http://www.cnblogs.com/xrq730/p/5005347.html)
- [Java LinkedList工作原理及实现](https://yikun.github.io/2015/04/05/Java-LinkedList%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [LinkedList的实现原理浅析](https://my.oschina.net/wangmengjun/blog/1538726)

#### <a name="Vector">3.1.3 Vector</a>

|关  注  点| 结      论|
| --------  | :----: |
| Vector是否允许空  | 允许   |
| Vector是否允许重复数据  | 允许   |
| Vector是否有序  | 有序   |
| Vector是否线程安全  | 线程安全   |

类似ArrayList，内部实现的原理形同，也是通过Object[]数组实现的。但是是**线程安全**的,但是为了同步，尽量少使用vector，因为vector的方法都是通过synchronized实现的，代价很大

- [Vector 是线程安全的吗？](http://yuanfentiank789.github.io/2016/11/25/vectorsafe/)

### <a name="Set">3.2 Set</a>

不包含重复元素的Collection，最多有一个null元素

#### <a name="HashSet">3.2.1 HashSet</a>


|关  注  点| 结      论|
| --------  | :----: |
| HashSet是否允许空  | 允许，但最多一个   |
| HashSet是否允许重复数据  | 不允许   |
| HashSet是否有序  | 不保证有序   |
| HashSet是否线程安全  | 非线程安全   |


HashSet是基于HashMap来实现的，操作很简单，更像是对HashMap做了一次“封装”，而且只使用了HashMap的key来实现各种特性。内部有多个不同的构造方法，对应初始化HashMap的操作也不同。

```
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;

    private transient HashMap<E,Object> map;
    
    public HashSet() {
        map = new HashMap<>();
    }
    
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
    
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }
    

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
}    
```

实际上HashSet存储的对象是HashMap的key，只不过HashMap的value是一个Object对象。

```
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
public boolean contains(Object o) {
    return map.containsKey(o);
}
public int size() {
    return map.size();
}
```


如果想获得线程安全的HashSet可以使用如下方法：

`Collections.synchronizedSet(new HashSet<String>());`


- [Java HashSet工作原理及实现](https://yikun.github.io/2015/04/08/Java-HashSet%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

#### <a name="TreeSet">3.2.2 TreeSet</a>

|关  注  点| 结      论|
| --------  | :----: |
| TreeSet是否允许空  | 不允许   |
| TreeSet是否允许重复数据  | 不允许   |
| TreeSet是否有序  | 不保证有序   |
| TreeSet是否线程安全  | 非线程安全   |


TreeSet是基于TreeMap实现的，也非常简单，同样的只是用key及其操作，然后把value置为dummy的object。

利用TreeMap的特性，实现了set的**有序性**(通过红黑树实现，这里的有序性指的是排序后的顺序，对于某些类型的元素，需要传递自定义的Comparable)。

TreeSet添加null时，如果再添加不是null的元素，就会报NullPointerException异常

- [Java TreeSet工作原理及实现](http://yikun.github.io/2015/04/10/Java-TreeSet%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

#### <a name="LinkedHashSet">3.2.3 LinkedHashSet</a>


|关  注  点| 结      论|
| --------  | :----: |
| LinkedHashSet是否允许空  | 允许，但最多一个   |
| LinkedHashSet是否允许重复数据  | 不允许   |
| LinkedHashSet是否有序  | 不保证有序   |
| LinkedHashSet是否线程安全  | 非线程安全   |

继承自HashSet，内部使用LinkedHashMap维持双向的链表。

```
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable

```

- [Java LinkedHashSet工作原理及实现](http://yikun.github.io/2015/04/09/Java-LinkedHashSet%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)


###  <a name="Queue与Deque">3.3 Queue与Deque</a>

#### <a name="ArrayDeque">3.3.1 ArrayDeque</a>

基于数组实现，实现了一个逻辑上的循环数组。

ArrayDeque的高效来源于head和tail这两个变量，它们使得物理上简单的从头到尾的数组变为了一个逻辑上循环的数组，避免了在头尾操作时的移动。我们来解释下循环数组的概念。


对于一般数组，比如arr，第一个元素为arr[0]，最后一个为arr[arr.length-1]。但对于ArrayDeque中的数组，它是一个逻辑上的循环数组，所谓循环是指元素到数组尾之后可以接着从数组头开始，数组的长度.第一个和最后一个元素都与head和tail这两个变量有关，具体来说：

- 如果head和tail相同，则数组为空，长度为0。
- 如果tail大于head，则第一个元素为elements[head]，最后一个为elements[tail-1]，长度为tail-head，元素索引从head到tail-1。
- 如果tail小于head，且为0，则第一个元素为elements[head]，最后一个为elements[elements.length-1]，元素索引从head到elements.length-1。
- 如果tail小于head，且大于0，则会形成循环，第一个元素为elements[head]，最后一个是elements[tail-1]，元素索引从head到elements.length-1，然后再从0到tail-1。


参考：

- [Java编程的逻辑 (48) - 剖析ArrayDeque](http://www.cnblogs.com/swiftma/p/6029547.html)
- [Java ArrayDeque源码剖析](https://www.cnblogs.com/CarpenterLee/p/5468803.html)

####  <a name="PriorityQueue">3.3.2 PriorityQueue</a>


- [Java编程的逻辑 (46) - 剖析PriorityQueue](http://www.cnblogs.com/swiftma/p/6014636.html)


## <a name=“tongbu”>4.同步集合</a>

### <a name="CopyOnWriteArrayList">4.1 CopyOnWriteArrayList </a>


###  <a name="CopyOnWriteArrayset">4.2 CopyOnWriteArrayset </a>

###  <a name="ArrayBlockkingQueue">4.3 ArrayBlockkingQueue </a>

###  <a name="LinkedBlockingQueue">4.4 LinkedBlockingQueue </a>





## <a name=“sanlie”>5.散列码产生规则</a>

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/%E6%95%A3%E5%88%97%E7%A0%81%E7%9A%84%E4%BA%A7%E7%94%9F%E8%A7%84%E5%88%99.png?raw=true)

