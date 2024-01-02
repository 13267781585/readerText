# key过期删除策略

redis中对过期key采用`定期+惰性`策略。

## 惰性删除

在对key进行读写前后调用`expireIfNeeded`方法判断是否过期，会根据设置的过期key策略同步或异步(4.0+)清除。

```c
int expireIfNeeded(redisDb *db, robj *key) {
    //判断key是否过期，不过期直接返回
    if (!keyIsExpired(db,key)) return 0;

   //...

    //删除数据并做数据同步
    deleteExpiredKeyAndPropagate(db,key);
    return 1;
}

void deleteExpiredKeyAndPropagate(redisDb *db, robj *keyobj) {
    mstime_t expire_latency;
    //监控删除耗时
    latencyStartMonitor(expire_latency);
    //根据lazyfree_lazy_expire配置是否开启异步删除
    if (server.lazyfree_lazy_expire)
        dbAsyncDelete(db,keyobj);
    else
        dbSyncDelete(db,keyobj);
    latencyEndMonitor(expire_latency);
    latencyAddSampleIfNeeded("expire-del",expire_latency);
    //触发键通知空间
    notifyKeyspaceEvent(NOTIFY_EXPIRED,"expired",keyobj,db->id);
    signalModifiedKey(NULL, db, keyobj);
    //把删除命令写入aof文件并同步命令给从库
    propagateExpire(db,keyobj,server.lazyfree_lazy_expire);
    server.stat_expiredkeys++;
}
```

## 定期删除

redis定时任务会定期抽样扫描过期键，扫描频率由`hz`配置项决定(1s/hz)。分为两种模式，快速清除和慢速清除：

* 快速清除：短时间内不限制资源快速回收
* 慢速清除：长时间限制资源慢回收
正常情况下慢速清除模式可以满足回收需求，只有在过期key数量大，且慢速回收频繁超时，才会启动快速回收。

