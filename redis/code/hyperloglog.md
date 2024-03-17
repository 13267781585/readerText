# HyperLogLog

使用小内存统计巨量数据

## 原理

* 基于`伯努利实验`：投掷一个硬币，直到出现正面，统计投掷的次数k，进行n组实验后，当n足够大，n=2^max(k)，为了提高准确率，会分组实验，取每组的max(k)计算调和平均数
* redis将输入的数值使用hash函数转化为64位二进制，低位的14位作为分组计算，高位50位计算k，共有2^14个桶，最后可以计算出去重后数据的数量。
* [HyperLogLog 算法的原理讲解以及 Redis 是如何应用它的](https://www.cnblogs.com/linguanh/p/10460421.html)
* [redis的HyperLogLog源码分析](https://zhuanlan.zhihu.com/p/368136350)
<img src="./image/30.jpg" alt="30" />

## 应用场景

统计基数大(几百上千万)，可以容忍少量不精确

* 结合bitmap使用，bitmap统计活跃的用户，hyperloglog统计数量

## 结构

```c
#define HLL_DENSE 0 /* Dense encoding. */
#define HLL_SPARSE 1 /* Sparse encoding. */
struct hllhdr {
    char magic[4];      /* "HYLL" */
    //编码 稀疏和密集
    uint8_t encoding;   
    //没有使用
    uint8_t notused[3]; 
    /*
        用小端字节序储存的64位整数，保存最近的基数计算结果。如果从上一次基数计算到现在，数据结构都没有修改过，card中的内容可以重新使用。redis使用card最高位表示该数据是否可用，如果最高位是1，表明数据被修改了，就需要重新计算基数并缓存到card中。
    */
    uint8_t card[8];    
    //数据寄存器 每个寄存器存放8bit
    uint8_t registers[]; /* Data bytes. */
};
```

## 编码

redis的HyperLogLog实现使用2^14(16384)个桶，每个桶6位，使用内存在12k左右，因为内存是宝贵的资源，所以在数据量比较小时，采用压缩算法，压缩registers数组，节约内存空间。

### 稀疏

#### 压缩方式

* ZERO操作码占用一个字节，表示为00xxxxxx，后六位xxxxxx+1表示有N个连续的寄存器设置为0，这个操作码可以表示1-64个连续的寄存器被设置为0。
* XZERO操作码占用两个字节，表示为01xxxxxx yyyyyyyy。xxxxxx是高位，yyyyyyyy是低位。这十四位+1表示有N个连续的寄存器设置为0.这个操作码可以表示0-16384个寄存器被设置为0。
* VAL操作码占用一个字节，表示为1vvvvvxx。它包含一个5bit的vvvvv表示寄存器值，2bit的xx+1表示有这么多个连续的寄存器被设置为vvvvv。这个操作码表示可以表示1-4个寄存器被设置1-32的值。

### 密集

### 稀疏转密集条件

* 桶需要表示的数值大于32，因为压缩方式最多表示的数值是32(1vvvvvxx，vvvvv五位表示最大数值32)
* 数据的长度大于3000字节(hll-sparse-max-bytes)
