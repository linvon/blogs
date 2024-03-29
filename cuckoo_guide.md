---
title: "比布隆过滤器更好：布谷鸟过滤器实战对比与调参指南"
date: "2021-02-25"
toc: "true"
math: "true"
keywords: 
- cuckoofilter
- 布谷鸟过滤器
- 布谷鸟
- 过滤器
- cuckoo
- filter
- 参数
- 布隆过滤器
- 布隆
- bloom
- go
- golang

---



“判断一个值是否在一个巨大的集合当中”（下文中统称为集合隶属测试），是一种常见的数据处理问题。在以往的经验中，如果允许一定的假阳性率，那么布隆过滤器是首选，而如今我们有了更好的选择：布谷鸟过滤器

## 布谷鸟过滤器

布谷鸟过滤器在网络上已经有很多的介绍文章了，本文不再去详细阐述布谷鸟过滤器是什么、其原理是什么，我们直接拿他与布隆过滤器作比较，看看其在实际使用中的优劣

如果想要知道更多的细节，可以参考 [原论文](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)，或者查看我的 [中文翻译版本](http://www.linvon.cn/posts/cuckoo/)

### 什么是布谷鸟过滤器？

是一种基于布谷鸟哈系算法实现的过滤器，本质上是存储了插入项部分哈希值的布谷鸟哈希表。

如果你了解布隆过滤器，你应该知道布隆过滤器原理是采用多种哈希方法，将存储项的不同哈希映射到位数组上，查询时通过对这些位进行检查来判断是否存在。

而布谷鸟过滤器是对存储项做哈希，将其哈希值取一定长度位数存储在数组中，查询时通过判断相等位数的哈希是否在数组中来判断是否存在。

### 为什么选择布谷鸟过滤器？

同样都是存储哈希值，本质上都是存储多位哈希，为什么布谷鸟过滤器更优？

- 一是由于布谷鸟哈希表更加紧凑，因此可以更加节省空间。

- 二是因为在查询时，布隆过滤器要采用多种哈希函数进行多次哈希，而布谷鸟过滤器只需一次哈希，因此查询效率很高

- 三是布谷鸟过滤器支持删除，而布隆过滤器不支持删除

优点有了，缺点呢？相比于布隆过滤器

- 布谷鸟过滤器采用一种备用候选桶的方案，候选桶与首选桶可以通过位置和存储值互相异或得出，这种对应关系要求桶的大小必须是 2 的指数倍
- 布隆过滤器插入时计算好哈希直接写入位即可，而布谷鸟过滤器在计算后可能会出现当前位置上已经存储了指纹，这时候就需要将已存储的项踢到候选桶，随着桶越来越满，产生冲突的可能性越来越大，插入耗时越来越高，因此其插入性能相比布隆过滤器很差
- 插入重复元素：布隆过滤器在插入重复元素时并没有影响，只是对已存在的位再置一遍。而布谷鸟过滤器对已存在的值会做踢出操作，因此重复元素的插入存在上限
- 布谷鸟过滤器的删除并不完美：有上述重复插入的限制，删除时也会出现相关的问题：删除仅在相同哈希值被插入一次时是完美的，如果元素没有插入便进行删除，可能会出现误删除，这和假阳性率的原因相同；如果元素插入了多次，那么每次删除仅会删除一个值，你需要知道元素插入了多少次才能删除干净，或者循环运行删除直到删除失败为止

优缺点都列出来了，我们再来总结一下。对于这种集合隶属测试问题，我们通常是用该类数据结构实现缓存，减小读压力。大部分情景都是读多写少的，而重复插入并没有什么意义，布谷鸟过滤器的删除虽然不完美但总好过没有，同时还有更优的查询和存储效率，应该说在绝大多数情况下其都是一个性价比更高的选择。

## 实战指南

### 细节实现

先说一下布谷鸟过滤器中的概念，过滤器是由很多桶组成的，每个桶存储插入项经过哈希计算后的值，该值只会存储固定位数。

过滤器中有 n 个桶，桶的数量是根据要存储的项的数量计算得来的。通过哈希算法我们可以计算出一个项应该存储在哪个桶中，此外每增加一个哈希算法，就可以对一个项产生一个候选桶，当重复插入时，会把当前存储的项踢到候选桶中去。理论上哈希算法越多，空间利用率越高，但实际测试使用 k=2 个哈希函数就可以达到 98% 的利用率了。

每一个桶会存储多个指纹，这受制于桶的大小，不同的指纹可能映射到同一个桶中。桶越大，空间利用率越高，但同时每次查询扫描同一桶中指纹数越多，因此产生假阳性的概率越高，此时就需要提高存储的指纹位数，用以降低冲突率，维持假阳性率。

在论文中提到了实现布谷鸟过滤器所需的几个参数，主要是

- 哈希函数个数（k）：哈希个数，取 2 就足够
- 桶大小（b）：每个桶存储多少个指纹
- 指纹大小（f）：每个指纹存储键的哈希值的多少位

详细阅读论文，在第五章中作者依靠试验数据告诉了我们如何选择最合适的构建参数，我们可以得到如下结论

- 过滤器无法 100% 填满，存在最大负载因子 α，那么均摊到每个项上的存储占用空间就是 f/α
- 当保持过滤器总大小不变时，桶越大负载因子越高，即空间利用率越高，但每个桶存储的指纹越多，查询时可能发生冲突的概率也越高，为了维持假阳性率不变，桶越大，就需要越大的指纹

根据上述的理论依据，得出的相关实验数据有：

- 使用 k=2 个哈希函数时，当桶大小 b=1（即直接映射哈希表）时，负载因子 α 为 50%，但使用桶大小 b=2、4 或 8 时则分别会增加到 84%、95% 和 98%
- 为了保证假阳性率 r，需要保证 $2b/2^f\leq r$ ，那么指纹 f 大小约为 $f ≥ log_2(2b/r)=log_2(1/r) + log_2(2b)$ ，那每个项的均摊成本即为 $C ≤ [log_2(1/r) + log_2(2b)]/α$
- 实验数据表明，当 r>0.002 时。每桶有两个条目比每桶使用四个条目产生的结果略好；当 r 减小到 0.00001<r≤0.002 时，每个桶四个条目可以最小化空间
- 如果使用半排序桶，可以对每一个存储项减少 1bit 的存储空间，但其仅作用于 b=4 的过滤器

**这样一来我们便可以确定如何选择参数来构建我们的布谷鸟过滤器了**：

**首先哈希函数我们使用两个就足够了，这可以达到足够的空间利用率。根据我们需要的假阳性率，我们可以确定使用多大的桶大小，当然 b 的选择并不绝对，即使 r>0.002，你也可以使用 b=4 来启用半排序桶。之后我们可以根据 b 来计算为了达到目标假阳性率，我们需要的 f 的大小，这样所有的过滤器参数就确定了。**

将上面的结论与布隆过滤器的每项 $1.44log_2(1/r)$ 对比，可以发现启用半排序时，当 r<0.03 左右，布谷鸟过滤器空间更小，若不启用半排序，则退化到 r<0.003 左右。

#### 一些进阶解释

**哈希算法的优化**

虽然我们指定了需要两个哈希算法，但实际实现上我们使用一个哈希算法就足够了，因为在论文中提到了一种备选桶计算方法，第二个哈希值可以由第一个哈希值与该位置存储的指纹异或计算得来。如果你在担心我们还需要分别计算指纹的哈希和位置的哈希，我们可以只用一种算法制作 64 位的哈希，高 32 位用于计算位置，低 32 位用于计算指纹。

**为什么半排序桶只能用于 b=4 的情况？**

半排序的本质是对每个指纹取其四位，该四位可以表示为一个十六进制，对于 b 个指纹的四位存储就可以表示为 b 个 16 进制数，将其所有可能按顺序排列后，可以通过索引其位置来找到对应的排列，从而获取实际的存储值。

我们可以通过以下函数计算所有的情况种类数

``` go
func getNum(base, k, b, f int, cnt *int) {
	for i := base; i < 1<<f; i++ {
		if k+1 < b {
			getNum(i, k+1, b, f, cnt)
		} else {
			*cnt++
		}
	}
}

func getNextPow2(n uint64) uint {
	n--
	n |= n >> 1
	n |= n >> 2
	n |= n >> 4
	n |= n >> 8
	n |= n >> 16
	n |= n >> 32
	n++
	return uint(n)
}

func getNumOfKindAndBit(b, f int) {
	cnt := 0
	getNum(0, 0, b, f, &cnt)
	fmt.Printf("Num of kinds: %v, Num of needed bits: %v\n", cnt, math.Log2(float64(getNextPow2(uint64(cnt)))))
}

```

在 b=4 时，总共有 3786 种排列，小于 4096，即用 12 位即可存储所有的排列索引，而如果直接存储所有指纹，则需要 4X4=16 位，这样节省了 4 位，即对每一个指纹节省了一位。

可以发现，在 b 为 2 时是否启用半排序需要存储的位数相同，没有意义。如果 b 太大则需要存储的索引也会急剧扩张，会在查询性能上有很大的损耗，因此 b=4 是一个性价比最高的选择。

此外编码存储四位指纹的选择是因为其刚好可以用一个十六进制表示，利于存储

**使用半排序时的参数选择**

使用半排序时，应保证 $ceil(b*(f-1)/8)<ceil(b*f/8)$，否则是否使用半排序占用的空间是一样大的

**过滤器大小选择**

过滤器的桶总大小一定是 2 的指数倍，因此在设定过滤器大小时，尽量满足 $size/α ~=(<) 2^n$，size 即为想要一个过滤器存储的数据量，必要时应选择小一点的过滤器，使用多个过滤器达到目标效果

### Golang 实现

这部分主要是 Golang 库相关

在翻阅了 Github 上 cuckoofilter 的 golang 实现后，发现已有的实现都存在一些缺点：

- 绝大部分的库都是固定 b 和 f 的，即假阳性率也是固定的，适应性不好
- 所有的库 f 都是以字节为单位的，只能以 8 的倍数来调整，不方便调整假阳性率
- 所有的库都没有实现半排序桶，相比于布隆过滤器的优势大打折扣

因为自己的场景需要更优的空间和自定的假阳性率，因此移植了原论文的 C++ 实现，并做了一些优化，主要包括

- 支持调节参数

- 支持半排序桶

- 压缩空间到紧凑的位数组，按位存储指纹

- 支持二进制序列化

附上地址，欢迎贡献或 debug [Github](https://github.com/linvon/cuckoo-filter)
