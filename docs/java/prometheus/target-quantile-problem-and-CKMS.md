# 目标分位数问题 与 CKMS 算法

关于了解目标分位数问题和算法，可以简单参考：[分位数算法总结](https://caorong.github.io/2020/08/03/quartile- algorithm/)。CKMS 算法网上资料很少，只有一篇论文。

这里仅仅是大概看下 Prometheus client_java 中 `io.prometheus.metrics.core.metrics.CKMSQuantiles`的算法实现，了解怎么用以及算法的特点。

此算法可以确保通过百分位获取的值的 r 值（即在所有历史统计数据中的排序后的索引，CKMS samples 中的数据可能只占历史统计数据中的一少部分）处于 `[ckms.n*(q.quantile - 2*q.epsilon), ckms.n*(q.quantile + 2*q.epsilon)]` 闭区间，即获取的是误差允许范围内的近似值。

CKMS算法思想是不会存储全量数据，通过压缩算法，将误差允许范围内的多个近似值压缩为其中一个最大值存储，可以极大降低空间占用，允许误差越大空间占用越小，性能越高。

**数据结构**：

```java
final class CKMSQuantiles {
    // 分位数数组
    final Quantile[] quantiles;
    // 历史统计的观测值总数，并不是samples中观测值的个数
    // n 等于 samples 中各个观测值 g 字段的总和
    int n = 0;
    // 采样后的观测值列表，按 Sample.value 排序。
    // 这个才是真正存储观测值的容器
    final LinkedList<Sample> samples = new LinkedList<Sample>();
    // 压缩间隔，每插入128个数值压缩一次
    private final int compressInterval = 128;
    // 自从上次压缩后插入的观测值数量
    private int insertsSinceLastCompress = 0;
    // 临时存储观测值的 buffer，buffer 存满后以及get()前需要将观测值批量移动到samples
    // 引入 buffer 可以减少 flush() compress() 执行次数，从源码上看其实主要是降低 compress() 计算量
    private final double[] buffer = new double[compressInterval];
    // 最后插入数据的位置
    private int bufferPos = 0;
}

static class Quantile {
    // 分位数
    final double quantile;
    // 误差
    final double epsilon;
    // u = 2.0 * epsilon / (1.0 - quantile);
    // u v 本身没有意义，是计算 delta 公式的一部分
    final double u;
    // v = 2.0 * epsilon / quantile;
    final double v;
}

static class Sample {
	//观测值
    final double value;
	// 当前观测值与前一个观测值间的位置差值，所有元素的g值总和表示历史实际统计的数值总数
    // 没有压缩前所有Sample均为1,压缩后比如 1 2 3 4 5，被压缩成了 1 5, 那么5的g值是4，表示压缩了4个元素
    // 比如 Sample{val=5.000, g=4, delta=0} 这个值表示历史上有4个值被压缩为了这一个值
    // 原版注释说的有些抽象：
    // Difference between the lowest possible rank of this sample and its predecessor.
    // This always starts with 1, but will be updated when compress() merges Samples.
    int g = 1;
    //r 其实是元素在历史中所有被统计的元素中的索引
	//delta 的计算方式：
    //对于 r >= q.quantile * n 的元素， 即 >= 期望元素索引的元素
    // delta = (int) 2*epsilon*r/quantile - 1
    //对于 r < q.quantile * n 的元素
    // delta = (int) 2*epsilon*(n-r)/(1-quantile) - 1
    //个人理解 delta 就是压缩容忍范围，比如
    final int delta;
}
```

里面的核心的代码是 **delta 的计算** 和 **压缩条件**，从源码流程看，samples 的元素一直在变（有插入有删除），至于为何通过这样的计算可以最终通过分位值获取的元素在历史数据中的排序索引不会超出误差范围还是需要看论文，证明过程还是挺复杂的。

```java
int f(int r) {
    int minResult = Integer.MAX_VALUE;
    for (Quantile q : quantiles) {
        if (q.quantile == 0 || q.quantile == 1) {
            continue;
        }
        int result;
        // 以前面的例子第二批数据flush()，当 2.0 插入到 5.0 前面，r=1, quantile=0.5, n=8
	//delta 的计算方式：
        //对于 r >= q.quantile * n 的元素， 即 >= 期望元素索引的元素
        // delta = (int) 2*epsilon*r/quantile - 1
        //对于 r < q.quantile * n 的元素
        // delta = (int) 2*epsilon*(n-r)/(1-quantile) - 1， 当前例子就是 delta = 1
        if (r >= q.quantile * n) {
            result = (int) (q.v * r + 0.00000000001);
        } else {
            result = (int) (q.u * (n - r) + 0.00000000001);
        }
        if (result < minResult) {
            minResult = result;
        }
    }
    return Math.max(minResult, 1);
}

void compress() {
    if (samples.size() < 3) {
        return;
    }
    Iterator<Sample> descendingIterator = samples.descendingIterator();
    // 假设 1 2 3 4 5，从未压缩过，初始 r= 5
    int r = n; // n is equal to the sum of the g's of all samples
    Sample right; //右值，即大值
    Sample left = descendingIterator.next(); //挨着右值的小值
    r -= left.g;    // r 其实是元素在历史中所有被统计的元素中的索引，这里 r 指 left 元素的历史索引
    // 倒序遍历，即从大值开始压缩
    while (descendingIterator.hasNext()) {
        right = left;
        left = descendingIterator.next();
        r = r - left.g;
        if (left == samples.getFirst()) {
            // The min sample must never be merged.
            break;
        }
        // 压缩条件：left.g + right.g + right.delta < f(r)
        // 小值删除，大值的g加上小值的g
        if (left.g + right.g + right.delta < f(r)) {
            right.g += left.g;
            descendingIterator.remove();
            left = right;
        }
    }
}
```

