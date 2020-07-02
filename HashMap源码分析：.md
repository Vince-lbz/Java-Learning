**写在开头：这篇博客是我个人在学习Java的时候的记录，相当于个人的学习笔记，为了形成积累，对于相关内容也是参照网上博客、网络视频资源以及个人理解，望大家客观看待，如有不正确的地方也欢迎留言，帮助我改正以及给其他阅读者引导。**

 

 - HashMap的简要介绍：

 	HashMap是一个散列表，用于存储Key-Value键值对数据，需要注意它的特点是不是同步的，也就是线程不安全的；Key或Value可以为null值；存储的数据是无序的。
 	


 - HashMap在Java8前后做出了改动：

   - Java8之前，HashMap的数组结构是数组+链表，链表中添加数据采用的是头插法，这种方法会导致null值的丢失，容易造成死循环。扩容时链表中的元素先后顺序可能发生变化，且哈希冲突严重时链表中的元素变多，会导致查询速度下降，影响性能。
    - Java8之后，HashMap的数组结构是数组+链表/红黑树，并且链表中添加数据采用的是尾插法，解决了死循环及乱序的问题，当链表中元素个数大于8并且数组长度达到64时，将链表转化为红黑树，减少了查询时间（O(n)变为O( log(n) )）。

 - 源码分析（基于Java8的源码）：

**成员变量：**
	 

```java
int DEFAULT_INITIAL_CAPACITY = 16		//默认初始容量：16
float DEFAULT_LOAD_FACTOR = 0.75F	//默认负载因子: 0.75
int TREEIFY_THRESHOLD = 8			//链表转变为红黑树的阈值： 8
int UNTREEIFY_THRESHOLD = 6		//红黑树转变为链表的阈值： 6
int MIN_TREEIFY_CAPACITY = 64		//链表转变为红黑树的数组长度的阈值： 64
int size				//数组中元素个数
int modCount			//数组结构改变次数
int threshold			//扩容阈值
final float loadFactor	//负载因子
```

**构造方法：**

```java
//定义了四种重载的构造方法
//第一种是无参构造，默认的负载因子是0.75
public HashMap() {
    this.loadFactor = 0.75F;
}
//第二种是参数个数为1的构造方法，给定初始容量，内部调用参数个数为2的构造方法
public HashMap(int initialCapacity) {
    this(initialCapacity, 0.75F);
}
//第三种是参数个数为2的构造方法，给定初始容量和负载因子
//内部调用了tableSizeFor方法，使得threshold会自动转换为比给定容量大的最小的2^n的数值。
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    } else {
        if (initialCapacity > 1073741824) {
            initialCapacity = 1073741824;
        }

        if (loadFactor > 0.0F && !Float.isNaN(loadFactor)) {
            this.loadFactor = loadFactor;
            this.threshold = tableSizeFor(initialCapacity);
        } else {
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        }
    }
}
//------------------------------------------------------------------------------------//
//这里给出tableSizeFor方法
//计算cap-1的二进制的0的个数，再用-1无符号右移这些位数（-1的二进制全是1），最终返回这个结果+1作为构造方法中的threshold。
static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return n < 0 ? 1 : (n >= 1073741824 ? 1073741824 : n + 1);
}
//------------------------------------------------------------------------------------//
//第四种是参数是另一个Map，默认负载因子是0.75
//在这个构造方法中调用了putMapEntries方法
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = 0.75F;
    this.putMapEntries(m, false);
}
//------------------------------------------------------------------------------------//
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();//插入map的元素个数
    if (s > 0) {
        if (this.table == null) {//map只是刚创建，并未添加数据
            float ft = (float)s / this.loadFactor + 1.0F;//新添加的map的容量
            int t = ft < 1.07374182E9F ? (int)ft : 1073741824;
            if (t > this.threshold) {//容量大于threshold要将threshold转化为比需求容量大的最小的2^n数值
                this.threshold = tableSizeFor(t);
            }
        } else if (s > this.threshold) {//原hashmap中已经添加元素，也就是原本有一个容量
            this.resize();//扩容
        }

        Iterator var8 = m.entrySet().iterator();
        //将新map放入
        while(var8.hasNext()) {
            Entry<? extends K, ? extends V> e = (Entry)var8.next();
            K key = e.getKey();
            V value = e.getValue();
            this.putVal(hash(key), key, value, false, evict);
        }
    }
}
```

**哈希值计算：**

