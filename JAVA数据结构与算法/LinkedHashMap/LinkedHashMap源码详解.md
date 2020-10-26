# 目的
JDK中已经有了HashMap为什么还需要有个LinkedHashMap呢？答案相信大家也知道，HashMap是无序的，而LinkedHashMap是有序的。那么当HashMap变成LinkedHashMap后它是如何去存储的？查询的性能是否是发生了变化，如果没有，它是怎么做到保证了查询性能的同时又保证了顺序的？这篇文章将深入LinkedHashMap源码对以上问题进行剖析，在了解LinkedHashMap源码之前请先看<<HashMap源码详解>>，我们将在它的基础上进行讲解，因为90%都是由HashMap实现的，对于HashMap的源码实现部分将直接粗略带过不再进行细讲
# HashMap与LinkedHashMap数据结构对比
首先，LinkedHashMap继承了HashMap，而且LinkedHashMap中定义了三个新字段分别是`head`、`tail`和`accessOrder`，我们将在后面讲解它们的作用

![LinkedHashMap_UML.png](https://upload-images.jianshu.io/upload_images/18758351-dd9a454e0dad2e53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时，LinkedHashMap的`table`使用的也是HashMap的`table`，但是LinkedHashMap的`table`实现类并没有使用`HashMap.Node`，而是使用继承了`HashMap.Node`的`Entry`类，`Entry`类新增了两个字段`before`和`after`，将在后面讲解它们的作用![HashMap.Node_UML.png](https://upload-images.jianshu.io/upload_images/18758351-73fe7276d593cae3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) ![LinkedHashMap.Node_UML.png](https://upload-images.jianshu.io/upload_images/18758351-0b8cc3a4fe662f2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# put方法
LinkedHashMap使用的`put(K key, V value)`继承至HashMap，所以不再进行讲解
# putVal方法
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
LinkedHashMap使用的`putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)`同样继承至HashMap，所以同样不再进行讲解，这段代码里面有三个方法是需要讲解的分别是第7行的`newNode(hash, key, value, null)`、第33行的`afterNodeAccess(e)`和第40行的`afterNodeInsertion(evict)`，这三个方法LinkedHashMap都进行了重写
# newNode方法
```
// HashMap.class中的实现
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}

// LinkedHashMap.class中的实现
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```
HashMap中的`newNode(int hash, K key, V value, Node<K,V> next)`新创建的数据存储类是`HashMap.Node`，而LinkedHashMap重写了该方法，创建的数据存储类是`LinkedHashMap.Entry`。同时在LinkedHashMap中还调用了`linkNodeLast(LinkedHashMap.Entry<K,V> p)`将LinkedHashMap中的`head`和`tail`指向新建的数据存储类，如果又新建一个`LinkedHashMap.Entry`则将新`LinkedHashMap.Entry`添加至链表尾部
>`LinkedHashMap.Entry`本身是链表结构，`head`和`tail`分别指向链表的头部和尾部，同时它们也是LinkedHashMap为什么能保证与插入顺序一致的原因所在

# afterNodeAccess方法
```
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
LinkedHashMap支持两种排序，一种是默认的`按插入顺序排序`，一种是`按最近最少使用排序（LRU）`。`按插入顺序排序`好理解，这里解释下`按最近最少使用排序（LRU）`，意思是将最近最少使用过的或最少访问过的排在最前面，方便以后进行淘汰和删除，经常访问或使用的放在最后面。那如何才能做到呢，其实很简单就是元素每被使用或访问过后就把它放到最后面，当经过一段时间后经常被使用或访问的就都在后面了，越前面的就表示越没被使用或访问过的。
`afterNodeAccess(Node<K,V> e)`就是为了`按最近最少使用排序（LRU）`而存在，当`accessOrder`为`true`时，默认为`false`，就会启用`按最近最少使用排序（LRU）`算法，会将最近被使用过或访问过的元素放到链表的最后面，从而改变了原来链表的顺序也就改变了默认的`按插入顺序排序`
>`putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)`添加新元素也算是被使用访问，所以当`accessOrder`为`true`，这时使用的是`按最近最少使用排序（LRU）`，那么就会将元素移动到最后
# afterNodeInsertion方法
```
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```
`afterNodeInsertion(boolean evict)`在HashMap中是空实现，也是留给LinkedHashMap实现的一个钩子，该方法的作用是是否删除最老的元素，在LinkedHashMap中不管是`按插入顺序排序`还是`按最近最少使用排序（LRU）`在链表最前面的就是最老的或最少使用的，所以会将头部第一个元素进行删除。
当是从`put(K key, V value)`进来时`evict`是`true`但是LinkedHashMap中的`removeEldestEntry(Map.Entry<K,V> eldest)`永远返回`false`所以在LinkedHashMap中该方法不会触发
# remove方法
```
public V remove(Object key) {
	Node<K,V> e;
	return (e = removeNode(hash(key), key, null, false, true)) == null ?
		null : e.value;
}

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
LinkedHashMap使用的`remove(Object key)`同样继承至HashMap，所以同样不再进行讲解，这段代码里面唯一也有一个方法是需要讲解的就是`afterNodeRemoval(node)`，该方法在LinkedHashMap中进行了重写
# afterNodeRemoval方法
```
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```
当从`table`中删除了元素后，LinkedHashMap同时调用`afterNodeRemoval(Node<K,V> e)`删除维持顺序的链表中的元素
# 总结
当LinkedHashMap进行查询的时候调用的同样是HashMap的查询方法，同样是根据需要查询的`key`进行哈希取值从`table`中进行查找，当我们需要按顺序输出时，则通过LinkedHashMap中维护的链表头`head`到链表尾`tail`便能进行顺序输出
# 简书地址
https://www.jianshu.com/p/27377b59d6bb
