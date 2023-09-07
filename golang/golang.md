# golang

## nil

* nil 不是关键字，可以定义变量名问nil的变量

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
    func do() error {   // error(*doError, nil)
        var err *doError  
        return err  
    }

    //正确处理方式
    func do1() error{
        var err *doError
        ...
        if err != nil{
            return err
        }
        return nil
    }

    func main() {
        err := do()  // 类型不为空 值为空
        // true
        if err != nil{
            ...
        } 
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

## Slice是否公用底层数组

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

数组是长度固定的，切片底层是基于数组实现的，是对数组的扩展，可以实现自动扩容，[2]int和[3]int是不同的类型

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

检查程序是否有并发读写统一变量的调试方式。
[golang中的race检测](https://www.cnblogs.com/yjf512/p/5144211.html)

## noCopy

因为接口含有状态，用于使结构在使用时不可复制避免出错的机制。
[Go 的 noCopy 是什么机制？](https://blog.csdn.net/EDDYCJY/article/details/125883888)
[深入理解Golang nocopy原理](https://int64.ink/blog/golang_%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3golang_nocopy%E5%8E%9F%E7%90%86/)

## context

[Golang Context 源码剖析](http://www.17bigdata.com/study/programming/godeep/goddeep-context-src.html)

## make和new区别

* make专门用于创建slice、map、chan类型数据并初始化；new用于创建任意类型
* make返回的是类型的引用(Type)，new返回的是类型引用的指针(*Type)

## panic和throw的区别

* panic表示可以捕获处理的奔溃，使用recover修复，throw表示不可修复的错误(例如未加锁情况下解锁，map并发使用等)，不行可以修复，程序会直接退出
* panic可以再程序中使用，throw存在底层源码，用户不可以使用