```java
//是将key的hashcode值与hashcode值无符号右移16位的结果做异或运算，目的是为了充分利用hashcode值的高位和低位，减少哈希碰撞的发生。
//这里需要注意异或操作的优先级很低，所以是先运算h>>>16，再异或。
static final int hash(Object key) {
    int h;
    return key == null ? 0 : (h = key.hashCode()) ^ h >>> 16;
}
```

**扩容：**
根据计算后的新容量及新扩容阈值创建空数组，遍历旧数组，旧数组元素不为null，则用临时变量保存，并将对应原数组数据设置为null，便于垃圾回收；如果原数组中该数据没有next数据，则直接存放到新数组中；如果该数据是红黑树结构，则将红黑树拆分，存入新的数组；如果是链表，创建一个高位链表，一个低位链表，用该元素的hashcode值与旧数组容量做与运算，结果位0，则插入新链表中的当前位置，如果结果不为0，插入新链表中的当前位置+旧数组长度的索引位置上，最后将高低位链表插入新数组中。

```java
final HashMap.Node<K, V>[] resize() {
    HashMap.Node<K, V>[] oldTab = this.table;
    int oldCap = oldTab == null ? 0 : oldTab.length;
    int oldThr = this.threshold;
    int newThr = 0;
    int newCap;
    if (oldCap > 0) {//hashmap中有元素
        if (oldCap >= 1073741824) {
            this.threshold = 2147483647;
            return oldTab;
        }

        if ((newCap = oldCap << 1) < 1073741824 && oldCap >= 16) {//新容量扩大1倍，同时旧容量>=16
            newThr = oldThr << 1;//扩容阈值扩大1倍
        }
    } else if (oldThr > 0) {//此情况是hashmap刚创建，给定了初始的容量，并没有创建数组，newcap==0
        newCap = oldThr;//新的容量=构造参数计算出的扩容阈值，也就是比给定容量大的最小的2^n
    } else {
        newCap = 16;
        newThr = 12;
    }

    if (newThr == 0) {//此情况是hashmap刚创建，并没有创建数组
        float ft = (float)newCap * this.loadFactor;//threshold在首次插入数据扩容后时将2^n传递给新容量，之后都将满足容量*负载因子
        newThr = newCap < 1073741824 && ft < 1.07374182E9F ? (int)ft : 2147483647;
    }

    this.threshold = newThr;
    HashMap.Node<K, V>[] newTab = new HashMap.Node[newCap];
    this.table = newTab;
    if (oldTab != null) {//如果旧的数组不为空，遍历数组
        for(int j = 0; j < oldCap; ++j) {
            HashMap.Node e;
            if ((e = oldTab[j]) != null) {//取出的数组元素不为空，保存在临时变量中
                oldTab[j] = null;
                if (e.next == null) {
                    newTab[e.hash & newCap - 1] = e;
                } else if (e instanceof HashMap.TreeNode) {
                    ((HashMap.TreeNode)e).split(this, newTab, j, oldCap);
                } else {
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null) {
                                loHead = e;
                            } else {
                                loTail.next = e;
                            }
                            loTail = e;
                        } else {
                            if (hiTail == null) {
                                hiHead = e;
                            } else {
                                hiTail.next = e;
                            }
                            hiTail = e;
                        }

                        e = next;
                    } while(next != null);
					//将高低位链表插入到新数组中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }

                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

**添加数据：**
对外的方法接口put实际上是调用了putVal方法。而putVal方法是根据路由寻址n - 1 & hash计算出在数组中的位置，如果当前位置为null，则创建一个新的Node节点，插入数组当前位置；如果数组当前位置不为null，但是hashcode值和key值都与当前位置的数据相同，则替换新数据即可；如果当前位置是红黑树，则在红黑树中做添加的判断，添加元素。上述都不符合，则在该索引处应为链表，遍历链表，如果链表的next有元素，先比较hashcode值再比较key值，都相同则替换，全部不同或没有next元素则在链表中新增一个Node节点，链表中元素超过8个则转变为红黑树。最后在存放对应的value值即可。

```java
public V put(K key, V value) {
    return this.putVal(hash(key), key, value, false, true);
}
//------------------------------------------------------------------------------------//
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    HashMap.Node[] tab;
    int n;
    if ((tab = this.table) == null || (n = tab.length) == 0) {
        n = (tab = this.resize()).length;
    }

    Object p;
    int i;
    if ((p = tab[i = n - 1 & hash]) == null) {
        tab[i] = this.newNode(hash, key, value, (HashMap.Node)null);
    } else {
        Object e;
        Object k;
        if (((HashMap.Node)p).hash == hash && ((k = ((HashMap.Node)p).key) == key || key != null && key.equals(k))) {
            e = p;
        } else if (p instanceof HashMap.TreeNode) {
            e = ((HashMap.TreeNode)p).putTreeVal(this, tab, hash, key, value);
        } else {
            int binCount = 0;

            while(true) {
                if ((e = ((HashMap.Node)p).next) == null) {
                    ((HashMap.Node)p).next = this.newNode(hash, key, value, (HashMap.Node)null);
                    if (binCount >= 7) {
                        this.treeifyBin(tab, hash);
                    }
                    break;
                }

                if (((HashMap.Node)e).hash == hash && ((k = ((HashMap.Node)e).key) == key || key != null && key.equals(k))) {
                    break;
                }

                p = e;
                ++binCount;
            }
        }

        if (e != null) {
            V oldValue = ((HashMap.Node)e).value;
            if (!onlyIfAbsent || oldValue == null) {
                ((HashMap.Node)e).value = value;
            }

            this.afterNodeAccess((HashMap.Node)e);
            return oldValue;
        }
    }

    ++this.modCount;
    if (++this.size > this.threshold) {
        this.resize();
    }

    this.afterNodeInsertion(evict);
    return null;
}
```

**获取数据：**
对外的方法接口get，内部实际上是调用getNode方法，getNode方法寻址计算得到的索引位置上的对应元素的hashcode值和键值是否都相同，相同则返回数组当前位置的元素，不同的话如果当前索引是红黑树结构，则遍历红黑树，如果是链表结构，遍历链表，返回key的hashcode值和key值相同的元素，没有返回null。

```java
public V get(Object key) {
    HashMap.Node e;
    return (e = this.getNode(hash(key), key)) == null ? null : e.value;
}
//------------------------------------------------------------------------------------//
final HashMap.Node<K, V> getNode(int hash, Object key) {
    HashMap.Node[] tab;
    HashMap.Node first;
    int n;
    if ((tab = this.table) != null && (n = tab.length) > 0 && (first = tab[n - 1 & hash]) != null) {
        Object k;
        if (first.hash == hash && ((k = first.key) == key || key != null && key.equals(k))) {
            return first;
        }

        HashMap.Node e;
        if ((e = first.next) != null) {
            if (first instanceof HashMap.TreeNode) {
                return ((HashMap.TreeNode)first).getTreeNode(hash, key);
            }

            do {
                if (e.hash == hash && ((k = e.key) == key || key != null && key.equals(k))) {
                    return e;
                }
            } while((e = e.next) != null);
        }
    }

    return null;
}
```

**包含数据：**
如果有相同的value值，返回true。外层for循环遍历Node数据上的元素，内层for循环遍历链表。

```java
public boolean containsValue(Object value) {
    HashMap.Node[] tab;
    if ((tab = this.table) != null && this.size > 0) {
        HashMap.Node[] var4 = tab;
        int var5 = tab.length;

        for(int var6 = 0; var6 < var5; ++var6) {
            for(HashMap.Node e = var4[var6]; e != null; e = e.next) {
                Object v;
                if ((v = e.value) == value || value != null && value.equals(v)) {
                    return true;
                }
            }
        }
    }

    return false;
}
```

**包含键：**
对外的方法接口containsKey，内部调用的是上面提到的getNode方法，hashcode值和key值相同，返回true。

```java
public boolean containsKey(Object key) {
    return this.getNode(hash(key), key) != null;
}
```

**移除数据：**
对外的方法接口remove，内部调用的是removeNode方法，removeNode是循环遍历map集合，找到hashcode值和key值与传入参数一致的元素e，如果该元素在数组上，则将数组该索引处改为e.next元素，如果在链表上，将该节点对象移除链表中。

```java
public boolean remove(Object key, Object value) {
    return this.removeNode(hash(key), key, value, true, true) != null;
}
//------------------------------------------------------------------------------------//
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

