# intset源码

## 结构
```c
typedef signed char    int8_t;
typedef unsigned int uint32_t;
typedef long long     int64_t;

typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
### 为什么encoding和length不使用花费更小的int去存储？   
猜测：
1. 考虑内存排列，32位和64位这两个变量的排列都不会影响到contents数据。

## 初始化
```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    //默认使用最小编码
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```

## 添加
* 是否需要升级编码
* 如果已经存在数据，直接返回
* 找到插入的位置，移动后续数据腾出空间插入数据
```c
// success:添加是否成功 1成功 0失败
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    //如果添加数据长度大于现有编码长度，升级set并添加
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    } else {
        //判断set中是否已经存在该元素
        //pos会被更新为插入的位置
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }
        //扩容长度
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //现有元素向后移
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
    //插入数据
    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    //当前编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    //新的编码
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    //从后往前迁移数据，为了不覆盖数据
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    //因为升级了编码，说明插入的数据要么最大，要么最小，通过数据的正负决定放在头部或者尾部
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

//因为是有序数据 采用二分查找法查询 
//pos:1.数据存在 pos=对应位置 2.边界情况判断数据不存在 pos=0 3.查询数据不存在 pos=set中<value的最大值
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    //过滤边界情况 length=0 || value<最小值 || value>最大值
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
    //二分查找法
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

## 删除
* 找到对应元素位置，移动后续数据覆盖删除数据
* 缩容
```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        is = intsetResize(is,len-1);
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```