## defer

#### 规则

* defer执行的延迟函数如果发生panic，不会干扰到其他延迟函数的执行(一个资源关闭出错不会导致其他资源的关闭)

    ```go
    func main() {
        defer fmt.Println("a")
        defer func() { panic("111") }()
        defer fmt.Println("b")
    }

    b
    a
    panic: 111
    ```

* defer 的规则
  * 延迟函数的参数在defer语句出现时就已经确定下来

    ```go
    func a() {
        i := 0
        defer fmt.Println(i)
        i++
        return
    }
    ```

    defer语句中的fmt.Println()参数i值在defer出现时就已经确定下来，实际上是拷贝了一份。后面对变量i的修改不会影响fmt.Println()函数的执行，仍然打印"0"。

    注意：对于指针类型参数，规则仍然适用，只不过延迟函数的参数是一个地址值，这种情况下，defer后面的语句对变量的修改可能会影响延迟函数。

    ```go
    func main() {
        a, b := 1, 2
        defer cal("1", a, cal("10", a, b))   //延迟函数的参数在defer语句时就会被确定，所以需要先执行cal("10", a, b)
        a = 0
        defer cal("2", a, cal("20", a, b))
    }

    func cal(index string, a, b int) int {
        ret := a + b
        fmt.Println(index, a, b, ret)
        return ret
    }

    // 10 1 2 3
    // 20 0 2 2
    // 2 0 2 2
    // 1 1 3 4
    ```

  * 延迟函数执行按后进先出顺序执行
  * 延迟函数可能操作主函数的具名返回值
    return不是一个原子操作，先将返回值存放到栈中，再执行延迟函数，最后跳转返回，所以延迟函数可能会影响返回值
    * 主函数拥有匿名返回值，返回字面值

        ```go
        func foo() int {
            var i int
        
            defer func() {
                i++
            }()
            // 延迟函数无法改变字面量
            return 1
        }
        ```

    * 主函数拥有匿名返回值，返回变量

        ```go
        func foo() int {
            var i int
        
            defer func() {
                //会把i+1
                i++
            }()
            //因为返回的是值类型，会复制，在返回时还没有+1，所以时0
            return i
        }
        ```

    * 主函数拥有具名返回值

        ```go
        func foo() (ret int) {
            defer func() {
                //指向同一个局部变量，修改会影响返回值
                ret++
            }()
        
            return 0
        }
        ```

#### 原理

defer是通过_defer结构体实现的，结构体中有_defer的指针指向下一个延迟函数，当增加延迟函数，会生成_defer结构体插入链表头部，最后执行也是从链表头部开始，所以表现为先进后执行。代码编译后新增延迟函数处会插入deferproc()函数，最后在ret指令(返回值)之前调用deferreturn函数执行延迟函数。

```go
//延迟函数结构体
// ./src/runtime/runtime2.go
type _defer struct {
 started bool
 heap    bool     // 是否为堆分配
 openDefer bool   //是否开启开放性编码

 sp        uintptr // 函数栈指针
 pc        uintptr // deferproc函数返回后要继续执行的指令地址
 fn        func()  // 延迟函数
 _panic    *_panic // 触发defer函数执行的panic指针，正常流程执行defer时它就是nil
 link      *_defer // 指向下一个延迟函数

 fd   unsafe.Pointer //开放性编码的一些信息
 varp uintptr        
 framepc uintptr
}


//./src/runtime/panic.go
func deferproc(fn func()) {
// 获取当前协程
 gp := getg()
//新建延迟函数结构
 d := newdefer()
 //插入链表头结点
 d.link = gp._defer
 gp._defer = d
 d.fn = fn
 d.pc = getcallerpc()
 d.sp = getcallersp()

 return0()
}


func deferreturn() {
//获取当前协程
 gp := getg()
 //从头结点循环执行延迟函数
 for {
  d := gp._defer
  if d == nil {
   return
  }
  sp := getcallersp()
  if d.sp != sp {
   return
  }
  //开放性编码
  if d.openDefer {
   done := runOpenDeferFrame(gp, d)
   if !done {
    throw("unfinished open-coded defers in deferreturn")
   }
   gp._defer = d.link
   freedefer(d)
   // If this frame uses open defers, then this
   // must be the only defer record for the
   // frame, so we can just return.
   return
  }
    //非开放性编码
  fn := d.fn
  d.fn = nil
  //删除延迟函数
  gp._defer = d.link
  freedefer(d)
  fn()
 }
}
```

#### defer性能降低的原因

* 使用defer需要注册和在函数返回之前执行延迟函数，相比直接调用函数，增了多更多的步骤
* 多个defer需要更多的内存空间记录信息
* defer函数只有在函数返回前才会被执行，若函数执行时间较长或者并发量大，会造成积累非常多的延迟函数
* defer结构体是分配在堆上，执行时copy到调用者的栈中执行
* defer延迟函数采用链表存放，链表节点在内存中散列分布，效率低

#### defer版本性能优化

* 1.12性能问题
    1. defer结构体分配到堆上(即使有deferpool减少堆内存的分配和回收，defer注册仍然需要分配到堆上，执行时复制到栈)

    2. defer结构体采用链表连接，效率低

* 1.13
将defer结构体分配到栈，减少堆的分配(不适用在for中的defer函数)，性能提升30%

```go
func A() {
    defer B(10)
    // code to do something
}
func B(i int) {
    //......
}

func A() {
    var d struct {
        runtime._defer
        i int
    }
    d.siz = 0
    d.fn = B
    d.i = 10
    r := runtime.deferprocStack(&d._defer)  //将结构体分配到栈上
    if r > 0 {
        goto ret
    }
    // code to do something
    runtime.deferreturn()
    return
ret:
    runtime.deferreturn()
}


```

