# dict源码
## 结构
<img src="./image/18.png" alt="18" />   
<img src="./image/19.png" alt="19" />
<img src="./image/20.png" alt="20" />  

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    //数据迁移的位置，没迁移时=-1
    long rehashidx;
    int16_t pauserehash; 
    /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
    //判断扩容
    int (*expandAllowed)(size_t moreMem, double usedRatio);
} dictType;

typedef struct dictht {
    dictEntry **table;
    //数组的内存=sizeof(dictEntry)*数组的长度
    unsigned long size;
    //size-1
    unsigned long sizemask;
    //数据的数量
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```



## 初始化
创建一个空的dict，不会预分配数组内存。
```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}

int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->pauserehash = 0;
    return DICT_OK;
}

static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

## 添加
添加只有在dict中不存在该key才会成功，插入前会判断是否需要扩容，如果dict正在扩容，会帮助rehash一个位置。
```c
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)
//插入数据要求dict中没有相同的key，否则会插入失败 0->插入成功 1->插入失败(可能是已经存在or..)
int dictAdd(dict *d, void *key, void *val)
{
    //若dict中没有该key，创建并返回；若有责返回NULL
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}


#define dictSetKey(d, entry, _key_) do { \
    //key复制函数不为空，复制后设置，否则直接设置
    if ((d)->type->keyDup) \
        (entry)->key = (d)->type->keyDup((d)->privdata, _key_); \
    else \
        (entry)->key = (_key_); \
} while(0)
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    //若在rehash，帮忙迁移数据
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    //头插法->因为最近加入的数据被访问的几率更大
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    //设置key
    dictSetKey(d, entry, key);
    return entry;
}
```
```c
//通过rehashidx值判断是否正在rehash
#define dictIsRehashing(d) ((d)->rehashidx != -1)
//查找d中是否有key的节点，如果有记录在existing中，没有则返回插入的位置
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    //判断是否需要扩容
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    //遍历表0和表1
    for (table = 0; table <= 1; table++) {
        //计算index
        idx = hash & d->ht[table].sizemask;
        //因为采用链地址法，遍历查询是否有该值
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                //查询到有相同key记录existing
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        //如果再rehash，需要查找表1，否则直接跳出
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}

//再插入之前判断是否需要进行扩容，需要则扩容，并返回是否成功，不需要直接返回成功 0->成功 1->失败
static int _dictExpandIfNeeded(dict *d)
{
    //正在扩容，直接返回
    if (dictIsRehashing(d)) return DICT_OK;

    //长度为0，扩容
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    //数据的数据大于长度的5倍 && dictTypeExpandAllowed
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio) &&
        dictTypeExpandAllowed(d))
    {
        return dictExpand(d, d->ht[0].used + 1);
    }
    return DICT_OK;
}

//todo expandAllowed
static int dictTypeExpandAllowed(dict *d) {
    if (d->type->expandAllowed == NULL) return 1;
    return d->type->expandAllowed(
                    _dictNextPower(d->ht[0].used + 1) * sizeof(dictEntry*),
                    (double)d->ht[0].used / d->ht[0].size);
}

#define DICT_HT_INITIAL_SIZE     4
//用于计算dict扩容需要申请的内存长度，申请的长度=第一个大于等于size的2的整数倍
static unsigned long _dictNextPower(unsigned long size)
{
    //最小申请的长度
    unsigned long i = DICT_HT_INITIAL_SIZE;

    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    while(1) {
        if (i >= size)
            return i;
        i *= 2;
    }
}
```

## 扩容
//初始化长度为4，每次扩容是原来的2倍。
```c
//size->现有节点数量+1
//如果malloc_failed不为空，因为内存申请失败导致扩容失败，malloc_failed会被置为1，否则为0
int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
{
    if (malloc_failed) *malloc_failed = 0;

    //如果已经在扩容或者size溢出，返回错误
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n;
    //计算要申请的dict长度
    unsigned long realsize = _dictNextPower(size);

    //检查申请长度是否溢出
    if (realsize < size || realsize * sizeof(dictEntry*) < realsize)
        return DICT_ERR;

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    if (malloc_failed) {
        n.table = ztrycalloc(realsize*sizeof(dictEntry*));
        *malloc_failed = n.table == NULL;
        if (*malloc_failed)
            return DICT_ERR;
    } else
        n.table = zcalloc(realsize*sizeof(dictEntry*));

    n.used = 0;

    //表0为空代表第一次初始化，不需要设置rehashidx，直接返回成功
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```

## rehash
```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

## Q&A
### 渐进式扩容好处
耗时操作平均分配到每个操作中，防止客户端被长时间阻塞。

### 头插法原因
因为最近插入的数据被访问的几率更大。