**替换数据：**
两种重载的方法，第一种传入两个参数，用getNode方法，查找到hashcode值和key值相同的Node对象，把value属性更改，并返回被替换的值。第二种传入三个参数，如果hashcode值、key值或value与原来value不同，都返回false，否则更改value属性。

```java
public V replace(K key, V value) {
    HashMap.Node e;
    if ((e = this.getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        this.afterNodeAccess(e);
        return oldValue;
    } else {
        return null;
    }
}

public boolean replace(K key, V oldValue, V newValue) {
    HashMap.Node e;
    Object v;
    if ((e = this.getNode(hash(key), key)) == null || (v = e.value) != oldValue && (v == null || !v.equals(oldValue))) {
        return false;
    } else {
        e.value = newValue;
        this.afterNodeAccess(e);
        return true;
    }
}
```

**至此，HashMap中的常用关键方法都提到了，但是注意的是其中涉及到红黑树的部分没有讲解，笔者自认为对其理解不透，待学习一段时间之后再补充上，但不影响对于数组+链表的操作的理解。**
//-------------------------------------------------------------------------------------------------------------------------------------//
**想着把HashMap的一些面试问题也集合到一起，便于自己之后的复习。**

1. 比较JAVA7和JAVA8中的hashMap？
   JAVA7采用的是数组加链表的结构，采用头插法，会导致null值的丢失，容易造成死循环，在扩容时桶内数据的先后顺序发生改变。当链表中元素较多时，性能较差。
   JAVA8采用的是数组加链表/红黑树的结构，采用尾插法，避免死循环，且加快查询性能。当链表中元素个数大于8且数组中元素个数大于64时，由链表转化为红黑树；当红黑树种元素个数小于6转化为链表。

