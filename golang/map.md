# Map

<img src="./image/14.jpg" alt="14" />    

## 底层数据结构

```go
type hmap struct {
 // 数据个数
 count     int // # live cells == size of map.  Must be first (used by len() builtin)
 flags     uint8
 B         uint8  // log_2 桶的数量 = 2^B
 noverflow uint16 // 溢出的桶的近似数
 hash0     uint32 // hash seed

 buckets    unsafe.Pointer // 底层数据数组指针  指向bmap[]
 oldbuckets unsafe.Pointer // 旧的数组指针 扩容时才不为nil
 nevacuate  uintptr        // 扩容的搬迁进度

 extra *mapextra // optional fields
}

//只有在map被标记为不包含指针才会使用该结构
type mapextra struct {
 overflow    *[]*bmap    //溢出指针
 oldoverflow *[]*bmap
 nextOverflow *bmap
}

//源码中只包含一个字段，实际编码完成后会增加更多属性
type bmap struct {
 tophash [bucketCnt]uint8
}

type bmap struct {
   topbits  [8]uint8         //key hash高8位的值，通过比较该值判断是否存在key
   keys     [8]keytype         //key和value分开存放
   values   [8]valuetype
   pad      uintptr
   overflow uintptr         //每个桶只能存放8个数据，超出新建新的桶，该字段指向溢出的桶
}

```

* 当key和value都不包含指针且key和value的大小都小于128字节，会将该map标记为不含指针，在gc回收时不需要扫描整个map(不包含指针，不需要继续向下标记)，因为bmap结构中包含由overflow溢出指针，所以将该字段转移到mapextra结构中存放。
* topbits用于存放key的hash的前8位值
<img src="./image/15.jpg" alt="15" />    
* tophash的特殊值

```go
// 空的 cell，也是初始时 bucket 的状态
empty          = 0
// 空的 cell，表示 cell 已经被迁移到新的 bucket
evacuatedEmpty = 1
// key,value 已经搬迁完毕，但是 key 都在新 bucket 前半部分，
// 后面扩容部分会再讲到。
evacuatedX     = 2
// 同上，key 在后半部分
evacuatedY     = 3
// tophash 的最小正常值
minTopHash     = 4
```

* 根据 key 的不同类型，编译器还会将查找、插入、删除的函数用更具体的函数替换，以优化效率：
  * mapaccess1_fast32(t *maptype, h*hmap, key uint32) unsafe.Pointer
  * mapaccess2_fast32(t *maptype, h*hmap, key uint32) (unsafe.Pointer, bool)
  * mapaccess1_faststr(t *maptype, h*hmap, ky string) unsafe.Pointer
  * mapaccess2_faststr(t *maptype, h*hmap, ky string) (unsafe.Pointer, bool)

## 扩容
### 扩容条件
* 装载因子=map中元素的个数/map的容量 > 负载因子(默认6.5)
* 溢出桶的数量过多   
  map的容量 = 2^B   
    * 当 B < 15，溢出桶的数量大于 2^B   
    * 当 B >= 15，溢出桶的数量大于 2^15

