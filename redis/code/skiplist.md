# 跳表

## 结构
```c
typedef struct zskiplistNode {
    //数据 char *->sds
    sds ele;
    //分值 用于排序
    double score;
    //前置节点
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        //后置节点指针
        struct zskiplistNode *forward;
        //和后一个节点间隔节点数，用于数据排名
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    //node数量
    unsigned long length;
    //最高层级
    int level;
} zskiplist;
```

## 初始化

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    // 头节点不存放数据 level层数最大32 头节点申请最大层级
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

## 随机层数

跳表最大层数32，循环每次给层数+1，概率为1/4，层数越高概率越小，这样设计使得底层节点更加密集，高层稀疏，提高搜索效率。

```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

## 插入

* 随机生成插入数据的层级
* 从高层向下寻找到新节点插入的位置，并记录插入位置前置节点+统计排名信息

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    //update每一层级插入位置的前置节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    //插入节点的前一个节点排名
    unsigned long rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        //排名初始化值，最开始从最高层的头节点开始遍历，排名为0，后续排名=为上一层停留节点的排名
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        //下一个节点部位空 && (下一个节点socre<当前插入节点score || (下一个节点socre==当前插入节点score，排序大的))
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            //排名加上对应节点和前置节点的跨度
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    //随机生成一个level
    level = zslRandomLevel();
    //如果生成的level>现有最大level，需要初始化超过部分数据，因为上述只初始化了现存最大level下的数据
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    //针对每一层级，插入节点
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /*  rank[0]:第一层前一个节点排名，+1相当于插入数据的排名
            rank[i]:第i层前一个节点排名
            (rank[0] - rank[i]):第0层前一个节点和i层前一个节点中间的节点数
            update[i]->level[i].span:第i层前一个节点和后一个节点跨度
            求插入节点到后一个节点的跨度=前一个节点跨度-前一个节点和插入节点的跨度
                                   =update[i]->level[i].span - (rank[0] - rank[i])
        */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    //前一个高于新节点的level的节点跨度+1
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

## 删除

```c
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;
    //找到对应删除节点的前置节点
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}

/*
    删除节点：
    1. 更新前节点指针、span
    2. 更新后节点指针
    3. 更新跳表length、level
*/  
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            //前节点span += 删除节点span
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            //说明删除节点没有这一层级
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    //删除的节点是最高level才需要更新level
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```

## 跳跃表API和复杂度

<img alt="9" src="./image/9.png"/>

## Question

### 跳表如何实现排名?(跳表span解析)

每个节点变量span保存这一层级中到下一个节点中间跨越的节点，在搜索的过程中，从上层到下层遍历，累加span直至找到目标节点，就可以得到目标数据的排名。
好处：每次增删节点不需要全量更新节点的排名，只需要变更前置节点的span变量即可。
[如何理解redis跳表源码中的span？](https://blog.csdn.net/qq_19648191/article/details/85381769)  

摘抄自《Redis设计与实现》
