# NFS-Ganesha 缓存系统 - 关键代码详解

---

## 1. 哈希表分区结构详解

### 源文件：[src/include/hashtable.h](src/include/hashtable.h)

**分区设计：**

```c
/**
 * @brief Represents an individual partition
 *
 * This structure holds the per-subtree data making up each partition in
 * a hash table.
 */
struct hash_partition {
    size_t count;                    /* 此分区内的条目数 */
    struct rbt_head rbt;             /* 红黑树头 */
    pthread_rwlock_t ht_lock;        /* 分区级锁（读写锁） */
    struct rbt_node **cache;         /* 期望条目缓存（加速命中） */
};

/**
 * @brief A hash table
 *
 * This structure defines an entire hash table.
 */
typedef struct hash_table {
    struct hash_param parameter;     /* 参数（哈希函数、大小等） */
    pool_t *node_pool;               /* 红黑树节点池 */
    pool_t *data_pool;               /* 键值对数据池 */
    struct hash_partition partitions[]; /* 动态大小的分区数组 */
} hash_table_t;
```

**双重哈希流程：**

```c
/* 步骤 1：计算分区索引 */
index = hash_func_key(params, key_buffer) % index_size

/* 步骤 2：计算 RBTree 内哈希值 */
rbthash = hash_func_rbt(params, key_buffer)

/* 步骤 3：在分区[index]的红黑树中查找 */
partition = &ht->partitions[index]
PTHREAD_RWLOCK_rdlock(&partition->ht_lock)
    node = rbt_find(&partition->rbt, rbthash, key_buffer)
PTHREAD_RWLOCK_unlock(&partition->ht_lock)
```

**缓存页优化：**

```c
/* 计算缓存页大小（基于 cache_entry_count） */
static inline size_t cache_page_size(const hash_table_t *ht) {
    return (ht->parameter.cache_entry_count) * sizeof(struct rbt_node *);
}

/* 计算偏移量 */
static inline int cache_offsetof(struct hash_table *ht, uint64_t rbthash) {
    return rbthash % ht->parameter.cache_entry_count;
}
```

---

## 2. MDCache 内部结构

### 源文件：[src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h)

**导出结构：**

```c
struct mdcache_fsal_export {
    struct fsal_export mfe_exp;         /* 导出基础结构 */
    char *name;                         /* 导出名称 */
    
    /** My up_ops 向量 */
    struct fsal_up_vector up_ops;
    
    /** Higher level up_ops for ops we don't consume */
    struct fsal_up_vector super_up_ops;
    
    /** The list of cache entries belonging to this export */
    struct glist_head entry_list;
    
    /** Lock protecting entry_list */
    pthread_mutex_t mdc_exp_lock;
    
    /** Flags for the export. */
    uint8_t flags;
    
    /** Mapping of ck -> name for whence-is-name（目录条目映射） */
    mdc_dirmap_t dirent_map;
    
    /** Thread for dirmap processing */
    struct fridgethr *dirmap_fridge;
    
    /** Count of entries pending cleanup for this export
     *  before release. Incremented in unexport/unmount for each 
     *  entry we mark; decremented in mdcache_lru_clean when that 
     *  entry is fully cleaned. Always >= 0.
     */
    int32_t cleanup_pending;
};
```

**缓存键结构：**

```c
/**
 * @brief Structure representing a cache key.
 *
 * Wraps an underlying FSAL-specific key.
 */
typedef struct mdcache_key {
    uint64_t hk;              /* hash key（用于快速哈希） */
    void *fsal;               /* sub-FSAL module pointer */
    struct gsh_buffdesc kv;   /* fsal handle buffer */
} mdcache_key_t;

/* 键比较函数（用于红黑树） */
static inline int mdcache_key_cmp(
    const struct mdcache_key *k1,
    const struct mdcache_key *k2) {
    /* 快速路径：比较哈希值 */
    if (likely(k1->hk < k2->hk))
        return -1;
    if (likely(k1->hk > k2->hk))
        return 1;

    /* 长度比较 */
    if (unlikely(k1->kv.len < k2->kv.len))
        return -1;
    if (unlikely(k1->kv.len > k2->kv.len))
        return 1;

    /* FSAL 指针比较 */
    if (unlikely(k1->fsal < k2->fsal))
        return -1;
    if (unlikely(k1->fsal > k2->fsal))
        return 1;

    /* 深层内存比较（最后手段） */
    return memcmp(k1->kv.addr, k2->kv.addr, k1->kv.len);
}
```

