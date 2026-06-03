# NFS-Ganesha 缓存系统 - 快速参考指南

## 项目结构速查表

### 核心模块
```
MainNFSD/          → NFS 服务主程序、线程管理
Protocols/NFS/     → NFSv3/v4 协议处理
SAL/               → 状态管理（锁、委托、开启）
FSAL/              → 文件系统访问层
  FSAL_MDCACHE/    → ★ 元数据缓存（核心）
  FSAL_VFS/        → Linux VFS 后端
  FSAL_CEPH/       → Ceph 后端
  ...
hashtable/         → ★ 通用哈希表实现
monitoring/        → ★ Prometheus 指标
```

---

## 缓存系统三层架构

### Layer 1: 哈希表（查询层）
- **文件**: `src/hashtable/hashtable.c`, `src/include/hashtable.h`
- **算法**: 分区红黑树（RBTree）
- **特性**: 
  - index_size 个分区（质数）
  - 每个分区有独立的 RWLock
  - 缓存的 cache_entry_count（2^10 到 2^15）

### Layer 2: MDCache（缓存层）
- **文件**: `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/`
- **核心实现**:
  - `mdcache_int.h` - 数据结构
  - `mdcache_hash.h` - 哈希查询
  - `mdcache_lru.h` - LRU 定义
  - `mdcache_lru.c` - LRU 算法
  - `mdcache_handle.c` - 文件操作

### Layer 3: 监控（指标层）
- **文件**: `src/monitoring/dynamic_metrics.cc`
- **指标**:
  - `mdcache_cache_hits_total` - 命中计数
  - `mdcache_cache_misses_total` - 未命中计数
  - 支持按导出分组

---

## 缓存算法核心

### LRU 多级队列（4 个队列）

| 队列 | 目的 | 特点 |
|------|------|------|
| L1 | 热数据 | 新条目/最近使用 |
| L2 | 冷数据 | 验证过的条目/较少使用 |
| ACTIVE | 活跃 | 有活跃请求的条目 |
| CLEANUP | 清理 | 待异步清理的条目 |

### 淘汰流程

```
请求来临 → ACTIVE 队列
请求结束 → L1 队列
时间流逝 → L2 队列（检查条件）
继续冷却 → LRU 末端
达到哨兵 → CLEANUP 队列
后台清理 → 释放内存
```

### 关键数据结构

```c
/* 缓存条目 LRU 信息 */
struct mdcache_lru {
    struct glist_head q;      /* 队列链接 */
    enum lru_q_id qid;        /* 所在队列 */
    int32_t refcnt;           /* 总引用计数 */
    int32_t active_refcnt;    /* 活跃引用计数 */
    uint32_t flags;           /* 标志位 */
    uint32_t lane;            /* 车道号（0-16） */
};

/* 缓存键 */
struct mdcache_key {
    uint64_t hk;              /* 哈希值 */
    void *fsal;               /* FSAL 模块 */
    struct gsh_buffdesc kv;   /* 数据缓冲区 */
};

/* 统计信息 */
struct mdcache_stats {
    uint64_t inode_req;       /* 请求数 */
    uint64_t inode_hit;       /* 命中数 */
    uint64_t inode_miss;      /* 未命中数 */
};
```

---

## 命中率计算

### 公式
```
命中率 = 命中数 / (命中数 + 未命中数) × 100%
```

### 记录点

**缓存命中：**
```c
dynamic_metrics__mdcache_cache_hit("lookup", export_id);
```

**缓存未命中：**
```c
dynamic_metrics__mdcache_cache_miss("lookup", export_id);
```

### Prometheus 查询

**全局命中率：**
```promql
rate(mdcache_cache_hits_total[5m]) / 
(rate(mdcache_cache_hits_total[5m]) + rate(mdcache_cache_misses_total[5m]))
```

**按导出分组：**
```promql
rate(mdcache_cache_hits_by_export_total{export_id="1"}[5m]) /
(rate(mdcache_cache_hits_by_export_total{export_id="1"}[5m]) + 
 rate(mdcache_cache_misses_by_export_total{export_id="1"}[5m]))
```

---

## 关键 API 速查

### 引用管理
```c
/* 增加引用 */
mdcache_lru_ref(entry, LRU_ACTIVE_REF | LRU_PROMOTE);

/* 释放引用 */
mdcache_lru_unref(entry, LRU_ACTIVE_REF);

/* 获取条目 */
mdcache_entry_t *entry = mdcache_lru_get(sub_handle, flags);

/* 插入活跃队列 */
mdcache_lru_insert_active(entry);
```

