## chan

#### 底层数据结构

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

#### 接受数据

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
    3. chan 已关闭但是buf不为空
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
   if raceenabled {
    raceacquire(c.raceaddr())
   }
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
  if raceenabled {
   raceacquire(c.raceaddr())
  }
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
  if raceenabled {
   racenotify(c, c.recvx, nil)
  }
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