**目录条目映射（DirentMap）：**

```c
typedef struct mdcache_dmap_entry__ {
    /** AVL node in tree by cookie */
    struct avltree_node node;
    
    /** Entry in LRU */
    struct glist_head lru_entry;
    
    /** Cookie value */
    uint64_t ck;
    
    /** Entry name */
    char *name;
    
    /** Timestamp on entry */
    struct timespec timestamp;
} mdcache_dmap_entry_t;

typedef struct {
    /** Lock protecting this structure */
    pthread_mutex_t dm_mtx;
    
    /** Mapping of ck -> name for whence-is-name 
     *  (AVL 树保证 O(log n) 查找)
     */
    struct avltree map;
    
    /** LRU of dirent map entries */
    struct glist_head lru;
    
    /** Count of entries in LRU */
    uint32_t count;
} mdc_dirmap_t;
```

---

## 3. LRU 多级队列实现

### 源文件：[src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h) 和 [mdcache_lru.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c)

**LRU 队列 ID：**

```c
enum lru_q_id {
    LRU_ENTRY_NONE = 0,     /* entry not queued */
    LRU_ENTRY_L1,           /* 热数据（最近使用） */
    LRU_ENTRY_L2,           /* 冷数据（较少使用） */
    LRU_ENTRY_CLEANUP,      /* 待清理队列（异步清理） */
    LRU_ENTRY_ACTIVE,       /* 活跃引用队列（refcnt > 0） */
};

/* 标志定义 */
#define LRU_CLEANUP 0x00000001      /* Entry is on cleanup queue */
#define LRU_CLEANED 0x00000002      /* Entry has been cleaned */
#define LRU_EVER_PROMOTED 0x00000004 /* Will promote after last ref released */
#define LRU_SENTINEL_HELD 0x00000008 /* Sentinel reference is held */
```

**LRU 条目结构：**

```c
typedef struct mdcache_lru__ {
    /** Link in the physical deque implementing a portion 
     *  of the logical LRU.
     */
    struct glist_head q;
    
    /** Queue identifier */
    enum lru_q_id qid;
    
    /** Reference count. This is signed to make mistakes easy to see. */
    int32_t refcnt;
    
    /** Active Reference count. This is signed to make mistakes easy to see. */
    int32_t active_refcnt;
    
    /** Status flags; MUST use atomic ops */
    uint32_t flags;
    
    /** The lane in which an entry currently resides, so we can lock 
     *  the deque and decrement the correct counter when moving or 
     *  deleting the entry.
     */
    uint32_t lane;
    
    /** Confounder（防止哈希冲突） */
    uint32_t cf;
} mdcache_lru_t;
```

**LRU 队列车道结构：**

```c
/**
 * A single queue lane, holding all entries.
 */
struct lru_q_lane {
    struct lru_q L1;         /* L1 队列（新条目） */
    struct lru_q L2;         /* L2 队列（验证过的条目） */
    struct lru_q cleanup;    /* 清理队列（待异步清理） */
    struct lru_q ACTIVE;     /* 活跃队列（有引用的条目） */
    pthread_mutex_t ql_mtx;  /* 保护此车道的锁 */

    CACHE_PAD(0);  /* 缓存行对齐，避免伪共享 */
};

/**
 * A single queue structure.
 */
struct lru_q {
    struct glist_head q;     /* LRU 在 HEAD，MRU 在 TAIL */
    enum lru_q_id id;        /* 队列标识 */
    uint64_t size;           /* 队列大小 */
};
```

**LRU 状态结构：**

```c
struct lru_state {
    uint64_t entries_hiwat;        /* 缓存高水位（条目数） */
    uint64_t entries_used;         /* 当前使用的条目数 */
    uint32_t entries_release_size; /* 每次淘汰的条目数 */
    
    uint64_t chunks_hiwat;         /* 分块高水位 */
    uint64_t chunks_lowat;         /* 分块低水位 */
    uint64_t chunks_used;          /* 当前使用的分块数 */
    
    uint32_t per_lane_work;        /* 每条车道的工作量 */
    time_t prev_time;              /* 上次 GC 线程运行时间 */
};

extern struct lru_state lru_state;
```

