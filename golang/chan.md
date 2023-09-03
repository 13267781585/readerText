# chan(1.18)

## 使用说明

### chan=nil

1. 获取数据 阻塞->协程挂起 非阻塞(select)->直接返回
2. 存入数据 阻塞->协程挂起 非阻塞(select)->直接返回
3. 关闭通道->panic

### chan已经被关闭

1. 获取数据->通道没有数据返回零值，false 通道有数据正常返回，true
2. 存入数据->panic
3. 重复关闭通道->panic

## 底层数据结构

```go
type hchan struct {
 qcount   uint           // 数组中元素的个数
 dataqsiz uint           // 底层数据的大小
 buf      unsafe.Pointer // 存放数据数组(缓存chan使用)
 elemsize uint16
 closed   uint32    // chan 是否关闭
 elemtype *_type // element type
 sendx    uint   // send index
 recvx    uint   // receive index
 recvq    waitq  // 接受队列
 sendq    waitq  // 发送队列
 // 保证安全锁
 lock mutex
}

```

## 接受数据

```go
<- chan

// 当出现错误返回对应类型零值，无法判断是正常读取还是返回类型的零值
func chanrecv1(c *hchan, elem unsafe.Pointer) {
 chanrecv(c, elem, true)
}

//返回bool 提示是否正常读取
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
 _, received = chanrecv(c, elem, true)
 return
}
/*
    ep->接受元素的地址
    block->是否阻塞
    1. chan == nil 非阻塞 -- 直接返回
                   阻塞  --- 挂起协程，知道关闭channel发生pannic
        返回 false，false
    2. 快速判断接受失败
       非阻塞 && 加下列任意情况：
       a.chan未被关闭 &&
        1. 非缓冲，发送队列为空
        2. 缓冲，底层数组元素为空
        返回 false，false
       b.chan关闭
        返回 true，false
    3. chan 已关闭但是buf不为空仍然会返回数据
        即使在chan关闭，buf中没有元素的情况下仍会返回类型零值
        返回 true，false

    4. 发送队列中有协程，可能是一下情况之一：
        1. 缓存chan，buf是满的 --- 返回循环数组头部元素，将发送的元素写入数组尾部
        2. 非缓冲chan  --- 将发送者的元素写入接收者的ep地址
        返回 true，true
    5. buf中有数据(针对缓存且没有等待发送的协程)
        直接读取数组头部数组
        返回 true，true
    
    6. chan中没有数据 && 非阻塞
        返回 false，false

    7. 阻塞
        记录接受的协程信息，挂起协程
*/
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
//如果chan 是nil 
 if c == nil {
    //不阻塞 直接返回
  if !block {
   return
  }
  // 阻塞 挂起协程
  gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
  throw("unreachable")
 }

 // 非阻塞且没有数据
 if !block && empty(c) {
  //通道没有关闭 直接返回
  if atomic.Load(&c.closed) == 0 {
   return
  }
  //通道关闭状态
  if empty(c) {
   //返回值没有被忽略 返回类型零值
   if ep != nil {
    typedmemclr(c.elemtype, ep)
   }
   return true, false
  }
 }

 var t0 int64
 if blockprofilerate > 0 {
  t0 = cputicks()
 }
 //加锁
 lock(&c.lock)
//通道被关闭且没有数据
 if c.closed != 0 && c.qcount == 0 {
  unlock(&c.lock)
  //返回类型零值
  if ep != nil {
   typedmemclr(c.elemtype, ep)
  }
  return true, false
 }

//有等待的发送协程
 if sg := c.sendq.dequeue(); sg != nil {
  // Found a waiting sender. If buffer is size 0, receive value
  // directly from sender. Otherwise, receive from head of queue
  // and add sender's value to the tail of the queue (both map to
  // the same buffer slot because the queue is full).
  recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
  return true, true
 }
 //缓冲的情况下 数组中有数据
 if c.qcount > 0 {
  // 获取头部元素
  qp := chanbuf(c, c.recvx)
  if ep != nil {
   typedmemmove(c.elemtype, ep, qp)
  }
  //清除循环数组中的数据
  typedmemclr(c.elemtype, qp)
  c.recvx++
  if c.recvx == c.dataqsiz {
   c.recvx = 0
  }
  c.qcount--
  unlock(&c.lock)
  return true, true
 }

//没有数据 如果是非阻塞的直接返回
 if !block {
  unlock(&c.lock)
  return false, false
 }

 // 获取协程信息记录
 gp := getg()
 mysg := acquireSudog()
 mysg.releasetime = 0
 if t0 != 0 {
  mysg.releasetime = -1
 }
 mysg.elem = ep
 mysg.waitlink = nil
 gp.waiting = mysg
 mysg.g = gp
 mysg.isSelect = false
 mysg.c = c
 gp.param = nil
 //链入接收队列
 c.recvq.enqueue(mysg)

 atomic.Store8(&gp.parkingOnChan, 1)
 //挂起协程
 gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

 // 协程被唤醒
 if mysg != gp.waiting {
  throw("G waiting list is corrupted")
 }
 gp.waiting = nil
 gp.activeStackChans = false
 if mysg.releasetime > 0 {
  blockevent(mysg.releasetime-t0, 2)
 }
 success := mysg.success
 gp.param = nil
 mysg.c = nil
 releaseSudog(mysg)
 return true, success
}

```

