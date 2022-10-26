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
内部使用 any 类型，可以存放所有的数据类型，ifaceWords 是 interface{} 的运行时内置描述符，通过将 v 转化为 ifaceWords 类型，可以获取到 v 的类型和对应的数据(类型转化问题可见[Point In Go](./Pointer%20In%20Go.md))。
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
* 过程
interfaca{} 类型的数据运行时描述类型是 ifaceWords，写入时需要修改两个指针字段。   
    写入操作分为两种情况：   
        1. 初始状态下，typ 和 data 都是为 nil，因此需要设置两个值，上述基本类型虽然提供了原子操作，但是只针对一个字段，两个字段无法保证原子性，因此需要程序自行增加判断保证。
        解决办法：使用自旋的方式，各个协程竞争将 typ 字段设置为一个标志位，设置成功的协程修改 data 字段，再将 typ 修改为对应类型，第一个竞争成功的协程完成对两个字段的更新且能保证原子性；竞争失败的协程自旋重试，会进入第二种情况。
        2. 第一个协程写入成功后，后续的协程之需要更新 data 字段即可，应该放入的数据类型默认一致。
* 优化措施   
    * runtime_procPin()/runtime_procUnpin() 禁止/启动协程抢被抢占，主要有两个作用：1.第一次写入时，当协程成功设置 typ 为标志位，因为在并发情况下，采用自旋的方式，会存在其他协程自旋等待第一个写入完成，防止协程被抢占可以尽快完成写入操作。2.防止 GC 线程看到一个莫名其妙的指向 firstStoreInProgress 的类型（这是赋值过程中的中间状态）。
```go
func (v *Value) Store(val any) {
    //...
	vp := (*ifaceWords)(unsafe.Pointer(v))
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	for {
        //原子操作，加载旧值的类型指针
		typ := LoadPointer(&vp.typ)
        //第一次写入
		if typ == nil {
			//禁止其他协程抢占该协程，也就是说尽快让这个协程的写入操作完成，因为采用的是自旋的模式，其他协程可能占用着cpu自旋等待这个操作的完成(协程在占用一定时间片后会被切换)
			runtime_procPin()
            //设置失败 启用抢占模式
			if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(&firstStoreInProgress)) {
				runtime_procUnpin()
				continue
			}
			//设置标志成功 写入值并开启抢占模式
			StorePointer(&vp.data, vlp.data)
			StorePointer(&vp.typ, vlp.typ)
			runtime_procUnpin()
			return
		}
        //typ ！= nil 检查第一次写入是否完成
		if typ == unsafe.Pointer(&firstStoreInProgress) {
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

### 读取操作
```go
func (v *Value) Load() (val any) {
	vp := (*ifaceWords)(unsafe.Pointer(v))
	typ := LoadPointer(&vp.typ)
	if typ == nil || typ == unsafe.Pointer(&firstStoreInProgress) {
		// First store not yet completed.
		return nil
	}
	data := LoadPointer(&vp.data)
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
    //将传入的val的类型和数据指针指向存储的数据
	vlp.typ = typ
	vlp.data = data
	return
}
```
## 疑问
* Mutex 和 atomic原子操作 的区别  
本质上就是 乐观锁 和 悲观锁 不同的应用。  
    * Mutex 悲观锁，操作系统实现，在并发激烈的情况下使用
    * atomic原子操作 乐观锁，底层硬件实现，自旋可以发挥多cpu的优势，只适合在并发不激烈的情况下使用，并发激烈情况下，大量自旋会浪费cpu资源


[Go 语言标准库中 atomic.Value 的前世今生](https://blog.betacat.io/post/golang-atomic-value-exploration/)