**关键常数：**

```c
/** Cache entries pool */
extern pool_t *mdcache_entry_pool;

/** The number of lanes comprising a logical queue. 
 *  This must be prime. */
#define LRU_N_Q_LANES 17

/** The minimum reference count for a cache entry not being recycled. */
#define LRU_SENTINEL_REFCOUNT 1

/** Reference type Flags for functions in the LRU package */
#define LRU_ACTIVE_REF 0x0004     /* 活跃引用 */
#define LRU_PROMOTE 0x0008        /* 晋升标志 */
#define LRU_FLAG_SENTINEL 0x0001  /* 哨兵标志 */
#define LRU_TEMP_REF 0x0002       /* 临时引用 */
```

---

## 4. 缓存统计数据

### 源文件：[mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h) 和 [mdcache_main.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_main.c)

**统计结构：**

```c
/**
 * MDCACHE statistics.
 */
struct mdcache_stats {
    uint64_t inode_req;     /* inode 请求总数 */
    uint64_t inode_hit;     /* inode 缓存命中数 */
    uint64_t inode_miss;    /* inode 缓存未命中数 */
    uint64_t inode_conf;    /* inode 冲突数（键相同但值不同） */
    uint64_t inode_added;   /* 新添加的 inode 数 */
    uint64_t inode_mapping; /* inode 映射数 */
};
```

**全局声明（mdcache_main.c）：**

```c
/* 统计数据实例 */
struct mdcache_stats cache_st;
struct mdcache_stats *cache_stp = &cache_st;

/* 条目内存池 */
pool_t *mdcache_entry_pool;

/* 初始化 */
int mdcache_param_set_from_file(const char *filename) {
    if (mdcache_entry_pool)
        pool_destroy(mdcache_entry_pool);
    
    mdcache_entry_pool = 
        pool_init(
            "mdcache_entry_pool",
            sizeof(mdcache_entry_t),
            pool_flags);
    
    if (!mdcache_entry_pool) {
        LogCrit(COMPONENT_MDCACHE,
                "Could not allocate mdcache entry pool");
        return -1;
    }
    
    return 0;
}
```

---

## 5. 命中率指标实现

### 源文件：[src/monitoring/dynamic_metrics.cc](src/monitoring/dynamic_metrics.cc)

**Prometheus 指标定义：**

```cpp
class DynamicMetrics {
public:
    DynamicMetrics(prometheus::Registry &registry);

    /* === MDCache Hit Rate Metrics === */
    
    // 全局命中数计数器
    CounterInt::Family &mdcacheCacheHitsTotal;
    
    // 全局未命中数计数器
    CounterInt::Family &mdcacheCacheMissesTotal;
    
    // 按导出分组的命中数计数器
    CounterInt::Family &mdcacheCacheHitsByExportTotal;
    
    // 按导出分组的未命中数计数器
    CounterInt::Family &mdcacheCacheMissesByExportTotal;

    // ... 其他指标 ...
};

/* 构造函数初始化 */
DynamicMetrics::DynamicMetrics(prometheus::Registry &registry)
    // === MDCache Metrics ===
    , mdcacheCacheHitsTotal(
        prometheus::Builder<CounterInt>()
            .Name("mdcache_cache_hits_total")
            .Help("Counter for total cache hits in mdcache.")
            .Register(registry))
    
    , mdcacheCacheMissesTotal(
        prometheus::Builder<CounterInt>()
            .Name("mdcache_cache_misses_total")
            .Help("Counter for total cache misses in mdcache.")
            .Register(registry))
    
    , mdcacheCacheHitsByExportTotal(
        prometheus::Builder<CounterInt>()
            .Name("mdcache_cache_hits_by_export_total")
            .Help("Counter for total cache hits in mdcache, by export.")
            .Register(registry))
    
    , mdcacheCacheMissesByExportTotal(
        prometheus::Builder<CounterInt>()
            .Name("mdcache_cache_misses_by_export_total")
            .Help("Counter for total cache misses in mdcache, by export.")
            .Register(registry))
    
    // ... 其他初始化 ...
{
}
```

**指标标签结构：**

```cpp
/* 典型的 Prometheus 标签化格式 */

/* 全局指标 */
mdcache_cache_hits_total{operation="lookup"}
mdcache_cache_misses_total{operation="lookup"}

/* 按导出分组 */
mdcache_cache_hits_by_export_total{export_id="1", operation="getattr"}
mdcache_cache_misses_by_export_total{export_id="1", operation="getattr"}
```