```go
  //在插入后判断是否需要扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		//...
	}

  loadFactorDen = 2
  loadFactorNum = 13
  //map中元素的数量大于 bucketCnt => 相当于至少有一个桶放了两个元素的情况下才发起扩容，也就是说已经有hash冲突
  //bucketShift(B) => 1<B
  //  count / bucketShift(B) > 13 / 2
  func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// B > 15 B = 15 (B&15) = 15
// B <= 15 (B&15) = B
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	if B > 15 {
		B = 15
	}
	return noverflow >= uint16(1)<<(B&15)
}
```
### 扩容过程
扩容采用渐进式的方式，扩容操作只是初始化了新的 bucket 和一些变量，真正的迁移过程平摊到每一次的 插入 和 删除 的操作中，可以减少扩容带来的阻塞。触发不同扩容条件采用不同的策略：   
* 装载因子大于负载因子 => B+1
* 溢出桶的数量过多，这种情况下本身存在的元素不多，只是元素分散到各个溢出桶中，查找的效率很低，这种情况下不需要扩展map的容量，需要对元素进行整理，减少溢出桶的数量，提高查找插入的效率 => B不变
```
对于 溢出桶的数量过多 的解决方案，曹大的博客里还提出了一个极端的情况：如果插入 map 的 key 哈希都一样，就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 O(n)。
```
```go
  //扩容初始化操作
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
  //如果不是装载因子大于负载因子，不需要扩容，设置sameSize的标志
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

  /*
  段对 flags 一顿操作的代码的意思是：先把 h.flags 中 iterator 和 oldIterator 对应位清 0，然后如果发现 iterator 位为 1，那就把它转接到 oldIterator 位，使得 oldIterator 标志位变成 1。潜台词就是：buckets 现在挂到了 oldBuckets 名下了，对应的标志位也转接过去吧。
  */
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
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
}
```
```go
	if h.growing() {
		growWork(t, h, bucket)
	}
  //一次搬迁两个
  func growWork(t *maptype, h *hmap, bucket uintptr) {
  //搬迁需要操作的bucket
	evacuate(t, h, bucket&h.oldbucketmask())

  //按照扩容进度再搬迁一个
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

type evacDst struct {
	b *bmap          // 当前操作的bucket
	i int            // 当前插入的cell的位置
	k unsafe.Pointer // 指向下一个要插入的key位置指针
	e unsafe.Pointer // 下一个value要插入的位置指针
}
/*
  oldbucket 扩容的进度 桶的位置
*/
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  //定位需要迁移的旧的桶的指针
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
  //判断该桶是否已经搬迁
	if !evacuated(b) {
		// x -> 元素要搬迁到的low低位
    // y -> 元素要搬迁到的high高位
		var xy [2]evacDst
		x := &xy[0]
    //定位low的bucket
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))
    //如果长度扩展 需要定位high的bucket
		if !h.sameSizeGrow() {
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}
    //循环桶->溢出桶
		for ; b != nil; b = b.overflow(t) {
      //旧的桶第一个需要迁移的cell
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
      //遍历各个cell
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
				top := b.tophash[i]
        //空的cell 标记已迁移
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
        //key存放的是指针 拿出指针指向的数据
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
        //标志变量 是否迁移到high位
				var useY uint8
        //是否扩展了长度 没有扩展的话 只是整理数据->useY=0 放在原来的位置
				if !h.sameSizeGrow() {
					hash := t.hasher(k2, uintptr(h.hash0))
          	/*
              有一个特殊情况是：有一种 key，每次对它计算 hash，得到的结果都不一样。这个 key 就是 math.NaN() 的结果，当搬迁碰到 math.NaN() 的 key 时，只通过 tophash 的最低位决定分配到 X part 还是 Y part（如果扩容后是原来 buckets 数量的 2 倍）。如果 tophash 的最低位是 0 ，分配到 X part；如果是 1 ，则分配到 Y part。
            */
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						useY = top & 1
            //更新tophash
						top = tophash(hash)
					} else {
            //判断扩容后最高一位是否为1 为1说明要迁入high 否则low
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

        //新的桶的数据满了 需要开辟一个新的来存数据
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
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
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
      // 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}
  // 更新搬迁进度
  // 如果此次搬迁的 bucket 等于当前进度
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```



