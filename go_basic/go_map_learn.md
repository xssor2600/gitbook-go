
在源码中，表示 map 的结构体是 hmap，它是 hashmap,底层golang对`map`的实现主要是通过`hmap+maptype`运行的。

对golang底层源代码的理解学习，要习惯用动态的角度去考虑，因为golang具体底层运行是包括**编译器**的。

> 会有不少内建数据类型，在编译器期间，编译器会为了某种实现目的，会动态给源代码中数据结构进行加工处理。
>
> 在go的源代码中，出现很多地址类型，在看的时候，需要结合起始内存地址+偏移量的思维去理解。

##### Map的底层实现原理

Go runtime 中的map是如何实现的？**Go 中的 map 是一个 hashmap**

> Hashmap 是一种经典的数据结构，提供了平均 O(1) 的查询时间复杂度，即使在最糟的情况下也有 O(n) 的复杂度。也就是说，正常情况下，执行 map 函数的时间是个常量。
>
> 这个常量的大小部分取决于 hashmap 的设计方式，而 map 存取时间从 O(1) 到 O(n) 的变化则取决于它的 hash 函数。

Hashmap 的使用方面有两个重要的特点:

1. 第一个是稳定性。Hash 函数必须是稳定的。给定相同的 key，你的 hash 函数必须返回相同的值。
2. 第二个特点是良好的分布。给定两个相类似的 key，结果应该是极其不同的。hashmap 中的 value 值应当均匀地分布于 buckets 之间，否则存取的复杂度不会是 O(1).

