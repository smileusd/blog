---
title: golang map 内存布局和原理
update: 2018-07-02 21:10:01
tags: golang
categories: coding
---

#### map数据结构

golang的map是hashmap实现的, 代码在/src/runtime/hashmap.go. 对比C++用红黑树实现的map，Go的map是unordered map，即无法对key值排序遍历。跟传统的hashmap的实现方法一样，它通过一个buckets数组实现，所有元素被hash到数组的bucket中，**buckets**就是指向了这个内存连续分配的数组. 

```go
type maptype struct {
	typ           _type
	key           *_type
	elem          *_type
	bucket        *_type // internal type representing a hash bucket
	hmap          *_type // internal type representing a hmap
	keysize       uint8  // size of key slot
	indirectkey   bool   // store ptr to key instead of key itself
	valuesize     uint8  // size of value slot
	indirectvalue bool   // store ptr to value instead of value itself
	bucketsize    uint16 // size of bucket
	reflexivekey  bool   // true if k==k for all keys
	needkeyupdate bool   // true if we need to update key on an overwrite
}

// A header for a Go map.
type hmap struct {
	count int // len()返回的map的大小 即有多少kv对
	flags uint8
	B     uint8  // 表示hash table总共有2^B个buckets 
	hash0 uint32 // hash seed

	buckets    unsafe.Pointer // 一系列桶的头指针, 按照low hash值可查找的连续分配的数组，初始时为16个Buckets.
	oldbuckets unsafe.Pointer 
	nevacuate  uintptr      

	overflow *[2]*[]*bmap //溢出链 当初始buckets都满了之后会使用overflow
}

// A bucket for a Go map.
type bmap struct {
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

如上所示, maptype表示map类型, 其中的hmap表示hashmap, 它的指针就是map的实体.  桶一个连续分配的数组组成, 而buckets是这个数组的头指针(起始地址), bmap是每个bucket的实体. 每个bucket有一个长度为8的数组叫做tophash, 他存储了8个key的高八位的值, 这样当我们找的时候, 先用key hash的低八位找到对应的桶, 再匹配key高8位值找到对应的tophash, 如果正确了再去找对应的key是否相等. 在对key/value对增删查的时候，先比较key的hash值高八位是否相等，然后再比较具体的key值。根据官方注释在tophash数组之后跟着8个key/value对，每一对都对应tophash当中的一条记录。最后bucket中还包含指向链表下一个bucket的指针。内存布局如下图。

![image](https://ninokop.github.io/2017/10/24/Go-Hashmap%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%92%8C%E5%AE%9E%E7%8E%B0/hashmap.png)

```
之所以把所有k1k2放一起而不是k1v1是因为key和value的数据类型内存大小可能差距很大，比如map[int64]int8，考虑到字节对齐，kv存在一起会浪费很多空间。
```



#### map创建和初始化

我们首先来看make过程, 看一个map是如何创建的.

```go
// makemap implements a Go map creation make(map[k]v, hint)
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If bucket != nil, bucket can be used as the first bucket.
// hint是map大小, h过不等于空就直接使用这个hmap, bucket不为空就当做第一个bucket.
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    if sz := unsafe.Sizeof(hmap{}); sz > 48 || sz != t.hmap.size {
		println("runtime: sizeof(hmap) =", sz, ", t.hmap.size =", t.hmap.size)
		throw("bad hmap size")
	}

    // hint值不符合规范, 置0
	if hint < 0 || hint > int64(maxSliceCap(t.bucket.size)) {
		hint = 0
	}
    // 按照是否实现hash()来判断是否支持map类型
	if !ismapkey(t.key) {
		throw("runtime.makemap: unsupported map key type")
	}

	// 检查key和value大小
	if t.key.size > maxKeySize && (!t.indirectkey || t.keysize != uint8(sys.PtrSize)) ||
		t.key.size <= maxKeySize && (t.indirectkey || t.keysize != uint8(t.key.size)) {
		throw("key size wrong")
	}
	if t.elem.size > maxValueSize && (!t.indirectvalue || t.valuesize != uint8(sys.PtrSize)) ||
		t.elem.size <= maxValueSize && (t.indirectvalue || t.valuesize != uint8(t.elem.size)) {
		throw("value size wrong")
	}
	// 检查各种编译的规范, 跳过
	// invariants we depend on. We should probably check these at compile time
	// somewhere, but for now we'll do it here.
	if t.key.align > bucketCnt {
		throw("key align too big")
	}
	if t.elem.align > bucketCnt {
		throw("value align too big")
	}
	if t.key.size%uintptr(t.key.align) != 0 {
		throw("key size not a multiple of key align")
	}
	if t.elem.size%uintptr(t.elem.align) != 0 {
		throw("value size not a multiple of value align")
	}
	if bucketCnt < 8 {
		throw("bucketsize too small for proper alignment")
	}
	if dataOffset%uintptr(t.key.align) != 0 {
		throw("need padding in bucket (key)")
	}
	if dataOffset%uintptr(t.elem.align) != 0 {
		throw("need padding in bucket (value)")
	}

    // 把hint参数对应成2进制的上确界那个数. 例如size=6, B就是3, 因为2^2 < 6 < 2^3
	B := uint8(0)
	for ; overLoadFactor(hint, B); B++ {
	}

    // 使用malloc分配2^B个buckets给h.
	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	buckets := bucket
	var extra *mapextra
	if B != 0 {
		var nextOverflow *bmap
		buckets, nextOverflow = makeBucketArray(t, B)
		if nextOverflow != nil {
			extra = new(mapextra)
			extra.nextOverflow = nextOverflow
		}
	}

	// initialize Hmap
	if h == nil {
		h = (*hmap)(newobject(t.hmap))
	}
	h.count = 0
	h.B = B
	h.extra = extra
	h.flags = 0
	h.hash0 = fastrand()
	h.buckets = buckets
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0

	return h
}
```



#### map存值

存储的步骤和第一部分的分析一致。首先用key的hash值低8位找到bucket，然后在bucket内部比对tophash和高8位与其对应的key值与入参key是否相等，若找到则更新这个值。若key不存在，则key优先存入在查找的过程中遇到的空的tophash数组位置。若当前的bucket已满则需要另外分配空间给这个key，新分配的bucket将挂在overflow链表后。

```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc(unsafe.Pointer(&t))
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write.
	h.flags |= hashWriting

	if h.buckets == nil {
		h.buckets = newarray(t.bucket, 1)
	}