## 查找
若map在扩容，旧的桶迁移搜索新的bucket，没迁移搜索旧的bucket，然后根据hash值定位桶的位置，根据hash的前八位比对tophash，若相同，定位key的位置，调用equals方法比较，相同定位value返回，若不同则继续下一轮比较，若当前节点搜索完毕，则沿着溢出桶向下搜索。
```go
/*
    两种查询模式
    value := map[key]->mapaccess1
    value,ok := map[key]->mapaccess2
*/
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  //...
 //并发操作
 if h.flags&hashWriting != 0 {
  throw("concurrent map read and map write")
 }

 //计算hash值
 hash := t.hasher(key, uintptr(h.hash0))
 //掩码 数组的长度-1
 m := bucketMask(h.B)
 //key存放桶的指针
 b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
 //正在扩容 若旧的桶还没迁移，搜索旧的，迁移则搜索新的桶
 if c := h.oldbuckets; c != nil {
  if !h.sameSizeGrow() {
   // 长度/2
   m >>= 1
  }
  //定位旧的桶的位置
  oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
  //如果旧的桶数据还没迁移
  if !evacuated(oldb) {
   b = oldb
  }
 }
 top := tophash(hash)
bucketloop:
//沿着当前桶查询，查询不出则向溢出桶搜索
 for ; b != nil; b = b.overflow(t) {
  //遍历 tophash 数组
  for i := uintptr(0); i < bucketCnt; i++ {
   if b.tophash[i] != top {
    if b.tophash[i] == emptyRest {
     break bucketloop
    }
    continue
   }
   //若 tophash 相同，根据tophash的位置定位key的位置
   k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
   if t.indirectkey() {
    k = *((*unsafe.Pointer)(k))
   }
   //比较key是否相同
   if t.key.equal(key, k) {
    e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
    if t.indirectelem() {
     e = *((*unsafe.Pointer)(e))
    }
    return e
   }
  }
 }
 //查询无结果，返回零值
 return unsafe.Pointer(&zeroVal[0])
}
```

## 插入和修改

```go

func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
 if h == nil {
 //...
 hash := t.hasher(key, uintptr(h.hash0))

 // 修改标志为写入
 h.flags ^= hashWriting

 if h.buckets == nil {
  h.buckets = newobject(t.bucket)
 }

again:
 bucket := hash & bucketMask(h.B)
 //在扩容 帮忙迁移
 if h.growing() {
  growWork(t, h, bucket)
 }
 //映射位置
 b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
 top := tophash(hash)

 var inserti *uint8           //要插入的cell位置的tophash
 var insertk unsafe.Pointer   //要插入的key的位置
 var elem unsafe.Pointer      //要插入的value的位置
bucketloop:
 for {
  for i := uintptr(0); i < bucketCnt; i++ {
   if b.tophash[i] != top {
    if isEmpty(b.tophash[i]) && inserti == nil {
     inserti = &b.tophash[i]
     insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
     elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
    }
    if b.tophash[i] == emptyRest {
     break bucketloop
    }
    continue
   }
   k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
   if t.indirectkey() {
    k = *((*unsafe.Pointer)(k))
   }
   if !t.key.equal(key, k) {
    continue
   }
   // already have a mapping for key. Update it.
   if t.needkeyupdate() {
    typedmemmove(t.key, k, key)
   }
   elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
   goto done
  }
  ovf := b.overflow(t)
  if ovf == nil {
   break
  }
  b = ovf
 }

 // Did not find mapping for key. Allocate new cell & add entry.

 // If we hit the max load factor or we have too many overflow buckets,
 // and we're not already in the middle of growing, start growing.
 if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
  hashGrow(t, h)
  goto again // Growing the table invalidates everything, so try again
 }

 if inserti == nil {
  // 当前bucket和溢出桶全都满了 需要重新申请一个溢出桶链接到最后面
  newb := h.newoverflow(t, b)
  inserti = &newb.tophash[0]
  insertk = add(unsafe.Pointer(newb), dataOffset)
  elem = add(insertk, bucketCnt*uintptr(t.keysize))
 }

 // 拷贝一个数据放到对应位置
 if t.indirectkey() {
  kmem := newobject(t.key)
  *(*unsafe.Pointer)(insertk) = kmem
  insertk = kmem
 }
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
 //清楚写入标志
 h.flags &^= hashWriting
 if t.indirectelem() {
  elem = *((*unsafe.Pointer)(elem))
 }
 return elem
}
```
## 遍历
正常情况下数据都存在于新的bucket中，但是当map在扩容时，因为采用的是渐进式的机制，所以一部分还没有迁移过来的数据仍在存放在旧的bucket中，需要特殊考虑这种情况，因此在读取每一个bucket时，需要判断旧的bucket对应位置的数据是否已经迁移过来，如果否，还需要判别数据在迁移后去到low还是hign位。