---

## 6. 缓存查询实现

### 源文件：[src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c)

**查询函数模板（简化版）：**

```c
/**
 * @brief Lookup a name in the cache
 * 
 * @param[in] parent   Parent directory entry
 * @param[in] name     Name to look up
 * @param[in] new_entry Whether this is a new lookup
 * @param[out] entry   Found entry (if success)
 * @param[out] attrs   Returned attributes
 *
 * @return FSAL status
 */
fsal_status_t mdc_lookup(mdcache_entry_t *parent,
                         const char *name,
                         bool new_entry,
                         mdcache_entry_t **entry,
                         struct fsal_attrlist *attrs) {
    fsal_status_t status;
    mdcache_entry_t *mdc_entry = NULL;

    /* 第1步：在缓存中查找 */
    status = mdcache_find_keyed(parent, name, 0, &mdc_entry);
    
    if (FSAL_IS_SUCCESS(status) && mdc_entry) {
        /* 缓存命中 */
        *entry = mdc_entry;
        
        /* 记录命中统计 */
        dynamic_metrics__mdcache_cache_hit(
            "lookup",              /* 操作名 */
            op_ctx->export->export_id  /* 导出 ID */
        );
        
        /* 可选：刷新属性 */
        if (attrs) {
            fsal_copy_attrs(attrs, &mdc_entry->attrs, false);
        }
        
        return FSAL_IS_SUCCESS;
    }
    
    /* 第2步：缓存未命中，查询下层 FSAL */
    dynamic_metrics__mdcache_cache_miss(
        "lookup",
        op_ctx->export->export_id
    );
    
    /* 调用下层 FSAL 的 lookup */
    struct fsal_obj_handle *sub_handle = NULL;
    struct fsal_attrlist sub_attrs;
    
    fsal_prepare_attrs(&sub_attrs, ATTR_MASK_ALL);
    
    subcall(status = parent->sub_handle->obj_ops->lookup(
        parent->sub_handle,
        name,
        &sub_handle,
        &sub_attrs));
    
    if (FSAL_IS_ERROR(status)) {
        fsal_release_attrs(&sub_attrs);
        return status;
    }
    
    /* 第3步：将结果插入缓存 */
    status = mdcache_new_entry(
        mdc_cur_export(),
        sub_handle,
        &sub_attrs,
        false,
        attrs,      /* 返回给调用者的属性 */
        false,      /* not a directory */
        &mdc_entry,
        NULL,       /* no state */
        LRU_ACTIVE_REF);
    
    fsal_release_attrs(&sub_attrs);
    
    if (FSAL_IS_ERROR(status)) {
        return status;
    }
    
    /* 可选：添加到目录条目缓存 */
    if (get_readdir_mode() == FSAL_RDDIR_CHUNK_ALWAYS) {
        bool invalidate = false;
        mdcache_dirent_add(parent, name, mdc_entry, &invalidate);
    }
    
    *entry = mdc_entry;
    return FSAL_IS_SUCCESS;
}
```

---

## 7. 引用计数管理 API

### 源文件：[mdcache_lru.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c)

**获取缓存条目：**

```c
/**
 * @brief Get a logical reference to a cache entry
 *
 * Increment reference count and manage LRU state transitions.
 * 
 * @param[in] entry   Cache entry being referenced
 * @param[in] flags   Set of flags to specify type of reference
 *
 * Flags:
 *   LRU_ACTIVE_REF   - Active request reference
 *   LRU_PROMOTE      - Promote after releasing active ref
 *   LRU_FLAG_SENTINEL - Sentinel reference
 *   LRU_TEMP_REF     - Temporary reference
 */
void _mdcache_lru_ref(mdcache_entry_t *entry,
                      uint32_t flags,
                      const char *func,
                      int line) {
    mdcache_lru_t *lru = &entry->lru;
    struct lru_q_lane *lane = &lru_lanes[lru->lane];
    
    /* 增加引用计数 */
    QLOCK(lane);
    
    if (flags & LRU_ACTIVE_REF) {
        atomic_inc_int32_t(&lru->active_refcnt);
    }
    
    atomic_inc_int32_t(&lru->refcnt);
    
    /* 处理晋升标志 */
    if (flags & LRU_PROMOTE) {
        atomic_set_uint32_t_bits(&lru->flags, LRU_EVER_PROMOTED);
    }
    
    QUNLOCK(lane);
    
    LogDebugAlt(COMPONENT_MDCACHE, COMPONENT_MDCACHE_LRU,
                "Ref entry %p: refcnt=%d, active_refcnt=%d (%s:%d)",
                entry, lru->refcnt, lru->active_refcnt, func, line);
}

#define mdcache_lru_ref(e, f) _mdcache_lru_ref(e, f, __func__, __LINE__)
```