```c
/*
    有两种扫描模式：快速 or 慢速，主要是扫描时间的限制
*/
void activeExpireCycle(int type) {
    /* Adjust the running parameters according to the configured expire
     * effort. The default effort is 1, and the maximum configurable effort
     * is 10. */
    unsigned long
    //参数的微调参数
    effort = server.active_expire_effort-1, 
    //每次循环抽样的数据量 ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP->20
    config_keys_per_loop = ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP +
                           ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP/4*effort,
    //快速清理模式下每次清理的持续时间 ACTIVE_EXPIRE_CYCLE_FAST_DURATION->1000
    config_cycle_fast_duration = ACTIVE_EXPIRE_CYCLE_FAST_DURATION +
                                 ACTIVE_EXPIRE_CYCLE_FAST_DURATION/4*effort,
    //慢速清理模式下cpu最大使用率 ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC->25
    config_cycle_slow_time_perc = ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC +
                                  2*effort,
    //可接受的过期数据量占比，如果本次采样中过期数量小于这个阈值就结束本次清理。 ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE->10
    config_cycle_acceptable_stale = ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE-
                                    effort;

    /* This function has some global state in order to continue the work
     * incrementally across calls. */
    static unsigned int current_db = 0; /* Next DB to test. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    int j, iteration = 0;
    int dbs_per_call = CRON_DBS_PER_CALL;
    long long start = ustime(), timelimit, elapsed;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        //timelimit->是否因为超时退出，如果上一次快速清理不是因为超时退出且系统的过期数据比例小于设定的值，说明过期数据不多，直接返回
        if (!timelimit_exit &&
            server.stat_expired_stale_perc < config_cycle_acceptable_stale)
            return;
        // 如果距离上次快速清理时间小于两倍快速清理时间，直接返回
        if (start < last_fast_cycle + (long long)config_cycle_fast_duration*2)
            return;

        last_fast_cycle = start;
    }

    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    //用最大占用cpu转化为扫描时间限制，取决于配置项hz
    timelimit = config_cycle_slow_time_perc*1000000/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    // 如果是快速模式，超时时间设置1000毫秒
    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = config_cycle_fast_duration; /* in microseconds. */

    /* Accumulate some global stats as we expire keys, to have some idea
     * about the number of keys that are already logically expired, but still
     * existing inside the database. */
    long total_sampled = 0;
    long total_expired = 0;

    //遍历所有数据库或者超时退出循环
    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
        /* Expired and checked in a single loop. */
        unsigned long expired, sampled;

        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        current_db++;

        /* Continue to expire if at the end of the cycle there are still
         * a big percentage of keys to expire, compared to the number of keys
         * we scanned. The percentage, stored in config_cycle_acceptable_stale
         * is not fixed, but depends on the Redis configured "expire effort". */
        do {
            //字典数据的个数，字典槽的个数
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;
            iteration++;

            /* If there is nothing to expire try next DB ASAP. */
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            slots = dictSlots(db->expires);
            now = mstime();

            //如果槽的填充率小于 1%，直接跳过
            if (slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. */
            expired = 0;
            sampled = 0;
            ttl_sum = 0;
            ttl_samples = 0;

            if (num > config_keys_per_loop)
                num = config_keys_per_loop;

            //扫描数据个数是num个，但是字典中可能大部分槽都是空的，所以我们需要考虑到槽的数量
            long max_buckets = num*20;
            long checked_buckets = 0;
            //从上次结束的位置开始扫描 db->expires_cursor
            while (sampled < num && checked_buckets < max_buckets) {
                for (int table = 0; table < 2; table++) {
                    if (table == 1 && !dictIsRehashing(db->expires)) break;

                    unsigned long idx = db->expires_cursor;
                    idx &= db->expires->ht[table].sizemask;
                    dictEntry *de = db->expires->ht[table].table[idx];
                    long long ttl;

                    /* Scan the current bucket of the current table. */
                    checked_buckets++;
                    while(de) {
                        /* Get the next entry now since this entry may get
                         * deleted. */
                        dictEntry *e = de;
                        de = de->next;

                        ttl = dictGetSignedIntegerVal(e)-now;
                        //如果过期则清除数据
                        if (activeExpireCycleTryExpire(db,e,now)) expired++;
                        if (ttl > 0) {
                            /* We want the average TTL of keys yet
                             * not expired. */
                            ttl_sum += ttl;
                            ttl_samples++;
                        }
                        sampled++;
                    }
                }
                db->expires_cursor++;
            }
            total_expired += expired;
            total_sampled += sampled;

            /* Update the average TTL stats for this database. */
            if (ttl_samples) {
                long long avg_ttl = ttl_sum/ttl_samples;

                /* Do a simple running average with a few samples.
                 * We just use the current estimate with a weight of 2%
                 * and the previous estimate with a weight of 98%. */
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    server.stat_expired_time_cap_reached_count++;
                    break;
                }
            }
            //如果扫描的数据个数0或者过期键的比例不可接受，重复上述扫描
        } while (sampled == 0 ||
                 (expired*100/sampled) > config_cycle_acceptable_stale);
    }

    elapsed = ustime()-start;
    server.stat_expire_cycle_time_used += elapsed;
    latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);

    /* Update our estimate of keys existing but yet to be expired.
     * Running average with this sample accounting for 5%. */
    double current_perc;
    if (total_sampled) {
        current_perc = (double)total_expired/total_sampled;
    } else
        current_perc = 0;
    server.stat_expired_stale_perc = (current_perc*0.05)+
                                     (server.stat_expired_stale_perc*0.95);
}
```

```c
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
    //获取过期时间
    long long t = dictGetSignedIntegerVal(de);
    if (now > t) {
        sds key = dictGetKey(de);
        robj *keyobj = createStringObject(key,sdslen(key));
        deleteExpiredKeyAndPropagate(db,keyobj);
        decrRefCount(keyobj);
        return 1;
    } else {
        return 0;
    }
}
```

## 删除方式

对于过期数据的删除有两种方式：同步或异步。区别在于异步会使用后台线程异步释放内存，同步使用主线程释放内存，在清楚大对象时可能会有性能问题。

### 同步

```c
int dbSyncDelete(redisDb *db, robj *key) {
    //删除过期记录表中的数据
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    //将数据从key字典中移除(数据还在)
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        /* Tells the module that the key has been unlinked from the database. */
        moduleNotifyKeyUnlink(key,val);
        //清除key+val
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```

### 异步

在值对象清除成本比较大时，会采用后台线程执行，键还是同步清除。

```c
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    //删除过期记录表中的数据
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    //将数据从key字典中移除(数据还在)
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Tells the module that the key has been unlinked from the database. */
        moduleNotifyKeyUnlink(key,val);
        //计算清除的成本
        size_t free_effort = lazyfreeGetFreeEffort(key,val);

        //如果成本大于LAZYFREE_THRESHOLD(64)且共享计数器=1，创建异步任务清除值(不是key)
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            bioCreateLazyFreeJob(lazyfreeFreeObject,1, val);
            dictSetVal(db->dict,de,NULL);
        }
    }

    //如果上述逻辑符合异步条件，这里只会清除key，如果不符合，这里会同步清除key+val
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```

[Redis源码剖析与实战-17LazyFree会影响缓存替换吗？](https://www.oomspot.com/post/ouxiyushizhan17lazyfreehuiyingxianghuancuntihuanma)
