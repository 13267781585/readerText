# golang

## 值选择器 vs 指针选择器
在Go语言中选择使用值接收器（`func (a A)`）还是指针接收器（`func (a *A)`）通常取决于以下几个因素：

1. **是否需要修改原始结构体的数据**
2. **结构体的大小**
3. **方法的调用频率和性能要求**

### 场景示例

#### 场景一：不需要修改原始数据

如果你的方法只是读取结构体的数据，而不需要修改它，那么使用值接收器是合适的。这样可以避免在方法调用时不小心修改了数据。

```go
type Book struct {
    Title  string
    Author string
    Pages  int
}

// 使用值接收器，因为只需要读取数据
func (b Book) Description() string {
    return fmt.Sprintf("%s by %s, %d pages", b.Title, b.Author, b.Pages)
}
```

#### 场景二：需要修改原始数据

如果你需要在方法中修改结构体的数据，并且希望这些修改反映到原始数据上，那么应该使用指针接收器。

```go
type Counter struct {
    Value int
}

// 使用指针接收器，因为需要修改原始数据
func (c *Counter) Increment() {
    c.Value++
}
```

#### 场景三：结构体较大

对于包含大量字段或大型字段（如大数组）的结构体，使用值接收器可能会导致性能问题，因为Go语言在调用方法时会复制整个结构体。在这种情况下，即使不需要修改原始数据，使用指针接收器也可能更有效，因为它只传递一个指针。

```go
type LargeData struct {
    LotsOfNumbers [10000]int
}

// 使用指针接收器，即使只是读取数据，也是为了避免复制大量数据
func (d *LargeData) Sum() int {
    sum := 0
    for _, num := range d.LotsOfNumbers {
        sum += num
    }
    return sum
}
```

#### 场景四：方法频繁调用

如果一个方法会被非常频繁地调用，那么即使结构体不大，使用指针接收器也可能更合适，因为这样可以减少复制的开销，提高程序的运行效率。

```go
type Position struct {
    X, Y float64
}

// 使用指针接收器，优化性能，尤其在频繁调用时
func (p *Position) Move(dx, dy float64) {
    p.X += dx
    p.Y += dy
}
```

### 总结

选择`func (a A)`还是`func (a *A)`应根据是否需要修改数据、数据的大小和性能要求来决定。通常，如果需要修改原始数据或考虑到性能（如结构体较大或方法频繁调用），使用指针接收器是更好的选择。如果只是读取数据且结构体较小，使用值接收器可以简化代码并避免意外修改数据。

## nil

* nil 不是关键字，可以定义变量名为nil的变量

```go
 nil := 1
 fmt.Println(nil)
```

* nil 是pointer、func、map、slice、interface、channel的零值

  * slice   
    为nil的slice，除了不能索引外，其他的操作都是可以的，当你需要填充值的时候可以使用append函数，slice会自动进行扩充。根据官方的文档，slice有三个元素，分别是长度、容量、指向数组的指针。

    ```go
    // nil slices
    var s []slice
    len(s)  // 0
    cap(s)  // 0
    for range s  // iterates zero times
    s[i]  // panic: index out of range
    ```

  * map   
  对于nil的map，我们可以把它看成是一个只读的map，不能进行写操作，否则就会panic。对于只被用于读操作的map，在传参的是时候可以传递nil值。

    ```go
    //map用于只读
    func a(m map[string]string)  {}

    //使用
    a(nil)  -- 空map
    ```

  * channel   
    关闭一个nil的channel会导致程序panic。
  * interface   
    interface并不是一个指针，底层实现由两部分组成，一个是类型，一个值，(Type, Value)。声名接口指针类型没有赋值时，类型和值都为nil，这时候接口==nil，显示赋值带有类型值为nil时，接口类型不为nil，值为nil，这时候接口!=nil。

    ```go
    func Test1(t *testing.T) {
        var inter TestInterface // ==nil

        inter = &Test11{} // !=nil

        var test11 *Test11
        inter = test11 // !=nil
    }

    type TestInterface interface {
        A()
    }
    ```

