# ziplist源码
>* ziplist是为了解决小数据使用quicklist浪费内存的问题(节点指针占用空间远远大于数据)  
>* ziplist没有预容量，每次增删都设计内存重新分配   
>* 有小概率会有连锁更新的风险

## 构成
<img alt="2" src="./image/2.png"/>
<img alt="3" src="./image/3.png"/>
<img alt="4" src="./image/4.png"/>
<img alt="7" src="./image/7.png"/>

<img alt="5" src="./image/5.png"/>

## 连锁更新
<img alt="6" src="./image/6.png"/>

## 初始化->压缩的字符串
源代码巧妙利用了define，用带有语义的变量代替冗杂的指针操作，大大提高了代码的可读性。通过指针位移操作，设置不用参数。
```c
//大小端转换 使用小端
#if (BYTE_ORDER == LITTLE_ENDIAN)
//...
#define intrev64ifbe(v) (v)
#else
//...
#define intrev64ifbe(v) intrev64(v)
#endif

//zlbytes+zltail+zllen字段长度
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))

//zlend字段长度
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

//转化zl指针指向zlbytes参数
#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))

//转化zl指针指向zltail参数
#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))

//转化zl指针指向zllen参数
#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))

//结束标志
#define ZIP_END 255        

unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    //设置zlbytes参数
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    //设置zltail参数
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    //设置zllen参数
    ZIPLIST_LENGTH(zl) = 0;
    //设置zlend参数
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

## 获取
```c
//用于方便ziplist操作的结构体
typedef struct zlentry {
    unsigned int prevrawlensize; //前一个数据的长度需要用多少字节表示 1 或者 5 字节
    unsigned int prevrawlen;     //前一个数据字节数
    unsigned int lensize;        //数据长度 1 2 5
    unsigned int len;            //数据的长度需要用多少字节表示
    unsigned int headersize;     //头部占用的长度 prevrawlensize+lensize->也就是元数据占用的字节数
    unsigned char encoding;      //数据的类型 整形 or 字节数组
    unsigned char *p;            //数据指针，指向previous_entry_length
} zlentry;
```
```c
//获取指针p指向的数据，若是字节数组，则将数组的指针存入sstr，长度存入slen，若是整数，则存入sval。
//p为空 || p指向尾端，返回0，否则返回1
unsigned int ziplistGet(unsigned char *p, unsigned char **sstr, unsigned int *slen, long long *sval) {
    zlentry entry;
    if (p == NULL || p[0] == ZIP_END) return 0;
    if (sstr) *sstr = NULL;

    //解析数据
    zipEntry(p, &entry); 
    //字节数组
    if (ZIP_IS_STR(entry.encoding)) {
        if (sstr) {
            *slen = entry.len;
            //移动*sstr指针到数据开始的位置
            *sstr = p+entry.headersize;
        }
    } else {
        if (sval) {
            //将整形数据存入sval中
            *sval = zipLoadInteger(p+entry.headersize,entry.encoding);
        }
    }
    return 1;
}

