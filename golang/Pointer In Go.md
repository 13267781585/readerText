# Point In Go
Go中的指针及与指针对指针的操作主要有以下三种：   
* 普通的指针类型，例如 var intptr *T，定义一个T类型指针变量。

* 内置类型uintptr，本质是一个无符号的整型，它的长度是跟平台相关的，它的长度可以用来保存一个指针地址。

* 是unsafe包提供的Pointer，表示可以指向任意类型的指针。

## 普通指针类型(safe pointer)
### Base Type 基本类型
* 假设有指针 *T,那么他的基本类型就是 T。
* 两个未命名的类型有相同的基本类型，那他们就是相同的类型。
```go
*int  // An unnamed pointer type whose base type is int.
**int // An unnamed pointer type whose base type is *int.

// Ptr is a named pointer type whose base type is int.
type Ptr *int
// PP is a named pointer type whose base type is Ptr.
type PP *Ptr
```
### 限制
* 不支持算术运算
* 普通指针类型之间不能随意转换，除非满足以下条件之一:   
    * 两个类型的基础类型(Underlying Type)一致  
    * 两个类型都是未命名的指针类型，且指针的基本类型(Base Type)的基础类型(Underlying Type)一致
    以下举例说明:
    ```go
    type MyInt int64
    type Ta    *int64
    type Tb    *MyInt

    /*
        *int64->Ta 基础类型都是*int64，反之亦然
        *MyInt->Tb 基础类型都是*MyInt，反之亦然
        *MyInt->*int64 基本类型 MyInt 和 int64，基础类型 int64 和 int64,反之亦然
        Ta不能直接转化成Tb，但是可以根据上述操作，隐式地通过三个显示转换，假设有 Ta 类型变量 pa，Tb((*MyInt)((*int64)(pa)))
    */
    ```
* 不同普通指针类型不能用于比较(==、!=等)  
不同普通类型不支持比较操作，除非满足以下条件之一:   
    * 两个指针类型相同
    * 两个指针的基础类型相同且都是未命名类型。
    * 其中一个指针是 nil
* 不同类型的指针不能相互赋值

