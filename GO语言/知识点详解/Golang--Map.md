

map主要有两种实现：哈希查找表和搜索树（Search tree）

哈希查找表：查找效率高，会存在碰撞问题，链表法和开放地址法解决冲突

搜索树：AVL 树，红黑树，

### map底层实现

Go 语言采用的是**哈希查找表**，并且使用**链表**解决哈希冲突

##### 数据结构

```go
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
	count     int
	flags     uint8
	// buckets 的对数 log_2
	B         uint8
	// overflow 的 bucket 近似数
	noverflow uint16
	// 计算 key 的哈希的时候会传入哈希函数
	hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer
	// 等量扩容的时候，buckets 长度和 oldbuckets 相等
	// 双倍扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成，哈希在扩容时用于保存之前 buckets 的字段，它的大小是当前 buckets 的一半；
  
	nevacuate  uintptr
	extra *mapextra // 
}
```

buckets中是一个bmap数组，每一个元素中可以装8个bmap元素，当哈希表中存储的数据过多，单个桶已经装满时就会使用 `extra.nextOverflow` 中桶存储溢出的数据。分为正常桶和溢出桶，溢出桶的设计可以减少扩容的频率

<img src="assets/hmap-and-buckets.png" alt="hmap-and-buckets" style="zoom:45%;" />

```go
type bmap struct {
    topbits  [8]uint8//高八位
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

##### 创建map

```golang
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)
// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int
```

底层调用的是 `makemap` 函数，主要做的工作就是初始化 `hmap` 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等。

makemap主要工作是初始化map中的数据结构，构建数组，变量赋值等操作

1.   计算哈希占用的内存是否溢出或者超出能分配的最大值；
2.   调用 runtime.fastrand 获取一个随机的哈希种子；
3.   根据传入的 hint 计算出需要的最小需要的桶的数量；
4.   使用 runtime.makeBucketArray 创建用于保存桶的数组；





##### 哈希函数

在程序启动时，会检测 cpu 是否支持 `aes`，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 `alginit()` 中完成，位于路径：`src/runtime/alg.go` 下。

hash 函数，有加密型和非加密型。 加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种； 非加密型的一般就是查找。在 map 的应用场景中，用的是查找。 选择 hash 函数主要考察的是两点：<u>性能、碰撞概率</u>。

##### key定位过程

key经过hash函数得到64bit, 只会用到最后 B 个 bit 位来定位桶的位置，再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ……
	
	// 如果 h 什么都没有，返回零值
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
	// 写和读冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	
	// 不同类型 key 使用的 hash 算法在编译期确定
	alg := t.key.alg
	
	// 计算哈希值，并且加入 hash0 引入随机性
	hash := alg.hash(key, uintptr(h.hash0))
	
	// 比如 B=5，那 m 就是31，二进制是全 1
	// 求 bucket num 时，将 hash 与 m 相与，
	// 达到 bucket num 由 hash 的低 8 位决定的效果
	m := uintptr(1)<<h.B - 1
	
	// b 就是 bucket 的地址
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	
	// oldbuckets 不为 nil，说明发生了扩容
	if c := h.oldbuckets; c != nil {
	    // 如果不是同 size 扩容（看后面扩容的内容）
	    // 对应条件 1 的解决方案
		if !h.sameSizeGrow() {
			// 新 bucket 数量是老的 2 倍
			m >>= 1
		}
		
		// 求出 key 在老的 map 中的 bucket 位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		
		// 如果 oldb 没有搬迁到新的 bucket
		// 那就在老的 bucket 中寻找
		if !evacuated(oldb) {
			b = oldb
		}
	}
	
	// 计算出高 8 位的 hash
	// 相当于右移 56 位，只取高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	
	// 增加一个 minTopHash
	if top < minTopHash {
		top += minTopHash
	}
	for {
	    // 遍历 bucket 的 8 个位置
		for i := uintptr(0); i < bucketCnt; i++ {
		    // tophash 不匹配，继续
			if b.tophash[i] != top {
				continue
			}
			// tophash 匹配，定位到 key 的位置
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			// key 是指针
			if t.indirectkey {
			    // 解引用
				k = *((*unsafe.Pointer)(k))
			}
			// 如果 key 相等
			if alg.equal(key, k) {
			    // 定位到 value 的位置
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				// value 解引用
				if t.indirectvalue {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
		
		// bucket 找完（还没找到），继续到 overflow bucket 里找
		b = b.overflow(t)
		// overflow bucket 也找完了，说明没有目标 key
		// 返回零值
		if b == nil {
			return unsafe.Pointer(&zeroVal[0])
		}
	}
}
```

##### get操作

Go 语言中读取 map 有两种语法：<u>带 comma 和 不带 comma</u>。

当要查询的 key 不在 map 里，带 comma 的用法会返回一个 bool 型变量提示 key 是否在 map 中；

而不带 comma 的语句则会返回一个 key 对应 value 类型的零值。如果 value 是 int 型就会返回 0，如果 value 是 string 类型，就会返回空字符串。

##### 扩容操作

在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：loadFactor := count / (2^B)；

1.  装载因子超过阈值，源码里定义的阈值是 6.5。
2.  overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，<u>如果 overflow 的 bucket 数量超过 2^B</u>；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，<u>如果 overflow 的 bucket 数量超过 2^15</u>。

对于1情况：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）。

对于2情况：解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间

渐进式扩容：每次最多只会搬迁 2 个 bucket

具体扩容代码：TODO