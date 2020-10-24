#目的
深入讲解HashMap源码，如题，将从`put(K key, V value)`方法调用开始
#put方法
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
当传入一个`key`和`value`时，第一步将调用`hash(Object key)`方法计算`key`的哈希值，返回int类型。内部调用`key`的`hashCode()`返回32位int类型的哈希值，同时将获取的哈希值无符号向右移动16位，再和原来的32位哈希值进行异或运算得到一个新的哈希值作为`hash(Object key)`的返回值返回
>这里会有一个问题，为什么要无符号右移16位之后再和原来的进行异或运算再返回？
这里涉及到获取哈希值后计算存放在数组中的哪一个下标上，稍后会在文章结尾进行说明

#putVal方法
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
`put(K key, V value)`内部调用`putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)`，首先判断`table（最终存放结果的数据结构）`是否为空，如果是则调用`resize()`进行初始化
>`resize()`既是`table`的初始化方法，同时也是`table`的扩容方法，将在后面一小节会进行讲解

初始化完成后将`table`赋值给`tab`并将`table`的长度赋值给`n`，接着将长度`n - 1`和前面计算出来的`key`的哈希值`hash`进行与运算计算出`key`应该存放在`table`的哪个下标上
>这里和我们想象的不太一样，按照我们的想法是直接将算好的哈希值`hash`和数组长度`n`进行取模运算就可以得出`key`将落在哪个下标上，但是JDK中并没有采用这种算法，而是采用了一种更加高效的`位运算`来完成，下面将详细讲解这种算法和限制

###· JDK位运算计算下标位置
当`table`初始化时长度为`16（详解下面resize小节）`转化成二进制`00000000000000000000000000010000`减1得`00000000000000000000000000001111`由于数组是从0开始进行计算此时二进制表示的范围正好是0 ~ 15，这时再将上面`key`计算的`hash`进行与运算（两者为1则为1，否则为0）即可得出一个0 ~ 15范围内的数字，这个数字就是`key`将要存放的下标位置。比如：
```
   hash:10000100011100010000011110000001
   n-1 :00000000000000000000000000001111
outcome:00000000000000000000000000000001
```
上面随机写的一个`hash`，由于`n-1`的出的二进制限制，`hash`相对于`n-1`中为0的位将全部被忽略，只有当`n-1`的位上是`1`时相对应的`hash`位才有计算的意义，例子中最终结果是`00000000000000000000000000000001`转十进制得`1`则该`key`将存放到`table`中下标为`1`的位置上
###· JDK位运算计算下标位置算法的限制
为了能使上述算法成立，`n`的值必须是`2的N次幂`，由于int类型为32位，所以N的取值范围为0 ~ 31，即2^0~31。在二进制中表现则是在所有的位中最多只有一位是`1`其他的都必须为`0`，因为假设`n`为`3`时（不是2的N次幂），二进制为`00000000000000000000000000000011`此时减1得到`00000000000000000000000000000010`再和`hash`进行与运算的时候只能得出0或者2的下标，永远不可能得出1的下标，显然是有问题的，如下
```
   hash:10000100011100010000011110000001
   n-1 :00000000000000000000000000000010
outcome:00000000000000000000000000000000

   hash:10000100011100010000011110000010
   n-1 :00000000000000000000000000000010
outcome:00000000000000000000000000000010
```
我们知道new HashMap(3)时是可以自定义初始化长度的，JDK为了保证`n`永远是`2的N次幂`，当我们传入了不是2的N次幂时，JDK会在构造函数中调用`tableSizeFor(int cap)`检查并重置传入的长度，重置完成的长度`大于等于`自定义初始化长度并保证是`2的N次幂`
```
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
解释下上面的算法，以长度为3作为例子，二进制为`00000000000000000000000000000011`此时减1得`00000000000000000000000000000010`，此时无符号向右移动一位得`00000000000000000000000000000001`再和原来减1后得到的n进行与运算得`00000000000000000000000000000011`再无符号向右移动两位得`00000000000000000000000000000000`
再和最后的结果n进行与运算得`00000000000000000000000000000011`，重复上述动作移动四位再进行与运算，移动八位再进行与运算，移动16位再进行与运算，最终得`00000000000000000000000000000011`，最后判断是否大于允许的最大长度，不大于则加1得`00000000000000000000000000000100`，转为十进制为4，计算过程如下
```
      n :00000000000000000000000000000011
  n - 1 :00000000000000000000000000000010
