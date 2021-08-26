---
title: ConcurrentHashMap
---
### 线程安全

利用CAS自旋锁+Synchronized保证并发更新安全

底层使用数组+链表+红黑树实现

key和value不能为null

### 创建ConcurrentHashMap

- 和HashMap不同, 创建对象时指定初始容量会进行计算，计算方式是  n + n / 2 + 1
- 实际设置的初始容量会比传入的大，同时是2的幂的一个值.

### 关键参数sizeCtl

1. sizeCtl为0, 表示数组未初始化, 且初始容量为16
2. sizeCtl为正数, 如果数组未初始化，则记录的是数组初始容量，如果已经初始化，则记录数组扩容阈值（初始容量*0.75）
3. sizeCtl为-1， 表示数组正在初始化
4. sizeCtl小于0且不是-1，表示数组正在扩容,-(1-n)表示n个线程正在对数组扩容

### 结构

和HashMap一样， 用数组保存数据， Hash碰撞的保存成链表或者红黑树，转换条件是链表长度8同时数组长度大于等于64

### 注意

1. 初始化容量计算

    初始容量采用公式 n + n / 2 + 1, 例如创建时指定32, 则根据公式  32 + 32 / 2 + 1 = 49.  然后会取49之后的2的幂数是64

2. 多线程协助扩容

    在扩容过程中会根据CPU按段对数组进行迁移处理, 对处理过的元素进行标记, 当有另一个线程添加元素发现是标记元素时, 则协助扩容, 也领取一段元素进行扩容处理

    处理完成后再次循环领取任务处理, 处理是从数组末尾向前处理的, 最小的段大小是16个元素

3. 元素数量计数

    元素计数采用CAS赋值和赋值失败对数组元素赋值的方式进行, CAS对baseCount累加失败, 则循环对数组中的元素进行累加操作, 数组累加操作成功一次就跳出, 失败则重新计算索引再次尝试累加

    总元素个数是baseCount+数组元素累加的值

4. 大量的使用CAS自旋操作, 以避免使用锁

### 获取索引

获取索引和HashMap不同，不是直接tab[i]

使用Unsafe.getObjectVolatie()获取索引元素， 虽然table是用volatile修饰的，但是无法保证线程每次都能拿到table最新元素，用这个方法可以直接获取内存的数据，保证每次拿到数据都是最新的

### 初始化Table

在添加元素时， 如果Table为空则进行初始化

- 初始化过程中， 会判断sizeCtl, 如果小于0则表示已经有线程在初始化, 当前线程让出CPU
- sizeCtl在初始化前保存的时候初始化容量, 初始化完成后保存扩容阈值
- sc = n - (n >>> 2);计算扩容阈值, n>>>2就是n除以4, n - n / 4就是n乘以4分之一n. 即sc = 0.75*n. 用右移计算避免除法的性能损耗

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
				// 小于0表示其他线程正在初始化Table, 当前线程调用yield方法让出CPU时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
				// 以原子性操作对sizeCtl赋值成-1, 表示正在扩容
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
										// 初始化容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
										// 创建数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 计算扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
								// 初始化完成, sizeCtl保存扩容阈值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

### 添加元素

- 添加元素时判断Table是否为空, 如果为空则执行上述的初始化方法
- 初始化完成后, 添加操作会有3种情况发生
    1. 要添加的key对应的Table数组索引位置为空, 则以CAS形式将添加的元素放到对应的索引位置上, 此处可能出现多个线程同时进入数组为空的结点, 所以要CAS赋值, 一个线程赋值成功, 第二个线程赋值就会失败走下一次循环, 进入下边的逻辑

        ```java
        // 计算要添加的key对于数组的索引下标
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        		// 以原子性操作对这个索引赋值, 将要添加的结点放到索引位置上
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
        		    break;                   // no lock when adding to empty bin
        }
        ```

    2. Table数组正在扩容, 则触发协助扩容逻辑
    3. 其他情况, 出现了hash碰撞, 即通过key的hash获取到了元素, 则要判断这个key是否在这个元素对应的桶内
        - 对数组的这个元素进行加锁, 确保只有一个线程能操作这个元素对于的桶（链表或红黑树）
        - 再次获取元素判断
        - 通过hash值来判断这个元素结点是链表还是红黑树
            - 链表, 则循环链表, 如果hash和key都相同, 则表示是覆盖已有key的操作, 否则遍历到链表尾, 将新元素结点添加到链表尾部
            - 红黑树, 将新元素结点添加到树里

            ```java
            // 加锁, f是数组Table的元素, 对其加锁保证多个线程只能一个线程操作当前元素对应的桶
            synchronized (f) {
                if (tabAt(tab, i) == f) {
            				// 链表的hash值是正数或0
                    if (fh >= 0) {
                        binCount = 1;
            						// 循环链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
            								// 当key相同时, 表示key已经存在了, 进行值覆盖操作
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
            								// 把新的元素结点添加到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
            				// 元素结点是红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            ```

### 链表转红黑树

- 当添加元素完成后, 判断当前链表长度是否大于8, 如果是8则调用转换红黑树方法

    ```java
    if (binCount != 0) {
    		// 链表长度大于等于8则进入转换方法
        if (binCount >= TREEIFY_THRESHOLD)
            treeifyBin(tab, i);
        if (oldVal != null)
            return oldVal;
        break;
    }
    ```

