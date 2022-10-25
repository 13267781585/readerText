# atomic-package
sync/atomic 将底层硬件的原子能力封装成了函数，在调用的时候可以保证原子性，可以在并发中使用。

## 出现并发安全的原因
对于编码中的一个语句，可能会被解析成多条机器指令，因此如果这几个指令不能在同一时间点执行，就会发生并发安全问题。例如：对于一个基本类型的赋值 i := int64(1)，在 64bit 的系统下不会出现问题，但是如果是 32bit 系统，会被拆成两个写指令，先写高32位，在写低32位。


## 基本类型原子函数
不只是int有对应的原子问题，unsafe.Pointer，uintptr 本质上也是一个整形，也有原子的问题。
```go
func SwapInt32(addr *int32, new int32) (old int32)

// SwapInt64 atomically stores new into *addr and returns the previous *addr value.
func SwapInt64(addr *int64, new int64) (old int64)

// SwapUint32 atomically stores new into *addr and returns the previous *addr value.
func SwapUint32(addr *uint32, new uint32) (old uint32)

// SwapUint64 atomically stores new into *addr and returns the previous *addr value.
func SwapUint64(addr *uint64, new uint64) (old uint64)

// SwapUintptr atomically stores new into *addr and returns the previous *addr value.
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)

// SwapPointer atomically stores new into *addr and returns the previous *addr value.
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)

// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

//...

// AddInt32 atomically adds delta to *addr and returns the new value.
func AddInt32(addr *int32, delta int32) (new int32)

//...

// LoadInt32 atomically loads *addr.
func LoadInt32(addr *int32) (val int32)
//...

// StoreInt32 atomically stores val into *addr.
func StoreInt32(addr *int32, val int32)
//...
```

## 万能类型 atomic.Value
### 原理
本质上使用的是上述基本类型的原子操作+自旋完成的。

### 数据结构
内部使用 any 类型，可以存放所有的数据类型，ifaceWords 是 interface{} 的运行时内置描述符，通过将 v 转化为 ifaceWords 类型，可以获取到 v 的类型和对应的数据。
```go
type Value struct {
	v any
}

type ifaceWords struct {
	typ  unsafe.Pointer  //v的类型指针
	data unsafe.Pointer  //v的数据指针
}
```

### 写入操作
```go
func (v *Value) Store(val any) {
	if val == nil {
		panic("sync/atomic: store of nil value into Value")
	}
	vp := (*ifaceWords)(unsafe.Pointer(v))
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	for {
		typ := LoadPointer(&vp.typ)
		if typ == nil {
			// Attempt to start first store.
			// Disable preemption so that other goroutines can use
			// active spin wait to wait for completion.
			runtime_procPin()
			if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(&firstStoreInProgress)) {
				runtime_procUnpin()
				continue
			}
			// Complete first store.
			StorePointer(&vp.data, vlp.data)
			StorePointer(&vp.typ, vlp.typ)
			runtime_procUnpin()
			return
		}
		if typ == unsafe.Pointer(&firstStoreInProgress) {
			// First store in progress. Wait.
			// Since we disable preemption around the first store,
			// we can wait with active spinning.
			continue
		}
		// First store completed. Check type and overwrite data.
		if typ != vlp.typ {
			panic("sync/atomic: store of inconsistently typed value into Value")
		}
		StorePointer(&vp.data, vlp.data)
		return
	}
}
```

## 疑问
* Mutex 和 原子操作 的区别

[Go 语言标准库中 atomic.Value 的前世今生](https://blog.betacat.io/post/golang-atomic-value-exploration/)