n >>> 1 :00000000000000000000000000000001
      n :00000000000000000000000000000011
n >>> 2 :00000000000000000000000000000000
      n :00000000000000000000000000000011
n >>> 4 :00000000000000000000000000000000
      n :00000000000000000000000000000011
n >>> 8 :00000000000000000000000000000000
      n :00000000000000000000000000000011
n >>> 16:00000000000000000000000000000000
      n :00000000000000000000000000000011
  n + 1 :00000000000000000000000000000100
```
>这里在计算前有一个减1的操作，到最后有一个加1的操作，是为了使最后得出来的长度能`大于等于`我们所自定义的长度，如果不这样操作的话不能保证计算后得出的长度一定`大于等于`我们所自定义的长度

回到`putVal`方法，当计算完得出`key`所要存放的下标时，判断该链表上是否为空，为空则直接new一个Node并放入数组相对应的下标中，不为空则将当前下标的链表赋值给`p`然后判断当前链表头的`hash`值和`key`是否相等，相等则执行替换，不相等则继续判断`p`是否是TreeNode节点（LinkedHashMap数据结构存放数据的节点），HashMap中为否，此时走到最后的else逻辑，循环遍历链表`p`判断是否有相同的`key`有则替换，无则新增。当else走完后，在方法最后会计算整个HashMap的元素总数`size`和阈值`threshold（后面resize()那节将会讲解如何计算出来的）`进行比较，如果大于`threshold`则会调用`resize()`进行扩容
>注意！！！在else逻辑里面循环遍历链表`p`中`binCount`记录着当前链表循环到了第几个，也表示当前存在多少个元素，当元素个数`binCount`大于等于`TREEIFY_THRESHOLD（常量8） - 1`时，即`binCount`大于等于`7`时，`binCount`从0开始计算，当为7时表示第8个元素，此时会将链表`p`转换成为`红黑树`

#resize方法

`resize()`方法除了对Hashmap进行扩容外还有对Hashmap进行初始化
```
final Node<K,V>[] resize() {
	Node<K,V>[] oldTab = table;
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	int oldThr = threshold;
	int newCap, newThr = 0;
	if (oldCap > 0) {
		if (oldCap >= MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return oldTab;
		}
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
				 oldCap >= DEFAULT_INITIAL_CAPACITY)
			newThr = oldThr << 1; // double threshold
	}
	else if (oldThr > 0) // initial capacity was placed in threshold
		newCap = oldThr;
	else {               // zero initial threshold signifies using defaults
		newCap = DEFAULT_INITIAL_CAPACITY;
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	if (newThr == 0) {
		float ft = (float)newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
				  (int)ft : Integer.MAX_VALUE);
	}
	threshold = newThr;
	@SuppressWarnings({"rawtypes","unchecked"})
	Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab;
	if (oldTab != null) {
		for (int j = 0; j < oldCap; ++j) {
			Node<K,V> e;
			if ((e = oldTab[j]) != null) {
				oldTab[j] = null;
				if (e.next == null)
					newTab[e.hash & (newCap - 1)] = e;
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				else { // preserve order
					Node<K,V> loHead = null, loTail = null;
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;
					do {
						next = e.next;
						if ((e.hash & oldCap) == 0) {
							if (loTail == null)
								loHead = e;
							else
								loTail.next = e;
							loTail = e;
						}
						else {
							if (hiTail == null)
								hiHead = e;
							else
								hiTail.next = e;
							hiTail = e;
						}
					} while ((e = next) != null);
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
当`table`为空时且new HashMap()时没有指定初始化长度，这时代码22 ~ 24行获取初始化的默认值，默认长度`newCap = DEFAULT_INITIAL_CAPACITY = 1 << 4 = 16`，阈值为`newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY) = 12`，`DEFAULT_LOAD_FACTOR`的值是`0.75f`也就是会把初始化长度的三分之一作为阈值，然后创建`table`并返回

当添加的元素总个数超过阈值时则会调用`resize()`进行扩容，这时代码11 ~ 13行对原来的长度和阈值同时向左移动一位（相当于乘以2），相对第一次初始化的长度`16`和阈值`12`扩容后就长度为`32`阈值为`24`，并用新的长度创建新的`table`后，将原来的旧数据迁移到新的`table`上

这里同样JDK并没有使用我们想到的方法，即轮询旧的`table`并让每个`key`的`hash`和新的长度`n`进行重新计算，得出在新`table`中的下标并设值，只有当旧`table`的链表个数`只有1个`时才是使用我们所想的方法，即代码35 ~ 36行。如果链表的长度`超过1个`时则走else代码，接下来详细对else中的代码进行讲解

>首先要明白一点！！！同一个链表上的元素在旧的长度上面相对应的`有效hash位`的值是相同的，假设我们的初始化长度为`16`，扩容后的长度为`32`，其中一条链表上面有两个元素，`hash`分别为`00000100001010100100000000010011`、`00000100101010110000010000000011`，两个元素在同一条链表上，旧长度为`16`化为二进制并减1得`00000000000000000000000000001111`那么只有后四位会进行计算，也就是只有后四位为`有效hash位`，所以只有新元素的`有效hash位`也就是后四位为`0011`时才会获得同一个下标和上面两个元素在同一个链表中。

明白后我们接着看，接下来的计算会有点神奇！！！当长度从16扩展到32时，32的二进制为`00000000000000000000000000100000`减`1`得`00000000000000000000000000011111`这时会发现长度为`16减1`后的二进制的后四位和`32减1`的后四位相同，那么也就是说决定旧元素在新`table`中，需要增加多少才能算出在新`table`中的准确下标位置，这个由`最左边的1`决定
比如上面的两个`hash`，其中一个是`00000100001010100100000000010011`那么后四位决定了它在旧`table`中的下标为`3`，那么新`table`中需要增加多少才能算出准确下标位置由`hash`从右往左数`第5位`决定，它上面是`1`，该位在二进制中表示`16`，所以该`hash`在新`table`中的准确下标是原本的位置`3`加上需要移动的长度`16`等于`19`。第二个`hash`的二进制是`00000100101010110000010000000011`，从右往左数`第5位`是`0`，所以移动的长度为`0`，故在新的`table`中下标保持不变，还是原来的位置，那如何才能知道从右往左数`第5位`是`0`还是`1`，只需要将旧`table`的长度转成二进制再和`hash`进行计算即可
>！！！也就是说在同一个链表中要么就是保持原来的位置不变，要么就是增加移动旧`table`的长度作为新`table`中的位置

此时再来看代码就简单多了，现在的逻辑就变成了同一条链表上的元素，要么保持原来的位置，要么移动相对应的长度。JDK中创建了两个链表来保存保持不变的元素`loHead`和需要移动相对应的长度的链表`hiHead`，接着用旧`table`长度的二进制便能计算出是否需要移动，再添加到相对应的链表中，最后再存放到新`table`中
>`resize()`要注意，当结构为红黑树时，元素个数`小于等于6`时会被转化回链表，由`UNTREEIFY_THRESHOLD = 6`常量定义，
除了`resize()`还有`remove(Object key)`也有可能将红黑树转化回链表
#remove方法
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

final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
						  boolean movable) {
	int n;
	if (tab == null || (n = tab.length) == 0)
		return;
	int index = (n - 1) & hash;
	TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
	TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
	if (pred == null)
		tab[index] = first = succ;
	else
		pred.next = succ;
	if (succ != null)
		succ.prev = pred;
	if (first == null)
		return;
	if (root.parent != null)
		root = root.root();
	if (root == null
		|| (movable
			&& (root.right == null
				|| (rl = root.left) == null
				|| rl.left == null))) {
		tab[index] = first.untreeify(map);  // too small
		return;
	}
	TreeNode<K,V> p = this, pl = left, pr = right, replacement;
	if (pl != null && pr != null) {
		TreeNode<K,V> s = pr, sl;
		while ((sl = s.left) != null) // find successor
			s = sl;
		boolean c = s.red; s.red = p.red; p.red = c; // swap colors
		TreeNode<K,V> sr = s.right;
		TreeNode<K,V> pp = p.parent;
		if (s == pr) { // p was s's direct parent
			p.parent = s;
			s.right = p;
		}
		else {
			TreeNode<K,V> sp = s.parent;
			if ((p.parent = sp) != null) {
				if (s == sp.left)
					sp.left = p;
				else
					sp.right = p;
			}
			if ((s.right = pr) != null)
				pr.parent = s;
		}
		p.left = null;
		if ((p.right = sr) != null)
			sr.parent = p;
		if ((s.left = pl) != null)
			pl.parent = s;
		if ((s.parent = pp) == null)
			root = s;
		else if (p == pp.left)
			pp.left = s;
		else
			pp.right = s;
		if (sr != null)
			replacement = sr;
		else
			replacement = p;
	}
	else if (pl != null)
		replacement = pl;
	else if (pr != null)
		replacement = pr;
	else
		replacement = p;
	if (replacement != p) {
		TreeNode<K,V> pp = replacement.parent = p.parent;
		if (pp == null)
			root = replacement;
		else if (p == pp.left)
			pp.left = replacement;
		else
			pp.right = replacement;
		p.left = p.right = p.parent = null;
	}

	TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

	if (replacement == p) {  // detach
		TreeNode<K,V> pp = p.parent;
		p.parent = null;
		if (pp != null) {
			if (p == pp.left)
				pp.left = null;
			else if (p == pp.right)
				pp.right = null;
		}
	}
	if (movable)
		moveRootToFront(tab, r);
}
```
`remove(Object key)`的话没有什么好说的，通过获取要删除的`key`的`hash`计算出在`table`中的那个下标，轮询该下标上的链表比较并进行移除并返回被删除的`key`的值
>需要注意的是，在`remove(Object key)`中当数据结构不是链表而是红黑树的时候，当红黑树的元素个数`小于等于6`时，也会把红黑树转化回链表结构，但在`remove(Object key)`中这个`小于等于6`的说法也不是一定是绝对的，因为判断是否将红黑树转化回链表的依据是红黑树的`root节点的左子树是否为空`或者`root节点的右子树是否为空`或者`root节点的左子树的左子树是否为空`，这里`不同于resize()`有可能出现`小于6`但依然是红黑树，只能说`最理想最快`会被转化回链表的情况是`root节点的左子树的左子树为空其他均不为空`，这时删除元素后的红黑树的个数`等于6`，如下图，代码为67 ~ 71行![删除元素后最理想最快被转回链表的红黑树结构](https://upload-images.jianshu.io/upload_images/18758351-98dc1c1bc1ab8921.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#回顾
回到文章之前，我们还有一个问题没有回答，为什么要无符号右移16位之后再和原来的哈希值进行异或运算再返回计算后的值作为`hash`？
这里首先需要说明hashCode是一个32位int类型，分为高16位（从左往右数16位）和低16位（从右往左数16位），无符号右移16位意味着删除低16位后将高16位向右移动，高16位变成了低16位，而原来的高16位用0填充，这是考虑到如果低16位一直重复，而高16位不重复，那么在数据量少的情况下都是低16位在起到决定下标的作用，所以为了防止这种情况的发生，JDK将高16位的影响带到低16位中进行一个异或运算，使得得出来的`hash`尽可能的离散

#总结
至此，HashMap的源码解析结算，红黑树部分并没有讲解，因为也只是数据结构上面的存储方式不同而已