### 缓存查询
```c
/* 查询文件 */
fsal_status_t status = mdc_lookup(
    parent,       /* 父目录条目 */
    name,         /* 文件名 */
    true,         /* 新查询 */
    &entry,       /* 返回条目 */
    &attrs        /* 返回属性 */
);

/* 创建新条目 */
status = mdcache_new_entry(
    export,       /* MDCache 导出 */
    sub_handle,   /* 下层句柄 */
    attrs_in,     /* 输入属性 */
    false,        /* not directory */
    attrs_out,    /* 返回属性 */
    false,        /* not new dir */
    &entry,       /* 新条目 */
    NULL,         /* state */
    LRU_ACTIVE_REF
);
```

### 淘汰管理
```c
/* 释放条目 */
size_t freed = mdcache_lru_release_entries(want_release);

/* 推入清理队列 */
mdcache_lru_cleanup_push(entry);

/* 杀死条目 */
mdcache_lru_kill(entry);
```

---

## 文件查询速查

### 哈希表实现
- `src/hashtable/hashtable.c` - 双重哈希逻辑
- `src/include/hashtable.h` - 数据结构定义

### MDCache 核心
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h` - 数据结构
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_hash.c` - 哈希查询实现
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c` - LRU 算法实现
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c` - mdc_lookup 等
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_handle.c` - 文件句柄操作

### 监控指标
- `src/monitoring/dynamic_metrics.cc` - Prometheus 集成
- `src/monitoring/include/dynamic_metrics.h` - 指标 API

### 后台管理
- `src/FSAL/commonlib.c` - FD LRU 线程（lru_try_one, fd_lru_run）
- `src/MainNFSD/nfs_metrics.c` - 度量统计

---

## 配置参数

### MDCACHE 块
```ini
MDCACHE {
    # 属性过期时间（秒）
    Attr_Expiration_Time = 3600;
    
    # 最大缓存条目
    Max_Cache_Entries = 1000000;
    
    # 目录块配置
    Chunks_LowWat = 1000;
    Chunks_HiWat = 10000;
    
    # 目录读取模式
    #  FSAL_RDDIR_CHUNK_ALWAYS - 缓存目录条目
    #  FSAL_RDDIR_CHUNK_NEVER - 不缓存
    Readdir_Mode = FSAL_RDDIR_CHUNK_ALWAYS;
}
```

---

## 性能优化点

### 1. 分区数
```c
index_size = 127  /* 或其他质数，分散锁竞争 */
```

### 2. 车道数
```c
#define LRU_N_Q_LANES 17  /* 质数，17 个独立的 LRU 队列 */
```

### 3. 缓存页大小
```c
cache_entry_count = 1024  /* 2^10，预期缓存大小 */
```

### 4. 水位管理
```c
entries_hiwat = 1000000   /* 高水位，触发淘汰 */
entries_lowat = 500000    /* 低水位，淘汰目标 */
```

---

## 故障排查

### 问题 1: 命中率低

**检查项：**
1. `Attr_Expiration_Time` 是否过短？
2. 是否频繁更新同一文件？
3. 缓存大小（`Max_Cache_Entries`）是否足够？
4. `Readdir_Mode` 是否为 `FSAL_RDDIR_CHUNK_ALWAYS`？

**查询：**
```promql
# 查看 5 分钟内的命中率变化
rate(mdcache_cache_hits_total[5m]) /
(rate(mdcache_cache_hits_total[5m]) + rate(mdcache_cache_misses_total[5m]))
```

### 问题 2: 内存占用高

**检查项：**
1. 缓存条目数是否超过 `Max_Cache_Entries`？
2. 是否有未释放的引用（refcnt 泄漏）？
3. 后台淘汰线程是否正常运行？

**日志组件：**
```
COMPONENT_MDCACHE      - 主模块
COMPONENT_MDCACHE_LRU  - LRU 管理
COMPONENT_HASHTABLE    - 哈希表
```

### 问题 3: 并发竞争

**检查：**
- 分区锁（`ht_lock`）竞争
- 车道锁（`ql_mtx`）竞争
- 增加分区数或车道数（需重新编译）

---

## 实时监控

### ganesha-top（NFS Ganesha 监视工具）
```bash
ganesha-top
# 显示 MDCache 信息、缓存大小、命中率等
```

### Prometheus 导出
```bash
curl http://localhost:8000/metrics | grep mdcache
```

### DBUS 查询
```bash
dbus-send --system --print-reply \
  --dest=org.ganesha.nfsd \
  /org/ganesha/nfsd/admin \
  org.ganesha.nfsd.admin.ShowCache