```go
//编写demo探究map遍历过程：
func main() {
	m := make(map[string]string, 0)
	m["1"] = "1"
	for key, value := range m {
		fmt.Println(key + value)
	}
}
//go tool compile -S main.go
//关键指令
//通过mapiterinit初始化，然后不停调用 mapiternext 至没有数据
  0x00cc 00204 (main.go:8)        CALL    runtime.mapiterinit(SB)
  0x0130 00304 (main.go:8)        CALL    runtime.mapiternext(SB)

//遍历的迭代器
type hiter struct {
	key         unsafe.Pointer
	elem        unsafe.Pointer 
	t           *maptype        //map的类型信息 记录了key和value的类型和长度
	h           *hmap           //遍历的bmap
	buckets     unsafe.Pointer // 初始化时指向的bucket
	bptr        *bmap          // 当前遍历到的bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // 开始桶的位置 用于判断是否结束
	offset      uint8          // 遍历的偏移量 下一个读取cell的索引 startBucket + offset
	wrapped     bool           // 是否是从头(0号cell)开始遍历bucket bucket被设置为0时 wrapped会被设置为 true
	B           uint8
	i           uint8          // 当前遍历的cell位置
	bucket      uintptr        // 当前遍历的bucket
	checkBucket uintptr        // 因为扩容需要检查的bucket，检查低位是否有数据迁移到高位，遍历出这些数据
}

//迭代器初始化
func mapiterinit(t *maptype, h *hmap, it *hiter) {
  //...
	it.t = t
	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/goarch.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/reflectdata/reflect.go
	}
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
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

func mapiternext(it *hiter) {
	h := it.h
  //...
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
    //遍历结束
		if bucket == it.startBucket && it.wrapped {
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// 在扩容情况下 遍历选择的对象是扩容后的hmap 所以每一个位置上的数据对应旧的hmap中一部分数据
      // 例如:扩容后长度8 3号位置 -> 旧的3号中扩展长度位=0的数据
      //                6号位置 -> 旧的2号中扩展长度位=1的数据
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
      //旧的hmap对应位置上bucket没有迁移
			if !evacuated(b) {
				checkBucket = bucket
			} else {
        //旧的bucket已经迁移 直接遍历新的bucket即可
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
    //在扩容且map的长度扩展的情况下 新的bucket对应的旧的bucket还没有搬迁 
		if checkBucket != noCheck && !h.sameSizeGrow() {
			//t.reflexivekey -> k==k -> true
      //如果 k==k 的情况下
			if t.reflexivekey() || t.key.equal(k, k) {
				hash := t.hasher(k, uintptr(h.hash0))
        //比较cell的数据是否迁移的位置是否和当前遍历的bucket一致
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
        //k !=k (NaNs)
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
    //那key和value的操作
    //key != key
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || t.key.equal(k, k)) {
			// This is the golden data, we can return it.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
      //key == key
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}

```

## 小点
### 状态位
flag 是一个状态变量，记录map状态。各个状态只有 0 和 1 两种，因此uint8类型可以记录八种不同含义的状态。
```go
	flags     uint8

  iterator     = 1 // 有迭代器使用
	oldIterator  = 2 // 有迭代器使用
	hashWriting  = 4 // map 是否有协程在写操作
	sameSizeGrow = 8 // 扩容是否长度是否增加
```

### 按位置0运算符&^ 
例如：  
x = 01010011
y = 01010100
z = x &^y
  = 00000011

如果 y bit 位为 1，那么结果 z 对应 bit 位就为 0，否则 z 对应 bit 位就和 x 对应 bit 位的值相同。

### TopHash
tophash是hash值的前八位，作为查找该桶中是否有对应值的索引，前判断tophash是否相同，再定位key进行比较。
```go
	emptyRest      = 0 // 这个cell是空的，且高位没有数据(包括溢出桶)
	emptyOne       = 1 // cell是空的
	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell是空的，对应的数据搬迁到新的桶中，存在于旧的bucket中
	minTopHash     = 5 // 最小的tophash值，0-5有对应的含义，> 5 表示数据高八位的数值
```

