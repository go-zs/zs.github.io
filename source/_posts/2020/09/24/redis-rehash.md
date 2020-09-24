---
title: redis源码分析(2)--rehash
date: 2020-09-24 14:51:58
tags:
---

`Redis`底层数据结构是`hash`，`hash`在发生扩容、缩容的时候都会发生`rehash`。它的`rehash`不是一蹴而就的，而是一个渐近式的过程。

<!-- more -->

## dict

字典及hash表的定义如下

```C
// dict.h
// 字典
typedef struct dict {
    dictType *type;   //  类型
    void *privdata;
    dictht ht[2];     //  ht[0] 旧数组, ht[1]rehash后的数组
    long rehashidx;   // 当前rehash的索引位置，初始为0  -1表示未rehash
    unsigned long iterators; 
} dict;

// hash表
typedef struct dictht {
    dictEntry **table;
    unsigned long size;      // 数组长度
    unsigned long sizemask;  // sizemask=size-1,用于计算索引值
    unsigned long used;      // 已经用的节点数量
} dictht;

```

从源码我们可以知道，`dict`中存储了一个长度为2的`dictht`类型数组，用于记录，`rehash`前后的两个哈希表。


## rehash

```C
// dict.c
/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    // 正在rehash或者新的size小于当前使用的key的数量，返回错误
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    // 字典初始化
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    // ht[1]中存储新的hash
    d->ht[1] = n;
    // 记录rehash index
    d->rehashidx = 0;
    return DICT_OK;
}

int dictRehash(dict *d, int n) {
    // empty_visits 旧key迁移到新key的数量
    int empty_visits = n*10; 
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

`rehash`的做的事其实就是逐步将旧`hash`中的`key`迁移到新`hash`中。

## 操作

```C
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```
`_dictRehashStep`在dict的增删改查操作中都会被调用,每次调用都会触发一次`rehash`

```C
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}
```


## 定时器

`dictRehashMilliseconds`函数会进行`n`毫秒的`rehash`。

```C
int dictRehashMilliseconds(dict *d, int ms) {
    if (d->iterators > 0) return 0;

    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```

`databasesCron`是`redis`中的一个定时器，它会定时调用`dictRehashMilliseconds`进行`rehash`

```C
void databasesCron(void) {
    /* Expire keys by random sampling. Not required for slaves
     * as master will synthesize DELs for us. */
    if (server.active_expire_enabled) {
        if (iAmMaster()) {
            activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
        } else {
            expireSlaveKeys();
        }
    }

    /* Defrag keys gradually. */
    activeDefragCycle();

    /* Perform hash tables rehashing if needed, but only if there are no
     * other processes saving the DB on disk. Otherwise rehashing is bad
     * as will cause a lot of copy-on-write of memory pages. */
    if (!hasActiveChildProcess()) {
        /* We use global counters so if we stop the computation at a given
         * DB we'll be able to start from the successive in the next
         * cron loop iteration. */
        static unsigned int resize_db = 0;
        static unsigned int rehash_db = 0;
        int dbs_per_call = CRON_DBS_PER_CALL;
        int j;

        /* Don't test more DBs than we have. */
        if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;

        /* Resize */
        for (j = 0; j < dbs_per_call; j++) {
            tryResizeHashTables(resize_db % server.dbnum);
            resize_db++;
        }

        /* Rehash */
        if (server.activerehashing) {
            for (j = 0; j < dbs_per_call; j++) {
                int work_done = incrementallyRehash(rehash_db);
                if (work_done) {
                    /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
                    break;
                } else {
                    /* If this db didn't need rehash, we'll try the next one. */
                    rehash_db++;
                    rehash_db %= server.dbnum;
                }
            }
        }
    }
}
```



总结下，`redis`的`rehash`是在在字典的读写操作，以及定时事件中每次完成一定量的迁移。