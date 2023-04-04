### GO 中map的底层是如何实现的

首先**Go 语言采用的是哈希查找表，并且使用链表解决哈希冲突。**

#### GO的内存模型

先看这一张map原理图

![image-20230102180419004](img/image-20230102180419004.png)

再来看看源码中map的定义

```go
//src/runtime/map.go  line 115

// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    //
    count     int  //len(map)时，返回的值
    flags     uint8   //表示是否有其他协程在操作map
    B         uint8    //上图中[]bmap的''长度'' 2^B
    noverflow uint16  //// 溢出的bucket个数
    hash0     uint32 // hash seed, key 计算hash 时的hash 因子

    buckets    unsafe.Pointer    //buckets是一个指针指向一个长度为2^B的数组，数组的每个元素是bmap类型，该结构包含8个key/value，称为一个桶
    oldbuckets unsafe.Pointer  // 扩容的时候用于赋值的buckets数组
    nevacuate  uintptr       // 搬迁进度

    extra *mapextra   // 用于扩容的指针

    type mapextra struct {
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    nextOverflow *bmap
}
```



但这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构：

```go
type bmap struct {
	// tophash是一个大小为8的数组，记录了key计算hash值得高8位，其正常情况下其取值大于5(如果key哈希的8八位计算结果小于5会被强制设置为5)，0-5是有特殊含义：
    // 0: 该处的元素是空的没有值，并且其后的元素也是空的。所以再遍历该桶的时候遇到此值就不用再往后进行遍历了
    // 1: 该处的元素是空的
    // 2: key值是有效的，元素已经被迁移到新桶的前半部分
    // 3: 同上，元素已经被迁移到新桶的后半部分？这个后半部分怎么理解
    // 4: 此处元素是空的，旧桶的数据已经全部被迁移到新桶了
    // 5: key哈希的8八位计算结果小于5会被强制设置为5
	tophash [bucketCnt]uint8
	
    
    // 接下来是map的8个key和value。结构体省略了没有体现出来
    key[8]
    value[8]
 
    // 注意一个细节是Bucket中key/value的放置顺序，是将keys放在一起，values放在一起，
    // 为什么不将key和对应的value放在一起呢？如果那么做，存储结构将变成key1/value1/key2/value2… 
    // 设想如果是这样的一个map[int64]int8，考虑到字节对齐，会浪费很多存储空间
 
    // 溢出桶的指针,如果该桶的8个key/value存满了，有新的元素需要加进来就重新创建一个桶，将该指针指向新桶地址形成一个链表结构
    overflow_ptr
}
```

>
>
>通过对key进行hash得到的最后B位可以锁定到哪一个桶, 也是就 在 hmap 中 通过key hash 得到最后的B位，确定在哪个 bmap 中

bmap 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的(低位的B位决定bucket)。

在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。如上图所示

>
>
>但是每个bmap桶中还有8个元素，并且还有可能有溢出桶，在每个桶中查找key时并不是将传入的key和桶中的key进行hash运算然后做比较因为做hash运算太耗时了。所以这里利用了tophash数组在map赋值时就已经提前计算好了key的hash值并将高8位存入tophash中，这样在比较时直接和tophash里的值进行比较，可以大大的节约时间，相当于用空间换时间。当然在进行查找key时tophash比较结果相同后还有再次对传入的key和桶中的key进行hash运算比较，避免出现高8位hash冲突的情况，虽然几率很小。

bmap 是存放 k-v 的地方，我们把视角拉近，仔细看 bmap 的内部组成。

<img src="img/image-20230102180655899.png" alt="image-20230102180655899" style="zoom:30%;" />

上图就是 bucket 的内存模型， HOBHash 指的就是 top hash。注意到 key 和 value 是各自放在一起的，并不是 key/value/key/value/… 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来。



### 创建map

从语法上来说，创建一个map很简单（记得key的类型必须为可比较类型）

实际上底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 ；给buckets分配内存等。