### Math.NaN 和 float 可以作为 key 吗
可以。只要类型支持 == 和 !=，都可以作为 key，value的类型没有限制。
math.NaN() 的含义是 not a number，类型是 float64，两个 NaN 在进行 == 比较时返回的是 false。当它作为 map 的 key，在搬迁的时候，会遇到一个问题：再次计算它的哈希值和它当初插入 map 时的计算出来的哈希值不一样！这样带来的一个后果是，这个 key 是永远不会被 Get 操作获取的！当我使用 m[math.NaN()] 语句的时候，是查不出来结果的。这个 key 只有在遍历整个 map 的时候，才有机会现身。所以，可以向一个 map 插入任意数量的 math.NaN() 作为 key。
```go
const	uvnan    = 0x7FF8000000000001
func NaN() float64 { return Float64frombits(uvnan) }
func f64hash(p unsafe.Pointer, h uintptr) uintptr {
	f := *(*float64)(p)
	switch {
	case f == 0:
		return c1 * (c0 ^ h) // +0, -0
	case f != f:
		return c1 * (c0 ^ h ^ uintptr(fastrand())) // any kind of NaN  每次计算hash加入随机值
	default:
		return memhash(p, h, 8)
	}
}
```

### 将key和value分开存放的意义？
* key和value是分开存放的，以key1/key2/.../value1/value2...的形式，好处是可以减少内存的补偿padding，节省内存空间，例如，有这样一个类型的 map：map[int64]int8 ，如果按照 key/value/key/value/... 这样的模式存储，那在每一个 key/value 对之后都要额外 padding 7 个字节；而将所有的 key，value 分别绑定到一起，这种形式 key/key/.../value/value/...，则只需要在最后添加 padding。


### map中提升效率的方式
* 将key和value分开存放，减少内存补偿，提高磁盘读的效率
* 用hash值的前八位作为索引，先比较索引，再具体比较key，减少key之间equals方法的调用
* 根据key的具体类型替换为对应类型定制化方式，提升效率

### map中有减少hash冲突的方式
* 链表法，每一个节点可以存放八个元素，超出部分申请新的节点并链接到后面
* 扩容

### 为什么go中map不使用红黑树提高搜索效率？
提升考虑的方式不同。java采用红黑树的方式提高搜索的效率，go则是在每一个节点中存放八个数据，每次进行搜索都是顺序查询，不像红黑树一样频繁根据引用查找下一个节点。   

### map中没有缩容的操作
map没有提供缩容的操作，也就是说map只会不停的增长，不会因为数据变少而减少容量，这就会导致一个问题，当map中存放上百万的数据后，即使删除了数据，map也会占用很大的内存空间。解决办法：
* 定时重建map，将原来的数据copy到新的map中
* 使用指针作为value   
  就算map容量非常大，但是指针占用的空间是固定的，可以减少内存损耗。   
[Maps and Memory Leaks in Go](https://teivah.medium.com/maps-and-memory-leaks-in-go-a85ebe6e7e69)   
[学习了！GoMap 会内存泄露？？](https://mp.weixin.qq.com/s/TcYo3VWpM3uDpya1XXrX3w)


### extra *mapextra 的意义
当map的key/value不包含指针且size小于128byte时，会标记这个map，在gc时不需要扫描bucket，减少gc的压力。   
但是bucket中的overflow是一个指针类型，生成函数会将overflow的类型转化为uintptr，而uintptr虽然是地址，但不会被gc认为是指针，指向的数据有被回收的风险。
此时为保证其中的overflow指针指向的数据存活，就用mapextra结构指向了这些buckets，这样bmap有被引用就不会被回收了。
### 根据 key 的不同类型，编译器还会将查找、插入、删除的函数用更具体的函数替换，如何做到？




本文是以下文章的读书笔记：   
[Go 语言 map 的底层实现完整剖析](https://zhuanlan.zhihu.com/p/406751292)
[深度解密Go语言之map](https://mp.weixin.qq.com/s/Jq65sSHTX-ucSG8TlI5Zxg)
