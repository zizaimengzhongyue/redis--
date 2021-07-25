# dict 结构定义
## 结构定义
dict 结构定义如下
```c
// dict 结构
typedef struct dict {
    dictType *type; // dict 类型信息，在 server 启东时默认初始化为 dbDictType，具体信息见下文
    void *privdata; //
    dictht ht[2];  // dict 哈希表结构，一般数据保存在 ht[0] 中，ht[1] 主要是在 rehash 过程中使用
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;

// dict hash table 
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

// dictEntry hash 桶，保存 key-val 值
// dictEntry key 为 dict 结构中 key 的值；v 为 value 的值，联合类型；next 通过拉链的方式解决 hash 冲突？
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
## dict 初始化
在 server.c 中会对 dict 结构进行初始化
```c
void initServer(void) {
    ...
    /* Create the Redis databases, and initialize other internal state. */
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL); // dbDictType 是 dict 的类型，包含了与类型相关的一系列函数
        ...
    }
    ...
}
```
dbDictType 结构定义如下
```c
/* Db->dict, keys are sds strings, vals are Redis objects. */
dictType dbDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictObjectDestructor,       /* val destructor */
    dictExpandAllowed           /* allow to expand */
};

```
## 写入数据
dictCreate 创建一个新的 dict 结构
```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d)); // zmalloc 是 redis 内部定义的内存申请方法，底层调用的是 malloc

    _dictInit(d,type,privDataPtr);
    return d;
}
/* Initialize the hash table */
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]); // ht 初始化
    _dictReset(&d->ht[1]);
    d->type = type; // server 启动时默认初始化为 dbDictType
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->pauserehash = 0;
    return DICT_OK;
}
```
增加一个 key-value 信息
```c
// dictAddRaw 会检查当前 key 是否存在，如果存在返回原来的 entry，如果不存在则创建新的 entry，写入 key 值，并把 entry 加入 ht
// dictSetVal 会把值写入 entry
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL); // 

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}
// dictAddRaw 先判断 key 是否存在，如果存在返回 null
// dictHashKey 为 dbDictType 中 hashFunction，计算 key 的 hash 值
// 如果 key 不存在，先为这个 key 申请空间再加入 hashtable，然后把 key 值写入 entry
// 这个函数返回值是一个 key 为目标写入 key 的 entry，但是还没有值或还是旧值
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry.
     * Insert the element in top, with the assumption that in a database
     * system it is more likely that recently added entries are accessed
     * more frequently. */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0]; // 在 dictRehash 过程中，新创建的 key 写入新的 hash table 中
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}
// _dictKeyIndex 搜索 key 的时候直接用 key&ht.sizemast 
// 设计一种 hash 算法，该算法中出现了两个元素 hash 到了同一个桶里，通过拉链法解决；此时在查找时怎么判断是否是同一个元素呢？
// 把原始数据的 key 或者基于另外一套算法生成的 key 放进去比较
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```
## dictRehash
在 redis 认为 dict.ht.used > dict.ht.size 且 dict.ht.used/dict.ht.size > dict_force_resize_ratio 的时候，会触发 dict_expand；dict_expand 执行之后，会标记开始 rehash；
dict_force_resize_ratio 值为 5
在 rehash 过程中，redis 占用的内存空间会增加；增加的空间是当前 2 的整数倍
代码如下：
```c
// 判断是否需要执行 dictExpand
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio) &&
        dictTypeExpandAllowed(d))
    {
        return dictExpand(d, d->ht[0].used + 1); // 这里的 d->ht[0].used + 1 并不是最终要生成的 ht 的 size，最终实际 expand 的 size 是第一个大于 d->ht[0].used + 1 的 2 的平方数
    }
    return DICT_OK;
}
// 需要扩张的 size 的计算方法
```c
static unsigned long _dictNextPower(unsigned long size)
{
    unsigned long i = DICT_HT_INITIAL_SIZE;

    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    while(1) {
        if (i >= size)
            return i;
        i *= 2;
    }
}
```
// 在执行 expand 操作最后，标记 rehashindex 为 0:
```c
int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
{
    if (malloc_failed) *malloc_failed = 0;

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

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

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
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
dictRehash 过程不是一次完成，而是分阶段，每次 rehash 一个桶或者遇到连续十个空桶就结束
```c
static void _dictRehashStep(dict *d) {
    if (d->pauserehash == 0) dictRehash(d,1);
}
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */ // 这里值默认为 10 
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
    // rehash 完成之后，替换 hash table 并 reset 
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