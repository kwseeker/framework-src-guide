# 消费者负载均衡策略 — 一致性哈希分配策略  ConsistentHashRouter

这里分析 `ConsistentHashRouter` 一致性哈希算法实现原理。

看其内部实现是比较基础的一致性哈希算法实现，这种实现**物理节点对应的虚拟机节点是散乱交叉分布的**，只可以用于**解决集群动态伸缩问题**，**无法降低节点上下线的数据迁移成本**（比如数据库数据分片存储，删除一个数据库节点，这个数据库节点上的数据可能需要向其他多个节点迁移数据）。

对比物理节点对应的虚拟节点**连续分布**的图片：

![](https://ucc.alicdn.com/pic/developer-ecology/8aa02f948488428d99a8b73cd1a4bea9.png)

数据结构：

```java
public class ConsistentHashRouter<T extends Node> {
    // 一致性哈希算法哈希环实现，默认为每个物理节点（消费者）创建10个虚拟节点
    // TreeMap不是环，匹配到最大虚拟节点后需要特殊处理，使用第一个虚拟节点的物理节点进行处理
    private final SortedMap<Long, VirtualNode<T>> ring = new TreeMap<>();
    // 计算 hash 值的函数，默认使用 MD5Hash
    // 先进行 MD5 摘要，然后
    // long h = 0;
    // for (int i = 0; i < 4; i++) {
    //     h <<= 8;
    //     h |= ((int) digest[i]) & 0xFF;
    // }
    // return h;
    private final HashFunction hashFunction;
    ...
}
```

哈希环初始化（添加物理节点和虚拟节点）：

一个物理节点可能对应多个虚拟节点，虚拟节点分配到的消息队列最终是交给对应的物理节点处理；
注意哈希环上仅仅展示虚拟节点，物理节点则是通过虚拟节点内部的引用获取。

```java
// pNode是消费者对应的物理节点
public void addNode(T pNode, int vNodeCount) {
    if (vNodeCount < 0)
        throw new IllegalArgumentException("illegal virtual node counts :" + vNodeCount);
    // 遍历哈希环当前所有虚拟节点，看是否有当前物理节点对应的虚拟节点，返回虚拟节点的个数。
    int existingReplicas = getExistingReplicas(pNode);
    // 循环 vNodeCount, 即为每个物理节点创建了10个虚拟节点
    for (int i = 0; i < vNodeCount; i++) {
        // 为物理节点创建虚拟节点
        VirtualNode<T> vNode = new VirtualNode<>(pNode, i + existingReplicas);
        // 哈希环上仅仅展示虚拟节点，通过虚拟节点内部的引用获取物理节点
        // 使用物理节点即消费者ID+当前虚拟节点编号作为key求哈希，然后将虚拟节点放到哈希环上
        ring.put(hashFunction.hash(vNode.getKey()), vNode);
    }
}
```

消息队列的路由：

```java
// MessageQueue 重写了 toString() 方法，使用 toString 返回值作为 key
public T routeNode(String objectKey) {
    if (ring.isEmpty()) {
        return null;
    }
    Long hashVal = hashFunction.hash(objectKey);
    // 查找右子树
    SortedMap<Long, VirtualNode<T>> tailMap = ring.tailMap(hashVal);
    // 如果匹配到最大的虚拟节点，则没有子树，就返回第一个虚拟节点，即模拟环的逻辑
    Long nodeHashVal = !tailMap.isEmpty() ? tailMap.firstKey() : ring.firstKey();
    // 先获取虚拟节点，再获取对应的物理节点
    return ring.get(nodeHashVal).getPhysicalNode();
}
```