//安全模式 解析数据的信息到entry中 validate_prevlen->是否需要解析前一个节点是否超出范围
//会校验指针的合法范围
//返回值1:解析成功 0:解析异常
static inline int zipEntrySafe(unsigned char* zl, size_t zlbytes, unsigned char *p, zlentry *e, int validate_prevlen) {
    //数据的开始和结束范围
    unsigned char *zlfirst = zl + ZIPLIST_HEADER_SIZE;
    unsigned char *zllast = zl + zlbytes - ZIPLIST_END_SIZE;
//校验p指针是否超出范围
#define OUT_OF_RANGE(p) (unlikely((p) < zlfirst || (p) > zllast))
    //该函数只会访问previous_length和encoding字段解析数据，因此假设这两个字段采用最大的5字节，如果p+10不超出范围，可以快速解析数据
    if (p >= zlfirst && p + 10 < zllast) {
        ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
        ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
        ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
        e->headersize = e->prevrawlensize + e->lensize;
        e->p = p;
        /* We didn't call ZIP_ASSERT_ENCODING, so we check lensize was set to 0. */
        if (unlikely(e->lensize == 0))
            return 0;
        /* Make sure the entry doesn't reach outside the edge of the ziplist */
        if (OUT_OF_RANGE(p + e->headersize + e->len))
            return 0;
        /* Make sure prevlen doesn't reach outside the edge of the ziplist */
        if (validate_prevlen && OUT_OF_RANGE(p - e->prevrawlen))
            return 0;
        return 1;
    }

    /* Make sure the pointer doesn't reach outside the edge of the ziplist */
    if (OUT_OF_RANGE(p))
        return 0;

    /* Make sure the encoded prevlen header doesn't reach outside the allocation */
    ZIP_DECODE_PREVLENSIZE(p, e->prevrawlensize);
    if (OUT_OF_RANGE(p + e->prevrawlensize))
        return 0;

    /* Make sure encoded entry header is valid. */
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    e->lensize = zipEncodingLenSize(e->encoding);
    if (unlikely(e->lensize == ZIP_ENCODING_SIZE_INVALID))
        return 0;

    /* Make sure the encoded entry header doesn't reach outside the allocation */
    if (OUT_OF_RANGE(p + e->prevrawlensize + e->lensize))
        return 0;

    /* Decode the prevlen and entry len headers. */
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    e->headersize = e->prevrawlensize + e->lensize;

    /* Make sure the entry doesn't reach outside the edge of the ziplist */
    if (OUT_OF_RANGE(p + e->headersize + e->len))
        return 0;

    /* Make sure prevlen doesn't reach outside the edge of the ziplist */
    if (validate_prevlen && OUT_OF_RANGE(p - e->prevrawlen))
        return 0;

    e->p = p;
    return 1;
#undef OUT_OF_RANGE
}
```
```c
static inline void zipEntry(unsigned char *p, zlentry *e) {
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    assert(e->lensize != 0); /* check that encoding was valid. */
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}

//判断长度用1还是5字节表示 如果是5 首字节为254
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)

#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do {                     \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);                                  \
    if ((prevlensize) == 1) {                                                  \
        (prevlen) = (ptr)[0];                                                  \
    } else { /* prevlensize == 5，因为ptr是char数组，char占用1个字节，分别取1、2、3、4字节上的数值向右移动0、8、16、24相加还原出长度 */                                            
        (prevlen) = ((ptr)[4] << 24) |                                         \
                    ((ptr)[3] << 16) |                                         \
                    ((ptr)[2] <<  8) |                                         \
                    ((ptr)[1]);                                                \
    }                                                                          \
} while(0)

#define ZIP_STR_MASK 0xc0 //11000000
//解析数据的类型
#define ZIP_ENTRY_ENCODING(ptr, encoding) do {  \
    (encoding) = ((ptr)[0]); \
    //字节数据的编码只需要获取前两位即可
    if ((encoding) < ZIP_STR_MASK) (encoding) &= ZIP_STR_MASK; \
} while(0)

#define ZIP_STR_06B (0 << 6) //00000000
#define ZIP_STR_14B (1 << 6) //01000000
#define ZIP_STR_32B (2 << 6) //10000000

//0xc0->192->11000000 整形前两位是类型编码
#define ZIP_INT_16B (0xc0 | 0<<4) //11000000 int16_t
#define ZIP_INT_32B (0xc0 | 1<<4) //11010000 int32_t
#define ZIP_INT_64B (0xc0 | 2<<4) //11100000 int64_t
#define ZIP_INT_24B (0xc0 | 3<<4) //11110000 24有符号
#define ZIP_INT_8B 0xfe           //11111110 8位有符号

#define ZIP_INT_IMM_MIN 0xf1    /* 11110001  241*/
#define ZIP_INT_IMM_MAX 0xfd    /* 11111101  253*/