* 1.14(延迟函数在栈中如何存放和定位的？)
优化了分配堆的缺点，使用 open coded 的方式，将延迟函数以普通函数调用的方式在主函数返回之前调用，省去了创建 _defer 结构体和链表的开销，在正常情况下延迟函数会被调用，当发生panic时，程序会终止panic后续逻辑执行，这是因为 panic程序 会通过扫描栈的方式保证延迟函数正确执行(如何扫描)，因此优化后defer的性能提升但是panic的处理过程变慢了。在下列条件中会禁止使用 open coded 的方法:
  * defer数量 <= 8个

  * return数量 * defer数量 <= 15

  * defer外部无循环(可以用匿名函数优化)

    ```go
    for {
    //...
    defer func() {}()
    }

        for{
            func(){
                //...
        defer func() {}()
            }()
        }

    ```

  * gcflags使用-N来禁止编译器优化

    ```go
    /*
        defer的开放编码优化方法会把defer函数转化为静态函数调用，并使用变量的bit记录函数调用的情况，调用成功会清除对应bit的标识，在panic后会扫描变量的bit，如果没有执行会补偿。这种方式不需要创建_defer结构体，减少了对堆的依赖，加快了defer的性能，但是panic后需要额外的扫描成本，降低了panic处理的性能。
    */
    func A(i int) {
        defer A1(i, 2*i)
        if(i > 1){
            defer A2("Hello", "eggo")
        }
        // code to do something
        return
    }
    func A1(a,b int){
        //......
    }
    func A2(m,n string){
        //......
    }

    //只是机制的模拟
    func A(i int){
        //Go1.14通过增加一个标识变量df来解决这类问题。用df中的每一位对应标识当前函数中的一个defer函数是否要执行。
        var df byte
        //A1的参数
        var a, b int = i, 2*i
        df |= 1

        //A2的参数
        var m,n string = "Hello", "eggo"
        if i > 1 {
            df |= 2
        }
        //code to do something
            
        //判断A2是否要调用
        if df&2 > 0 {
            //执行后清除标记位
            df = df&^2
            A2(m, n)
        }
        //判断A1是否要调用
        if df&1 > 0 {
            df = df&^1
            A1(a, b)
        }
        return
        //省略部分与recover相关的逻辑
    }

    //实际处理方式  
    //在 deferreturn中调用，通过_defer中fd和varp字段定位延迟函数并执行，panic后调用，扫描哪些defer函数没有被执行
    func runOpenDeferFrame(gp *g, d *_defer) bool {
    done := true
    fd := d.fd

    deferBitsOffset, fd := readvarintUnsafe(fd)
    nDefers, fd := readvarintUnsafe(fd)
    deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))  //延迟函数比特位，记录函数是否需要被执行

    for i := int(nDefers) - 1; i >= 0; i-- {
    // // 读取函数funcdata地址和参数信息
    var closureOffset uint32
    closureOffset, fd = readvarintUnsafe(fd)
    //不需要执行
    if deferBits&(1<<i) == 0 {
    continue
    }
    closure := *(*func())(unsafe.Pointer(d.varp - uintptr(closureOffset)))
    d.fn = closure
    deferBits = deferBits &^ (1 << i)
    *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
    p := d._panic
    //执行延迟函数
    deferCallSave(p, d.fn)
    if p != nil && p.aborted {
    break
    }
    d.fn = nil
    if d._panic != nil && d._panic.recovered {
    done = deferBits == 0
    break
    }
    }

    return done
    }

    func deferCallSave(p *_panic, fn func()) {
    if p != nil {
    p.argp = unsafe.Pointer(getargp())
    p.pc = getcallerpc()
    p.sp = unsafe.Pointer(getcallersp())
    }
    //执行延迟函数
    fn()
    if p != nil {
    p.pc = 0
    p.sp = unsafe.Pointer(nil)
    }
    }
    ```

#### defer何时处理的？

代码编译后会由.\src\cmd\compile\internal\ssagen\ssa.go根据不同条件处理defer

```go
func (s *state) stmt(n ir.Node) {
    //...
 case ir.ODEFER:
  n := n.(*ir.GoDeferStmt)
  //如果运行程序没有禁止编译优化 -N
  if base.Debug.Defer > 0 {
   var defertype string
   if s.hasOpenDefers {
       //开放编码
    defertype = "open-coded"
   } else if n.Esc() == ir.EscNever {
       //分配在栈上
    defertype = "stack-allocated"
   } else {
       //分配在堆上
    defertype = "heap-allocated"
   }
   base.WarnfAt(n.Pos(), "%s defer", defertype)
  }
  
  if s.hasOpenDefers {
   s.openDeferRecord(n.Call.(*ir.CallExpr))
  } else {
   d := callDefer
   if n.Esc() == ir.EscNever {
    d = callDeferStack
   }
   s.callResult(n.Call.(*ir.CallExpr), d)
  }

  //...

    }
```

[go语言系列7 - Go defer优化之open_coded](https://www.modb.pro/db/57017)  
[golang defer原理](https://blog.csdn.net/qq_49723651/article/details/121509818)
[Go defer实现原理剖析](https://blog.csdn.net/Tybyqi/article/details/83827140)  
[1.14版本defer性能大幅度提升，内部实现了开放编码优化](https://www.q578.com/s-5-2421209-0/)
