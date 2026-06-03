# NFS-Ganesha 源码架构详细梳理

> 时间: 2026-05-16  
> 分析对象: NFS-Ganesha 缓存系统、算法和命中率计算机制

---

## 目录
1. [项目整体架构](#1-项目整体架构)
2. [缓存模块详情](#2-缓存模块详情)
3. [缓存算法深析](#3-缓存算法深析)
4. [命中率计算实现](#4-命中率计算实现)
5. [关键数据结构](#5-关键数据结构)

---

## 1. 项目整体架构

### 1.1 核心模块层次结构

```
NFS-Ganesha
│
├── MainNFSD/                    # NFS 服务主程序
│   ├── nfs_main.c              # 主程序入口
│   ├── nfs_init.c              # 初始化模块
│   ├── nfs_metrics.c           # 度量统计
│   ├── nfs_admin_thread.c      # 管理线程
│   ├── nfs_rpc_dispatcher_thread.c   # RPC 分发线程
│   └── nfs_worker_thread.c     # 工作线程
│
├── FSAL/                        # 文件系统访问层 (File System Abstraction Layer)
│   ├── FSAL_VFS/               # VFS 后端（Linux VFS）
│   ├── FSAL_CEPH/              # Ceph 后端
│   ├── FSAL_GLUSTER/           # Gluster 后端
│   ├── FSAL_GPFS/              # GPFS 后端
│   ├── FSAL_PROXY_V3/          # NFSv3 代理
│   ├── FSAL_PROXY_V4/          # NFSv4 代理
│   ├── FSAL_PSEUDO/            # 伪文件系统（导出树）
│   ├── Stackable_FSALs/
│   │   └── FSAL_MDCACHE/       # **元数据缓存（关键模块）**
│   │       ├── mdcache_int.h          # 内部接口定义
│   │       ├── mdcache_hash.h         # 哈希表实现
│   │       ├── mdcache_lru.h          # LRU 算法定义
│   │       ├── mdcache_avl.h          # AVL 树定义
│   │       ├── mdcache_main.c         # 主实现
│   │       ├── mdcache_lru.c          # LRU 算法实现
│   │       ├── mdcache_hash.c         # 哈希查询实现
│   │       ├── mdcache_handle.c       # 文件句柄处理
│   │       ├── mdcache_helpers.c      # 辅助函数
│   │       └── mdcache_read_conf.c    # 配置读取
│   └── common/
│       ├── access_check.c
│       ├── fsal_config.c
│       └── fsal_convert.c
│
├── SAL/                         # 状态抽象层 (State Abstraction Layer)
│   ├── state_lock.c
│   ├── state_deleg.c            # 委托状态
│   └── state_*.c                # 各种状态管理
│
├── Protocols/                   # 协议层
│   ├── NFS/                     # NFS 协议实现
│   │   ├── nfs3_*.c             # NFSv3 协议处理
│   │   ├── nfs4_*.c             # NFSv4 协议处理
│   │   └── nfs_rpc_*.c          # RPC 处理
│   ├── NLM/                     # 网络锁定管理
│   ├── NFSACL/                  # NFS ACL
│   └── 9P/                      # 9P 协议支持
│
├── hashtable/                   # **通用哈希表实现**
│   └── hashtable.c              # RBTree 分区哈希表
│
├── monitoring/                  # **监控和度量模块**
│   ├── dynamic_metrics.cc       # Prometheus 指标
│   └── include/
│       └── dynamic_metrics.h    # 动态指标定义
│
└── support/                     # 支持库和工具
    ├── log/                     # 日志系统
    ├── include/                 # 通用头文件
    └── os/                      # OS 抽象层
```

### 1.2 架构分层说明

| 层次 | 组件 | 功能 | 关键文件 |
|------|------|------|--------|
| **协议层** | NFS, NLM, 9P | NFS 协议处理和命令分发 | `Protocols/NFS/nfs_*_*.c` |
| **状态层** | SAL | 文件锁、委托、开启状态管理 | `SAL/state_*.c` |
| **缓存层** | MDCACHE | 元数据缓存（基于哈希表+LRU）| `FSAL_MDCACHE/*` |
| **文件系统层** | FSAL | 底层文件系统访问抽象 | `FSAL/FSAL_*/*` |
| **核心层** | MainNFSD | 服务初始化、线程管理、主循环 | `MainNFSD/nfs_*.c` |

---

## 2. 缓存模块详情

### 2.1 MDCache 模块概述

**MDCACHE** (MetaData CACHE) 是一个可堆叠的 FSAL 模块，位于其他 FSAL 之上，提供透明的元数据缓存。

**关键特性：**
- 基于哈希表的高效查询
- LRU 多级队列淘汰策略
- 支持目录条目（dirent）缓存
- 支持属性过期时间管理
- 原子操作和并发控制

### 2.2 MDCache 核心文件清单

| 文件 | 行数 | 功能描述 |
|------|------|--------|
| [mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h) | ~200 | **内部接口和数据结构定义** |
| [mdcache_hash.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_hash.h) | ~100 | 哈希表和分区定义 |
| [mdcache_lru.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h) | ~150 | LRU 队列和引用计数管理 |
| [mdcache_avl.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_avl.h) | ~50 | AVL 树定义（目录条目映射） |
| [mdcache_main.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_main.c) | ~1000+ | 初始化、配置管理 |
| [mdcache_lru.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c) | ~2000+ | **LRU 算法核心实现** |
| [mdcache_hash.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_hash.c) | ~500+ | 哈希表查询和管理 |
| [mdcache_handle.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_handle.c) | ~2000+ | 文件句柄操作 |
| [mdcache_helpers.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c) | ~1500+ | 辅助函数和缓存查询 |

### 2.3 哈希表实现

**文件:** [src/hashtable/hashtable.c](src/hashtable/hashtable.c)

**关键描述：**
```c
/**
 * @brief Implement an RBTree-based partitioned hash lookup
 *
 * This file implements a partitioned, tree-based, concurrent
 * hash-lookup structure. For every key, two values are derived that
 * determine its location within the structure: an index, which
 * determines which of the partitions (each containing a tree and each
 * separately locked), and a hash which acts as the key within an
 * individual Red-Black Tree.
 */
```

**核心特点：**
- **分区机制**：多个独立的红黑树分区，每个分区有自己的锁
- **缓存优化**：支持预期条目缓存（cache_entry_count: 2^10 到 2^15）
- **并发控制**：分区级别的 RWLock，支持高并发

### 2.4 哈希表数据结构

**[src/include/hashtable.h](src/include/hashtable.h) 关键结构：**

```c
/* 哈希参数 */
struct hash_param {
    uint32_t flags;                    /* 创建标志 */
    uint32_t cache_entry_count;        /* 缓存条目数（2的幂） */
    uint32_t index_size;               /* 分区树数量（质数） */
    index_function_t hash_func_key;    /* 分区函数 */
    rbthash_function_t hash_func_rbt;  /* 树内哈希函数 */
    both_function_t hash_func_both;    /* 组合计算函数 */
    hash_comparator_t compare_key;     /* 比较函数 */
};

/* 分区结构 */
struct hash_partition {
    size_t count;                      /* 分区内条目数 */
    struct rbt_head rbt;               /* 红黑树 */
    pthread_rwlock_t ht_lock;          /* 分区锁 */
    struct rbt_node **cache;           /* 期望条目缓存 */
};

/* 哈希表 */
typedef struct hash_table {
    struct hash_param parameter;       /* 参数 */
    pool_t *node_pool;                 /* 节点池 */
    pool_t *data_pool;                 /* 数据池 */
    struct hash_partition partitions[]; /* 分区数组 */
} hash_table_t;
```

---

## 3. 缓存算法深析

### 3.1 多级 LRU 算法（MQ 启发）

**算法描述：** 基于论文 [Zhou 2004] 的 MQ (Multi-Queue) 算法

**文件:** [mdcache_lru.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h) 和 [mdcache_lru.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c)

**多级队列结构：**

```c
enum lru_q_id {
    LRU_ENTRY_NONE = 0,      /* 不在队列中 */
    LRU_ENTRY_L1,            /* L1 队列（热数据） */
    LRU_ENTRY_L2,            /* L2 队列（冷数据） */
    LRU_ENTRY_CLEANUP,       /* 清理队列 */
    LRU_ENTRY_ACTIVE,        /* 活跃引用队列 */
};

struct lru_q_lane {
    struct lru_q L1;         /* L1 队列 */
    struct lru_q L2;         /* L2 队列 */
    struct lru_q cleanup;    /* 清理队列 */
    struct lru_q ACTIVE;     /* 活跃队列 */
    pthread_mutex_t ql_mtx;  /* 队列锁 */
};
```

### 3.2 缓存淘汰策略

**LRU 数据结构：** [mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h)

```c
typedef struct mdcache_lru__ {
    struct glist_head q;         /* 物理双端队列 */
    enum lru_q_id qid;           /* 队列 ID */
    int32_t refcnt;              /* 总引用计数（有符号） */
    int32_t active_refcnt;       /* 活跃引用计数（有符号） */
    uint32_t flags;              /* 标志位（原子操作） */
    uint32_t lane;               /* 车道号（分片锁分组） */
    uint32_t cf;                 /* 混淆因子 */
} mdcache_lru_t;
```

**队列转移流程：**

```
      L1 队列（热数据）
      └─> 检查条件（开放文件、锁等）
          └─> L2 队列（冷数据）
              └─> 淘汰（LRU 端）或回收

      活跃队列
      └─> 引用计数 > 0
          └─> 放回 L1 或 L2
```

### 3.3 缓存分片策略

**车道数：** `LRU_N_Q_LANES = 17`（质数）

**目的：**
- 减少锁竞争
- 并行化缓存管理
- 每个车道独立管理一组队列

**引用计数规则：**

| 标志 | 值 | 含义 |
|------|-----|------|
| `LRU_SENTINEL_REFCOUNT` | 1 | 最小未回收引用计数 |
| `LRU_ACTIVE_REF` | - | 活跃引用标志 |
| `LRU_PROMOTE` | - | 晋升标志（L1→L2） |
| `LRU_FLAG_SENTINEL` | - | 哨兵引用 |
| `LRU_TEMP_REF` | - | 临时引用 |

### 3.4 水位管理

**文件:** [mdcache_lru.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c)

```c
struct lru_state {
    uint64_t entries_hiwat;        /* 高水位 */
    uint64_t entries_used;         /* 当前使用 */
    uint32_t entries_release_size; /* 释放大小 */
    uint64_t chunks_hiwat;         /* 分块高水位 */
    uint64_t chunks_lowat;         /* 分块低水位 */
    uint64_t chunks_used;          /* 分块使用 */
    uint32_t per_lane_work;        /* 每车道工作量 */
    time_t prev_time;              /* 前次运行时间 */
};
```

**异步淘汰机制：**
- 缓存大小异步管理（避免内联请求延迟）
- 后台清理线程定期运行
- 当使用 > 高水位时触发淘汰

---

## 4. 命中率计算实现

### 4.1 监控指标架构

**文件:**
- [src/monitoring/dynamic_metrics.cc](src/monitoring/dynamic_metrics.cc)
- [src/monitoring/include/dynamic_metrics.h](src/monitoring/include/dynamic_metrics.h)

### 4.2 命中率指标定义

**Prometheus 指标：**

```cpp
class DynamicMetrics {
    // 缓存命中和未命中计数器
    CounterInt::Family &mdcacheCacheHitsTotal;           // 总命中数
    CounterInt::Family &mdcacheCacheMissesTotal;         // 总未命中数
    CounterInt::Family &mdcacheCacheHitsByExportTotal;   // 按导出分组命中
    CounterInt::Family &mdcacheCacheMissesByExportTotal; // 按导出分组未命中
};
```

**指标注册：**

```cpp
mdcacheCacheHitsTotal(
    prometheus::Builder<CounterInt>()
        .Name("mdcache_cache_hits_total")
        .Help("Counter for total cache hits in mdcache.")
        .Register(registry))

mdcacheCacheMissesTotal(
    prometheus::Builder<CounterInt>()
        .Name("mdcache_cache_misses_total")
        .Help("Counter for total cache misses in mdcache.")
        .Register(registry))
```

### 4.3 命中率记录点

**文件:** [mdcache_handle.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_handle.c)

```c
/* 行 1005 - 缓存未命中 */
dynamic_metrics__mdcache_cache_miss(OPERATION, export_id);

/* 行 1019 - 缓存命中 */
dynamic_metrics__mdcache_cache_hit(OPERATION, export_id);

/* 行 1029 - 缓存命中 */
dynamic_metrics__mdcache_cache_hit(OPERATION, export_id);

/* 行 1033 - 缓存未命中 */
dynamic_metrics__mdcache_cache_miss(OPERATION, export_id);
```

**文件:** [mdcache_helpers.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c)

```c
/* 行 1283 - 未命中 */
dynamic_metrics__mdcache_cache_miss(OPERATION, export_id);

/* 行 1337 - 命中 */
dynamic_metrics__mdcache_cache_hit(OPERATION, export_id);

/* 行 1368 - 未命中 */
dynamic_metrics__mdcache_cache_miss(OPERATION, export_id);
```

### 4.4 统计数据结构

**文件:** [mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h)

```c
/**
 * MDCACHE statistics.
 */
struct mdcache_stats {
    uint64_t inode_req;     /* inode 请求总数 */
    uint64_t inode_hit;     /* inode 缓存命中数 */
    uint64_t inode_miss;    /* inode 缓存未命中数 */
    uint64_t inode_conf;    /* inode 冲突数 */
    uint64_t inode_added;   /* 新增 inode 数 */
    uint64_t inode_mapping; /* inode 映射数 */
};

extern struct mdcache_stats *cache_stp;
```

**初始化：** [mdcache_main.c](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_main.c)

```c
struct mdcache_stats cache_st;
struct mdcache_stats *cache_stp = &cache_st;
```

### 4.5 命中率计算公式

**命中率 = 命中数 / (命中数 + 未命中数)**

```python
# 实时计算示例
hit_rate = mdcacheCacheHitsTotal / (
    mdcacheCacheHitsTotal + mdcacheCacheMissesTotal
)

# 按导出分组
hit_rate_by_export = mdcacheCacheHitsByExportTotal[export_id] / (
    mdcacheCacheHitsByExportTotal[export_id] + 
    mdcacheCacheMissesByExportTotal[export_id]
)
```

### 4.6 指标收集接口

**头文件定义：** [dynamic_metrics.h](src/monitoring/include/dynamic_metrics.h)

```c
/* MDCache hit rates. */
void dynamic_metrics__mdcache_cache_hit(
    const char *operation,
    export_id_t export_id);

void dynamic_metrics__mdcache_cache_miss(
    const char *operation,
    export_id_t export_id);
```

**调用上下文：**
- `operation`: 操作类型（如 "lookup", "getattr" 等）
- `export_id`: 导出 ID（用于按导出分组）

### 4.7 Prometheus 采集

**导出端点：** 通常为 `/metrics` 或 `/prometheus`

**Prometheus 查询示例：**

```promql
# 总命中率
rate(mdcache_cache_hits_total[5m]) / 
(rate(mdcache_cache_hits_total[5m]) + rate(mdcache_cache_misses_total[5m]))

# 按导出分组
rate(mdcache_cache_hits_by_export_total[5m]) / 
(rate(mdcache_cache_hits_by_export_total[5m]) + rate(mdcache_cache_misses_by_export_total[5m]))
```

---

## 5. 关键数据结构

### 5.1 缓存条目结构

**文件:** [mdcache_int.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h)

```c
/* MDCache 导出结构 */
struct mdcache_fsal_export {
    struct fsal_export mfe_exp;         /* 导出基础结构 */
    char *name;                         /* 导出名称 */
    struct fsal_up_vector up_ops;       /* UP 操作向量 */
    struct fsal_up_vector super_up_ops; /* 上级 UP 操作 */
    struct glist_head entry_list;       /* 属于此导出的条目列表 */
    pthread_mutex_t mdc_exp_lock;       /* entry_list 保护锁 */
    uint8_t flags;                      /* 导出标志 */
    mdc_dirmap_t dirent_map;            /* 目录条目映射 */
    struct fridgethr *dirmap_fridge;    /* dirmap 处理线程 */
    int32_t cleanup_pending;            /* 待清理条目计数 */
};

/* 缓存键结构 */
typedef struct mdcache_key {
    uint64_t hk;                   /* 哈希键 */
    void *fsal;                    /* 子 FSAL 模块 */
    struct gsh_buffdesc kv;        /* FSAL 句柄 */
} mdcache_key_t;

/* 目录条目映射 */
typedef struct mdcache_dmap_entry__ {
    struct avltree_node node;      /* AVL 树节点（按 cookie） */
    struct glist_head lru_entry;   /* LRU 链表 */
    uint64_t ck;                   /* cookie */
    char *name;                    /* 名称 */
    struct timespec timestamp;     /* 时间戳 */
} mdcache_dmap_entry_t;
```

### 5.2 缓存键比较函数

```c
static inline int mdcache_key_cmp(
    const struct mdcache_key *k1,
    const struct mdcache_key *k2) {
    /* 比较哈希值 */
    if (likely(k1->hk < k2->hk))
        return -1;
    if (likely(k1->hk > k2->hk))
        return 1;

    /* 比较长度 */
    if (unlikely(k1->kv.len < k2->kv.len))
        return -1;
    if (unlikely(k1->kv.len > k2->kv.len))
        return 1;

    /* 比较 FSAL */
    if (unlikely(k1->fsal < k2->fsal))
        return -1;
    if (unlikely(k1->fsal > k2->fsal))
        return 1;

    /* 深比较 */
    return memcmp(k1->kv.addr, k2->kv.addr, k1->kv.len);
}
```

### 5.3 引用计数管理 API

**文件:** [mdcache_lru.h](src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h)

```c
/* 获取缓存条目 */
mdcache_entry_t *mdcache_lru_get(
    struct fsal_obj_handle *sub_handle,
    uint32_t flags);

/* 增加引用 */
void _mdcache_lru_ref(
    mdcache_entry_t *entry,
    uint32_t flags,
    const char *func,
    int line);
#define mdcache_lru_ref(e, f) \
    _mdcache_lru_ref(e, f, __func__, __LINE__)

/* 释放引用 */
bool _mdcache_lru_unref(
    mdcache_entry_t *entry,
    uint32_t flags,
    const char *func,
    int line);
#define mdcache_lru_unref(e, f) \
    _mdcache_lru_unref(e, f, __func__, __LINE__)

/* 转移到活跃队列 */
void mdcache_lru_insert_active(mdcache_entry_t *entry);

/* 淘汰和清理 */
size_t mdcache_lru_release_entries(int32_t want_release);
```

---

## 6. 工作流程示例

### 6.1 查询流程（缓存命中）

```
[NFS 客户端] 
    ↓
[Protocols/NFS] (协议处理)
    ↓
[mdc_lookup()] (缓存查询)
    ↓
[mdcache_hash.c: cih_lookup()] (哈希查找)
    │
    ├─→ 分区索引计算 (partition_index)
    │
    └─→ 红黑树查找 (RBTree)
         │
         ├─ 命中 → [dynamic_metrics__mdcache_cache_hit()]
         │              └─→ Prometheus 计数器 ++
         │
         └─ 未命中 → [dynamic_metrics__mdcache_cache_miss()]
                     └─→ 调用下层 FSAL
                         └─→ Prometheus 计数器 ++

    ↓
[FSAL_* 后端]
    ↓
[返回结果给客户端]
```

### 6.2 LRU 淘汰流程

```
[fd_lru_run()] (后台线程)
    ↓
检查缓存大小
    │
    ├─ 使用 ≤ 低水位 → 无操作
    │
    └─ 使用 > 高水位 → 触发淘汰
         ↓
    [mdcache_lru_release_entries(want_release)]
         ↓
    对每个车道：
         │
         ├─ 检查 L2 队列末端条目
         │   └─ refcnt ≤ 1 → 回收
         │
         └─ 检查 L1 队列末端条目
             └─ 晋升条件 → L2 队列
```

### 6.3 属性过期管理

```
[mdcache 条目]
    ↓
[Attr_Expiration_Time] (可配置)
    ↓
到期时间计算：
    entry->attr_time + Attr_Expiration_Time
    ↓
时间到期后：
    ├─ MDCACHE_TRUST_ATTRS 标志被清除
    └─ 下次访问时刷新属性
```

---

## 7. 配置参数（主要）

**配置文件位置：** `ganesha.conf` 或导出配置文件

**MDCache 相关参数：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `MDCACHE { }` | - | MDCache 模块配置块 |
| `Attr_Expiration_Time` | 秒数 | 属性缓存过期时间 |
| `Max_Cache_Entries` | - | 最大缓存条目数 |
| `Chunks_LowWat` | - | 块低水位 |
| `Chunks_HiWat` | - | 块高水位 |

---

## 8. 关键函数映射

| 功能 | 文件 | 函数 | 行号 |
|------|------|------|------|
| 哈希查询 | mdcache_hash.c | `cih_lookup()` | - |
| 查询操作 | mdcache_helpers.c | `mdc_lookup()` | ~1280 |
| 创建条目 | mdcache_handle.c | `mdcache_alloc_and_check_handle()` | ~995 |
| LRU 获取 | mdcache_lru.c | `mdcache_lru_get()` | - |
| LRU 引用增加 | mdcache_lru.c | `_mdcache_lru_ref()` | - |
| LRU 引用释放 | mdcache_lru.c | `_mdcache_lru_unref()` | - |
| 异步淘汰 | commonlib.c | `fd_lru_run()` | ~1553 |
| 命中计数 | mdcache_handle.c | `dynamic_metrics__mdcache_cache_hit()` | 1019 |
| 未命中计数 | mdcache_handle.c | `dynamic_metrics__mdcache_cache_miss()` | 1005 |

---

## 9. 性能相关点

### 9.1 并发优化

1. **分区锁** - 分散锁竞争
2. **车道分片** - 17 个独立的 LRU 队列
3. **读写锁** - 分区级 RWLock
4. **原子操作** - 引用计数的原子更新

### 9.2 缓存效率

1. **预期缓存** - hash_partition.cache 数组
2. **多级队列** - L1/L2/CLEANUP/ACTIVE
3. **异步清理** - 避免同步淘汰延迟
4. **属性过期** - 自动失效机制

### 9.3 故障处理

1. **STALE 检测** - ERR_FSAL_STALE 时标记条目为不可达
2. **引用计数** - 确保条目安全回收
3. **锁顺序** - 分区锁 < 队列锁，防止死锁

---

## 10. 监控和调试

### 10.1 日志组件

```c
#define COMPONENT_MDCACHE          /* MDCache 主模块 */
#define COMPONENT_MDCACHE_LRU      /* LRU 管理 */
#define COMPONENT_HASHTABLE        /* 哈希表 */
#define COMPONENT_HASHTABLE_CACHE  /* 哈希表缓存 */
```

### 10.2 Prometheus 指标端点

```
# 全局命中率
mdcache_cache_hits_total
mdcache_cache_misses_total

# 按导出分组
mdcache_cache_hits_by_export_total{export_id="X"}
mdcache_cache_misses_by_export_total{export_id="X"}
```

### 10.3 DBUS 接口（可选）

```
# Ganesha-top 工具可显示：
- MDCache 信息
- 缓存大小
- 命中率统计
```

---

## 总结

NFS-Ganesha 的缓存系统是一个精密的多层次设计：

1. **哈希表层** - 分区 RBTree 实现 O(log n) 查询
2. **LRU 层** - 多级队列（L1/L2）提供低开销淘汰
3. **分片层** - 17 车道并行管理减少竞争
4. **指标层** - Prometheus 集成实时命中率统计
5. **协议层** - NFSv3/v4 等协议透明使用缓存

**命中率计算** 基于原子计数器，支持全局和按导出分组统计，通过 Prometheus 指标向外暴露。
