## 缓存淘汰算法

#### redis中的lru

https://redis.io/topics/lru-cache

redis使用lru来进行过期淘汰的策略有两种：allkeys-lru和volatile-lru。第一种是从所有key集合中按lru算法来淘汰key，第二种是从设置了过期时间的集合中来按lru算法进行淘汰。

首先redis在创建对象的时候，每个对象都会又一个lruclock，记录访问时间

```c
robj *createObject(int type, void *ptr) {

    robj *o = zmalloc(sizeof(*o));

    o->type = type;
    o->encoding = REDIS_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK(); 
    return o;
}
```
根据设置的淘汰策略（当然可以动态修改），进行内存释放。在redis.c的freeMemoryIfNeeded方法中，主要看下lru处理的分支。
```c
             /* volatile-lru and allkeys-lru policy */
            // 如果使用的是 LRU 策略，
            // 那么从一集 sample 键中选出 IDLE 时间最长的那个键
            else if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
                server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
            {
                struct evictionPoolEntry *pool = db->eviction_pool;  // 这里防止要淘汰的对象集合，按idle time升序排列

                while(bestkey == NULL) {
                    evictionPoolPopulate(dict, db->dict, db->eviction_pool);// 按配置的maxmemory_samples来筛选淘汰keys
                    /* Go backward from best to worst element to evict. */
                    for (k = REDIS_EVICTION_POOL_SIZE-1; k >= 0; k--) {
                        if (pool[k].key == NULL) continue;
                        de = dictFind(dict,pool[k].key);

                        /* Remove the entry from the pool. */
                        sdsfree(pool[k].key);
                        /* Shift all elements on its right to left. */
                        memmove(pool+k,pool+k+1,
                            sizeof(pool[0])*(REDIS_EVICTION_POOL_SIZE-k-1));
                        /* Clear the element on the right which is empty
                         * since we shifted one position to the left.  */
                        pool[REDIS_EVICTION_POOL_SIZE-1].key = NULL;
                        pool[REDIS_EVICTION_POOL_SIZE-1].idle = 0;

                        /* If the key exists, is our pick. Otherwise it is
                         * a ghost and we need to try the next element. */
                        if (de) {
                            bestkey = dictGetKey(de);
                            break;
                        } else {
                            /* Ghost... */
                            continue;
                        }
                    }
                }
            }
 ```
 
官网提到，redis的这种lru过期算法并不是精准的lru算法，不能准确淘汰掉idle时间最长的那个key，主要就是因为内存考虑设置了maxmemory_samples，当然增大这个值可以提高在lru过程中的准确性，但是相应的内存压力会比较大，这对redis server这种内存敏感型的存储来说并不一定合适。实际情况下，还是要根据业务情况分析key的访问情况，调整不同的参数，包括设置不同的过期策略。
 
 
 #### tomcat中使用的lru
 
 CsrfPreventionFilter,内部使用LruCache来进行请求判断。具体实现就是LinkedHashMap,使用双端链表。
 
 
 #### 其他
 
 一些改进的lru算法实现机制，比如multi-queue 或者lru-k，需要继续研究，找一下使用场景

 
 