2. 什么时候扩容？为什么要扩容？
   元素个数超过数组长度*负载因子时扩容；链表长度超过8但数组长度没超过64时扩容。
   为了解决哈希碰撞导致的链化影响查询速度的问题。

3. hashMap构造函数初始化容量？
   (需要存储的元素个数/负载因子)+1

4. HashMap有参的构造函数是为了给负载因子和扩容阈值赋值，其中扩容阈值的初始化计算为给定初始容量左侧有多少位零，就将-1无符号右移多少位，再加1。第一次扩容时（也就是第一次put），扩容阈值赋值给新的容量，再利用公式计算新的扩容阈值。（cap*loadfactor）之后的扩容就直接按照计算后的阈值进行。

5. 扩容时，如果链表头结点不为null且无next节点，则扩容后仍是该索引位置，若有next节点，说明发生哈希碰撞，将链表扩容时分组，拆分为高位与低位两个链表，利用hashcode值与oldcap与运算，得到高位是0，则扩容后仍在原位置，得到高位是1，则扩容后变为原下标+原数组长度的位置，重排好链表后，将头结点插入扩容的数组中，将Tail.next设置为null。

6. 为什么容量是2^n？
   hashMap的路由寻址计算是key的hashcode值与数组长度-1进行与运算，这样对hashcode值进行按位与运算能迅速得到数组下标，并使散列均匀。

7. 负载因子为什么是0.75？
   是性能与存储资源之间的折中。

8. 为什么链表转化为红黑树的阈值选择8？
   因为链表存放节点个数的概率是服从泊松分布，一个链表中有8个节点的概率已经非常小了。

9. 哈希表底层采用何种算法计算哈希值？还有哪些算法可以计算出哈希值？
   采用key的hashcode值结合数组长度进行按位与操作。还有取模法、伪随机数法。

10. 何时发生哈希碰撞？什么是哈希碰撞？如何解决哈希碰撞？
    key计算出的哈希值相同就会发生哈希碰撞，JDK8前使用链表解决，JDK8后使用链表加红黑树解决。

11. 传统HashMap的缺点，1.8为什么引入红黑树？
    哈希碰撞严重时，桶内的链表节点个数较多，查询性能慢，时间复杂度为O(n)，红黑树为了在节点数多时提升性能，红黑树的结构时间复杂度为O(log n)。

12. 为什么不直接用key的hashcode值与n-1进行位运算，而是将hashcode值与hashcode值右移16位做异或操作来计算hashcode值？
    实际上位运算使用的是低位的hashcode值，如果hashcode值高位变化很大，低位变化很小，很容易造成哈希碰撞，上述方法可以将高低位都利用起来，减少冲突。

13. HashMap的扩容为什么是2倍？
    为了使得扩容后的容量也是2的幂次。

14. hashmap的寻址计算：
    计算Entry在数组中的索引，将HashMap的key与数组长度做取模运算，但是取模运算比位运算开销高很多，所以利用与数组长度-1做位运算来达到和取模一样的目的。

15. hashmap线程不安全的体现：
    体现JDK7在扩容时的产生死循环或数据丢失的问题；JDK8中扩容时会出现数据覆盖的问题。同时还要注意在使用迭代器时会有fast-fail。
    fast-fail比较EntryIterator的exceptModCount与HashMap的modCount，如果不相同，则有线程修改过HashMap，会抛出异常，意在提醒用户及早注意线程安全问题。