- 进入转换方法, 再次判断Table元素结点是否小于64个, 如果小于则只进行扩容, 大于等于64个则转成红黑树

    ```java
    private final void treeifyBin(Node<K,V>[] tab, int index) {
    		Node<K,V> b; int n, sc;
    		if (tab != null) {
    				// 先判断数组Table的元素个数是否小于64个, 小于的话调用扩容方法
    		    if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
    		        tryPresize(n << 1);
    				// 大于等于64个元素进行转成红黑树操作
    		    else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
    		        synchronized (b) {
    		            if (tabAt(tab, index) == b) {
    		                TreeNode<K,V> hd = null, tl = null;
    		                for (Node<K,V> e = b; e != null; e = e.next) {
    		                    TreeNode<K,V> p =
    		                        new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
    		                    if ((p.prev = tl) == null)
    		                        hd = p;
    		                    else
    		                        tl.next = p;
    		                    tl = p;
    		                }
    		                setTabAt(tab, index, new TreeBin<K,V>(hd));
    		            }
    		        }
    		    }
    		}
    }
    ```

### 统计操作

- 添加完成后进行元素个数统计操作

    ```java
    addCount(1L, binCount);
    ```

- 进入统计方法

    当添加操作完成后需要对当前元素个数据进行累加操作, 而多线程操作时的累加操作会发生并发问题, 此处采用一个bashCount和一个数组的形式进行计算, 当bashCount累加操作失败时, 则将累加的值在数组中进行累加, 最终的元素个数是bashCount和数组各个元素的总和

    统计元素个数完成后, 计算当前元素的总个数, 再判断是否超出阈值进行扩容处理

    ```java
    // 添加计数
    private final void addCount(long x, int check) {
    		// CounterCell数组就是保存元素个数的数组
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 || (a = as[ThreadLocalRandom.getProbe() & m]) == null || !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                // 关键方法, 进行元素个数添加操作
    						fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
    				// 统计数组元素个数
            s = sumCount();
        }
    		// 当发生添加操作时进入此逻辑判断是否要对数组扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
    				// 此时sizeCtl保存的是扩容阈值, s为当前数组元素个数, 此处判断是否需要扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null && (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
    						// 小于0则表示当前有线程在扩容, 进行协助扩容操作
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 || sc == rs + MAX_RESIZERS || (nt = nextTable) == null || transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
    						// 大于等于0表示开始扩容, 同时将sizeCtl的值设置成负数
                else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
    								// 扩容方法
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
    ```

### 添加计数

```java
private final void fullAddCount(long x, boolean wasUncontended) {
	  int h;
		// 计算统计数组的hash值
	  if ((h = ThreadLocalRandom.getProbe()) == 0) {
	      ThreadLocalRandom.localInit();      // force initialization
	      h = ThreadLocalRandom.getProbe();
	      wasUncontended = true;
	  }
	  boolean collide = false;                // True if last slot nonempty
	  for (;;) {
	      CounterCell[] as; CounterCell a; int n; long v;
				// baseCount添加失败了, 同时数组不为空
	      if ((as = counterCells) != null && (n = as.length) > 0) {
						// hash对应数组元素为空, 进行创建元素
	          if ((a = as[(n - 1) & h]) == null) {
	              if (cellsBusy == 0) {            // Try to attach new Cell
										// 调用有参构造器进行创建对象
	                  CounterCell r = new CounterCell(x); // Optimistic create
	                  if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
	                      boolean created = false;
	                      try {               // Recheck under lock
	                          CounterCell[] rs; int m, j;
	                          if ((rs = counterCells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) {
	                              rs[j] = r;
	                              created = true;
	                          }
	                      } finally {
	                          cellsBusy = 0;
	                      }
												// 经过一系列判断, 创建成功跳出循环, 否则下次循环
	                      if (created)
	                          break;
	                      continue;           // Slot is now non-empty
	                  }
	              }
	              collide = false;
	          }
	          else if (!wasUncontended)       // CAS already known to fail
	              wasUncontended = true;      // Continue after rehash
	          else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
	              break;
						// 数组大小超出CPU核数
	          else if (counterCells != as || n >= NCPU)
	              collide = false;            // At max size or stale
	          else if (!collide)
	              collide = true;
	          else if (cellsBusy == 0 && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
	              try {
	                  if (counterCells == as) {// Expand table unless stale
	                      CounterCell[] rs = new CounterCell[n << 1];
	                      for (int i = 0; i < n; ++i)
	                          rs[i] = as[i];
	                      counterCells = rs;
	                  }
	              } finally {
	                  cellsBusy = 0;
	              }
	              collide = false;
	              continue;                   // Retry with expanded table
	          }
	          h = ThreadLocalRandom.advanceProbe(h);
	      }
				// 数组为空, 则进行创建数组操作
	      else if (cellsBusy == 0 && counterCells == as && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
	          boolean init = false;
	          try {                           // Initialize table
	              if (counterCells == as) {
	                  CounterCell[] rs = new CounterCell[2];
	                  rs[h & 1] = new CounterCell(x);
	                  counterCells = rs;
	                  init = true;
	              }
	          } finally {
	              cellsBusy = 0;
	          }
	          if (init)
	              break;
	      }
				// 优先尝试对baseCount进行累加操作, 操作成功就跳出循环, 否则进行下次循环进入对数组操作逻辑
	      else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
	          break;                          // Fall back on using base
	  }
}
```

### 扩容方法

- 扩容有两种情况, 当前线程扩容和协助扩容

    扩容的原理, 多线程可能同时进行扩容, 那么将整个数组根据CPU核数计算出平均的段, 每个线程处理一个段, 当处理完一个元素结点后, 在这个位置放置一个ForwardingNode元素结点占位表示正在处理, 即MOVED

    新建一个2倍原数组大小的nextTable数组,用来保存扩容后的新数组

    ```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
    		// 根据当前CPU核数计算要处理的段大小, 最小16个元素
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
    		// 新数组为null, 则创建2倍大的新数组
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt (this, TRANSFERINDEX, nextIndex, nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
    						// 扩容完成, 将新的table赋值给原table
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
    						// 判断是否扩容完成
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
    ```