## uintptr类型
* uintptr用于指针运算，保存了一个指针地址，但它只是一个值，不引用任何对象，因此 uintptr 的值指向的内存可能会被回收。
```go
// uintptr is an integer type that is large enough to hold the bit pattern of any pointer.
type uintptr uintptr
```
## unsafe.Pointer(unsafe pointer)
* Poniter是一种特殊的指针，可以包含任意类型变量的地址
* 普通类型指针可以转化为 Pointer,反之亦然
* uintptr 可以转化为 Pointer,反之亦然
```go
package unsafe
// ArbitraryType is here for the purposes of documentation only and is not actually
// part of the unsafe package. It represents the type of an arbitrary Go expression.
type ArbitraryType int

type IntegerType int

type Pointer *ArbitraryType
//查询对象占用空间的多少
func Sizeof(x ArbitraryType) uintptr
//查询结构对象中字段的偏移量(只能用于结构体)
func Offsetof(x ArbitraryType) uintptr
//查询类型的内存对齐倍数
func Alignof(x ArbitraryType) uintptr
//返回ptr位置到+len长度的切片
func Slice(ptr *ArbitraryType, len IntegerType) []ArbitraryType
//在ptr的基础上+len，返回对于位置的指针
func Add(ptr Pointer, len IntegerType) Pointer

func main() {
	var struct1 struct {
		Int1 int8
		Int2 int32
		Int3 int64
	}
	fmt.Println(unsafe.Alignof(struct1.Int1))//1
	fmt.Println(unsafe.Alignof(struct1.Int2))//4
	fmt.Println(unsafe.Alignof(struct1.Int3))//8
	fmt.Println(unsafe.Sizeof(struct1.Int1))//1
	fmt.Println(unsafe.Sizeof(struct1.Int2))//4
	fmt.Println(unsafe.Sizeof(struct1.Int3))//8
	fmt.Println(unsafe.Offsetof(struct1.Int1))//0
	fmt.Println(unsafe.Offsetof(struct1.Int2))//4
	fmt.Println(unsafe.Offsetof(struct1.Int3))//8
	fmt.Println(unsafe.Alignof(struct1))//8 alignof != size
	fmt.Println(unsafe.Sizeof(struct1))//16
}
```
### unsafe pointer 是一个指针，uintptr 是一个整形
pointer 变量引用内存空间，而 uintptr 只是一个整形变量，假如一块内存空间的地址仅被 uintptr 的变量记录，是随时可能会被 gc 回收的。
```go
import "unsafe"

// Assume createInt will not be inlined.
//go:noinline
func createInt() *int {
	return new(int)
}

func foo() {
	p0, y, z := createInt(), createInt(), createInt()
	var p1 = unsafe.Pointer(y)
	var p2 = uintptr(unsafe.Pointer(z))

	/*
    虽然 z 指向的 int值 的地址被 p2 记录，但是 p2 仅仅是一个整形变量，在下列操作中 
    不涉及 z 变量，也就是说 z 对应的内存空间不再被使用，随时可能被 gc 回收。
    因此对于 p2的值指向的数据做操作是危险的！
    */
	p2 += 2; p2--; p2--

	*p0 = 1                         // okay
	*(*int)(p1) = 2                 // okay
	*(*int)(unsafe.Pointer(p2)) = 3 // dangerous!
}
```
### 在运行时值的内存地址可能会变化
[Memory Blocks](https://go101.org/article/memory-block.html#where-to-allocate) 中提到一种情况:当 协程 的 栈 大小发生变化，会重新分配内存。也就是说，存放在栈上的数据在运行时内存地址可能发生变化。

### 一个值在运行时的生命周期可能没有它在代码中看起来那么大
```go
type T struct {
	x int
	y *[1<<23]byte
}

func bar() {
	t := T{y: new([1<<23]byte)}
	p := uintptr(unsafe.Pointer(&t.y[0]))

	... // use T.x and T.y

	// 聪明的编译器可以捕获到 t.y 不会再被使用，因此认为 t.y 占用的内存空间可以被回收
	// Using *(*byte)(unsafe.Pointer(p))) 这么使用是危险的

	// Continue using value t, but only use its x field.
	println(t.x)
}
```


### 如何正确使用 *T、uintptr、unsafe.Pointer?
#### 模式一：*T1 -> unsafe.Pointer -> *T2
* 需要满足条件：只有在T1的尺寸不小于T2并且此转换具有实际意义的时候才应该实施这样的转换(个人理解：只能尺寸小的或相等的类型T1->T2，否则T2的尺寸大于T1，会发生内存读取问题，因为一个大的指针指向了小的内存，有一部分是未知的)
```go
// float64 -> int64
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}
//int64 -> float64
func Float64frombits(b uint64) float64 {
	return *(*float64)(unsafe.Pointer(&b))
}

func main() {
	i64 := int64(1)
	f64 := (*float64)(unsafe.Pointer(&i64))
	fmt.Println(*f64) //5e-324

	f644 := float64(1)
	i644 := (*int64)(unsafe.Pointer(&f644))
	fmt.Println(*i644) //4607182418800017408

    i8 := int32(1)
	i32 := (*float32)(unsafe.Pointer(&i8))
	fmt.Println(*i32) //1e-45 有答案 但是不合法
}
```
#### 模式二：unsafe.Pointer -> uintptr -> 算术运算 -> unsafe.Pointer
* 整个过程需要保证原子性，否则会发生意向不到的 panic
```go
type T struct {
	x bool
	y [3]int16
}

const N = unsafe.Offsetof(T{}.y)
const M = unsafe.Sizeof(T{}.y[0])

func main() {
	t := T{y: [3]int16{123, 456, 789}}
	p := unsafe.Pointer(&t)
	// "uintptr(p)+N+M+M" is the address of t.y[2].
	ty2 := (*int16)(unsafe.Pointer(uintptr(p)+N+M+M)) // unsafe.Pointer(uintptr(p)+N+M+M) -> 推荐使用 unsafe.Add 方法完成
	fmt.Println(*ty2) // 789
}

/*
    ty2 := (*int16)(unsafe.Pointer(uintptr(p)+N+M+M)) 这一步不能拆分为两个步骤，因为会发生意想不到的panic，假如：
    
    addr := uintptr(p) + N + M + M
	
	// ... (some other operations)
	
	两个风险：
     1. 现在 t 是被没有被使用的变量，gc 随时可能回收这部分内存，因此下列操作 ty2 指向的内存可能是无效的
     2. 假如协程的栈大小发生变化，会导致栈重新分配内存并迁移数据，因此 addr 值指向的数据是无效的
	ty2 := (*int16)(unsafe.Pointer(addr))
*/

```
#### 模式三：将非类型安全指针值转换为uintptr值并传递给syscall.Syscall函数调用
模式二的操作需要原子性是因为无法保证 uintptr 的变量指向的内存空间不被 gc 回收。然而，syscall标准库包 中的 Syscall函数 有特殊的逻辑，编译器针对每个 syscall.Syscall函数 调用中的每个被转换为uintptr类型的非类型安全指针实参添加了一些指令，从而保证此非类型安全指针所引用着的内存块在此调用返回之前不会被垃圾回收和移动。 
```go
syscall.Syscall(syscall.SYS_READ, uintptr(fd),
			uintptr(unsafe.Pointer(p)), uintptr(n))
//安全调用
syscall.Syscall(syscall.SYS_READ, uintptr(fd),
			uintptr(unsafe.Pointer(p)), uintptr(n))

u := uintptr(unsafe.Pointer(p))
// 被p所引用着的值在此时有可能会被回收掉，
// 或者它的地址已经发生了改变。
syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))

// 相关实参必须呈现为"uintptr(anUnsafePointer)"
// 这种形式。事实上，Go 1.15之前，此调用是合法的；
// 但是Go 1.15略改了一点规则。
syscall.Syscall(SYS_XXX, uintptr(uintptr(fd)),
			uint(uintptr(unsafe.Pointer(p))), uintptr(n))
```


#### 模式四：将一个reflect.SliceHeader或者reflect.StringHeader值的Data字段转换为非类型安全指针，以及其逆转换
* 字符串 和 切片 的内部定义(运行时的描述类型)
```go
type _slice struct {
	elements unsafe.Pointer // 引用着底层的元素
	len      int            // 当前的元素个数
	cap      int            // 切片的容量
}
type _string struct {
	elements *byte // 引用着底层的byte元素
	len      int   // 字符串的长度
}
```
* reflect.SliceHeader 和 reflect.StringHeader 是 切片 和 字符串 运行时的运行时的描述，Data字段的类型被指定为uintptr，而不是unsafe.Pointer    
* 可以将 字符串的指针值 转换为一个 *reflect.StringHeader指针值，从而可以对此字符串的内部进行修改。 类似地，可以将一个 切片的指针值 转换为一个 *reflect.SliceHeader指针值，从而可以对此切片的内部进行修改。
```go
type StringHeader struct {
	Data uintptr
	Len  int
}

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

func main() {
	a := [...]byte{'G', 'o', 'l', 'a', 'n', 'g'}
	s := "Java"
	hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
    //修改指向的数据
	hdr.Data = uintptr(unsafe.Pointer(&a))
	hdr.Len = len(a)
	fmt.Println(s) // Golang
	// 现在，字符串s和切片a共享着底层的byte字节序列，
	// 从而使得此字符串中的字节变得可以修改。
	a[2], a[3], a[4], a[5] = 'o', 'g', 'l', 'e'
	fmt.Println(s) // Google
}

func main() {
	a := [6]byte{'G', 'o', '1', '0', '1'}
	bs := []byte("Golang")
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	hdr.Data = uintptr(unsafe.Pointer(&a))

	hdr.Len = 2
	hdr.Cap = len(a)
	fmt.Printf("%s\n", bs) // Go
	bs = bs[:cap(bs)]
	fmt.Printf("%s\n", bs) // Go101
}
/*
    我们只应该从一个已经存在的字符串值得到一个*reflect.StringHeader指针， 或者从一个已经存在的切片值得到一个*reflect.SliceHeader指针， 而不应该从一个全新的StringHeader值生成一个字符串，或者从一个全新的SliceHeader值生成一个切片。因为 StringHeader 和 SliceHeader 只是已经存在的 字符串 或者 切片 的一个描述(记录)，所以不能利用一个记录反向生成需要内存空间的实体 比如，下面的代码是不安全的： 
*/
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(new([5]byte)))
// 在此时刻，上一行代码中刚开辟的数组内存块已经不再被任何值
// 所引用，所以它可以被回收了。
hdr.Len = 5
s := *(*string)(unsafe.Pointer(&hdr)) // 危险！
```
* 字符串 -> 字节数据，零拷贝，本质上是不同的指针指向同一份数据
```go
//str 在运行过程中转化为 _string  []byte 转化为 _slice
func String2ByteSlice(str string) (bs []byte) {
    //改变描述类型中指针的指向
	strHdr := (*reflect.StringHeader)(unsafe.Pointer(&str))
	sliceHdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	sliceHdr.Data = strHdr.Data
	sliceHdr.Cap = strHdr.Len
	sliceHdr.Len = strHdr.Len
	return
}

func main() {
	// str := "Golang"
	// 对于官方标准编译器来说，上面这行将使str中的字节
	// 开辟在不可修改内存区。所以这里我们使用下面这行。
	str := strings.Join([]string{"Go", "land"}, "")
	s := String2ByteSlice(str)
	fmt.Printf("%s\n", s) // Goland
	s[5] = 'g'
	fmt.Println(str) // Golang
}
```

[Golang学习笔记--unsafe.Pointer和uintptr](https://studygolang.com/articles/33151)   
[Pointers in Go](https://gfw.go101.org/article/pointer.html)   
[Type-Unsafe Pointers](https://gfw.go101.org/article/unsafe.html)