**释放缓存条目：**

```c
/**
 * @brief Release logical reference to a cache entry
 *
 * Decrement reference count and manage LRU state transitions.
 * If refcnt reaches 0, entry may be recycled.
 * 
 * @param[in] entry Cache entry being returned
 * @param[in] flags Flag to indicate what kind of reference is to be released
 *
 * @return true if entry was recycled/freed, false if still in use
 */
bool _mdcache_lru_unref(mdcache_entry_t *entry,
                        uint32_t flags,
                        const char *func,
                        int line) {
    mdcache_lru_t *lru = &entry->lru;
    struct lru_q_lane *lane = &lru_lanes[lru->lane];
    bool freed = false;
    
    QLOCK(lane);
    
    /* 减少活跃引用 */
    if (flags & LRU_ACTIVE_REF) {
        atomic_dec_int32_t(&lru->active_refcnt);
        
        /* 检查是否需要晋升 */
        if (lru->active_refcnt == 0 &&
            test_mde_flags(entry, LRU_EVER_PROMOTED)) {
            /* 从 L1 转移到 L2 */
            lru_q_move(lru, L1, L2);
        }
    }
    
    /* 减少总引用计数 */
    atomic_dec_int32_t(&lru->refcnt);
    
    /* 检查是否应该回收 */
    if (lru->refcnt == 0) {
        /* 条目可以回收 */
        lru_q_move(lru, lru->qid, LRU_ENTRY_CLEANUP);
        freed = true;
    }
    
    QUNLOCK(lane);
    
    LogDebugAlt(COMPONENT_MDCACHE, COMPONENT_MDCACHE_LRU,
                "Unref entry %p: refcnt=%d, freed=%s (%s:%d)",
                entry, lru->refcnt, freed ? "true" : "false", func, line);
    
    return freed;
}

#define mdcache_lru_unref(e, f) _mdcache_lru_unref(e, f, __func__, __LINE__)
```

**插入活跃队列：**

```c
/**
 * @brief Insert entry into ACTIVE queue
 *
 * Used when entry has active references and should not be
 * considered for eviction.
 *
 * @param[in] entry Cache entry
 */
void mdcache_lru_insert_active(mdcache_entry_t *entry) {
    mdcache_lru_t *lru = &entry->lru;
    struct lru_q_lane *lane = &lru_lanes[lru->lane];
    
    QLOCK(lane);
    
    if (lru->qid != LRU_ENTRY_NONE) {
        /* 从原队列移除 */
        glist_del(&lru->q);
    }
    
    /* 添加到活跃队列 */
    glist_add_tail(&lane->ACTIVE.q, &lru->q);
    lru->qid = LRU_ENTRY_ACTIVE;
    
    QUNLOCK(lane);
}
```

---

## 8. 异步 LRU 淘汰线程

### 源文件：[src/FSAL/commonlib.c](src/FSAL/commonlib.c)

**FD LRU 管理：**