```

---

## 调试技巧

### 启用详细日志
```c
/* mdcache_debug.h */
#ifdef DEBUG_MDCACHE
  LogFullDebug(COMPONENT_MDCACHE, "Debug message");
#endif
```

### 追踪引用计数
```c
#define mdcache_lru_ref(e, f) \
    _mdcache_lru_ref(e, f, __func__, __LINE__)

/* 日志会包含调用位置 */
```

### LTTNG 追踪（如果启用）
```bash
# 收集 MDCache 追踪事件
lttng create -o /tmp/mdcache mdcache
lttng enable-event -u mdcache:*
lttng start

# ... 运行测试 ...

lttng stop
lttng view
```

---

## 关键常数速查

```c
/* 引用计数 */
#define LRU_SENTINEL_REFCOUNT     1
#define LRU_ACTIVE_REF            0x0004
#define LRU_PROMOTE               0x0008
#define LRU_FLAG_SENTINEL         0x0001
#define LRU_TEMP_REF              0x0002

/* 队列 ID */
#define LRU_ENTRY_NONE            0
#define LRU_ENTRY_L1              1
#define LRU_ENTRY_L2              2
#define LRU_ENTRY_CLEANUP         3
#define LRU_ENTRY_ACTIVE          4

/* 并发 */
#define LRU_N_Q_LANES             17   /* 质数 */

/* 标志位 */
#define LRU_CLEANUP               0x00000001
#define LRU_CLEANED               0x00000002
#define LRU_EVER_PROMOTED         0x00000004
#define LRU_SENTINEL_HELD         0x00000008

/* 属性有效性 */
#define MDCACHE_TRUST_ATTRS       0x0001
#define MDCACHE_TRUST_CONTENT     0x0002
```

---

## 编译和集成

### 依赖
- Red-Black Tree 库（`<rbt_node.h>`, `<rbt_tree.h>`）
- AVL Tree（用于目录条目映射）
- Prometheus C++ 库（监控）

### 编译选项
```bash
cmake -DUSE_MDCACHE=ON \
      -DENABLE_MONITORING=ON \
      -DENABLE_LTTNG=OFF \
      ..
make -j$(nproc)
```

### 模块加载
```ini
# ganesha.conf
FSAL {
    # 在底层 FSAL 上堆叠 MDCACHE
    MDCACHE {
        ...
    }
    VFS {
        ...
    }
}
```

---

## 性能基准

**典型指标（大规模部署）：**
- 命中率: 80-95%（取决于工作集大小）
- 查询延迟: < 1μs（缓存命中）
- 并发度: 支持数千并发
- 内存开销: ~100 bytes/条目

**优化建议：**
1. 设置 `Attr_Expiration_Time` 为 3600 秒
2. `Max_Cache_Entries` = 工作集大小 × 1.2
3. `Chunks_HiWat` 和 `Chunks_LowWat` 比例 = 10:1
4. 使用 FSAL_RDDIR_CHUNK_ALWAYS 模式

---

## 相关论文参考

1. **Johnson, T., & Shasha, D. (1994).** "2Q: A Low Overhead High Performance Buffer Management Replacement Algorithm"

2. **Zhou, Y., Chen, Z., & Li, K. (2004).** "Second-Level Buffer Cache Management"

这些论文说明了 NFS-Ganesha LRU 算法的理论基础。

---

## 更多资源

- **官方文档**: https://github.com/nfs-ganesha/nfs-ganesha/wiki
- **配置指南**: `src/doc/man/ganesha-config.rst`
- **API 参考**: `src/doc/Resources.txt`
- **社区讨论**: GitHub Issues

---

## 快速问题排查流程

```
问题描述
  ↓
检查 Prometheus 指标
  ├─ 命中率 → metrics 查询
  ├─ 缓存大小 → entries_used
  └─ 操作类型分布 → by operation
  ↓
检查日志
  ├─ COMPONENT_MDCACHE
  ├─ COMPONENT_MDCACHE_LRU
  └─ COMPONENT_HASHTABLE
  ↓
检查配置
  ├─ Attr_Expiration_Time
  ├─ Max_Cache_Entries
  └─ Readdir_Mode
  ↓
性能分析
  ├─ ganesha-top
  ├─ dbus 接口
  └─ LTTNG 追踪
  ↓
优化建议
```

---

**最后更新**: 2026-05-16  
**版本**: 1.0  
**维护**: NFS-Ganesha 社区