//解析数据占用的字节数
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    //字节数组
    if ((encoding) < ZIP_STR_MASK) {                                           \
        //判断使用多少字节记录长度并计算
        //使用1字节
        if ((encoding) == ZIP_STR_06B) {                                       \
            (lensize) = 1;                                                     \
            //需要去掉encoding编码的前两位 0x3f->00111111
            (len) = (ptr)[0] & 0x3f;                                           \
        //使用2字节
        } else if ((encoding) == ZIP_STR_14B) {                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        //使用5字节
        } else if ((encoding) == ZIP_STR_32B) {                                \
            (lensize) = 5;                                                     \
            (len) = ((uint32_t)(ptr)[1] << 24) |                               \
                    ((uint32_t)(ptr)[2] << 16) |                               \
                    ((uint32_t)(ptr)[3] <<  8) |                               \
                    ((uint32_t)(ptr)[4]);                                      \
        } else {                                                               \
            (lensize) = 0; /* bad encoding, should be covered by a previous */ \
            (len) = 0;     /* ZIP_ASSERT_ENCODING / zipEncodingLenSize, or  */ \
                           /* match the lensize after this macro with 0.    */ \
        }                                                                      \
    } else {                                                                   \
        //整形数据
        (lensize) = 1;                                                         \
        if ((encoding) == ZIP_INT_8B)  (len) = 1;                              \
        else if ((encoding) == ZIP_INT_16B) (len) = 2;                         \
        else if ((encoding) == ZIP_INT_24B) (len) = 3;                         \
        else if ((encoding) == ZIP_INT_32B) (len) = 4;                         \
        else if ((encoding) == ZIP_INT_64B) (len) = 8;                         \
        //存储0-12整数，长度部分存放的是数据 len=0
        else if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)   \
            (len) = 0; /* 4 bit immediate */                                   \
        else                                                                   \
            (lensize) = (len) = 0; /* bad encoding */                          \
    }                                                                          \
} while(0)                                                                        \

```

## 查询
```c
//zl压缩表 p指向数据 vstr查询的字节数组(字符串或者整形) vlen字节数组长度 skip跳跃查询
unsigned char *ziplistFind(unsigned char *zl, unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip) {
    int skipcnt = 0;
    unsigned char vencoding = 0;
    long long vll = 0;
    size_t zlbytes = ziplistBlobLen(zl);

    while (p[0] != ZIP_END) {
        struct zlentry e;
        unsigned char *q;
        //解析参数到entry
        assert(zipEntrySafe(zl, zlbytes, p, &e, 1));
        q = p + e.prevrawlensize + e.lensize;
        //判断是否需要跳过当前数据
        if (skipcnt == 0) {
            //字节数组
            if (ZIP_IS_STR(e.encoding)) {
                //比较两个字节数组是否相同
                if (e.len == vlen && memcmp(q, vstr, vlen) == 0) {
                    return p;
                }
            } else {
                //尝试把字符指针转化为整形 该逻辑只会被运行一次
                if (vencoding == 0) {
                    if (!zipTryEncoding(vstr, vlen, &vll, &vencoding)) {
                        /* If the entry can't be encoded we set it to
                         * UCHAR_MAX so that we don't retry again the next
                         * time. */
                        vencoding = UCHAR_MAX;
                    }
                    /* Must be non-zero by now */
                    assert(vencoding);
                }

                //比较两个整形是否相同
                if (vencoding != UCHAR_MAX) {
                    long long ll = zipLoadInteger(q, e.encoding);
                    if (ll == vll) {
                        return p;
                    }
                }
            }

            /* Reset skip count */
            skipcnt = skip;
        } else {
            /* Skip entry */
            skipcnt--;
        }

        /* Move to next entry */
        p = q + e.len;
    }

    return NULL;
}