again:
    // 用hash值的低八位找到bucket
	bucket := hash & (uintptr(1)<<h.B - 1)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    // 拿到高八位的hash值用于比对tophash
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	if top < minTopHash {
		top += minTopHash
	}

	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
            // 遍历tophash, 找到对应的key
			if b.tophash[i] != top {
                // 如果没有找到key, 说明是第一次, 将这个key插入到insertk(第i+1个key所在的)的位置, 将value插入到val(第i+1个value所在)的位置, 并且将tophash的当前地址赋值给inserti, 用于记录是否插入以及插入的位置.
				if b.tophash[i] == empty && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				continue
			}
            // 如果找到了key的tophash, 就将拿对应的key跟现在的key对比, 看是否相等, 如果不相等就跳过继续找key, 如果相等就更新它的value的值.
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
			if !alg.equal(key, k) {
				continue
			}
			// 如果需要更新key, 就覆盖key的值.
			if t.needkeyupdate {
				typedmemmove(t.key, k, key)
			}
            // 返回val的地址, 这个函数并不真正更新value, 只是找到value所在的地址.
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
        // 如这个bucket没找到, 找他的overflow链表, 拿下一个bucket循环操作.
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

    // 如果都没有找到对应的值, 就可能做两件事:
    // 1) 建立新的overflow, 然后把值加到这个overflow的bucket中.
    // 2) 如果此时map的len()超过了overLoadFactor(6.5默认值)*桶的数量(2^B, 每个桶最多8个kv), 或者overflow的bucket太多了, golang就会扩大map的容量. 
	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(int64(h.count), h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/value at insert position
	if t.indirectkey {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
    // 不支持并发map的写.
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectvalue {
		val = *((*unsafe.Pointer)(val))
	}
	return val
}
```

在往map中存值时若所有的bucket已满，需要在堆中new新的空间时需要计算是否需要扩容。扩容的时机是count > loadFactor(2^B)。这里的loadfactor选择为6.5。**扩容时机的物理意义的理解** 在没有溢出时hashmap总共可以存储8*(2^B)个KV对，当hashmap已经存储到6.5*(2^B)个KV对时表示hashmap已经趋于溢出，即很有可能在存值时用到overflow链表，这样会增加hitprobe和missprobe。为了使hashmap保持读取和超找的高性能，在hashmap快满时需要在新分配的bucket中重新hash元素并拷贝，源码中称之为evacuate。

> overflow溢出率是指平均一个bucket有多少个kv的时候会溢出。bytes/entry是指平均存一个kv需要额外存储多少字节的数据。hitprobe是指找一个存在的key平均需要找多少次。missprobe是指找一个不存在的key平均需要找多少次。选取6.5是为了平衡这组数据。

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
| ---------- | --------- | ----------- | -------- | --------- |
| 4.00       | 2.13      | 20.77       | 3.00     | 4.00      |
| 4.50       | 4.05      | 17.30       | 3.25     | 4.50      |
| 5.00       | 6.85      | 14.77       | 3.50     | 5.00      |
| 5.50       | 10.55     | 12.94       | 3.75     | 5.50      |
| 6.00       | 15.27     | 11.67       | 4.00     | 6.00      |
| 6.50       | 20.90     | 10.79       | 4.25     | 6.50      |
| 7.00       | 27.14     | 10.15       | 4.50     | 7.00      |
| 7.50       | 34.03     | 9.73        | 4.75     | 7.50      |
| 8.00       | 41.10     | 9.40        | 5.00     | 8.00      |

但这个迁移并没有在扩容之后一次性完成，而是逐步完成的，每一次insert或remove时迁移1到2个pair，即增量扩容。[增量扩容的原因](https://ninokop.github.io/2017/10/24/Go-Hashmap%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%92%8C%E5%AE%9E%E7%8E%B0/) 主要是缩短map容器的响应时间。若hashmap很大扩容时很容易导致系统停顿无响应。增量扩容本质上就是将总的扩容时间分摊到了每一次hash操作上。由于这个工作是逐渐完成的，导致数据一部分在old table中一部分在new table中。old的bucket不会删除，只是加上一个已删除的标记。只有当所有的bucket都从old table里迁移后才会将其释放掉。

#### Map的取值

map取值和存值前面的过程差不多, 代码在```mapaccess1```中, 这里不做赘述.

#### Map 的删除

删除的前面部分也是找到对应的key和value, 此处省略.

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
// 查找位置, 省略
	if t.indirectkey {
        // 如果存的是key的地址, 将地址变为nil
				*(*unsafe.Pointer)(k) = nil
			} else {
        // 如果存的是key的值, 清空key的值
				typedmemclr(t.key, k)
			}
			v := unsafe.Pointer(uintptr(unsafe.Pointer(b)) + dataOffset + bucketCnt*uintptr(t.keysize) + i*uintptr(t.valuesize))
    	// 同理
			if t.indirectvalue {
				*(*unsafe.Pointer)(v) = nil
			} else {
				typedmemclr(t.elem, v)
			}
    		//将tophash相应位置标记为空, kv数量-1.
			b.tophash[i] = empty
			h.count--
			goto done
	}
}
```