```c
/* 全局 FD LRU 链表 */
struct glist_head fsal_fd_global_lru = GLIST_HEAD_INIT(fsal_fd_global_lru);

/* LRU 运行间隔 */
time_t lru_run_interval;

/* FD LRU 线程指针 */
static struct fridgethr *fd_lru_fridge;

/* LRU 状态 */
struct fd_lru_state {
    uint32_t fds_hiwat;        /* FD 高水位 */
    uint32_t fds_lowat;        /* FD 低水位 */
    uint32_t futility;         /* 无效淘汰计数 */
};

static struct fd_lru_state fd_lru_state;

/**
 * @brief Execute LRU reclamation on a single FD entry
 *
 * @return Number of FDs reclaimed
 */
uint32_t lru_try_one(void) {
    struct fsal_fd *fsal_fd = NULL;
    
    /* 获取 LRU 队列末端条目（最少使用的） */
    fsal_fd = glist_last_entry(&fsal_fd_global_lru,
                               struct fsal_fd,
                               fd_lru);
    
    if (!fsal_fd)
        return 0;
    
    /* 尝试原子标记为回收中 */
    atomic_inc_int32_t(&fsal_fd->lru_reclaim);
    
    /* 尝试关闭文件描述符 */
    int status = close(fsal_fd->fd);
    
    if (status != 0) {
        /* 关闭失败，恢复标志 */
        atomic_dec_int32_t(&fsal_fd->lru_reclaim);
        return 0;
    }
    
    /* 从 LRU 链表移除 */
    glist_del(&fsal_fd->fd_lru);
    fsal_fd->fd = -1;
    
    /* 减少原子计数 */
    atomic_dec_int32_t(&fsal_fd->lru_reclaim);
    
    return 1;  /* 回收了1个FD */
}

/**
 * @brief Function that executes in the fd_lru thread
 *
 * This thread is responsible for keeping the count of open
 * file descriptors within configurable bounds.
 *
 * @param[in] ctx Fridge thread context
 */
void fd_lru_run(struct fridgethr_context *ctx) {
    uint32_t current_open;
    uint32_t fds_avg;
    bool extremis;
    uint32_t futility_count = 50;
    time_t threadwait = lru_run_interval;
    
    SetNameFunction("fd_lru");
    
    while (!fridgethr_should_exit(ctx)) {
        /* 计算平均值（高水位 - 低水位）/ 2 */
        fds_avg = (fd_lru_state.fds_hiwat - fd_lru_state.fds_lowat) / 2;
        
        /* 获取当前开启的 FD 数 */
        current_open = getOpenFDs();
        
        /* 检查是否处于紧急状态 */
        extremis = current_open > fd_lru_state.fds_hiwat;
        
        LogFullDebug(COMPONENT_FSAL,
                     "FD LRU awakes: current_open=%u, hiwat=%u, lowat=%u",
                     current_open, fd_lru_state.fds_hiwat,
                     fd_lru_state.fds_lowat);
        
        /* 执行淘汰循环 */
        uint32_t reclaimed = 0;
        while (current_open >= fd_lru_state.fds_lowat) {
            uint32_t reclaim_per_iteration = 5;
            uint32_t reclaimed_iter = 0;
            
            /* 每次尝试回收多个 FD */
            for (uint32_t i = 0; i < reclaim_per_iteration; ++i) {
                reclaimed_iter += lru_try_one();
            }
            
            if (reclaimed_iter == 0) {
                /* 无法回收更多 FD */
                fd_lru_state.futility++;
                
                if (fd_lru_state.futility >= futility_count) {
                    LogEvent(COMPONENT_FSAL,
                             "FD LRU giving up, cannot reduce FDs further");
                    break;
                }
            } else {
                fd_lru_state.futility = 0;
                reclaimed += reclaimed_iter;
            }
            
            current_open = getOpenFDs();
        }
        
        LogDebugAlt(COMPONENT_FSAL, COMPONENT_NONE,
                    "FD LRU reclaimed %u FDs", reclaimed);
        
        /* 等待下一个周期 */
        threadwait = lru_run_interval;
        if (extremis)
            threadwait = 1;  /* 紧急状态下缩短间隔 */
        
        fridgethr_sleep(ctx, threadwait);
    }
}
```

---

## 9. 缓存条目生命周期

