# Type Concept
## Basic Types 基本类型
* string->string
* boolean->bool
* numeric->
    * int8, uint8 (byte), int16, uint16, int32 (rune), uint32, int64, uint64, int, uint, uintptr.
    * float32, float64.
    * complex64, complex128.   
上述 17 中基本类型也是预先定义类型(predeclared types)。

## Composite Types 复合类型
* pointer types
* struct types
* function types
* container types
    * array
    * slice
    * map
* channel types
* interface types

## Named Type & Unnamed Type 命名类型 & 未命名类型
### 命名类型(可以通过一个标识符被引用)
* 预先定义类型(string、boolean、numeric)
* 新定义类型(type a int)
* 被实例化的泛型
* 泛型中的参数  
### 未命名类型
除去上述的其他类型。    
原文如下:   
A named type may be

* a predeclared type;
* a defined (non-custom-generic) type;
* an instantiated type (of a generic type);
* a type parameter type (used in custom generics).

Other value types are called unnamed types. An unnamed type must be a composite type (not vice versa). 


## Underlying Type 基础类型
Go中的每一种类型都有一个基础类型：
* 对于内置的int、float32等类型，他们的基础类型是他们自己
* 对于 unsafe 包中的 Pointer 类型，他的基础类型也是他自己(Pointer被认为是一种内置类型)
* 未命名类型，也就是说复合类型的基础类型是他们自己
* 对于新定义的类型，基础类型和新定义类型的源类型相同(in a type declaration, the newly declared type and the source type have the same underlying type.)

```go
// underlying type int.
type (
	MyInt int
	Age   MyInt
)

type (
	IntSlice   []int   // unnamed type []int,underlying type is []int
	MyIntSlice []MyInt // unnamed type []MyInt,underlying type is []MyInt
	AgeSlice   []Age   // unnamed type []Age,underlying type is []Age
)

// The underlying types of []Age, Ages, and AgeSlice
// are all the unnamed type []Age.
type Ages AgeSlice
```
如何判断一个类型的基础类型？从上往下找到 内置类型 或者 未命名类型 时，就停止。
```go
MyInt → int
Age → MyInt → int
IntSlice → []int
MyIntSlice → []MyInt stop []int
AgeSlice → []Age x []MyInt stop []int
Ages → AgeSlice → []Age stop []MyInt → []int
```
[Go Type System Overview](https://gfw.go101.org/article/type-system-overview.html)