这里有一个重要的概念装载因子6.5。6.5表示的含义是每平均每个桶存放的元素个数要小于6.5，因为每个桶可以装8个元素， 但是由于kay的hash计算并不能保证每个桶都能平均的分配到8个元素，可能会出现有的桶中元素小于8个有的桶中元素大于8个，这时会出现溢出桶。所以map的作者经过计算得到一个经验值每个桶的平均元素个数小于6.5达到最佳性能。

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
 
	// new一个hmap作为map的头结构
	if h == nil {
		h = new(hmap)
	}
	// 随机生成一个hash种子
	h.hash0 = fastrand()
 
	// 计算一个合适的B使装载因子小于6.5。6.5表示的含义是每平均每个桶存放的元素个数要小于6.5，因为每个桶可以装8个元素，
	// 但是由于kay的hash计算并不能保证每个桶都能平均的分配到8个元素，可能会出现有的桶中元素小于8个有的桶中元素大于8个，
	// 这时会出现溢出桶影响。所以map的作者经过计算得到一个经验值每个桶的平均元素个数小于6.5达到最佳性能。
	// 如果 map元素个数 > 6.5 * 2^B 表示已经桶过载了需要扩容
	B := uint8(0)
	for overLoadFactor(hint, B) {
		// 逐步增加B直到桶不在过载
		B++
	}
	h.B = B
 
	if h.B != 0 {
		var nextOverflow *bmap
		// 给buckets分配内存
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
 
	return h
}
```

### hash函数

map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 alginit() 中完成，位于路径：src/runtime/alg.go 下。

> hash 函数，有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；非加密型的一般就是查找。在 map 的应用场景中，用的是查找。选择 hash 函数主要考察的是两点：性能、碰撞概率。

### key的定位过程

key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，`只会用到最后 B 个 bit 位`。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。
例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

```
10010111 | 000011110110110010001111001010100010010110010101010 │ 01010 
高8位作用于bmap tophash 																					 低B位作用于hmap 确定bmap
```

- 用最后的 5 个 bit 位 （B），也就是 01010，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的`位操作`代替。 (hmap 查找 bmap 位置)
- 再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。 （bmap 查找 key）

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

这里参考曹大 github 博客里的一张图

<img src="img/image-20230102180953360.png" alt="image-20230102180953360" style="zoom:30%;" />

上图中，假定B=5，所以bucket总数就是2^5=32。首先计算出待查找key的哈希，使用低5位00110，找到对应的6号bucket，使用高8位10010111，对应十进制151，在6号bucket中寻找tophash值（HOBhash）为151的key，找到了2号槽位，这样整个查找过程就结束了。如果在bucket中没找到，并且overflow不为空，还要继续去overflowbucket中寻找，直到找到或是所有的key槽位都找遍了，包括所有的overflowbucket。



### 访问map的过程


```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
 
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		// map 是空的直接返回value的零值
		return unsafe.Pointer(&zeroVal[0])
	}
	// 有其他协程正在对map进行写操作，抛出异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	// 获取key类型对应的hash计算公式
	alg := t.key.alg
	// 对key进行hash运算，h.hash0为随机的种子，在map初始化时设置的随机值
	hash := alg.hash(key, uintptr(h.hash0))
 
	m := bucketMask(h.B)
	// 根据hash的低B位，计算key落在哪一个桶。
	// h.buckets(buckets起始地址) + hash&m(第几个桶) * t.bucketsize(一个桶的大小)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	// 如果有旧桶说明发生了扩容，那么先在旧桶中找
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// 新桶是旧桶的2倍，m偏移量要减半否则数据组会越界
			m >>= 1
		}
		// 计算key在旧桶中的位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		// 旧桶中的元素还没有全部转移到新桶
		if !evacuated(oldb) {
			// 接下来从旧桶中开始查
			b = oldb
		}
	}
	// 获取hash的高8位，如果小于5，则加上5.因为0-5是保留字段有特殊含义
	top := tophash(hash)