- Hashmap数据结构

  经典的 hashmap 结构是一个 bucket 数组，其中的每项包含一个指针指向一个 key/value entry 数组。

  **golang底层的map结构是如何的？**

  底层数据结构也是一个bucket数组，并且每个 bucket 最多持有 8 个 key/value entry。使用 2 的次方便于做位掩码和移位，而不必做昂贵的除法操作。[宏观看map底层实现](https://mp.weixin.qq.com/s/j7_D0vj7ZpYgM5NOLkh35g)

  当一个`map` 操作被执行时会根据 key 的名字来生成一个散列 `key`，比如`（colors["Black"] = "#000000"）`会根据字符串`“ Black ”` 来生成散列 `key`，根据这个散列 `key`的低阶位`（LOB）`来选择放入哪个`bucket` 中（&011）。

  

  一旦确定了 `bucket`，那么就可以对键值对进行相应的操作，比如储存、删除或查找。如果我们观察 `bucket` 的内部，那么会发现两个结构体。首先是一个数组，它从之前用来选择 `bucket` 的散列 `key` 中获取`8`个高阶位`（HOB）`，这个数组区分了每一个被储存在`bucket` 中的键值对，然后是一个储存键值对内容的 `byte`数组，这个数组把键值对结合起来并储存到所在的 `bucket` 中。![image](https://s1.ax1x.com/2020/09/06/wmd00A.md.png)

  > 一旦 bucket 中的 entry 数量超过总数的某个百分比，也就是所说的 负载因子（load factor），那么 map 就会翻倍 bucket 的数量并重新分配原先的 entry。要实现一个 hashmap 有四个要点

  要实现一个 hashmap 有三个要点:

  1. 需要一个给 key 做计算的 hash 函数
  2. 需要一个判断 key 相等的算法
  3. 需要知道 key 的大小和value的大小（考虑到会影响bucket大小，对编译器处理分配内存影响）


  golang中对map的抽象数据结构是`hmap`，该结构体贯穿底层map的实现原理：

  ```go
  // Go map的hashmap结构
  type hmap struct {
  	count     int //size of map.
  	flags     uint8
  	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
  	noverflow uint16 
  	hash0     uint32 // hash seed
  
  	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
  	oldbuckets unsafe.Pointer 
  	nevacuate  uintptr       
  
  	extra *mapextra // optional fields
  }
  ```

  

- **bucket**

  `bucket`是map中最基础的存储key-value的元素。buckets 是一个指针，最终它指向的是一个结构体。这里会些许奇怪，为什么一个容器bucket中没有保存键值对key,value的描述？

  ```go
  // A bucket for a Go map.
  type bmap struct {
  	tophash [bucketCnt]uint8
  }
  
  	// Maximum number of key/elem pairs a bucket can hold.
  bucketCntBits = 3
  bucketCnt     = 1 << bucketCntBits
  ```

  可以看到，默认bucket中的`tophash`是数组大小为8的无符号整数（其实可以理解为8个字节）。uint8与byte可以说是一样的，因为文档中有这样的定义：

  ```go
  //The Go Programming Language Specification
  //Numeric types
  uint8       the set of all unsigned  8-bit integers (0 to 255)
  byte        alias for uint8
  ```

  但这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构：

  > 这种做法是因为 Go 语言在执行哈希的操作时会**直接操作内存空间**，与此同时**由于键值类型的不同占用的空间大小也不同**，也就导致了类型和占用的内存不确定，所以没有办法在 Go 语言的源代码中进行声明。

  此时的`bmap`才包含了key,value的描述，类似java中hashmap中的`entity`描述。

  ```go
  type bmap struct{
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
  }
  ```

  `bmap` 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）

  ![image](https://s1.ax1x.com/2020/09/06/wmdzA1.md.png)



- Go如何对map进行实现

  **需要编译器和 runtime （运行时）之间的相互协作。**可以看到编译器中定义的在初始化map时候调用的`makemap`方法，入参数是`maptype`，创建返回一个`hamp`。

  ```go
  
  	///usr/local/go/src/cmd/compile/internal/gc/builtin/runtime.go
  	func makemap(mapType *byte, hint int, mapbuf *any) (hmap map[any]any)
  
  	// /usr/local/go/src/runtime/map.go
  	func makemap(t *maptype, hint int, h *hmap) *hmap {
  	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
  	if overflow || mem > maxAlloc {
  		hint = 0
  	}
  
  	// initialize Hmap
  	if h == nil {
  		h = new(hmap)
  	}
  	h.hash0 = fastrand()
  
  	// Find the size parameter B which will hold the requested # of elements.
  	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
  	B := uint8(0)
  	for overLoadFactor(hint, B) {
  		B++
  	}
  	h.B = B
  
  	// allocate initial hash table
  	// if B == 0, the buckets field is allocated lazily later (in mapassign)
  	// If hint is large zeroing this memory could take a while.
  	if h.B != 0 {
  		var nextOverflow *bmap
  		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
  		if nextOverflow != nil {
  			h.extra = new(mapextra)
  			h.extra.nextOverflow = nextOverflow
  		}
  	}
  		return h
  	}
  ```

  1. 编译时间重写
     在编译期间 map 的操作被重写去调用了 runtime。

     ```go
     v := m["key"]     → runtime.mapaccess1(m, "key", &v)
     v, ok := m["key"] → runtime.mapaccess2(m, "key", &v, &ok)
     m["key"] = 9001   → runtime.mapinsert(m, "key", 9001)
     delete(m, "key")  → runtime.mapdelete(m, "key")
     ```

     > 值得注意的是，channel 中也做了相同的事，slice 却没有。
     > 这是因为 channel 是复杂的数据类型。发送，接收和 select 操作和调度器之间都有复杂的交互，所以就被委托给了 runtime。相比较而言，slice 就简单很多了。像 slice 的存取，len 和 cap 这些操作编译器就自己做了，而像 copy 和 append 这种复杂的还是委托给了 runtime。

  2. runtime运行时map 代码解释

     我们知道编译器重写了 map 的操作去调用了 runtime。我们也知道了在 runtime 内部，有一个叫 mapaccess1 的函数，一个叫 mapaccess2 的函数等等。

     先看下runtime中对map重写函数的定义：

     ```go
     func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
     
     // key 是指向你提供的作为 key 值的指针
     // h 是个指向 runtime.hmap 结构的指针。hmap 是一个持有 buckets 和其他一些值的 runtime 的 hashmap 结构。
     // t 是个指向 maptype 的指针
     ## /usr/local/go/src/runtime/type.go
     type maptype struct {
     	typ    _type
     	key    *_type
     	elem   *_type
     	bucket *_type // internal type representing a hash bucket
     	// function for hashing keys (ptr to key, seed) -> hash
       // hash函数，带着seed=>hmap.hash0
     	hasher     func(unsafe.Pointer, uintptr) uintptr 
     	keysize    uint8  // size of key slot
     	elemsize   uint8  // size of elem slot
     	bucketsize uint16 // size of bucket
     	flags      uint32
     }
     
     ## /usr/local/go/src/runtime
     // Go map的hashmap结构
     type hmap struct {
     	count     int // # live cells == size of map.  Must be first (used by len() builtin)
     	flags     uint8
     	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
     	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
     	hash0     uint32 // hash seed
     
     	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
     	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
     	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
     
     	extra *mapextra // optional fields
     }
     
     ```

     **为什么我们已经有了 *hmap 之后还需要一个 *maptype？**

     `*maptype` 是个特殊的类型，使得通用的 *hmap 可以服务于（几乎）任意 key 和 value 类型的组合。在你的程序中对于每一个独立的 map 定义都会有一个特定的 maptype 值。`(map  1:1 *maptype)`

     Go在编译期间创建了一个 maptype 并在调用 runtime 的 map 函数的时候使用了它。例如上面的runtime重写的方法`mapaccess1`。

     每个 maptype 中都包含了特定 map 中从 key 映射到 elem 所需的各种属性细节。它包含了关于 key 和 element 的信息。maptype.key 包含了指向我们传入的 key 的指针的信息。我们称之为 类型描述符。

     ```go
     type _type struct {
     	size       uintptr
     	ptrdata    uintptr // size of memory prefix holding all pointers
     	hash       uint32
     	tflag      tflag  // 代表key类型
     	align      uint8
     	fieldAlign uint8
     	kind       uint8
     	// function for comparing objects of this type
     	// (ptr to object A, ptr to object B) -> ==?
       // key类型是如何比较相等的函数算法
     	equal func(unsafe.Pointer, unsafe.Pointer) bool
     	// gcdata stores the GC type data for the garbage collector.
     	// If the KindGCProg bit is set in kind, gcdata is a GC program.
     	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
     	gcdata    *byte
     	str       nameOff
     	ptrToThis typeOff
     }
     ```

     在 _type 类型中，包含了它的大小`size`。这很重要，**因为我们只有一个指向 key 的指针，而不知道它实际多大并且是什么类型**。它到底是一个整数，还是一个结构体，等等。我们也需要知道如何比较这种类型的值和如何 hash 这种类型的值，这也就是` _type.tflag` 字段的意义所在。(用size大小可以来具体推测map中key的类型)

     ![image](https://s1.ax1x.com/2020/08/22/daGiAs.png)

     

     

- map的扩容机制

  一个 `bucket`被设定为只储存`8` 个键值对，当向一个已满的`bucket` 插入`key` 时，就会创建出一个新的`bucket` 和先前的`bucket`关联起来，并将`key` 加入到这个新的 `bucket` 中。

  ```go
  type hmap struct {
  	count     int // # live cells == size of map. 
  	flags     uint8
  	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
  	noverflow uint16 // approximate number of overflow buckets
  	hash0     uint32 // hash seed
  
  	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
  	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
  	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
  
  	extra *mapextra // optional fields
  }
  ```

  

  `hash map` 增长的时机由装载阈值`（load threshold values）`基于下面四个因素来确定：

  1. % overflow : 已满的 bucket 在所有 bucket 中的所占比例
  2. bytes/entry : 每个键值对的额外字节使用数量
  3. hitprobe  : 寻找一个 key 所需要检查的项数量
  4. missprobe  : 寻找一个不存在的 key 所需要检查的项数量


  `hash table` 在开始增长时会先将名叫 “ `old bucket` ” 的指针指向当前的 `bucket` 数组，然后会分配一个比原来 `bucket` 大两倍的新 `bucket` 数组，这可能会涉及到大量的内存分配，不过这些分配的内存并不会马上进行初始化。 


  当新的 `bucket` 数组内存可用时，旧的 `bucket` 数组中的键值对会被移动或者迁移到新的 `bucket` 数组中。迁移一般在 `map` 中的键值对增加或者删除时产生，在旧的 `bucket` 中作为一个整体的键值对可能会被移动到不同的新 bucket 数组中，迁移算法会让这些键值对均匀地分配。



#### Map内存和bucket溢出

一个 `bucket`被设定为只储存`8` 个键值对，当向一个已满的`bucket` 插入`key` 时，就会创建出一个新的`bucket` 和先前的`bucket`关联起来，并将`key` 加入到这个新的 `bucket` 中。

>  `B` 是 buckets 数组的长度的对数，也就是说 buckets 数组的长度就是 2^B

```go
type hmap struct {
....

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // map扩容之后关联的旧的bucket
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
}
```

hmap中重要的就是`buckets`，就是所谓的桶，在底层是通过`bmap`来描述定义：

```go
type bmap struct{
  topbits  [8]uint8
  keys     [8]keytype
  values   [8]valuetype
  pad      uintptr
  overflow uintptr // 保存超过8个相同hash(key)新创建的bucket地址
}
```

bmap 是存放 k-v 的地方，需要仔细看 bmap 的内部组成,**bmap内存模型:**

![image](https://s1.ax1x.com/2020/09/06/wmwaCV.md.png)

`HOBHash` 指的就是 top hash。注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 `overflow` 指针连接起来。



#### Map操作底层流程

- key的定位过程

  key 经过哈希计算后得到哈希值，共 64 个 bit 位，计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。

  例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：`10010111 | 000011110110110010001111001010100010010110010101010 │ 01010`,

  1. 用最后的 5 个 bit 位，也就是 `01010`，值为 10，也就是 10 号桶。
  2. 再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。
  3. buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

  ![image](https://s1.ax1x.com/2020/08/23/d0DbF0.png)

  上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 `00110`，找到对应的 6 号 bucket，使用高 8 位 `10010111`，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，这样整个查找过程就结束了。

  > 如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。
  >
  > hash(key) ：
  >
  > 1. 低8位，确定bucket索引
  > 2. 高8位，用于key冲突的value的存放位置的定位（连表法）

  ```go
  	// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
  	// it will return a reference to the zero object for the elem type if
  	// the key is not in the map.
  	// NOTE: The returned pointer may keep the whole map live, so don't
  	// hold onto it for very long.
  	func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  	if raceenabled && h != nil {
  		callerpc := getcallerpc()
  		pc := funcPC(mapaccess1)
  		racereadpc(unsafe.Pointer(h), callerpc, pc)
  		raceReadObjectPC(t.key, key, callerpc, pc)
  	}
  	if msanenabled && h != nil {
  		msanread(key, t.key.size)
  	}
  	if h == nil || h.count == 0 {
  		if t.hashMightPanic() {
  			t.hasher(key, 0) // see issue 23734
  		}
  		return unsafe.Pointer(&zeroVal[0])
  	}
      
    // 写和读冲突
  	if h.flags&hashWriting != 0 {
  		throw("concurrent map read and map write")
  	}
    // 计算key哈希值，并且加入 hash0 引入随机性
  	hash := t.hasher(key, uintptr(h.hash0))
    // 达到 bucket num 由 hash 的低 8 位决定的效果
    // 确定桶数量个数
  	m := bucketMask(h.B)
    // b 就是 bucket 的地址bmap
    // 先用(hash&m)找到bucket的索引index值，然后在内存地址h.buckets基础上计算地址偏移量
  	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // oldbuckets 不为 nil，说明发生了扩容
  	if c := h.oldbuckets; c != nil {
      //如果不是同 size 扩容,新 bucket 数量是老的 2 倍
  		if !h.sameSizeGrow() {
  			// There used to be half as many buckets; mask down one more power of two.
  			m >>= 1
  		}
      //  求出 key 在老的 map 中的 bucket 位置,起始内存地址+偏移量
  		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
      //如果 oldb 没有搬迁到新的 bucket,那就在老的 bucket 中寻找
  		if !evacuated(oldb) {
  			b = oldb
  		}
  	}
    // 计算出高 8 位的 hash值，用于在bucket中8个地址中索引
  	top := tophash(hash)
  	bucketloop:
    // 外层整个大循环：bucket 找完（还没找到），继续到 overflow bucket 里找
  	for ; b != nil; b = b.overflow(t) {
  		for i := uintptr(0); i < bucketCnt; i++ {
        // b就是bmap=> 一个桶，有属性topbits  [8]uint8
        // 存放8个tophash值
  			if b.tophash[i] != top {
  				if b.tophash[i] == emptyRest {
  					break bucketloop
  				}
  				continue
  			}
        // tophash 匹配，定位到 key 的位置
        // 定位位置是一个桶bmap的大小（起始偏移量）+ map中key类型的长度大小
        // 因为bmap中内存模型，key,value的存储是 k/k/k/v/v/v....
  			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
        // key 是指针,解引用
  			if t.indirectkey() {
  				k = *((*unsafe.Pointer)(k))
  			}
        // 如果 key 相等,bmap内存模型 k/k/k/v/v...
  			if t.key.equal(key, k) {
          // 定位到 value 的位置,从bmap起始位置+bucketCnt个key长度大小+第i个value大小偏移
  				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
          //value 解引用
  				if t.indirectelem() {
  					e = *((*unsafe.Pointer)(e))
  				}
  				return e
  				}
  			}
  		}
   	 //说明没有目标 key，返回零值
  		return unsafe.Pointer(&zeroVal[0])
  	}
  
  ```
  
  > uintptr是golang的内置类型，是能存储指针的整型，在64位平台上底层的数据类型是:
>
  > ```
  > typedef unsigned long long int  uint64;
  > typedef uint64          uintptr;
  > ```
  
  ```go
	// data offset should be the size of the bmap struct, but needs to be
  	// aligned correctly. For amd64p32 this means 64-bit alignment
  	// even though pointers are 32 bit.
  	dataOffset = unsafe.Offsetof(struct {
  		b bmap
  		v int64
  	}{}.v)
  ```
  
  dataOffset 是 key 相对于 bmap 起始地址的偏移,因此 bucket 里 key 的起始地址就是` unsafe.Pointer(b)+dataOffset`。**内存中：bmap大小长度+起始key地址**，第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。

  

  **外层是一个无限循环overflow:**

  ```go
for ; b != nil; b = b.overflow(t) {
    ....
  }
  
  // 单个桶定义
  type bmap struct{
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr // 保存超过8个相同hash(key)新创建的bucket地址
  }
  ```
  
  遍历所有的 bucket，这相当于是一个 bucket 链表，因为bmap中包含一个指向其他bucket的指针地址。

  当定位到一个具体的 bucket 时，里层循环就是遍历这个 bucket 里所有的 cell，或者说所有的槽位，也就是 bucketCnt=8 个槽位。外层循环就是遍历所有的bucket。

  ![image](https://s1.ax1x.com/2020/09/06/wmwsb9.md.png)

  

  - minTopHash

    当一个 cell 的 tophash 值小于 minTopHash 时，标志这个 cell 的迁移状态。因为这个状态值是放在 tophash 数组里，为了和正常的哈希值区分开，会给 key 计算出来的哈希值一个增量：minTopHash。这样就能区分正常的 top hash 值和表示状态的哈希值。

    ```go
  // entries in the evacuated* states (except during the evacuate() method, which only happens
    	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
    	emptyOne       = 1 // this cell is empty
    	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
    	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
    	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
    	minTopHash     = 5 // minimum tophash for a normal filled cell. 
    ```
  
    源码里判断这个 bucket 是否已经搬迁完毕，用到的函数:

    ```go
  func evacuated(b *bmap) bool {
    	h := b.tophash[0]
    	return h > emptyOne && h < minTopHash
    }
    ```
  
    只取了 tophash 数组的第一个值，判断它是否在 0-4 之间。对比上面的常量，当 top hash 是 `evacuatedEmpty`、 `evacuatedX`、 `evacuatedY` 这三个值之一，说明此 bucket 中的 key 全部被搬迁到了新 bucket。



#### Map底层扩容原理

随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。最理想的情况是一个 bucket 只装一个 key，这样，就能达到 `O(1)` 的效率，但这样空间消耗太大，用空间换时间的代价太高。

go底层的也是通过连表法处理冲突hash key,最坏情况是：所有的 key 都落在了同一个 bucket 里，直接退化成了链表，各种操作的效率直接降为 O(n)，是不行的。

所以，需要有一套机制去检测或者减少这种退化连表的情景。

有一个指标来衡量前面描述的情况，这就是 `装载因子`。Go 源码里这样定义 `装载因子`：

```go
loadFactor :=count /(2^B)
//count 就是 map 的元素个数，2^B 表示 bucket 数量。一个bucket8个cell
```

**有了衡量指标，那么什么时候map会进行扩容？也就是增加bucket？增加后的bucket和新的bucket有什么联系？**

在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

1. 装载因子超过阈值，源码里定义的阈值是 6.5  （6.5 *8 = 52）

2. overflow 的 bucket 数量过多：

   a. 当 B < 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B.

   b. 当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15.

```go
	// Maximum average load of a bucket that triggers growth is 6.5.
	// Represent as loadFactorNum/loadFactDen, to allow integer math.
	loadFactorNum = 13
	loadFactorDen = 2


//src/runtime/hashmap.go/mapassign	
// If we hit the max load factor or we have too many overflow buckets,
// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}


func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}


// 装载因子超过 6.5
// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}


// overflow buckets 太多
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}
```

针对上述两点的说明：

**第1点：**我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。(loadFactor :=count /(2^B) => loadFactor=8*8/2^3 = 8 )

**第2点：**是对第 1 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）

**什么处理操作情况下会出现上面的问题？**

不难想像造成这种情况的原因：不停地插入、删除元素。overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人。（在B不小的情况下，不断插入数据，但是一直未到达负载因子，然后又不断删除元素；又或者在没达到负载因子比例下，大量的key冲突，导致一个bucket中8个cell装不下，使用新的bucket，创建大量overflow bucket）

针对不同的情况，map底层的扩容策略也不会相同！

- 对于条件 1，**元素太多**，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）
- 对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。（将疏松的map压缩更紧凑）

**Go map扩容旧bucket元素迁移？**

由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为**“渐进式”**地方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

`hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。

```go
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h) // 只是分配了新buckets地址，没有真正进行元素迁移
		goto again // Growing the table invalidates everything, so try again
	}


func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets // 将当前bucket地址分配给旧bucket地址
  // 分配新buckets地址，buckets容量为2^(B+1)
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
  
  // 重新复制hmap的属性值
	// commit the grow (atomic wrt gc)
	h.B += bigger // 1<<2 B增加了1
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets   // 将新buckets地址分配给当前bucket,但是元素还未迁移
	h.nevacuate = 0  // 标志 nevacuate 被置为 0， 表示当前搬迁进度为 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}
  // 真正数据拷贝处理放在growWork函数中处理
	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}


```

**如何检查确认 oldbuckets 是否搬迁完毕？**具体来说就是检查 oldbuckets 是否为 nil。

```go
  // 如果oldbuckets不是nil,则表明旧元素没有迁移完成到新buckets
	if h.growing() {
		growWork(t, h, bucket)
	}

func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}
```

看真正执行搬迁工作的 growWork() 函数:

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 确认搬迁的旧的bucket到我们当前使用的bucket中
	evacuate(t, h, bucket&h.oldbucketmask())

	// 再搬迁一个 bucket，以加快搬迁进程
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

`bucket&h.oldbucketmask()` 这行代码，如源码注释里说的，是为了确认搬迁的 bucket 是我们正在使用的 bucket。`oldbucketmask()` 函数返回扩容前的 map 的 bucketmask。

**什么是bucketmask？**

所谓的 bucketmask，作用就是将 key 计算出来的哈希值与 bucketmask 相与，得到的结果就是 key 应该落入的桶。比如 B = 5，那么 bucketmask 的低 5 位是 `11111`，其余位是 `0`，hash 值与其相与的意思是，只有 hash 值的低 5 位决策 key 到底落入哪个 bucket。

hashmap扩容过程方法`evacuate`:

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 定位老的 bucket 地址,起始地址+n个buckets长度
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // 结果是 2^B，如 B = 5，结果为32，计算新bucket个数
	newbit := h.noldbuckets()
  // 如果 b 没有被搬迁过
	if !evacuated(b) {
		// xy contains the x and y (low and high) evacuation destinations.
    // evacDst is an evacuation destination.
    // type evacDst struct {
		// b *bmap          // current destination bucket
		// i int            // key/elem index into b
		// k unsafe.Pointer // pointer to current key storage
		// e unsafe.Pointer // pointer to current elem storage
	  // }
    // bucket移动的目标地址
		var xy [2]evacDst // 用两个新目的bucket分别处理等size和非等size扩容
		x := &xy[0]  // 使用 x 来进行搬迁
    //默认是等 size 扩容，前后 bucket 序号不变
    // x.b为目标地址x的新bucket起始地址+旧bucket长度地址
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
    // 注意bucket中key/elem内存模型是 KKKK/VVVV
    // 根据新bucket x起始内存地址+一个int64长度偏移量，得到key存储起始地址
		x.k = add(unsafe.Pointer(x.b), dataOffset)
    // KKKK/VVVV，elem的起始内存地址就是key起始地址+n个key长度的偏移量
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))
		
    // 如果不是等 size 扩容，前后 bucket 序号有变
    // 使用第二个y目标bucket地址处理迁移
		if !h.sameSizeGrow() {
			y := &xy[1]
      // y中新bucket地址 序号增加了 2^B
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

    // 遍历所有的 bucket，包括 overflow buckets
    // b 是老的 bucket 地址
		for ; b != nil; b = b.overflow(t) {
      // 获取旧bucket中key/element内存地址
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
      
			// 遍历 bucket 中的所有 cell
      // EEE/VVV每次迭代一个key和element偏移量
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
        // 当前 cell 的 top hash 值
				top := b.tophash[i]
        //如果 cell 为空，即没有 key
				if isEmpty(top) {
					// 那就标志它被"搬迁"过
					b.tophash[i] = evacuatedEmpty
					continue
				}
        // 正常不会出现这种情况 ，未被搬迁的 cell 只可能是 empty 
        // 或是正常的 top hash（大于 minTopHash）
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
        // 如果 key 是指针，则解引用
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
        //  如果不是等量扩容
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
          // 计算 hash 值，和 key 第一次写入时一样，根据hash计算结果判断是否能等容迁移
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible（可复制）. Reproducibility（还原性） is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
            // 使用tophash的低位字节位来决定该key的迁移方向，是在原bucket还是扩容新bucket
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
            // 第 B 位置 1
						useY = top & 1
            // 取hash值高 8 位作为 top hash 值
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}
				// evacuatedX     = 2 // key/elem is valid.
        // evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
        //老的 cell 的 top hash 值赋值
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
        // 根据useY=0或者等于1来判定新bucket是等size扩容还是非等size扩容
				dst := &xy[useY]                
				
        // key/elem index into b
        // 新目标bucket的cell个数等于8，说明要溢出了
				if dst.i == bucketCnt {
          // 新建一个 bucket
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0  // 重置cell索引位
          // 表示 key 要移动到的新内存地址位置
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
          // 表示value相对于key的偏移量=value起始内存地址
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
        
				// 设置 top hash 值
				dst.b.tophash[dst.i&(bucketCnt-1)] = top 
        //  key 是指针
				if t.indirectkey() {
          // 将原 key（是指针）复制到新位置 （unsafe.Pointer指针内存刷新）
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
           // 将原 key（是值）复制到新位置
					typedmemmove(t.key, dst.k, k) // copy elem
				}
         //  value 是指针，操作同 key
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
        // 定位到下一个 cell
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
    // 如果没有协程在使用老的 buckets，就把老 buckets 清除掉，帮助gc
		// Unlink the overflow buckets & clear key/elem to help GC.
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
      // 计算旧bucket内存起始地址
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
      // 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

  // 更新搬迁进度
  // 如果此次搬迁的 bucket 等于当前进度
	if oldbucket == h.nevacuate {
    // 标记处理迁移进度
		advanceEvacuationMark(h, t, newbit)
	}
}
```

搬迁的目的就是将老的 buckets 搬迁到新的 buckets。而通过前面的说明我们知道，应对条件 1（**元素太多**，而 bucket 数量太少），新的 buckets 数量是之前的一倍，应对条件 2（元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满），新的 buckets 数量和之前相等。

对于条件 1，从老的 buckets 搬迁到新的 buckets，由于 bucktes 数量不变，因此可以按序号来搬，比如原来在 0 号 bucktes，到新的地方后，仍然放在 0 号 buckets。

对于条件 2，就没这么简单了。要重新计算 key 的哈希，才能决定它到底落在哪个 bucket。例如，原来 B = 5，计算出 key 的哈希后，只用看它的低 5 位，就能决定它落在哪个 bucket。扩容后，B 变成了 6，因此需要多看一位，它的低 6 位决定 key 落在哪个 bucket。这称为 `rehash`。

![image](https://s1.ax1x.com/2020/09/06/wm0P5q.md.png)

**因此，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于 hash 值 第 6 bit 位是 0 还是 1。**



- 为什么遍历 map 是无序的？

  map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 2^B）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

  当然，Go 做得更绝，当我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。


> evacuate 函数每次只完成一个 bucket 的搬迁工作，因此要遍历完此 bucket 的所有的 cell，将有值的 cell copy 到新的地方。bucket 还会链接 overflow bucket，它们同样需要搬迁。因此会有 2 层循环，外层遍历 bucket 和 overflow bucket，内层遍历 bucket 的所有 cell。这样的循环在 map 的源码里到处都是，要理解透了。

源码里提到 X, Y part，其实就是我们说的如果是扩容到原来的 2 倍，桶的数量是原来的 2 倍，前一半桶被称为 X part，后一半桶被称为 Y part。一个 bucket 中的 key 可能会分裂落到 2 个桶，一个位于 X part，一个位于 Y part。所以在搬迁一个 cell 之前，需要知道这个 cell 中的 key 是落到哪个 Part。很简单，重新计算 cell 中 key 的 hash，并向前“多看”一位，决定落入哪个 Part，这个前面也说得很详细了。

![image](https://s1.ax1x.com/2020/09/06/wm0ZMF.md.png)

假设触发了 2 倍的扩容，那么扩容完成后，老 buckets 中的 key 分裂到了 2 个 新的 bucket。一个在 x part，一个在 y 的 part。依据是 hash 的 lowbits。新 map 中 `0-3`称为 x part， `4-7` 称为 y part。

> 上面描述的hash是重新计算 cell 中 key 的 hash，并向前“多看”一位，决定落入哪个 Part。
>
> 当没有扩容时候，同一个bucket中hash值的低B位是一样的，因为都低B位都索引到相同bucket index。
>
> 当扩容的时候，在进行hash值时候，有可能是获取hash的低（B+1），那么该key所在bucket index就有可能不一样，当B+1位是0时候，表示旧bucket index不变，但是当B+1位是1时候时候，那么旧bucke index就要改变迁移，是相比原来[index]加上 2^B（原来的 B 值）。
>
> 每个key的hash值并不是完全一样相等的，是一类hash，表示该类hash的低B位值相同。
>
> 					// Compute hash to make our evacuation decision (whether we need
> 					// to send this key/elem to bucket x or bucket y).
> 					hash := t.hasher(k2, uintptr(h.hash0))



##### Go Map遍历元素

本来 map 的遍历过程比较简单：遍历所有的 bucket 以及它后面挂的 overflow bucket，然后挨个遍历 bucket 中的所有 cell。每个 bucket 中包含 8 个 cell，从有 key 的 cell 中取出 key 和 value，这个过程就完成了。

是什么让map的遍历元素复杂度变高了？

扩容过程不是一个原子的操作，它每次最多只搬运 2 个 bucket，所以如果触发了扩容操作，那么在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。

**因此，遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程，这是难点所在。**

```go
func main() {
	ageMp:= make(map[string]int)
	ageMp["qcrao"] = 18
	for name,age := range ageMp {
		fmt.Println(name,age)
	}
}

// go tool comile -S main.go
// ....
0x012400292(test16.go:9) CALL runtime. mapiterinit(SB)
// ...
0x01fb00507(test16.go:9) CALL runtime. mapiternext(SB)
0x020000512(test16.go:9) MOVO ""..autotmp_4+160(SP), AX
0x020800520(test16.go:9) TESTO AX, AX
0x020b00523(test16.go:9) JNE 302
//....
```

关于 map 迭代，底层的函数调用关系一目了然。先是调用 `mapiterinit` 函数初始化迭代器，然后循环调用 `mapiternext` 函数进行 map 迭代。

```go
// A hash iteration structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end 
	elem        unsafe.Pointer // Must be in second position 
	t           *maptype  // map 类型，包含如 key size 大小等
	h           *hmap
	buckets     unsafe.Pointer // 初始化时指向的 bucket
	bptr        *bmap          //  当前遍历到的 bmap
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // 起始遍历的 bucet 索引编号
  offset      uint8          // 遍历开始时 cell 的编号（每个 bucket 中有 8 个 cell)
	wrapped     bool           // 是否从头遍历了
	B           uint8
	i           uint8   // 指示当前 cell 序号
	bucket      uintptr  // 指向当前的 bucket
	checkBucket uintptr  // 因为扩容，需要检查的 bucket
}
```

`mapiterinit` 就是对 hiter 结构体里的字段进行初始化赋值操作。

```go
// mapiterinit initializes the hiter struct used for ranging over maps.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
 ...

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
  // 生成随机数 r, map的遍历是随机扫描bucket的，不是都收默认从index 0开始
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
  //B = 2，那 bucketMask= uintptr(1)<<h.B-1 结果就是 3
  // 低 8 位为 00000011，将 r 与之相与，就可以得到一个 0~3 的 bucket 序号
  // 获取bucket索引位
	it.startBucket = r & bucketMask(h.B)
  //bucketCnt - 1 等于 7，低 8 位为 00000111
  //将 r 右移 2 位后，与 7 相与，就可以得到一个 0~7 号的 cell
	it.offset = uint8(r >> h.B & (bucketCnt - 1))
  //于是，在 mapiternext 函数中就会从 it.startBucket 的 it.offset 号的 cell 开始遍历，
  //取出其中的 key 和 value，直到又回到起点 bucket，完成遍历过程。

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

```



假设我们有下图所示的一个 map，起始时 B = 1，有两个 bucket，后来触发了扩容（这里不要深究扩容条件，只是一个设定），B 变成 2。并且， 1 号 bucket 中的内容搬迁到了新的 bucket， `1号`裂变成 `1号`和 `3号`；`0号` bucket 暂未搬迁。老的 bucket 挂在在 `*oldbuckets` 指针上面，新的 bucket 则挂在 `*buckets` 指针上面。

![image](https://s1.ax1x.com/2020/09/06/weqUzT.md.png)

这时，我们对此 map 进行遍历。假设经过初始化后，startBucket = 3，offset = 2。于是，遍历的起点将是 3 号 bucket 的 2 号 cell，下面这张图就是开始遍历时的状态：

![image](https://s1.ax1x.com/2020/09/06/wm0MI1.md.png)

**标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2。**

因为 3 号 bucket 对应老的 1 号 bucket，因此先检查老 1 号 bucket 是否已经被搬迁过。判断方法就是：

```go
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}
```

如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。

```go
	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
	emptyOne       = 1 // this cell is empty
	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash     = 5 // minimum tophash for a normal filled cell.
```

老 1 号 bucket 已经被搬迁过了。所以它的 tophash[0] 值在 (0,4) 范围内，因此只用遍历新的 3 号 bucket。

依次遍历 3 号 bucket 的 cell，这时候会找到第一个非空的 key：元素 e。到这里，mapiternext 函数返回，这时我们的遍历结果仅有一个元素：`key e,lowbits:11`.

由于返回的 key 不为空，所以会继续调用 mapiternext 函数。

继续从上次遍历到的地方往后遍历，从新 3 号 overflow bucket 中找到了元素 f 和 元素 g。

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。**那是不是把老 0 号 bucket 中的所有 key 都取出来呢？**

> 并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 `*bucket` 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。

因此， `lowbits==00` 的将进入遍历结果集：

![image](https://s1.ax1x.com/2020/09/06/wm08xO.md.png)

和之前的流程一样，继续遍历新 1 号 bucket，发现老 1 号 bucket 已经搬迁，只用遍历新 1 号 bucket 中现有的元素就可以了。结果集变成将`key h ,lowbit:01`追加到结果集中。

继续遍历新 2 号 bucket，它来自老 0 号 bucket，因此需要在老 0 号 bucket 中那些会裂变到新 2 号 bucket 中的 key，也就是 `lowbit==10` 的那些 key。

顺便说一下，如果碰到 key 是 `math.NaN()` 这种的，处理方式类似。核心还是要看它被分裂后具体落入哪个 bucket。只不过只用看它 top hash 的最低位。如果 top hash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。据此决定是否取出 key，放到遍历结果集里。

> map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。



##### Map的赋值

通过汇编语言可以看到，向 map 中插入或者修改 key，最终调用的是 `mapassign` 函数。

mapassign 有一个系列的函数，根据 key 类型的不同，编译器会将其优化为相应的“快速函数”。

整体来看，流程非常得简单：对 key 计算 hash 值，根据 hash 值按照之前的流程，找到要赋值的位置（可能是插入新 key，也可能是更新老 key），对相应位置进行赋值。

源码大体和之前讲的类似，核心还是一个双层循环，外层遍历 bucket 和它的 overflow bucket，内层遍历整个 bucket 的各个 cell。

```go
type hmap struct {
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed
  ....
}
```

函数首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。这也说明了**map 对协程是不安全的**。

通过前文我们知道扩容是渐进式的，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。

现在到了定位 key 应该放置的位置了，所谓找准自己的位置很重要。准备两个指针，一个（ `inserti`）指向 key 的 hash 值在 tophash 数组所处的位置，另一个( `insertk`)指向 cell 的位置（也就是 key 最终放置的地址），当然，对应 value 的位置就很容易定位出来了。这三者实际上都是关联的，在 tophash 数组中的索引位置决定了 key 在整个 bucket 中的位置（共 8 个 key），而 value 的位置需要“跨过” 8 个 key 的长度。

在循环的过程中，inserti 和 insertk 分别指向第一个找到的空闲的 cell。如果之后在 map 没有找到 key 的存在，也就是说原来 map 中没有此 key，这意味着插入新 key。那最终 key 的安置地址就是第一次发现的“空位”（tophash 是 empty）。

如果这个 bucket 的 8 个 key 都已经放置满了，那在跳出循环后，发现 inserti 和 insertk 都是空，这时候需要在 bucket 后面挂上 overflow bucket。当然，也有可能是在 overflow bucket 后面再挂上一个 overflow bucket。这就说明，太多 key hash 到了此 bucket。

在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。整个之前的查找定位 key 的过程，还得再重新走一次。因为扩容之后，key 的分布都发生了变化。

如果是插入新 key，map 的元素数量字段 count 值会加 1；在函数之初设置的 `hashWriting` 写标志出会清零。

有一个重要的点要说一下。前面说的找到 key 的位置，进行赋值操作，实际上并不准确。我们看 `mapassign` 函数的原型就知道，函数并没有传入 value 值，所以赋值操作是什么时候执行的呢？

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer 
```

**`mapassign` 函数返回的指针就是指向的 key 所对应的 value 值位置，有了地址，就很好操作赋值了.**



##### Map 删除元素

写操作底层的执行函数是 `mapdelete`：

```
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer)
```

根据 key 类型的不同，删除操作会被优化成更具体的函数。

`mapdelete` 函数。它首先会检查 h.flags 标志，如果发现写标位是 1，直接 panic，因为这表明有其他协程同时在进行写操作。

计算 key 的哈希，找到落入的 bucket。检查此 map 如果正在扩容的过程中，直接触发一次搬迁操作。

```go
			// Only clear key if there are pointers in it.
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
b.tophash[i] = emptyOne
```

最后，将 count 值减 1，将对应位置的 tophash 值置成 `Empty`。



参考：[https://mp.weixin.qq.com/s/Jq65sSHTX-ucSG8TlI5Zxg](https://mp.weixin.qq.com/s/Jq65sSHTX-ucSG8TlI5Zxg)