```go
// 发送队列中有协程，可能是一下情况之一：
//         1. 缓存chan，buf是满的 --- 返回循环数组头部元素，将发送的元素写入数组尾部
//         2. 非缓冲chan  --- 将发送者的元素写入接收者的ep地址
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
 if c.dataqsiz == 0 {
  // 非缓冲通道，buf数据长度为0，直接发阻塞的发送者数据复制给接收者
  if ep != nil {
   // copy data from sender
   recvDirect(c.elemtype, sg, ep)
  }
 } else {
  // 缓冲通道，buf满了，把队头数据复制给接收者，然后接受数据存入buf
  qp := chanbuf(c, c.recvx)
  // copy data from queue to receiver
  if ep != nil {
   typedmemmove(c.elemtype, ep, qp)
  }
  // copy data from sender to queue
  typedmemmove(c.elemtype, qp, sg.elem)
  c.recvx++
  if c.recvx == c.dataqsiz {
   c.recvx = 0
  }
  c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
 }
 sg.elem = nil
 gp := sg.g
 unlockf()
 gp.param = unsafe.Pointer(sg)
 sg.success = true
 if sg.releasetime != 0 {
  sg.releasetime = cputicks()
 }
 // 唤醒协程
 goready(gp, skip+1)
}
```

## 关闭通道

设置了通道关闭的标志，释放等待发送和接受的协程，不会释放通道底层buf数据结构。

```go
func closechan(c *hchan) {
 // 关闭nil的通道panic
 if c == nil {
  panic(plainError("close of nil channel"))
 }

 lock(&c.lock)
 // 重复关闭通道panic
 if c.closed != 0 {
  unlock(&c.lock)
  panic(plainError("close of closed channel"))
 }

 c.closed = 1

 var glist gList

 // 释放等待接受数据的协程
 for {
  sg := c.recvq.dequeue()
  if sg == nil {
   break
  }
  if sg.elem != nil {
   typedmemclr(c.elemtype, sg.elem)
   sg.elem = nil
  }
  if sg.releasetime != 0 {
   sg.releasetime = cputicks()
  }
  gp := sg.g
  gp.param = unsafe.Pointer(sg)
  sg.success = false
  glist.push(gp)
 }

 // 释放等待写入数据的协程(会panic)
 for {
  sg := c.sendq.dequeue()
  if sg == nil {
   break
  }
  sg.elem = nil
  if sg.releasetime != 0 {
   sg.releasetime = cputicks()
  }
  gp := sg.g
  gp.param = unsafe.Pointer(sg)
  sg.success = false
  glist.push(gp)
 }
 unlock(&c.lock)

 // Ready all Gs now that we've dropped the channel lock.
 for !glist.empty() {
  gp := glist.pop()
  gp.schedlink = 0
  goready(gp, 3)
 }
}
```