bucketloop:
	// 遍历所有的桶，如果有溢出桶也要遍历
	for ; b != nil; b = b.overflow(t) {
		// 遍历每个桶的8个元素
		for i := uintptr(0); i < bucketCnt; i++ {
			// tophash不相等
			if b.tophash[i] != top {
				// 并且tophash == emptyRest，说明该元素之后的所有元素都是空的没必要在往后遍历了
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// 走到这里说明tophash已经匹配了
 
			// 找到key的位置
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
 
			// t.flags取值在1,2,4,8,16
			// 1: key是一个指针类型，在计算hash时需要取指针里的内容
			// 2: value是一个指针类型
			// 4: k == k ??
			// 8: 需要更新覆盖key
			// 16: 计算hash时发生panic了
 
			// 判断key是一个指针类型，在计算hash时需要取指针里的内容
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 对传入的key和从桶中取到的key再次进行比较
			if alg.equal(key, k) {
				// 找到value对应的地址
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				// 如果value是个指针类型取指针里的内容
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				// 返回查到的value
				return e
			}
		}
	}
	// 没查到返回map对应的value的零值
	return unsafe.Pointer(&zeroVal[0])
}
```

### map的扩容

使用哈希表的目的就是要快速查找到目标key，然而，随着向map中添加的key越来越多，key发生碰撞的概率也越来越大。bucket中的8个cell会被逐渐塞满，查找、插入、删除key的效率也会越来越低。最理想的情况是一个bucket只装一个key，这样，就能达到O(1)的效率，但这样空间消耗太大，用空间换时间的代价太高。
Go语言采用一个bucket里装载8个key，定位到某个bucket后，还需要再定位到具体的key，这实际上又用了时间换空间。当然，这样做，要有一个度，不然所有的key都落在了同一个bucket里，直接退化成了链表，各种操作的效率直接降为O(n)，是不行的。因此，需要有一个指标来衡量前面描述的情况，这就是`装载因子`。Go源码里这样定义装载因子

`loadFactor:=count/(2^B)`

count就是map的元素个数，2^B表示bucket数量。

再来说触发map扩容的时机：在向map插入新key的时候，会进行条件检测，符合下面这2个条件，就会触发扩容：

①装载因子超过阈值，源码里定义的阈值是6.5。

②overflow的bucket数量过多，大于等于2^B：noverflow >= 1<<(B&15)，其中B最大只能是15。

通过汇编语言可以找到赋值操作对应源码中的函数是mapassign，对应扩容条件的如下：

第1点：我们知道，每个bucket有8个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是8。因此当装载因子超过6.5时，表明很多bucket都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。
第2点：是对第1点的补充。就是说在装载因子比较小的情况下，这时候map的查找和插入效率也很低，而第1点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即map里元素总数少，但是bucket数量多（真实分配的bucket数量多，包括大量的overflowbucket）。不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多bucket，但是装载因子达不到第1点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的overflowbucket，但就是不会触犯第1点的规定，你能拿我怎么办？overflowbucket数量太多，导致key会很分散，查找插入效率低得吓人，因此出台第2点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。
对于命中条件1，2的限制，都会发生扩容。但是扩容的策略并不相同，毕竟两种条件应对的场景不同。
对于条件1，元素太多，而bucket数量太少，很简单：将B加1，bucket最大数量（2^B）直接变成原来bucket数量的2倍。于是，就有新老bucket了。注意，这时候元素都在老bucket里，还没迁移到新的bucket来。而且，新bucket只是最大数量变为原来最大数量（2^B）的2倍（2^B*2）。
对于条件2，其实元素没那么多，但是overflowbucket数特别多，说明很多bucket都没装满。解决办法就是开辟一个新bucket空间，将老bucket中的元素移动到新bucket，使得同一个bucket中的key排列地更紧密。这样，原来，在overflowbucket中的key可以移动到bucket中来。结果是节省空间，提高bucket利用率，map的查找和插入效率自然就会提升。
对于条件2的解决方案，曹大的博客里还提出了一个极端的情况：如果插入map的key哈希都一样，就会落到同一个bucket里，超过8个就会产生overflowbucket，结果也会造成overflowbucket数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了O(n)。
再来看一下扩容具体是怎么做的。由于map扩容需要将原有的key/value重新搬迁到新的内存地址，如果有大量的key/value需要搬迁，会非常影响性能。因此Gomap的扩容采取了一种称为“渐进式”地方式，原有的key并不会一次性搬迁完毕，每次最多只会搬迁2个bucket。上面说的hashGrow()函数实际上并没有真正地“搬迁”，它只是分配好了新的buckets，并将老的buckets挂到了oldbuckets字段上。真正搬迁buckets的动作在growWork()函数中，而调用growWork()函数的动作是在mapassign和mapdelete函数中。也就是插入或修改、删除key的时候，都会尝试进行搬迁buckets的工作。先检查oldbuckets是否搬迁完毕，具体来说就是检查oldbuckets是否为nil。


```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 获取key的hash计算函数
	alg := t.key.alg
	// 对key进行hash计算，h.hash0为hash种子，随机值
	hash := alg.hash(key, uintptr(h.hash0))
 
	// 把map写标志置1
	h.flags ^= hashWriting
	// h.buckets为nil时需要先new一个
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}
 
