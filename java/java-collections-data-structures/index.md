## 1. 先按访问模式选集合

集合选型的第一问题不是“哪个更快”，而是数据是否有序、是否允许重复、按什么键查询、是否跨线程共享。

| 需求 | 常用选择 | 说明 |
|---|---|---|
| 按下标访问、尾部追加 | `ArrayList` | 连续存储，局部性好 |
| 去重、不要求顺序 | `HashSet` | 基于哈希表 |
| 保持插入顺序 | `LinkedHashMap/Set` | 额外维护链表 |
| 按键排序 | `TreeMap/Set` | 基于有序树 |
| 先进先出 | `ArrayDeque` | 通常优于 `LinkedList` |
| 优先级队列 | `PriorityQueue` | 堆结构，不保证整体有序 |
| 并发键值表 | `ConcurrentHashMap` | 支持高并发访问 |

## 2. List、Set 与 Queue

`ArrayList` 适合绝大多数列表场景。中间插入需要移动元素，但实际性能仍要结合访问局部性判断。`LinkedList` 每个节点有额外对象成本，随机访问慢，不能仅凭“插入是 O(1)”就选择它，因为找到插入位置本身可能是 O(n)。

`HashSet` 通过元素的 `hashCode` 和 `equals` 判重。放入集合后修改参与哈希计算的字段，会导致元素难以再次找到。实体作为键时应使用稳定、不可变的标识。

## 3. HashMap 的关键契约

哈希表先根据哈希值定位桶，再通过相等判断确定键。正确性依赖：相等对象必须拥有相同哈希值；对象不相等可以发生哈希冲突。

```java
public record UserId(long value) {
    public UserId {
        if (value <= 0) throw new IllegalArgumentException("value must be positive");
    }
}
```

不可变 record 很适合作为 Map 键。不要依赖 `HashMap` 的遍历顺序，也不要在多个线程中无保护地同时修改普通 `HashMap`。

## 4. 排序与比较

自然顺序由 `Comparable` 定义，外部排序规则由 `Comparator` 定义。比较器应满足反对称性、传递性，并尽量与 `equals` 一致。多字段排序可以组合：

```java
Comparator<User> order = Comparator
    .comparing(User::department)
    .thenComparing(User::name)
    .thenComparingLong(User::id);
```

## 5. 并发集合

`ConcurrentHashMap` 适合并发查询和更新，但复合操作仍要使用 `compute`、`merge`、`putIfAbsent` 等原子 API。`CopyOnWriteArrayList` 适合读多写极少、集合较小的监听器场景；每次写都会复制数组，不适合高频写入。

阻塞队列连接生产者和消费者：有界队列可以形成背压，无界队列可能在流量高峰耗尽内存。线程池队列必须明确容量和拒绝策略。

## 6. 常用工程结构

- LRU：可使用访问顺序的 `LinkedHashMap`，生产环境还要考虑并发、容量和过期。
- LFU：需要同时维护频率与同频次顺序，通常优先采用成熟缓存库。
- 布隆过滤器：用少量空间判断“一定不存在或可能存在”，存在误判但不会漏判已加入元素。
- 位图：适合范围明确的整数集合和状态标记。
- 堆：适合 Top K、任务调度和动态中位数问题。

## 7. 性能判断

复杂度只是上界模型。真实选择还受对象分配、缓存局部性、装箱、GC、数据规模和并发竞争影响。优化前应先测量；使用 JMH 时要预热并避免死代码消除，不能用一次 `System.nanoTime()` 循环得出结论。

## 检查表

- 是否真的需要保持顺序？
- 键是否不可变并正确实现相等契约？
- 是否需要允许 `null`？
- 是否跨线程修改？
- 队列是否有界？满时如何处理？
- 数据量是否大到需要专用结构？

## 参考资料

- [Java Collections Framework](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/doc-files/coll-overview.html)
- [Java 并发集合 API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)

