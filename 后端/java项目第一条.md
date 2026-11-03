## 前言

Caffeine 是一个高性能的、线程安全的缓存组件，本文将分成三个部分，第一部分，我将讲解Caffeine的总体实现原理，了解大致的模块划分，第二部分我会深入到细节，系统学习 统计元素访问频率的 Count-Min Sketch 数据结构、MPSC 多线程设计模式，第三部分将会带大家深入到源码学习。

---

## **一、Caffeine的实现原理**

![](https://i-blog.csdnimg.cn/direct/4823249563844cb3a410b6a7b1753c06.png)

caffeine使用了ConcurrentHashMap作为数据的存储结构，但为了提高缓存的命中率，做了些优化。

把存储数据的hashmap分成三个部分：窗口区、试用区、保护区，每个区域的是一个lru的双端队列，且大小随着该区域的命中率变化会自动调节。类 似jvm的新生代老年代设计，窗口区的生命周期最短，保护区的最长。

淘汰算法使用了tinylru，访问频率使用 Count-Min Sketch 数据结构记录。此外还涉及数据在各分区之间的晋级、降级、淘汰

读写操作类似mysql的wal机制，分别会向readbuffer和writebuffer添加读写任务，这两个buffer军采用mpcs多生产者单消费者的多线程设计模式。写缓存时，实际先把数据写到缓冲区，等合适的时机再把缓冲区的任务刷新到内存。

---

## 二、实现细节

### 1.FrequencySketch

FrequencySketch用于近似的统计各个元素的使用频次，它是一个long[]类型数组，每个元素长度64位，每4位为一组作为一个计数器。即数组每个槽位有16个计数器，每个计数器最多统计到15。 原理类似布隆过滤器，统计频率时根据hash计算槽位，更新多个槽位的数据+=1，而在计算频率时取多个槽位中的最小值

```cobol
final class FrequencySketch<E> {     static final long[] SEED = {     // A mixture of seeds from FNV-1a, CityHash, and Murmur3           0xc3a5c85c97cb3127L,0xb492b66fbe98f273L,0x9ae16a3b2f90404fL,0xcbf29ce484222325L};    static final long RESET_MASK = 0x7777777777777777L;    static final long ONE_MASK = 0x1111111111111111L;     // 采样大小    int sampleSize;    // tableMask = table.length - 1    int tableMask;    // 存储元素的频率    long[] table;    int size;     @NonNegative    public int frequency(@NonNull E e) {      // 如果缓存没有被初始化则返回频率为 0      if (isNotInitialized()) {        return 0;      }       // hash扰动，目的让hash均匀分布      int hash = spread(e.hashCode());      // 一个槽位有64个计数器，同一个e的会有四个计数器，且在槽位中的值是连续的，此处计算e的计数器的起始位置（是第几个计数器），start+0是第一个计数器，start+1是第二个...显然start只有四个位置，0,4,8,12      int start = (hash & 3) << 2;      // 算了四个hash，取对应四个槽位中的最小值作为元素e出现的频率      int frequency = Integer.MAX_VALUE;      for (int i = 0; i < 4; i++) {        // 计算e所在的table中的索引        int index = indexOf(hash, i);        // 这里计算出要提取的 4-bit 计数器的位置：        // 4个计数器的位置是相连的，start 表示起始位置， i 表示第几个计数器（逻辑上的索引）。        // << 2 表示将索引乘以 4（因为每个字段是 4 位的，占用 4 位）。        // 结果是该计数器的起始位在 64 位长整数中的偏移量。        int count = (int) ((table[index] >>> ((start + i) << 2)) & 0xfL);        frequency = Math.min(frequency, count);      }      return frequency;    }    /**   * Returns the table index for the counter at the specified depth.   *   * @param item the element's hash   * @param i the counter depth   * @return the table index   */    int indexOf(int item, int i) {      long hash = (item + SEED[i]) * SEED[i];      hash += (hash >>> 32);      return ((int) hash) & tableMask;    }    /**   * Applies a supplemental hash function to a given hashCode, which defends against poor quality   * hash functions.   */    int spread(int x) {      x = ((x >>> 16) ^ x) * 0x45d9f3b;      x = ((x >>> 16) ^ x) * 0x45d9f3b;      return (x >>> 16) ^ x;    } }
```

### 2.添加元素put 

`put` 添加元素时会先直接插入到 `ConcurrentHashMap` 中，然后执行afterWrite方法在 `WriteBuffer` 中添加 `AddTask/UpdateTask` 任务，添加成功后会往线程池中提交 `PerformCleanupTask` 任务，`PerformCleanupTask` 任务的主要作用是执行 `maintenance` 维护方法，该方法执行前需要先获取同步锁，单线程消费 `WriteBuffer` 中的任务。执行 `AddTask` 任务时会将元素先添加到窗口区，如果是 `UpdateTask`，它会修改三个不同区域的双端队列，这些队列采用LRU算法，最新被访问的元素会被放在尾节点处，并且试用区的元素被访问后会被晋升到保护区尾节点，元素对应的访问频率也会在频率草图中更新，如果被添加的节点权重超过缓存最大权重会直接被驱逐。会，首先将AddTask插入到双端队列writeBuffer

**1、afterWrite方法在 `WriteBuffer` 中添加 `AddTask/UpdateTask` 任务**

```scss
void afterWrite(Runnable task) {    // 重试最多100次    for (int i = 0; i < WRITE_BUFFER_RETRIES; i++) {      // mpsc队列插入任务，并在合适的时间调度执行      if (writeBuffer.offer(task)) {        // 写后调度        scheduleAfterWrite();        return;      }      scheduleDrainBuffers();    }     // The maintenance task may be scheduled but not running. This might occur due to all of the    // executor's threads being busy (perhaps writing into this cache), the write rate greatly    // exceeds the consuming rate, priority inversion, or if the executor silently discarded the    // maintenance task. In these scenarios then the writing threads cannot make progress and    // instead writers provide assistance by performing this work directly.    try {      performCleanUp(task);    } catch (RuntimeException e) {      logger.log(Level.SEVERE, "Exception thrown when performing the maintenance task", e);    }  }
```

### 2.1 writeBuffer

类型是MpscGrowableArrayQueue，从名字可以看出，writeBuffer是一个容量可变的、双端的队列，mpsc是multi producer，single consumer。继承的父类如下：

![](https://i-blog.csdnimg.cn/direct/a80754b81ec94071b333ef969c877668.png)

其中父类BaseMpscLinkedArrayQueuePad1内，定义了120字节，它的作用是缓存防伪共享

```cobol
abstract class BaseMpscLinkedArrayQueuePad1<E> extends AbstractQueue<E> {  byte p000, p001, p002, p003, p004, p005, p006, p007;  byte p008, p009, p010, p011, p012, p013, p014, p015;  byte p016, p017, p018, p019, p020, p021, p022, p023;  byte p024, p025, p026, p027, p028, p029, p030, p031;  byte p032, p033, p034, p035, p036, p037, p038, p039;  byte p040, p041, p042, p043, p044, p045, p046, p047;  byte p048, p049, p050, p051, p052, p053, p054, p055;  byte p056, p057, p058, p059, p060, p061, p062, p063;  byte p064, p065, p066, p067, p068, p069, p070, p071;  byte p072, p073, p074, p075, p076, p077, p078, p079;  byte p080, p081, p082, p083, p084, p085, p086, p087;  byte p088, p089, p090, p091, p092, p093, p094, p095;  byte p096, p097, p098, p099, p100, p101, p102, p103;  byte p104, p105, p106, p107, p108, p109, p110, p111;  byte p112, p113, p114, p115, p116, p117, p118, p119;}
```

> 复习下操作系统的知识
> 
> CPU 访问内存时，首先查询 cache 是否已缓存该数据。如果有，则返回数据，无需访问内存；如果不存在，则需把数据从内存中载入 cache，最后返回给理器。Cache 之所以有效，是因为程序对内存的访问存在一种概率上的局部特征：
> 
> - 对于刚被访问的数据，其相邻的数据在将来被访问的概率高。
> - 对于刚被访问的数据，其本身在将来被访问的概率高。
> 
> Cache是由很多个 Cache line 组成的。Cache line 是 cache 和 RAM 交换数据的最小单位，通常为 64 Byte。当 CPU 把内存的数据载入 cache 时，会把临近的共 64 Byte 的数据一同放入同一个Cache line，因为空间局部性：临近的数据在将来被访问的可能性大。
> 
> ![](https://i-blog.csdnimg.cn/direct/361297cd6c21459ba899a59f27299c54.png)
> 
> 缓存伪共享：处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2,L3）后再进行操作；如果对声明了volatile 变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在的缓存行的数据写回到系统内存。但就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读取到处理器缓存里。

因此通过增大对象间的间隔，保证两个对象在两个cache line中，可以避免伪共享。

**2、task插入writebuffer**

```cobol
public boolean offer(final E e) {    if (null == e) {      throw new NullPointerException();    }     long mask;    E[] buffer;    long pIndex;     while (true) {      long producerLimit = lvProducerLimit();      // volatile获取pIndex      pIndex = lvProducerIndex();      // 最低位表示是否在扩容，如果是，一直自旋      if ((pIndex & 1) == 1) {        continue;      }      // pIndex is even (lower bit is 0) -> actual index is (pIndex >> 1)      // mask、buffer会动态改变，取最新值      mask = this.producerMask;      buffer = this.producerBuffer;      // a successful CAS ties the ordering, lv(pIndex)-[mask/buffer]->cas(pIndex)       // assumption behind this optimization is that queue is almost always empty or near empty      if (producerLimit <= pIndex) {        int result = offerSlowPath(mask, pIndex, producerLimit);        switch (result) {          case 0:            break;          case 1:            continue;          case 2:            return false;          case 3:            resize(mask, buffer, pIndex, e);            return true;        }      }      // cas获取乐观锁成功      if (casProducerIndex(pIndex, pIndex + 2)) {        break;      }    }    // INDEX visible before ELEMENT, consistent with consumer expectation    final long offset = modifiedCalcElementOffset(pIndex, mask);    soElement(buffer, offset, e);    return true;  }
```

该双端队列通过保证pIndex的内存可见性，cas pIndex字段来获取锁

**3、如果task插入writebuffer成功，执行写后调度方法，加锁后往线程池插入PerformCleanupTask任务**

![](https://i-blog.csdnimg.cn/direct/40ab598e6d214aff9cf5e0a9f5ce20fc.png)

![](https://i-blog.csdnimg.cn/direct/de9307510af141b1af37b8880ae0edad.png)

4、任务的执行逻辑在maintenance中，通过加锁保证单消费者，依次消费读缓冲，写缓冲...

```less
  void performCleanUp(@Nullable Runnable task) {    evictionLock.lock();    try {      maintenance(task);    } finally {      evictionLock.unlock();    }    if ((drainStatus() == REQUIRED) && (executor == ForkJoinPool.commonPool())) {      scheduleDrainBuffers();    }  }
```

maintenance是一个重要的入口方法 

```cobol
void maintenance(@Nullable Runnable task) {        // 更新状态为执行中        setDrainStatusRelease(PROCESSING_TO_IDLE);         try {            // 处理读缓冲区中的任务            drainReadBuffer();             // 处理写缓冲区中的任务            drainWriteBuffer();            if (task != null) {                task.run();            }             // 处理 key 和 value 的引用            drainKeyReferences();            drainValueReferences();             // 过期和驱逐策略            expireEntries();            // 保护区和考察区的淘汰策略            evictEntries();             climb();        } finally {            if ((drainStatusOpaque() != PROCESSING_TO_IDLE) || !casDrainStatus(PROCESSING_TO_IDLE, IDLE)) {                setDrainStatusOpaque(REQUIRED);            }        }    }
```

其中evictEntries是主要的淘汰逻辑

```csharp
void evictEntries() {    if (!evicts()) {      return;    }    // 淘汰窗口区    int candidates = evictFromWindow();    // 淘汰考察区    evictFromMain(candidates);  }
```

窗口区淘汰逻辑，淘汰后写入到了考察区

```cobol
    @Nullable Node<K, V> evictFromWindow() {        Node<K, V> first = null;         //node = window queue的头部节点        Node<K, V> node = accessOrderWindowDeque().peekFirst();         //一直循环： 如果window区超过了最大的限制，那么就要把“多出来”的记录做处理        while (windowWeightedSize() > windowMaximum()) {            // The pending operations will adjust the size to reflect the correct weight            if (node == null) {                break;            }            //下一个节点            Node<K, V> next = node.getNextInAccessOrder();            if (node.getPolicyWeight() != 0) {                //设置 node 的类型： 为观察类型 probation                node.makeMainProbation();                 // 从window区去掉                //      node  = accessOrderWindowDeque().peekFirst()                accessOrderWindowDeque().remove(node);                  //加入到probation queue，相当于把节点移动到probation区（开始准备晋升）                accessOrderProbationDeque().offerLast(node);                 // 记录一下第一个 准备晋升的元素                if (first == null) {                    first = node;                }                //因为window移除了一个节点，所以需要调整 size                setWindowWeightedSize(windowWeightedSize() - node.getPolicyWeight());            }            node = next;        }        //第一个 准备晋升的元素        return first;    }
```

考察区的淘汰逻辑

```cobol
void evictFromMain(int candidates) {    int victimQueue = PROBATION;    // 第一个node是考察区马上淘汰的    Node<K, V> victim = accessOrderProbationDeque().peekFirst();    // 最后一个node是上个步骤中从window区晋升的    Node<K, V> candidate = accessOrderProbationDeque().peekLast();    while (weightedSize() > maximum()) {      // Search the admission window for additional candidates      if (candidates == 0) {        candidate = accessOrderWindowDeque().peekLast();      }       // Try evicting from the protected and window queues      if ((candidate == null) && (victim == null)) {        if (victimQueue == PROBATION) {          victim = accessOrderProtectedDeque().peekFirst();          victimQueue = PROTECTED;          continue;        } else if (victimQueue == PROTECTED) {          victim = accessOrderWindowDeque().peekFirst();          victimQueue = WINDOW;          continue;        }         // The pending operations will adjust the size to reflect the correct weight        break;      }       // Skip over entries with zero weight      if ((victim != null) && (victim.getPolicyWeight() == 0)) {        victim = victim.getNextInAccessOrder();        continue;      } else if ((candidate != null) && (candidate.getPolicyWeight() == 0)) {        candidate = (candidates > 0)            ? candidate.getPreviousInAccessOrder()            : candidate.getNextInAccessOrder();        candidates--;        continue;      }       // Evict immediately if only one of the entries is present      if (victim == null) {        @SuppressWarnings("NullAway")        Node<K, V> previous = candidate.getPreviousInAccessOrder();        Node<K, V> evict = candidate;        candidate = previous;        candidates--;        evictEntry(evict, RemovalCause.SIZE, 0L);        continue;      } else if (candidate == null) {        Node<K, V> evict = victim;        victim = victim.getNextInAccessOrder();        evictEntry(evict, RemovalCause.SIZE, 0L);        continue;      }       // Evict immediately if an entry was collected      K victimKey = victim.getKey();      K candidateKey = candidate.getKey();      if (victimKey == null) {        @NonNull Node<K, V> evict = victim;        victim = victim.getNextInAccessOrder();        evictEntry(evict, RemovalCause.COLLECTED, 0L);        continue;      } else if (candidateKey == null) {        @NonNull Node<K, V> evict = candidate;        candidate = (candidates > 0)            ? candidate.getPreviousInAccessOrder()            : candidate.getNextInAccessOrder();        candidates--;        evictEntry(evict, RemovalCause.COLLECTED, 0L);        continue;      }       // Evict immediately if the candidate's weight exceeds the maximum      if (candidate.getPolicyWeight() > maximum()) {        Node<K, V> evict = candidate;        candidate = (candidates > 0)            ? candidate.getPreviousInAccessOrder()            : candidate.getNextInAccessOrder();        candidates--;        evictEntry(evict, RemovalCause.SIZE, 0L);        continue;      }      // 此处是淘汰的核心逻辑，根据节点的统计频率frequency来做比较，看看要处理掉victim还是candidate。上面的一堆代码都是特殊情境下的特殊处理      if (admit(candidateKey, victimKey)) {        Node<K, V> evict = victim;        victim = victim.getNextInAccessOrder();        evictEntry(evict, RemovalCause.SIZE, 0L);        candidate = candidate.getPreviousInAccessOrder();      } else {        Node<K, V> evict = candidate;        candidate = (candidates > 0)            ? candidate.getPreviousInAccessOrder()            : candidate.getNextInAccessOrder();        evictEntry(evict, RemovalCause.SIZE, 0L);      }    }  }
```

上面的代码逻辑是从probation的头尾取出两个node进行比较频率，频率更小者将被remove，其中尾部元素就是上一部分从eden中淘汰出来的元素。如果将两步逻辑合并起来讲是这样的：在窗口队列通过lru淘汰出来的”候选者“与考察区队列通过lru淘汰出来的“被驱逐者“进行频率比较，失败者将被从cache中真正移除。

保护区的淘汰逻辑则在maintenance的最后一步climb中延后执行

**5、其中drainWriteBuffer消费写缓存的任务数据，从队列取task并执行**

caffeine性能高就是因为将队列分区，使其命中率高、污染率低，因此了解晋升、降级逻辑很重要。

caffeine使用了w-tinylru算法，当 window 区满了，就会根据 LRU 把 candidate（即淘汰出来的元素）放到 Probation区域， Probation区域则是一个观察区，当有新的缓存项需要进入Probation区时， 如果Probation区空间已满，则会将新进入的缓存项与Probation区中根据LRU规则需要被淘汰 （victim）的缓存项进行比较，两个进行“PK”，胜者留在 probation，输者就要被淘汰了。

```cobol
void drainWriteBuffer() {    for (int i = 0; i <= WRITE_BUFFER_MAX; i++) {      Runnable task = writeBuffer.poll();      if (task == null) {        return;      }      task.run();    }    lazySetDrainStatus(PROCESSING_TO_REQUIRED);  }
```

```cobol
    public void run() {      if (evicts()) {        // 更新权重        int oldWeightedSize = node.getPolicyWeight();        node.setPolicyWeight(oldWeightedSize + weightDifference);        // 如果在窗口区        if (node.inWindow()) {          if (node.getPolicyWeight() <= windowMaximum()) {            // 权重未超阈值，更新频率，移到队尾            onAccess(node);          } else if (accessOrderWindowDeque().contains(node)) {            // 权重超阈值，插到队头，下轮优先淘汰            accessOrderWindowDeque().moveToFront(node);          }          setWindowWeightedSize(windowWeightedSize() + weightDifference);        // 考察区        } else if (node.inMainProbation()) {            if (node.getPolicyWeight() <= maximum()) {              // 权重未超阈值，在考察区的元素在此访问后晋升到保护区              // 在维护方法maintenance的最后，会判断保护区是否达到权重最大值，会通过lru淘汰元素到考察区              // com.github.benmanes.caffeine.cache.BoundedLocalCache#climb              // com.github.benmanes.caffeine.cache.BoundedLocalCache#demoteFromMainProtected              onAccess(node);            } else if (accessOrderProbationDeque().remove(node)) {              accessOrderWindowDeque().addFirst(node);              setWindowWeightedSize(windowWeightedSize() + node.getPolicyWeight());            }        // 保护区        } else if (node.inMainProtected()) {           if (node.getPolicyWeight() <= maximum()) {            // 如果权重未满，更新频率，节点插到队尾            onAccess(node);            setMainProtectedWeightedSize(mainProtectedWeightedSize() + weightDifference);          } else if (accessOrderProtectedDeque().remove(node)) {            accessOrderWindowDeque().addFirst(node);            setWindowWeightedSize(windowWeightedSize() + node.getPolicyWeight());            setMainProtectedWeightedSize(mainProtectedWeightedSize() - oldWeightedSize);          } else {            setMainProtectedWeightedSize(mainProtectedWeightedSize() - oldWeightedSize);          }        }        setWeightedSize(weightedSize() + weightDifference);      } else if (expiresAfterAccess()) {        onAccess(node);      }      if (expiresAfterWrite()) {        reorder(writeOrderDeque(), node);      } else if (expiresVariable()) {        timerWheel().reschedule(node);      }    }
```

在维护方法maintenance的drainWriteBuffer中，涉及 更新元素的频率，同区域内重排序：

新增缓存数据首先写入 Window Cache 区域。当 Window Cache 空间满时，LRU 算法发挥作用，最久未被访问的缓存项会被移出 Window Cache 。这个被移出的缓存项称为 Candidate (候选项)，将面临 2 种情况：

- 如果 Probation Cache 未满，从 Window Cache 中移出的缓存项会直接写入 Probation Cache 中；
- 如果 Probation Cache 已满，则进入到 TinyLFU 过滤器，与 Protected Cache 移出的缓存项比较。根据 TinyLFU 算法确定该缓存项是淘汰还是写入 Probation Cache，后面会说
- Probation Cache 中的缓存项的访问频率达到一定次数后，就会晋升到 Protected Cache 中。（Ps. 到 Protected 保护区的都是经过 2 层选拔的真热门数据）如果 Protected Cache 也满了，根据 LRU 算法，最久未被访问的缓存项会被移出 Protected Cache ，降级成为 Candidate ，进入到 TinyLFU 过滤器，与 Window Cache 移出的缓存项作比较，根据 TinyLFU 算法确定该缓存项是淘汰还是写入到 Probation Cache 中。
- 此时把目光聚焦到 TinyLFU 过滤器，它汇聚了来自 Window Cache 、Protected Cache 和 Probation Cache 3 个区域中被移出的缓存项。其中，从 Window Cache 和 Protected Cache 中移出的缓存项称为 Candidate ；而从 Probation Cache 中移出的缓存项称为 Victim (受害者) 。这 3 个区域移出的缓存项都是各自区域中最久未被使用的数据，在 TinyLFU 过滤器中进行竞争，竞争算法如下：
    - 如果 Candidate 的访问频率 > Victim 的访问频率，则直接淘汰 Victim ；
    - 如果 Candidate 的访问频率 <= Victim 的访问频率，此时分为两种情况：
    - 如果 Candidate 的访问频率 < 5 ，则淘汰 Candidate ；
    - 如果 Candidate 的访问频率 >= 5 ，则在 Candidate 和 Victim 中随机淘汰一个。
    - 获胜者被称为 Winner ，就会被写入到 Probation Cache 中。

从上面的 W-TinyLFU 的原理描述可知，Caffeine 综合了 LFU 和 LRU 的优势，将不同特性的缓存项存入不同的缓存区域，最近刚产生的缓存项进入 Window Cache 区，不会被淘汰；访问频率高的缓存项进入 Protected Cache 区，也不会淘汰；介于这两者之间的缓存项存在 Probation 区，当缓存空间满了时，Probation 区的缓存项会根据访问频率判断是保留还是淘汰。

通过这种机制，很好的平衡了访问频率和访问时间新鲜程度两个维度因素，尽量将新鲜的缓存项和访问频率高的缓存项都保留在缓存中。同时在维护缓存项访问频率时，引入计数器饱和和衰减 (Count-Min Sketch, 下面会说) 机制，即节省了存储资源，也能较好的处理稀疏流量、短时超热点流量等传统 LRU 和 LFU 无法很好处理的场景。


## # Redis+Caffeine两级缓存，让访问速度纵享丝滑

### 什么场景可以使用：
1. 热点数据访问非常频繁，假设一个商品详情每秒被访问 1 万次。如果每次都请求 Redis，就需要产生大量网络请求。2. 数据变化不频繁3. 对响应延迟要求很高

如果使用 Caffeine，第一次从 Redis 获取后放进本地缓存，后续就可以直接从 JVM 内存读取。
在先不考虑并发等复杂问题的情况下，两级缓存的访问流程可以用下面这张图来表示：

![](https://i-blog.csdnimg.cn/direct/276cf583fffa48a7adb7bf13378142ff.png)

## 优点与问题

那么，使用两级缓存相比单纯使用远程缓存，具有什么优势呢?

- 本地缓存基于本地环境的内存，访问速度非常快
- 使用本地缓存能够减少和Redis类的远程缓存间的数据交互，减少网络I/O开销，降低这一过程中在网络通信上的耗时 ；

但是在设计中，还是要考虑一些问题的，例如数据一致性问题。首先，两级缓存与数据库的数据要保持一致，一旦数据发生了修改，在修改数据库的同时，本地缓存、远程缓存应该同步更新。

另外，如果是分布式环境下，一级缓存之间也会存在一致性问题，当一个节点下的本地缓存修改后，需要通知其他节点也刷新本地缓存中的数据，否则会出现读取到过期数据的情况，这一问题可以通过类似于Redis中的发布/订阅功能解决。

此外，缓存的过期时间、过期策略以及多线程访问的问题也都需要考虑进去，不过我们今天暂时先不考虑这些问题，先看一下如何简单高效的在代码中实现两级缓存的管理。

## 准备工作

代码实战，整合号称最强本地缓存的Caffeine作为一级缓存、性能之王的Redis作为二级缓存。
首先建一个springboot项目，引入缓存要用到的相关的依赖：
```
    com.github.ben-manes.caffeine
    caffeine
    2.9.2
 
    org.springframework.boot
    spring-boot-starter-data-redis
 
    org.springframework.boot
    spring-boot-starter-cache
 
    org.apache.commons
    commons-pool2
    2.8.1
```
配置Redis的连接信息：
```
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    timeout: 10000ms
    lettuce:
      pool:
        max-active: 8
        max-wait: -1ms
        max-idle: 8
        min-idle: 0
```

在下面的例子中，我们将使用RedisTemplate来对redis进行读写操作，RedisTemplate使用前需要配置一下ConnectionFactory和序列化方式，这一过程比较简单就不贴出代码了。

下面我们在单机环境下，将按照对业务侵入性的不同程度，分三个版本来实现两级缓存的使用。

## V1.0版本

我们可以通过手动操作Caffeine中的Cache对象来缓存数据，它是一个类似Map的数据结构，以key作为索引，value存储数据。在使用Cache前，需要先配置一下相关参数：

```
@Configuration
public class CaffeineConfig {
    @Bean
    public Cache caffeineCache(){
        return Caffeine.newBuilder()
                .initialCapacity(128)//初始缓存空大小
                .maximumSize(1024)//缓存的最大数量，设置这个值可以避免出现内存溢出
                .expireAfterWrite(60, TimeUnit.SECONDS)//指定缓存的过期时间，是最后一次写操作后的一个时间，这里
                //此外，缓存的过期策略也可以通过expireAfterAccess或refreshAfterWrite指定。
                .build();
    }
}
```

在创建完成Cache后，我们就可以在业务代码中注入并使用它了。在没有使用任何缓存前，一个只有简单的Service层代码是下面这样的，只有crud操作：
```
@Service
@AllArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final OrderMapper orderMapper;
 
    @Override
    public Order getOrderById(Long id) {  
        Order order = orderMapper.selectOne(new LambdaQueryWrapper()
              .eq(Order::getId, id));    
        return order;
    }
    
    @Override
    public void updateOrder(Order order) {      
        orderMapper.updateById(order);
    }
    
    @Override
    public void deleteOrder(Long id) {
        orderMapper.deleteById(id);
    }
}
```
现在：改造：
```
public Order getOrderById(Long id) {
    String key = CacheConstant.ORDER + id;
    Order order = (Order) cache.get(key,
            k -> {
                //先查询 Redis
                Object obj = redisTemplate.opsForValue().get(k);
                if (Objects.nonNull(obj)) {
                    log.info("get data from redis");
                    return obj;
                }
 
                // Redis没有则查询 DB
                log.info("get data from database");
                Order myOrder = orderMapper.selectOne(new LambdaQueryWrapper()
                        .eq(Order::getId, id));
                redisTemplate.opsForValue().set(k, myOrder, 120, TimeUnit.SECONDS);
                return myOrder;
            });
    return order;
}
```
在Cache的get方法中，会先从缓存中进行查找，如果找到缓存的值那么直接返回。如果没有找到则执行后面的方法，并把结果加入到缓存中。

因此上面的逻辑就是先查找Caffeine中的缓存，没有的话查找Redis，Redis再不命中则查询数据库，写入Redis缓存的操作需要手动写入，而Caffeine的写入由get方法自己完成。

在上面的例子中，设置Caffeine的过期时间为60秒，而Redis的过期时间为120秒，下面进行测试，首先看第一次接口调用时，进行了数据库的查询：

![](https://i-blog.csdnimg.cn/direct/9567b198bf3244f0bf967343b66d7deb.png)

而在之后60秒内访问接口时，都没有打印打任何sql或自定义的日志内容，说明接口没有查询Redis或数据库，直接从Caffeine中读取了缓存。

等到距离第一次调用接口进行缓存的60秒后，再次调用接口：

![](https://i-blog.csdnimg.cn/direct/497167c922414325849c826b4ec88a96.png)

可以看到这时从Redis中读取了数据，因为这时Caffeine中的缓存已经过期了，但是Redis中的缓存没有过期仍然可用。

下面再来看一下修改操作，代码在原先的基础上添加了手动修改Redis和Caffeine缓存的逻辑：
```
public void updateOrder(Order order) {
    log.info("update order data");
    String key=CacheConstant.ORDER + order.getId();
    orderMapper.updateById(order);
    //修改 Redis
    redisTemplate.opsForValue().set(key,order,120, TimeUnit.SECONDS);
    // 修改本地缓存
    cache.put(key,order);
}
```

看一下下面图中接口的调用、以及缓存的刷新过程。可以看到在更新数据后，同步刷新了缓存中的内容，在之后的访问接口时不查询数据库，也可以拿到正确的结果：

![](https://i-blog.csdnimg.cn/direct/25f267dac0304a409a5e2e161c07fb96.png)

最后再来看一下删除操作，在删除数据的同时，手动移除Reids和Caffeine中的缓存：

```
public void deleteOrder(Long id) {
    log.info("delete order");
    orderMapper.deleteById(id);
    String key= CacheConstant.ORDER + id;
    redisTemplate.delete(key);
    cache.invalidate(key);
}
```

我们在删除某个缓存后，再次调用之前的查询接口时，又会出现重新查询数据库的情况：

![](https://i-blog.csdnimg.cn/direct/5d5453337a234eaeba0493f0b81049e0.png)

简单地演示到此为止，可以看到上面这种使用缓存的方式，虽然看起来没什么大问题，但是对代码的入侵性比较强。在业务处理的过程中要由我们频繁地操作两级缓存，会给开发人员带来很大的负担。那么，有什么方法能够简化这一过程呢?我们看看下一个版本。

## V2.0注解版本

在spring项目中，提供了CacheManager接口和一些注解，允许让我们通过注解的方式来操作缓存。先来看一下常用的几个注解说明：

- @Cacheable：根据键从缓存中取值，如果缓存存在，那么获取缓存成功之后，直接返回这个缓存的结果。如果缓存不存在，那么执行方法，并将结果放入缓存中。
- @CachePut：不管之前的键对应的缓存是否存在，都执行方法，并将结果强制放入缓存。
- @CacheEvict：执行完方法后，会移除掉缓存中的数据。

如果要使用上面这几个注解管理缓存的话，我们就不需要配置V1版本中的那个类型为Cache的Bean了，而是需要配置spring中的CacheManager的相关参数，具体参数的配置和之前一样：

```java
@Configuration
public class CacheManagerConfig {
    @Bean
    public CacheManager cacheManager(){
        CaffeineCacheManager cacheManager=new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
                .initialCapacity(128)
                .maximumSize(1024)
                .expireAfterWrite(60, TimeUnit.SECONDS));
        return cacheManager;
    }
}
```

然后在启动类上再添加上@EnableCaching注解，就可以在项目中基于注解来使用Caffeine的缓存支持了。下面，再次对Service层代码进行改造。

首先，还是改造查询方法，在方法上添加@Cacheable注解：

```java
@Cacheable(value = "order",key = "#id")
//@Cacheable(cacheNames = "order",key = "#p0")
public Order getOrderById(Long id) {
    String key= CacheConstant.ORDER + id;
    //先查询 Redis
    Object obj = redisTemplate.opsForValue().get(key);
    if (Objects.nonNull(obj)){
        log.info("get data from redis");
        return (Order) obj;
    }
    // Redis没有则查询 DB
    log.info("get data from database");
    Order myOrder = orderMapper.selectOne(new LambdaQueryWrapper()
            .eq(Order::getId, id));
    redisTemplate.opsForValue().set(key,myOrder,120, TimeUnit.SECONDS);
    return myOrder;
}
```

@Cacheable注解的属性多达9个，好在我们日常使用时只需要配置两个常用的就可以了。其中value和cacheNames互为别名关系，表示当前方法的结果会被缓存在哪个Cache上，应用中通过cacheName来对Cache进行隔离，每个cacheName对应一个Cache实现。value和cacheNames可以是一个数组，绑定多个Cache。

而另一个重要属性key，用来指定缓存方法的返回结果时对应的key，这个属性支持使用SpringEL表达式。通常情况下，我们可以使用下面几种方式作为key：

```csharp
#参数名
#参数对象.属性名
#p参数对应下标
```

在上面的代码中，我们看到添加了@Cacheable注解后，在代码中只需要保留原有的业务处理逻辑和操作Redis部分的代码即可，Caffeine部分的缓存就交给spring处理了。

下面，我们再来改造一下更新方法，同样，使用@CachePut注解后移除掉手动更新Cache的操作：

```
@CachePut(cacheNames = "order",key = "#order.id")
public Order updateOrder(Order order) {
    log.info("update order data");
    orderMapper.updateById(order);
    //修改 Redis
    redisTemplate.opsForValue().set(CacheConstant.ORDER + order.getId(),
            order, 120, TimeUnit.SECONDS);
    return order;
}
```

注意，这里和V1版本的代码有一点区别，在之前的更新操作方法中，是没有返回值的void类型，但是这里需要修改返回值的类型，否则会缓存一个空对象到缓存中对应的key上。当下次执行查询操作时，会直接返回空对象给调用方，而不会执行方法中查询数据库或Redis的操作。

最后，删除方法的改造就很简单了，使用@CacheEvict注解，方法中只需要删除Redis中的缓存即可：
```
@CacheEvict(cacheNames = "order",key = "#id")
public void deleteOrder(Long id) {
    log.info("delete order");
    orderMapper.deleteById(id);
    redisTemplate.delete(CacheConstant.ORDER + id);
}
```
 
可以看到，借助spring中的CacheManager和Cache相关的注解，对V1版本的代码经过改进后，可以把全手动操作两级缓存的强入侵代码方式，改进为本地缓存交给spring管理，Redis缓存手动修改的半入侵方式。那么，还能进一步改造，使之成为对业务代码完全无入侵的方式吗?
## V3.0无侵害版本

模仿spring通过注解管理缓存的方式，我们也可以选择自定义注解，然后在切面中处理缓存，从而将对业务代码的入侵降到最低。

首先定义一个注解，用于添加在需要操作缓存的方法上：

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DoubleCache {
    String cacheName();
    String key(); //支持springEl表达式
    long l2TimeOut() default 120;
    CacheType type() default CacheType.FULL;
}
```
我们使用cacheName + key作为缓存的真正key(仅存在一个Cache中，不做CacheName隔离)，l2TimeOut为可以设置的二级缓存Redis的过期时间，type是一个枚举类型的变量，表示操作缓存的类型，枚举类型定义如下：

```
 public enum CacheType {
    FULL,   //存取
    PUT,    //只存
    DELETE  //删除
}
```

因为要使key支持springEl表达式，所以需要写一个方法，使用表达式解析器解析参数：

```
public static String parse(String elString, TreeMap map){
    elString=String.format("#{%s}",elString);
    //创建表达式解析器
    ExpressionParser parser = new SpelExpressionParser();
    //通过evaluationContext.setVariable可以在上下文中设定变量。
    EvaluationContext context = new StandardEvaluationContext();
    map.entrySet().forEach(entry->
        context.setVariable(entry.getKey(),entry.getValue())
    );
 
    //解析表达式
    Expression expression = parser.parseExpression(elString, new TemplateParserContext());
    //使用Expression.getValue()获取表达式的值，这里传入了Evaluation上下文
    String value = expression.getValue(context, String.class);
    return value;
}
```
参数中的elString对应的就是注解中key的值，map是将原方法的参数封装后的结果。简单进行一下测试：
```
public void test() {
    String elString="#order.money";
    String elString2="#user";
    String elString3="#p0";   
 
    TreeMap map=new TreeMap<>();
    Order order = new Order();
    order.setId(111L);
    order.setMoney(123D);
    map.put("order",order);
    map.put("user","Hydra");
 
    String val = parse(elString, map);
    String val2 = parse(elString2, map);
    String val3 = parse(elString3, map);
 
    System.out.println(val);
    System.out.println(val2);
    System.out.println(val3);
}
```
执行结果如下，可以看到支持按照参数名称、参数对象的属性名称读取，但是不支持按照参数下标读取，暂时留个小坑以后再处理。

```cobol
123.0
Hydra
null
```

至于Cache相关参数的配置，我们沿用V1版本中的配置即可。准备工作做完了，下面我们定义切面，在切面中操作Cache来读写Caffeine的缓存，操作RedisTemplate读写Redis缓存。

```cobol
@Slf4j @Component @Aspect @AllArgsConstructorpublic class CacheAspect {    private final Cache cache;    private final RedisTemplate redisTemplate;     @Pointcut("@annotation(com.cn.dc.annotation.DoubleCache)")    public void cacheAspect() {    }     @Around("cacheAspect()")    public Object doAround(ProceedingJoinPoint point) throws Throwable {        MethodSignature signature = (MethodSignature) point.getSignature();        Method method = signature.getMethod();         //拼接解析springEl表达式的map        String[] paramNames = signature.getParameterNames();        Object[] args = point.getArgs();        TreeMap treeMap = new TreeMap<>();        for (int i = 0; i < paramNames.length; i++) {            treeMap.put(paramNames[i],args[i]);        }         DoubleCache annotation = method.getAnnotation(DoubleCache.class);        String elResult = ElParser.parse(annotation.key(), treeMap);        String realKey = annotation.cacheName() + CacheConstant.COLON + elResult;         //强制更新        if (annotation.type()== CacheType.PUT){            Object object = point.proceed();            redisTemplate.opsForValue().set(realKey, object,annotation.l2TimeOut(), TimeUnit.SECONDS);            cache.put(realKey, object);            return object;        }        //删除        else if (annotation.type()== CacheType.DELETE){            redisTemplate.delete(realKey);            cache.invalidate(realKey);            return point.proceed();        }         //读写，查询Caffeine        Object caffeineCache = cache.getIfPresent(realKey);        if (Objects.nonNull(caffeineCache)) {            log.info("get data from caffeine");            return caffeineCache;        }         //查询Redis        Object redisCache = redisTemplate.opsForValue().get(realKey);        if (Objects.nonNull(redisCache)) {            log.info("get data from redis");            cache.put(realKey, redisCache);            return redisCache;        }         log.info("get data from database");        Object object = point.proceed();        if (Objects.nonNull(object)){            //写入Redis            redisTemplate.opsForValue().set(realKey, object,annotation.l2TimeOut(), TimeUnit.SECONDS);            //写入Caffeine            cache.put(realKey, object);                }        return object;    }}
```

切面中主要做了下面几件工作：

- 通过方法的参数，解析注解中key的springEl表达式，组装真正缓存的key。
- 根据操作缓存的类型，分别处理存取、只存、删除缓存操作。
- 删除和强制更新缓存的操作，都需要执行原方法，并进行相应的缓存删除或更新操作。
- 存取操作前，先检查缓存中是否有数据，如果有则直接返回，没有则执行原方法，并将结果存入缓存。

修改Service层代码，代码中只保留原有业务代码，再添加上我们自定义的注解就可以了：

```cobol
@DoubleCache(cacheName = "order", key = "#id",        type = CacheType.FULL)public Order getOrderById(Long id) {    Order myOrder = orderMapper.selectOne(new LambdaQueryWrapper()            .eq(Order::getId, id));    return myOrder;} @DoubleCache(cacheName = "order",key = "#order.id",        type = CacheType.PUT)public Order updateOrder(Order order) {    orderMapper.updateById(order);    return order;} @DoubleCache(cacheName = "order",key = "#id",        type = CacheType.DELETE)public void deleteOrder(Long id) {    orderMapper.deleteById(id);}
```

到这里，基于切面操作缓存的改造就完成了，Service的代码也瞬间清爽了很多，让我们可以继续专注于业务逻辑处理，而不用费心去操作两级缓存了。

总结本文按照对业务入侵的递减程度，依次介绍了三种管理两级缓存的方法。至于在项目中是否需要使用二级缓存，需要考虑自身业务情况，如果Redis这种远程缓存已经能够满足你的业务需求，那么就没有必要再使用本地缓存了。毕竟实际使用起来远没有那么简单，本文中只是介绍了最基础的使用，实际中的并发问题、事务的回滚问题都需要考虑，还需要思考什么数据适合放在一级缓存、什么数据适合放在二级缓存等等的其他问题。

# 多级缓存在项目中的运用
多级缓存包括很多层的缓存，**以一个购物网站为例，从前到后，可以涉及到5级缓存**

![](https://cdn.nlark.com/yuque/0/2025/png/54710028/1750332397844-dfb311bc-6dd8-489d-886a-3d2e9c859e1b.png)

1. **客户端缓存**：离用户最近的缓存，例如用电脑浏览网站，浏览器中的cookie，手机微信APP中的表情包，都是存储在用户自身电脑与手机中的。
2. **CDN缓存：**内容分发网络，是全球范围内离用户近的地方部署的用于存储网页、图片、视频等静态资源的一些边缘节点。例如，一个西藏用户浏览淘宝，淘宝的服务器假如部署在上海，那么数据从上海传输到西藏，流量存在浪费用户体验也不好，这时就可以提前将商品的图片，视频等静态资源部署在成都，大大缩短了流量成本，缩短用户体验。
3. **Nginx缓存**：WEB服务器缓存，Nginx除了做负载均衡外，还可以做一些上述客户端和CDN无法做到的缓存。像网络购物中风控系统会基于用户IP甄别是否为黄牛，那么就可以把这些IP地址放到nginx黑名单中，用于过滤。
4. **本地缓存**：是单台应用服务器维度的缓存，会占用服务器本身存储空间。假设一个分布式系统有5台应用服务器，那么这5台服务器中的缓存内容是独立且相同的，彼此不相互影响。本地缓存更多的是用来频繁地读，而非写。数据会随着应用程序的重启而丢失

- 优点：本地缓存不需要远程网络请求去操作内存空间，没有额外的性能消耗，所以读取速度快。
- 缺点：不能进行大数据量存储；应用程序集群部署时，会存在数据更新问题（数据更新不一致）

5. **分布式缓存**：一种专门做存储的系统（单台服务器或集群）如Redis。只要向其存储一份数据，那么接入该分布式缓存的所有应用服务器可以获取到相同的内容，保证了数据的一致性。举个例子，淘宝商品库存可以放到Redis中，用户在服务器A下了一单库存减少为99同步更新到Redis，其他用户在服务器B看到的库存也会变成99。

- 支持大数据量存储：分布式缓存是独立部署的进程，拥有自身独自的内存空间，不需要占用应用程序进程的内存空间，并且还支持横向扩展的集群方式部署，所以可以进行大数据量存储。
- 数据不会随着应用程序重启而丢失
- 数据集中存储，保证数据的一致性
- 数据读写分离，高性能，高可用

## _**本地缓存和数据库数据的数据一致性**_

**注意**：当然使用本地缓存，有一个问题是，当后端服务集群部署时，如果数据库的数据有更新的情况，本地缓存的数据和数据库的数据存在不一致的情况，如果要更新/删除本地缓存的数据，因为是集群部署，就要把所有节点的本地缓存的数据都进行更新/删除，此时这个实现稍微有些复杂，例如发送广播消息，所有实例节点监听广播消息，然后在本地缓存更新/删除。

但是其实可以使用一种更简单的方法，就是我们可以给本地缓存设置较短时间的TTL，这样我们可以不用去管本地缓存的数据更新，而是仅依靠TTL，去不断刷新本地缓存的数据。刷新机制

注意，用到本地缓存的数据，一般都是极少去改变的数据。