```
┌─────────────────────────────────────────────────────────┐
│  缓存条目生命周期                                        │
└─────────────────────────────────────────────────────────┘

┌─────────┐
│ 初始化  │ mdcache_new_entry()
│ refcnt=1│
└────┬────┘
     │
     v
┌─────────────────────┐
│   LRU_ENTRY_ACTIVE  │  (活跃）
│   refcnt >= 1       │  有活跃请求
│ active_refcnt >= 1  │  在 ACTIVE 队列
└────┬────────────────┘
     │
     │ mdcache_lru_unref()
     │ (active_refcnt --> 0)
     v
┌──────────────────┐
│ LRU_ENTRY_L1     │  (热数据）
│ refcnt >= 1      │  最近使用
│ active_refcnt=0  │  在 L1 队列
└────┬─────────────┘
     │
     │ 时间流逝
     │ 更新操作？
     v
┌──────────────────┐
│ LRU_ENTRY_L2     │  (冷数据）
│ refcnt >= 1      │  检查：开放文件？锁？
│ active_refcnt=0  │  在 L2 队列
└────┬─────────────┘
     │
     │ 继续不使用
     │ LRU 末端
     v
┌──────────────────┐
│ 回收资格         │  refcnt 达到哨兵值
│ refcnt = 1       │  (LRU_SENTINEL_REFCOUNT)
│ 准备清理         │
└────┬─────────────┘
     │
     │ mdcache_lru_release_entries()
     │ 或 mdcache_lru_cleanup_push()
     v
┌──────────────────────┐
│ LRU_ENTRY_CLEANUP    │  清理队列
│ 执行清理操作         │  释放资源
│ 关闭子 FSAL 句柄     │  删除 dirent
└────┬─────────────────┘
     │
     │ 清理完成
     v
┌──────────────────┐
│  释放内存         │  pool_free()
│  entry == NULL   │  条目死亡
└──────────────────┘
```

---

## 10. 命中率计算示例

### Prometheus 查询

**全局命中率（5 分钟）：**

```promql
rate(mdcache_cache_hits_total[5m]) / 
(rate(mdcache_cache_hits_total[5m]) + rate(mdcache_cache_misses_total[5m]))
```

**按操作分组的命中率：**

```promql
sum by (operation) (rate(mdcache_cache_hits_total[5m])) /
sum by (operation) (rate(mdcache_cache_hits_total[5m]) + rate(mdcache_cache_misses_total[5m]))
```

**按导出的命中率：**

```promql
sum by (export_id) (rate(mdcache_cache_hits_by_export_total[5m])) /
sum by (export_id) (rate(mdcache_cache_hits_by_export_total[5m]) + 
                    rate(mdcache_cache_misses_by_export_total[5m]))
```

### Python 计算示例

```python
#!/usr/bin/env python3
"""
NFS-Ganesha 缓存命中率计算示例
"""

import requests
from urllib.parse import urljoin

class GaneshaMetrics:
    def __init__(self, endpoint="http://localhost:8000"):
        self.endpoint = endpoint
        self.metrics_url = urljoin(endpoint, "/metrics")
    
    def fetch_metrics(self):
        """获取 Prometheus 指标"""
        response = requests.get(self.metrics_url)
        return response.text
    
    def parse_metric(self, metrics_text, metric_name):
        """解析指标值"""
        for line in metrics_text.split('\n'):
            if line.startswith(metric_name):
                return float(line.split()[-1])
        return 0.0
    
    def calculate_hit_rate(self):
        """计算缓存命中率"""
        metrics = self.fetch_metrics()
        
        hits = self.parse_metric(
            metrics, 
            'mdcache_cache_hits_total'
        )
        misses = self.parse_metric(
            metrics,
            'mdcache_cache_misses_total'
        )
        
        total = hits + misses
        if total == 0:
            return 0.0
        
        hit_rate = (hits / total) * 100
        return hit_rate
    
    def print_stats(self):
        """打印统计信息"""
        metrics = self.fetch_metrics()
        
        hits = self.parse_metric(metrics, 'mdcache_cache_hits_total')
        misses = self.parse_metric(metrics, 'mdcache_cache_misses_total')
        total = hits + misses
        
        if total == 0:
            print("No cache activity")
            return
        
        hit_rate = (hits / total) * 100
        
        print(f"Cache Statistics")
        print(f"  Hits:       {int(hits):,}")
        print(f"  Misses:     {int(misses):,}")
        print(f"  Total:      {int(total):,}")
        print(f"  Hit Rate:   {hit_rate:.2f}%")

if __name__ == "__main__":
    metrics = GaneshaMetrics()
    metrics.print_stats()
```

---

## 总结

NFS-Ganesha 的缓存系统通过以下关键技术实现高效率：

1. **分区哈希表** - O(log n) 查询，分散锁竞争
2. **多级 LRU** - L1/L2 队列，适应工作集大小
3. **引用计数** - 安全的内存生命周期管理
4. **原子操作** - 无锁或低锁设计
5. **异步淘汰** - 避免同步延迟
6. **Prometheus 集成** - 实时监控指标

这些机制共同提供了一个高性能、可扩展的元数据缓存系统。