## 存入数据

```go
/*
    1. chan==nil 阻塞--协程挂起
                 非阻塞--直接返回
    2. 

*/
func chansend1(c *hchan, elem unsafe.Pointer) {
 chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
 // 对nil的通道发送数据
 if c == nil {
  // 非阻塞直接返回
  if !block {
   return false
  }
  // 阻塞挂起协程
  gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
  throw("unreachable")
 }
 // 非阻塞，通道没有关闭且满了情况下直接返回
 if !block && c.closed == 0 && full(c) { 
  return false
 }

 var t0 int64
 if blockprofilerate > 0 {
  t0 = cputicks()
 }

 lock(&c.lock)

 // 向关闭的通道存入数据，直接panic
 if c.closed != 0 {
  unlock(&c.lock)
  panic(plainError("send on closed channel"))
 }

 // 如果有协程等待读取数据，说明通道为非缓存或者没有数据，直接把数据发送给等待者并唤醒
 if sg := c.recvq.dequeue(); sg != nil {
  // Found a waiting receiver. We pass the value we want to send
  // directly to the receiver, bypassing the channel buffer (if any).
  send(c, sg, ep, func() { unlock(&c.lock) }, 3)
  return true
 }

 // 如果buf还没有满，把数据放入
 if c.qcount < c.dataqsiz {
  // Space is available in the channel buffer. Enqueue the element to send.
  qp := chanbuf(c, c.sendx)
  typedmemmove(c.elemtype, qp, ep)
  c.sendx++
  if c.sendx == c.dataqsiz {
   c.sendx = 0
  }
  c.qcount++
  unlock(&c.lock)
  return true
 }

 if !block {
  unlock(&c.lock)
  return false
 }

 // 记录协程信息并挂起
 gp := getg()
 mysg := acquireSudog()
 mysg.releasetime = 0
 if t0 != 0 {
  mysg.releasetime = -1
 }
 // No stack splits between assigning elem and enqueuing mysg
 // on gp.waiting where copystack can find it.
 mysg.elem = ep
 mysg.waitlink = nil
 mysg.g = gp
 mysg.isSelect = false
 mysg.c = c
 gp.waiting = mysg
 gp.param = nil
 c.sendq.enqueue(mysg)
 // Signal to anyone trying to shrink our stack that we're about
 // to park on a channel. The window between when this G's status
 // changes and when we set gp.activeStackChans is not safe for
 // stack shrinking.
 atomic.Store8(&gp.parkingOnChan, 1)
 gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
 // 保证ep不会被释放
 KeepAlive(ep)

 // someone woke us up.
 if mysg != gp.waiting {
  throw("G waiting list is corrupted")
 }
 gp.waiting = nil
 gp.activeStackChans = false
 closed := !mysg.success
 gp.param = nil
 if mysg.releasetime > 0 {
  blockevent(mysg.releasetime-t0, 2)
 }
 mysg.c = nil
 releaseSudog(mysg)
 // 协程被唤醒后通道关闭panic
 if closed {
  if c.closed == 0 {
   throw("chansend: spurious wakeup")
  }
  panic(plainError("send on closed channel"))
 }
 return true
}
```

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
 if sg.elem != nil {
  sendDirect(c.elemtype, sg, ep)
  sg.elem = nil
 }
 gp := sg.g
 unlockf()
 gp.param = unsafe.Pointer(sg)
 sg.success = true
 if sg.releasetime != 0 {
  sg.releasetime = cputicks()
 }
 goready(gp, skip+1)
}
```

[通道](https://gfw.go101.org/article/channel.html)
[通道用例大全](https://gfw.go101.org/article/channel-use-cases.html)