[理解golang中什么是nil](https://blog.csdn.net/raoxiaoya/article/details/108705176)

## int和uint的意义

go提供了int8、int16、int32、int64等固定长度的数据类型，int和uint是根据cpu的位数来决定的，当使用的时候没有数据精度要求的时候推荐使用int和uint，因为按照cpu一个字的精度存放数据，有利于提高效率。

* 在32位的系统下使用int64会使得读取的速度变慢
* 在64位的系统下使用int64，编辑器可以根据地址和偏移量生成更加高效的代码，int32和int64混用会造成效率降低
[与Go语言中的特定类型（int64 / uint64）相比，常规类型（int / uint）有什么优势？](https://ask.csdn.net/questions/1011809)
[在go中使用大小型或无符号整数类型的原因是什么？](https://www.codenong.com/34796020/)

## type关键字的作用

* 定义结构体
* 定义接口
* 定义类型   
定义类型是一个新的类型，和原来的类型没有关系，新增加的方法不属于原来的类型

```go
type Cat int

func (*Cat) print() {
 fmt.Println("cat...")
}

type MyCat Cat

func (*MyCat) print1() {
 fmt.Println("mycat...")
}

func main() {
 var cat Cat
 cat.print()
 //cat.print1() //编译错误

 var mycat MyCat
 mycat.print1()
}

// cat...
// mycat...
```

* 类型别名
别名相当于类型的昵称，别名和类型没有区别，方法共享

```go

type Cat int

func (*Cat) print() {
 fmt.Println("cat...")
}

type CatAlais = Cat

func (*CatAlais) print2() {
 fmt.Println("catalais ...")
}

func main() {
 var cat Cat
 cat.print()
 cat.print2()

 var cat1 CatAlais
 cat1.print()
 cat1.print2()
}

// cat...
// catalais ...
// cat...
// catalais ...
```

* 类型查询
判断接口的类型

```go

type Cat struct {
}

type Int int
type Float = float64

func findType(name interface{}) {
 switch name.(type) {
 case int:
  fmt.Println("int ...")
 case float64:
  fmt.Println("float64...")
 case Cat:
  fmt.Println("cat...")
 default:
  fmt.Println("unkown...")
 }
}

func main() {
 var cat Cat
 var i Int
 var f Float
 findType(1)
 findType(cat)
 findType(i)
 findType(f)
}
// int ...
// cat...
// unkown...
// float64...
```

## Slice是否共用底层数组

* 当从一个切片或数组截取出来的切片共享同一个底层数组，对一个切片修改，其他的可见

```go
 arr := make([]int, 2, 6)
 arr[1] = 1
 fmt.Println(arr)
 arr1 := arr[:1]
 arr1[0] = 1
 fmt.Println(arr1)
 fmt.Println(arr)
  
// [0 1]
// [1]
// [1 1]
```

* 当调用append函数增加数据涉及到扩容时，会重新申请内存并copy数据，这时底层数组不共用，修改对其他切换不可见

```go
// arr1 := arr[start:end:max]  max不能大于容量
 arr := make([]int, 2, 6)
 arr[1] = 2
 fmt.Println(arr)
 arr1 := arr[:1]
 arr1 = append(arr1, 1)  // 容量足够，不涉及扩容，数据覆盖
 fmt.Println(arr1)
 fmt.Println(arr)

// [0 2]
// [0 1]
// [0 1]

 arr := make([]int, 2, 6)
 arr[1] = 2
 fmt.Println(arr)
 arr1 := arr[:1]
 //增加6个数据，这时候切片长度6，超过原来容量，需要扩容
 arr1 = append(arr1, 1)
 arr1 = append(arr1, 1)
 arr1 = append(arr1, 1)
 arr1 = append(arr1, 1)
 arr1 = append(arr1, 1)
 arr1 = append(arr1, 1)
 arr1[0] = 2    // 修改第一个元素 判断是否会影响arr
 fmt.Println(arr1)
 fmt.Println(arr)

// [0 2]
// [2 1 1 1 1 1 1]
// [0 1]
```

## 切片和数组的区别

* 数组是长度固定的，切片底层是基于数组实现的，是对数组的扩展，可以实现自动扩容，[2]int和[3]int是不同的类型
* 数组是值类型，赋值都会复制，切片是引用类型，赋值不会复制数据
* 数组在编译时确定大小和内存，切片在运行时动态扩容

## 切换扩容长度的计算

```go

func main() {
    s := make([]int, 0)

    oldCap := cap(s)

    for i := 0; i < 2048; i++ {
        s = append(s, i)

        newCap := cap(s)

        if newCap != oldCap {
            fmt.Printf("[%d -> %4d] cap = %-4d  |  after append %-4d  cap = %-4d\n", 0, i-1, oldCap, i, newCap)
            oldCap = newCap
        }
    }
}

// [0 ->   -1] cap = 0     |  after append 0     cap = 1   
// [0 ->    0] cap = 1     |  after append 1     cap = 2   
// [0 ->    1] cap = 2     |  after append 2     cap = 4   
// [0 ->    3] cap = 4     |  after append 4     cap = 8   
// [0 ->    7] cap = 8     |  after append 8     cap = 16  
// [0 ->   15] cap = 16    |  after append 16    cap = 32  
// [0 ->   31] cap = 32    |  after append 32    cap = 64  
// [0 ->   63] cap = 64    |  after append 64    cap = 128 
// [0 ->  127] cap = 128   |  after append 128   cap = 256 
// [0 ->  255] cap = 256   |  after append 256   cap = 512 
// [0 ->  511] cap = 512   |  after append 512   cap = 1024
// [0 -> 1023] cap = 1024  |  after append 1024  cap = 1280
// [0 -> 1279] cap = 1280  |  after append 1280  cap = 1696
// [0 -> 1695] cap = 1696  |  after append 1696  cap = 2304

func main() {
    s := []int{1,2}
    s = append(s,4,5,6)
    fmt.Printf("len=%d, cap=%d",len(s),cap(s))
}

// len=5, cap=6
```

* 在容量小于1024，新的容量等于旧的容量*2,当增加多个元素不一定
* 没有明显的规律

```go
//et 数据的类型  old 老的切片 cap 最小需要的容量
func growslice(et *_type, old slice, cap int) slice {
    // ……
    newcap := old.cap
    //双倍容量
    doublecap := newcap + newcap
    //如果最小需要容量大于双倍，直接申请最小所需容量 对应上述第二种添加多个元素的情形
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            //如果容量小于1024 新的容量翻倍
            newcap = doublecap
        } else {
            for newcap < cap {
                // 否则按 1/4 容量递增到大于最小所需容量
                newcap += newcap / 4
            }
        }
    }
    // ……
    // 需要进行内存对齐操作，这里就是导致容量增长不规律的原因
    capmem = roundupsize(uintptr(newcap) * ptrSize)
    newcap = int(capmem / ptrSize)
}



func roundupsize(size uintptr) uintptr {
    if size < _MaxSmallSize {
        if size <= smallSizeMax-8 {
            return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
        } else {
            //……
        }
    }
    //……
}

const _MaxSmallSize = 32768
const smallSizeMax = 1024
const smallSizeDiv = 8


var size_to_class8 = [smallSizeMax/smallSizeDiv + 1]uint8{0, 1, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15, 16, 16, 17, 17, 18, 18, 18, 18, 19, 19, 19, 19, 20, 20, 20, 20, 21, 21, 21, 21, 22, 22, 22, 22, 23, 23, 23, 23, 24, 24, 24, 24, 25, 25, 25, 25, 26, 26, 26, 26, 26, 26, 26, 26, 27, 27, 27, 27, 27, 27, 27, 27, 28, 28, 28, 28, 28, 28, 28, 28, 29, 29, 29, 29, 29, 29, 29, 29, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31}

var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

扩容新的容量的大致规律：

* 小于1024，翻倍
* 大于1024，增长 1/4

[深度解密Go语言之Slice](https://mp.weixin.qq.com/s/xik2YcHpgpbZd-8nCwACKw)

## uint8 和 byte，uint32 和 rune 没有区别，为什么要定义这两个别名?

因为uint8 和 uint32 直观上来看就是一个数值，但是也可以是一个字符，为了消除这个直观的错觉，定义这两个别名更好的表示。

## 信号量

[go中semaphore(信号量)源码解读](https://www.cnblogs.com/ricklz/p/14610213.html)

## race

检查程序是否有并发读写同一变量且没有做同步机制的调试方式。  
[golang中的race检测](https://www.cnblogs.com/yjf512/p/5144211.html)
```go
func main() {
	a, lock := 1, sync.Mutex{}
	go func() {
		lock.Lock()
		defer lock.Unlock()
		a = 2
	}()
	lock.Lock()
	defer lock.Unlock()
	a = 3
	fmt.Println("a is ", a)

	time.Sleep(2 * time.Second)
}


//a is  3

func main() {
	a, lock := 1, sync.Mutex{}
	go func() {
		a = 2
	}()
	a = 3
	fmt.Println("a is ", a)

	time.Sleep(2 * time.Second)
}

// a is  3
// ==================
// WARNING: DATA RACE
// Write at 0x00c0000ba018 by goroutine 7:
//   main.main.func1()
//       /Users/bytedance/work/workspace/TestProject/redis/main.go:11 +0x30

// Previous write at 0x00c0000ba018 by main goroutine:
//   main.main()
//       /Users/bytedance/work/workspace/TestProject/redis/main.go:13 +0xac

// Goroutine 7 (running) created at:
//   main.main()
//       /Users/bytedance/work/workspace/TestProject/redis/main.go:10 +0xa4
// ==================
// Found 1 data race(s)
// exit status 66
```

## 不可复制结构体

* 变量资源本身带状态且操作要配套的不能拷贝，携带有noCopy的结构体都不能复制。
* 因为接口含有状态，用于使结构在使用时不可复制避免出错的机制。

### noCopy(静态检测，不影响性能)

``使用 go vet 命令才能检查出来，如果直接编译或者运行，不会报错``

#### 原理

* nocopy 是底层源码使用的，用户无法使用，如果需要达到不可复制效果，可以实现sync.Locker接口，并作为结构体私有变量

#### 使用

```go
// 实现sync.Locker接口

type noCopy struct{}

func (*noCopy) Lock() {}

func (*noCopy) Unlock() {}

type SomethingCannotCopy struct {
    noCopy 
}
```

[Go 的 noCopy 是什么机制？](https://blog.csdn.net/EDDYCJY/article/details/125883888)   
[深入理解Golang nocopy原理](https://int64.ink/blog/golang_%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3golang_nocopy%E5%8E%9F%E7%90%86/)

### copyChecker(运行时检测，有损性能)

#### 原理

* 变量复制后指针会变化，可以通过记录并对比指针判断是否发生了复制行为。

```go
//sync.Cond源码
type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) && 
            !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
            uintptr(*c) != uintptr(unsafe.Pointer(c)) {

        panic("sync.Cond is copied")

    }
```

首先是checker初始化之后第一次调用：

* 当check第一次被调用，c的值肯定是0，而这时候c是有真实的地址的，所以step 1失败，进入step 2；
* 用原子操作把c的值设置成自己的地址值，注意只有c的值是0的时候才能完成设置，因为这里c的值是0，所以交换成功，step 2是False，判断流程直接结束；
* 因为不排除还有别的goroutine拿着这个checker在做检测，所以step 2是会失败的，这是要进入step 3；
* step 3再次比较c的值和它自己的地址是否相同，相同说明多个goroutine共用了一个checker，没有发生复制，所以检测通过不会panic。
    如果step 3的比较发现不相等，那么说明被复制了，直接panic

再看其他情况下checker的流程：

* 这时候c的值不是0，如果没发生复制，那么step 1的结果是False，判断流程结束，不会panic；
* 如果c的值和自己的地址不一样，会进入step 2，因为这里c的值不为0，所以表达式结果一定是True，所以进入step 3；
* step 3和step 1一样，结果是True，地址不同说明被复制，这时候if里面的语句会执行，因此panic。

#### 使用

```go
type SomethingCannotCopy struct {
    copyChecker // 这里只能作为值类型，不能作为引用类型，因为值类型在SomethingCannotCopy发生复制时会生成新的copyChecker对象
}
```

### 自定义

#### 原理

* 定义接口，实现包内私有类型，对外提供生成对象接口
* 通过反射和类型断言无法取出接口引用的数据；因为我们传给接口的是指针，因此源数据不会被复制

#### 使用

```go
// 对外只提供接口来访问数据
type Worker interface {
 Work()
}

// 内部类型不导出，以接口的形式供外部使用
type normalWorker struct {
 // data members
}

func (*normalWorker) Work() {
 fmt.Println("I am a normal worker.")
}

func NewNormalWorker() Worker {
 return &normalWorker{}
}

type specialWorker struct {
 // data members
}

func (*specialWorker) Work() {
 fmt.Println("I am a special worker.")
}

func NewSpecialWorker() Worker {
 return &specialWorker{}
}
```

[golang拾遗：实现一个不可复制类型](https://www.cnblogs.com/apocelipes/p/17137206.html#%E9%9D%99%E6%80%81%E6%A3%80%E6%B5%8B%E5%AE%9E%E7%8E%B0%E7%A6%81%E6%AD%A2%E5%A4%8D%E5%88%B6)

## context

[Golang Context 源码剖析](http://www.17bigdata.com/study/programming/godeep/goddeep-context-src.html)
[Golang Context 源码分析](https://mritd.com/2021/06/27/golang-context-source-code/)

## make和new区别

* make专门用于创建slice、map、chan类型数据并初始化；new用于创建任意类型
* make返回的是类型本身，因为创建的slice、map、chan是引用类型，不需要在返回指针，new返回的是类型指针，指针指向该类型的零值
* go模糊了堆和栈的界定，在编译时使用逃逸分析判断变量分配在什么地方

```go
make([]int,10)->长度为10的切片
new([]int)->[]int的指针*[]int，值为nil
```

## panic和throw的区别

* panic表示可以捕获处理的奔溃，使用recover修复，throw表示不可修复的错误(例如未加锁情况下解锁，map并发读写等)，修复没有意义，程序会直接退出
* panic可以在程序中使用，throw存在底层源码，用户不可以使用

## init执行顺序

* 同个文件按init定义顺序执行
* 同个包不同文件按文件名字典序执行
* main引入不同包，按引入顺序执行，再执行main的init函数
* 如果包之间有依赖关系，先执行被依赖包init
[一文读懂 Golang init 函数执行顺序](https://cloud.tencent.com/developer/article/2138066)

## CSP并发编程模型

两个并发实体通过消息(管道)通信的并发模型。go使用goroutine和channel实现部分csp理论，csp提倡通过通信共享内存，而不是通过共享内存通信。
#### 共享内存和csp区别
* 通信方式：csp使用管道发送消息通信；共享内存使用共享内存进行通信，例如java、c++实现了线程安全的数据结构
* 同步机制：csp发送消息后会阻塞，等到消息被接受后才能继续发送；共享内存随时可以写内存，需要配合同步机制
* 复制性：csp编程简单；共享内存需要开发者管理锁状态防止资源竞争和死锁   
[Golang的CSP并发模型](https://lushunjian.gitee.io/2021/03/02/golang-de-csp-bing-fa-mo-xin)

## 逃逸分析

编译阶段决定变量分配到堆还是栈上，提高内存使用效率和减少gc压力。

### 发生逃逸情况
变量被外引用或者饮用关系不明确。
* 函数把指针变量作为返回值
* interface{}类型的参数，编辑期间无法判断具体类型
* 大小超过设定的局部变量
* 闭包对象

## 闭包

闭包是一种特殊的匿名函数，可以捕获外部域变量，即使变量在外部域生命周期已经结束。由于使用闭包会导致代码不够清晰，使用不当还会导致得到错误的结果。所以一般不建议使用闭包。

### 闭包的作用和意义

* 捕获状态： 闭包可以捕获它被创建时候的环境，这意味着它可以记住和操作当时作用域中的变量。这种特性常被用来生成和维护状态。
* 数据隔离： 闭包允许封装变量，提供数据隐藏。即使闭包返回到它的外部作用域，这些私有变量依旧是被保护的，不会被外部作用域直接访问，只能通过闭包暴露的方法进行操作，通常用于优化全局变量。
* 回调函数： 闭包经常用作回调函数，特别是在异步操作中。它可以继承创建时候的上下文环境，并在将来某个时刻被调用。
* 实现接口： 通过闭包，可以在不定义新类型的情况下实现接口方法。

```go
func count() func() int {
 var i int
 return func() int {
  i++
  return i
 }
}

func Test111(t *testing.T) {
 c := count()
 fmt.Println(c())
 fmt.Println(c())
 c1 := count()
 fmt.Println(c1())
 fmt.Println(c1())
}

// Runner 接口定义了Run方法，任何实现了该方法的类型都满足此接口
type Runner interface {
    Run()
}

// FuncRunner 是一个函数类型，它也实现了Run方法
type FuncRunner func()

// Run 调用包含的闭包函数
func (f FuncRunner) Run() {
    f()
}

// NewTask 返回一个Runner接口，该接口背后是一个匿名函数闭包
func NewTask() Runner {
    var id int // id变量会被闭包捕获

    return FuncRunner(func() {
        id++
        fmt.Printf("Task id %d has been completed\n", id)
    })
}

func main() {
    task1 := NewTask() // 创建一个任务，它的闭包捕获了id变量
    task1.Run() // 输出: Task id 1 has been completed
    task1.Run() // 输出: Task id 2 has been completed

    task2 := NewTask() // 创建另一个任务，它持有一个新的独立id变量
    task2.Run() // 输出: Task id 1 has been completed
}
```

[探究Golang中的闭包](https://llmxby.com/2022/08/27/%E6%8E%A2%E7%A9%B6Golang%E4%B8%AD%E7%9A%84%E9%97%AD%E5%8C%85/)


## string
基础类型，不可变，线程安全，用于处理UTF-8文本。
### 底层实现
```golang
type stringStruct struct {
    str unsafe.Pointer // 指向数据数组的指针
    len int            // 数据的长度（字节）
}
```

### 字符串的内存管理
Go 的字符串通常是高效的，因为：
* 内存共享：当从一个字符串派生出一个子字符串时，新的字符串会复用原始字符串的内存。例如，s2 := s1[1:4] 不会导致复制，s2 会指向 s1 的内部数据的一个子区间。
* 字符串字面量的内存是静态分配的：在编译时，所有的字符串字面量都会被存储在可执行文件的只读部分，并在程序运行时共享。

### 不可变性
#### 优点
* 线程安全
* 内存优化
例如，编译器可以在编译时对相同的字符串字面量进行去重处理，使得所有相同的字符串字面量引用同一块内存区域。
* 高效

#### 缺点
* 大量修改操作会导致临时变量创建销毁，这可能会对性能和内存使用产生负面影响。在这种情况下，可以使用如 strings.Builder 或 []byte 等可变数据结构来优化性能