//尝试将字符指针entry数据转化为整形v并把编码存入encoding
//返回值0:失败 1:成功
int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding) {
    long long value;

    if (entrylen >= 32 || entrylen == 0) return 0;
    //尝试把entry字节数组转化为整形
    if (string2ll((char*)entry,entrylen,&value)) {
        /* Great, the string can be encoded. Check what's the smallest
         * of our encoding types that can hold this value. */
        if (value >= 0 && value <= 12) {
            *encoding = ZIP_INT_IMM_MIN+value;
        } else if (value >= INT8_MIN && value <= INT8_MAX) {
            *encoding = ZIP_INT_8B;
        } else if (value >= INT16_MIN && value <= INT16_MAX) {
            *encoding = ZIP_INT_16B;
        } else if (value >= INT24_MIN && value <= INT24_MAX) {
            *encoding = ZIP_INT_24B;
        } else if (value >= INT32_MIN && value <= INT32_MAX) {
            *encoding = ZIP_INT_32B;
        } else {
            *encoding = ZIP_INT_64B;
        }
        *v = value;
        return 1;
    }
    return 0;
}
```

## 插入
### 流程
先计算出插入需要的各种参数，再根据插入后新的长度扩容字符数据，移动数据腾出插入的空间，最后设置插入的数据。
```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    //curlen现在的ziplist长度 reqlen插入的数据需要的长度 newlen插入后新的ziplist长度
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
    //前一个数据需要的字节数和数据长度
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    //插入数据的编码
    unsigned char encoding = 0;
    //插入整形数据
    long long value = 123456789; 
    zlentry tail;

    //计算prelensize和prelen
    //如果p不是指向末端，prelensize和prelen通过p指向数据previous_length得到
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        //指向末端，说明要插入队尾，prelensize和prelen通过最后一个节点计算
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
        }
    }

    //尝试把s指针数据转化为整形
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        //返回整形存储长度
        reqlen = zipIntSize(encoding);
    } else {
        //字节数组 直接赋值数组长度
        reqlen = slen;
    }
    //加上previous_length和encoding字段的长度
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    //插入数据后需要确保下一个数据的previous_length字段有足够长度容纳
    int forcelarge = 0;
    //zipPrevLenByteDiff->返回=插入数据需要1或5字节表示-下一个字段的previous_length字节数1或5 只有三种结果 -4、4、0、0 4的情况表示下一个字段的previous_length字节数需要扩容为5
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    //存储p和zl指针的偏移量，因为realloc操作可能在原有内存上扩容，也有可能申请新的内存，需要重新初始化p指针
    offset = p-zl;
    newlen = curlen+reqlen+nextdiff;
    zl = ziplistResize(zl,newlen);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        //将原有数据向后移动腾出插入空间，需要注意后一个节点的previ_length可能发生扩容，所以需要考虑移动的位置向前-nextdiff，并且移动长度+nextdiff
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        //更新下一个数据previous_length字段
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        //更新zltail字段
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        //对于zltail的值，只有插入数据和队尾节点中间有数据时，才需要加上nextdiff，否则不需要增加
        /* 1.中间无数据
            |----
              1
            zltail=0
            |------ -----(扩容)
               0      1
            现在增加了0号数据，zltail=zltail+reqlen=6
           2.中间有数据
            |---- ----
              1    2
            zltail=4
            |------ ----- ----
               0      1    2
            现在增加了0号数据，zltail=zltail+reqlen+nextdiff=4+6+1=11
        */
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        //更新zltail字段
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    //插入操作可能会导致后续节点发生连锁更新反应
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    //增加zllen
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```
```c
//设置previous_length字段-5字节
int zipStorePrevEntryLengthLarge(unsigned char *p, unsigned int len) {
    uint32_t u32;
    if (p != NULL) {
        //第一字节设置为254
        p[0] = ZIP_BIG_PREVLEN;
        u32 = len;
        memcpy(p+1,&u32,sizeof(u32));
        memrev32ifbe(p+1);
    }
    return 1 + sizeof(uint32_t);
}
//设置previous_length字段-5字节or1字节
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len) {
    if (p == NULL) {
        return (len < ZIP_BIG_PREVLEN) ? 1 : sizeof(uint32_t) + 1;
    } else {
        if (len < ZIP_BIG_PREVLEN) {
            p[0] = len;
            return 1;
        } else {
            return zipStorePrevEntryLengthLarge(p,len);
        }
    }
}
```

## 连锁更新
### 流程
先遍历计算出需要扩容的节点数量和长度，然后扩容list，将后面不需要扩容的节点向后移动，然后倒序遍历，移动需要扩容的节点。
```c
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
    zlentry cur;
    //前一个数据的数据长度，占用多少字节数，当前节点和zl的偏移量
    size_t prevlen, prevlensize, prevoffset; 
    size_t firstentrylen; /* Used to handle insert at head. */
    //curlen现在的长度
    size_t rawlen, curlen = intrev32ifbe(ZIPLIST_BYTES(zl));
    //还需要扩容的长度，需要扩容的节点计数器
    size_t extra = 0, cnt = 0, offset;
    size_t delta = 4; //如果需要更新，需要增加的4个字节数，扩容后5字节-原有1字节
    //尾节点
    unsigned char *tail = zl + intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl));

    /* Empty ziplist */
    if (p[0] == ZIP_END) return zl;

    //解析当前指针p指向数据信息到entry
    zipEntry(p, &cur);
    firstentrylen = prevlen = cur.headersize + cur.len;
    prevlensize = zipStorePrevEntryLength(NULL, prevlen);
    prevoffset = p - zl;
    p += prevlen;

    /* Iterate ziplist to find out how many extra bytes do we need to update it. */
    while (p[0] != ZIP_END) {
        //解析当前指针p指向数据信息到entry
        assert(zipEntrySafe(zl, curlen, p, &cur, 0));

        //如果长度没有变化，说明连锁更新结束
        if (cur.prevrawlen == prevlen) break;

        //如果节点previous_length的字节数>=前一个数据长度需要多少字节表示，说明不需要进行扩容且后续节点也不需要更新，只需要更新previous_length的数值
        if (cur.prevrawlensize >= prevlensize) {
            //可能都为1字节or5字节
            if (cur.prevrawlensize == prevlensize) {
                zipStorePrevEntryLength(p, prevlen);
            } else {
                //节点previous_length是5字节，更新高位len
                zipStorePrevEntryLengthLarge(p, prevlen);
            }
            break;
        }

        /* cur.prevrawlen means cur is the former head entry. */
        assert(cur.prevrawlen == 0 || cur.prevrawlen + delta == prevlen);

        //节点的长度
        rawlen = cur.headersize + cur.len;
        //节点需要扩容，扩容后的长度+4
        prevlen = rawlen + delta; 
        prevlensize = zipStorePrevEntryLength(NULL, prevlen);
        prevoffset = p - zl;
        p += rawlen;
        extra += delta;
        cnt++;
    }

    /* Extra bytes is zero all update has been done(or no need to update). */
    if (extra == 0) return zl;

    //更新zltail字段，需要判断extra是否包含tail，因为zltail是到尾节点开始位置
    if (tail == zl + prevoffset) {
        if (extra - delta != 0) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra-delta);
        }
    } else {
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
    }

    /*
        ｜---------------｜--------------｜
            offset       p
        ｜---------------｜--------------｜---------|
            offset       p                  extra
        ｜---------------｜---------|--------------｜
            offset       p   extra         
    */
    //p指向第一个不需要更新的节点
    offset = p - zl;
    zl = ziplistResize(zl, curlen + extra);
    p = zl + offset;
    //把不需要扩容的节点向后移动
    memmove(p + extra, p, curlen - offset - 1);
    p += extra;

    //依次移动数据并更新previous_length
    while (cnt) {
        zipEntry(zl + prevoffset, &cur); /* no need for "safe" variant since we already iterated on all these entries above. */
        rawlen = cur.headersize + cur.len;
        /* Move entry to tail and reset prevlen. */
        memmove(p - (rawlen - cur.prevrawlensize), 
                zl + prevoffset + cur.prevrawlensize, 
                rawlen - cur.prevrawlensize);
        p -= (rawlen + delta);
        if (cur.prevrawlen == 0) {
            /* "cur" is the previous head entry, update its prevlen with firstentrylen. */
            zipStorePrevEntryLength(p, firstentrylen);
        } else {
            /* An entry's prevlen can only increment 4 bytes. */
            zipStorePrevEntryLength(p, cur.prevrawlen+delta);
        }
        /* Forward to previous entry. */
        prevoffset -= cur.prevrawlen;
        cnt--;
    }
    return zl;
}
```


## 删除
### 流程
计算出要删除的字节，记录删除的边界节点，向前移动后续未被删除的节点，需要考虑是否将整个尾部删除。
```c
//为什么用 **p？
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p) {
    size_t offset = *p-zl;
    zl = __ziplistDelete(zl,*p,1);

    *p = zl+offset;
    return zl;
}

unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    //删除的长度，删除的数量
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    //删除的第一个节点，
    zlentry first, tail;
    size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));

    zipEntry(p, &first); 
    //将指针p移动到第一个不删除的节点
    for (i = 0; p[0] != ZIP_END && i < num; i++) {
        p += zipRawEntryLengthSafe(zl, zlbytes, p);
        deleted++;
    }

    assert(p >= first.p);
    totlen = p-first.p; /* Bytes taken by the element(s) to delete. */
    if (totlen > 0) {
        //删除后zltail的值
        uint32_t set_tail;
        if (p[0] != ZIP_END) {
            //删除的是中间几个节点，需要考虑后续节点的previous_length长度是否可以记录前一个节点的长度
            //计算节点长度需要用多少字节数表示(1or5)和p节点的previous_length使用字节(1or5)的差值->-4、4、0、0

            //这一步相当于扩容或者缩容了，如果-4则缩容，p向后移动4字节，4则扩容，p向前移动4字节，后续操作会从p位置开始复制
            p -= nextdiff;
            assert(p >= first.p && p<zl+zlbytes-1);
            zipStorePrevEntryLength(p,first.prevrawlen);

            /* Update offset for tail */
            set_tail = intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen;

            assert(zipEntrySafe(zl, zlbytes, p, &tail, 1));
            //如果尾节点和和first前一个节点中间有节点，需要考虑nextdiff对于zltail的影响，上面详细说过，不再赘述
            if (p[tail.headersize+tail.len] != ZIP_END) {
                set_tail = set_tail + nextdiff;
            }

            //将删除后面的节点向前移动
            size_t bytes_to_move = zlbytes-(p-zl)-1;
            memmove(first.p,p,bytes_to_move);
        } else {
            /*
            整个尾部被删除，不需要拷贝数据，重新申请内存即可
                               first.prevrawlen
                ｜---------------｜--------|----------|-｜
                zl             first   first.p      p 255 
            */
            set_tail = (first.p-zl)-first.prevrawlen;
        }

        /* Resize the ziplist */
        offset = first.p-zl;
        zlbytes -= totlen - nextdiff;
        zl = ziplistResize(zl, zlbytes);
        p = zl+offset;

        /* Update record count */
        ZIPLIST_INCR_LENGTH(zl,-deleted);

        /* Set the tail offset computed above */
        assert(set_tail <= zlbytes - ZIPLIST_END_SIZE);
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(set_tail);

        //删除也可能触发连锁更新
        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }
    return zl;
}
```
## Q&A
### 为什么选择插入首部，而不是尾部，插入尾部可以省去复制的成本和减少连锁扩容风险？
* 连锁扩容发生的几率较小
* 插入的数据被认为是更可能将来会访问的，所以存入首部，可以充分利用缓存局部性原理？   



[Redis源码分析（四）—— ziplist的设计与实现](https://blog.csdn.net/pcj_888/article/details/122227334)   
摘抄自《Redis设计与实现》