again:
	// 根据hash的后B位找到key对应那个桶
	bucket := hash & bucketMask(h.B)
	// 旧桶不为nil，说明正在进行扩容
	if h.growing() {
		growWork(t, h, bucket)
	}
	// 找到key对应的桶的地址
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	// 获取key的hash的高8位
	top := tophash(hash)
 
	// 记住key对应的tophash地址
	var inserti *uint8
	// key的地址
	var insertk unsafe.Pointer
	// value的地址
	var elem unsafe.Pointer
bucketloop:
	for {
		// 遍历桶的8个元素
		for i := uintptr(0); i < bucketCnt; i++ {
			// 高8位的hash值不相等时还要判断，tophash的标志
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					// 找到tophash对应的地址
					inserti = &b.tophash[i]
					// key的地址
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					// value的地址
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					// 此元素以及之后的元素都是空的从来没有使用过
					break bucketloop
				}
				continue
			}
			// 运行到这里说明找到key了
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			// 如果key是指针类型获取指针指向的内容
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 只根据hash的高8位还不够，要再次判断整个key的hash是否相等
			if !alg.equal(key, k) {
				continue
			}
			// key是否更新？？既然一样了为啥还要更新
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			// 获取value的地址
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		// 查询溢出桶
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
 
	// 当map没有正在扩容 && （负载因子大于6.5 || 溢出桶太多时） 就需要进行扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again
	}
 
	// 走到这里说明桶和溢出桶已经全部遍历完了，既没有找到key也没有找到空位值存放新的key并且还不满足map的扩容条件，
	// 这时候需要创建一个新的溢出桶
 
	if inserti == nil {
		// new一个新的溢出桶
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}
 
	// key是指针类型，需要给key分配一块内存
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	// value是指针类型，需要给value分配一块内存
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++
 
done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

### map元素的删除

从代码中可以看到 在删除元素时只是将该桶的tophash标记为emptyOne，并没有删除该元素后将其后的元素向前移动也没有对溢出桶做释放处理。另外还做了一个处理是保证了桶中最后一个元素的标记是emptyReset，表示此位置及其以后的元素全部是初始状态，在遍历读/写时遇到这种标志就不用再往后查找了提升了效率。


```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
 
	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting
 
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		// 可以看到删除也会触发将数据从旧桶到新桶的转移
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					// 删除的元素不存在
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !alg.equal(key, k2) {
				continue
			}
			// key是指针，将key设置为nil，以便gc回收key的内存
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			// value是指针，将其设置为nil，以便gc回收内存
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
			// 将对应地址的tophash设置为emptyOne，可见emptyOne表示此处以前有数据只不过现在被清零了，和emptyRest表示含义不同
			b.tophash[i] = emptyOne
 
			// 以下的处理主要是保证如果删除的元素是最后一个，那么该元素及之后的所有元素全部是emptyRest标志
			if i == bucketCnt-1 {
				// 如果i是该桶的最后一个元素，要判断是否存在溢出桶，并且溢出桶
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}
 
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```





参考:

Go语言之map：map的用法到map底层实现分析_后打开撒打发了的博客-CSDN博客

go-internals/02.3.md at master · sunkaimr/go-internals · GitHub



参考：https://www.helloworld.net/p/3714029944