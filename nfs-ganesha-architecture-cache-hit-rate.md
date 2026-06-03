# NFS-Ganesha 架构、MDCACHE 缓存算法与实时命中率梳理

本文基于当前仓库源码梳理，重点覆盖 NFS-Ganesha 的整体分层、MDCACHE 元数据缓存实现、缓存淘汰算法，以及运行时命中率如何从现有指标计算。

## 1. 总体定位

NFS-Ganesha 是用户态 NFS 文件服务。仓库 `README.md` 将其定义为支持 NFSv3、NFSv4、NFSv4.1 的用户态 fileserver，同时也支持 9P.2000L。

整体上，它把协议处理、状态管理、后端文件系统适配和缓存拆成多个层：

```text
Client
  |
  | NFS / 9P / NLM / RQUOTA RPC
  v
RPCAL / libntirpc
  |
  v
MainNFSD worker + Protocol handlers
  |
  v
SAL: state / lock / lease / recovery
  |
  v
FSAL API
  |
  v
FSAL_MDCACHE stackable layer
  |
  v
Concrete FSAL: VFS / CEPH / GPFS / GLUSTER / RGW / PROXY / KVSFS ...
  |
  v
Backend filesystem or remote service
```

关键源码入口：

- `src/MainNFSD/`: daemon 初始化、线程、请求调度、监控启动。
- `src/Protocols/`: NFS、NLM、9P、RQUOTA 等协议实现。
- `src/RPCAL/`: RPC 连接、认证、duplicate request cache 等 RPC 抽象。
- `src/SAL/`: NFSv4 state、lock、lease、delegation、recovery。
- `src/FSAL/`: File System Abstraction Layer，统一后端文件系统接口。
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/`: 叠加式元数据缓存层。
- `src/monitoring/`: Prometheus 动态指标和暴露端点。

启动链路上，`start_fsals()` 会静态加载 `MDCACHE` 和 `PSEUDO` FSAL；`init_server_pkgs()` 会初始化 MDCACHE 包；`nfs_main.c` 在 `Enable_Metrics` 打开时启动 Prometheus exporter，并在 `Enable_Dynamic_Metrics` 打开时初始化动态指标。

## 2. FSAL 与 MDCACHE 的位置

FSAL 是协议层和具体后端之间的稳定接口。具体 FSAL 负责真实文件系统或对象存储访问，例如 `FSAL_VFS`、`FSAL_CEPH`、`FSAL_GPFS`、`FSAL_GLUSTER` 等。

`FSAL_MDCACHE` 是 stackable FSAL：它不是最终后端，而是包在具体 FSAL 上方，缓存对象句柄、属性和目录项。典型调用路径是：

```text
NFS LOOKUP / GETATTR / READDIR
  -> protocol handler
  -> FSAL object/export ops
  -> MDCACHE object/export ops
  -> sub-FSAL object/export ops on miss or refresh
```

MDCACHE 初始化路径：

- `src/FSAL/fsal_manager.c`: `load_fsal_static("MDCACHE", mdcache_fsal_init, NULL)`。
- `src/MainNFSD/nfs_init.c`: `mdcache_set_param_from_conf()` 读取 `MDCACHE {}` 配置。
- `src/MainNFSD/nfs_init.c`: `mdcache_pkginit()` 初始化 MDCACHE。
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_main.c`: 创建 entry pool，初始化 LRU，再初始化 file-handle hash。

## 3. MDCACHE 核心数据结构

### 3.1 缓存对象 `mdcache_entry_t`

`mdcache_entry_t` 实际类型是 `struct mdcache_fsal_obj_handle`，定义在 `mdcache_int.h`。它把 FSAL 对象句柄和缓存元数据放在同一个对象里：

- `obj_handle`: 对上层暴露的 MDCACHE FSAL handle。
- `sub_handle`: 指向下层真实 FSAL 的 handle。
- `attrs`: 缓存属性。
- `attr_generation`: 属性变更代数。
- `fh_hk.key`: file handle 缓存 key。
- `mde_flags`: 信任/失效标志，例如 `MDCACHE_TRUST_ATTRS`、`MDCACHE_TRUST_CONTENT`。
- `attr_time`、`acl_time`、`fs_locations_time`: 属性刷新时间。
- `lru`: LRU 引用计数和队列位置。
- `export_list`、`first_export_id`: entry 与 export 的多对多映射。
- `content_lock`: 目录项、目录 chunk、symlink 内容等对象内容锁。
- `attr_lock`: 属性和 export 映射锁。

目录对象额外维护：

- `chunks`: 当前目录的 dirent chunk 列表。
- `detached`: 未归入 chunk 的目录项列表。
- `avl.t`: 按名字查找 dirent。
- `avl.ck`: 按 FSAL cookie 查找 dirent。
- `avl.sorted`: 按 readdir 顺序查找 dirent。
- `first_ck`: 缓存中第一个目录项 cookie。

### 3.2 file-handle 查找表 CIH

CIH 由 `mdcache_hash.h/.c` 实现，用于通过 FSAL key/file handle 找到 `mdcache_entry_t`。

算法结构：

- 全局 `cih_fhcache` 包含 `npart` 个 partition。
- 每个 partition 有一个 mutex、一棵 AVL 树和一个 direct-mapped pointer cache。
- key 使用 `CityHash64WithSeed(fh, len, 557)` 计算 `hk`。
- partition 选择：`hk % npart`。
- direct cache slot 选择：`hk % cache_size`。

查找流程：

```text
cih_get_by_key_latch(key)
  -> 锁住 key 所属 partition
  -> 查 direct cache slot
     -> key 相等：O(1) 命中
  -> 查 partition AVL
     -> AVL 命中：更新 direct cache slot
     -> 未命中：按 flags 决定是否释放 partition lock
```

复杂度：

- direct cache 命中：近似 O(1)。
- direct cache 冲突后 AVL 查找：O(log n)，n 是该 partition entry 数。
- 多 partition 降低并发锁竞争。

### 3.3 目录项缓存

目录缓存由 `mdcache_avl.c` 和 `mdcache_helpers.c` 实现。

每个 dirent 保存：

- 文件名。
- FSAL cookie。
- child entry 的 cache key。
- `chunk` 指针。
- AVL 节点：name、cookie、sorted。
- 临时 `mde_entry` 指针，仅在持有引用时有效。

`mdc_try_get_cached()` 的目录 lookup 逻辑：

1. 如果当前 FSAL/readdir 模式不缓存 dirent，返回 stale，触发下层 FSAL lookup。
2. 如果父目录没有 `MDCACHE_TRUST_CONTENT`，返回 stale。
3. 在父目录 `avl.t` 中按 name 查找 dirent。
4. 找到后 bump chunk 或 detached dirent 的 LRU 位置。
5. 用 dirent 内保存的 child key 调 `mdcache_find_keyed_reason()` 找 child entry。
6. 如果 child entry 仍在 CIH 且 export mapping 有效，返回缓存命中。
7. 如果找不到 dirent，但父目录可信任 negative cache，则返回 `ERR_FSAL_NOENT`。
8. 其他情况返回 stale，调用下层 FSAL。

注意：当前 Prometheus 命中率埋点中，negative lookup 虽然可能由目录缓存直接回答，但最终会走 miss 计数口径，不能简单理解为“所有未访问下层 FSAL的请求都算 hit”。

### 3.4 属性缓存

属性缓存主要由 `mdcache_getattrs()`、`mdcache_refresh_attrs()` 和 `mdcache_is_attrs_valid()` 控制。

命中条件：

- 请求的 attribute mask 已被缓存且 `valid_mask` 满足。
- 对应 `MDCACHE_TRUST_*` 标志仍有效。
- `Attr_Expiration_Time` 不为 0。
- 若 `Attr_Expiration_Time > 0`，当前时间未超过 `attr_time/acl_time + expire_time_attr`。
- 如果目录启用了 `Use_Getattr_Directory_Invalidation`，目录 getattr 会强制不使用缓存。
- 文件 delegation 存在时，已有属性可被更强地信任，但 `expire_time_attr == 0` 仍禁用属性缓存。

`Attr_Expiration_Time` 是 export 级配置，文档说明它在 MDCACHE entry 创建时求值，因此动态修改可能只影响新 entry。

## 4. LRU 与缓存淘汰算法

MDCACHE 的 LRU 在 `mdcache_lru.c/.h` 中实现。源码注释明确说它是常数时间 cache management，基于 LRU，并借鉴 2Q 和 MQ。

### 4.1 队列结构

每个 logical queue 分成 17 个 lane，`LRU_N_Q_LANES` 必须是素数。每个 lane 有独立 mutex，降低并发竞争。

entry LRU 每个 lane 包含：

- `L1`: 最近被主动访问/提升过的对象。
- `L2`: 较冷对象或仅目录扫描使用过的对象。
- `ACTIVE`: 当前仍被请求持有 active reference 的对象。
- `cleanup`: 延迟清理对象。

dir chunk 也有独立的 chunk LRU lane，包含 `L1/L2/cleanup`。

### 4.2 引用计数模型

MDCACHE 不把 LRU 当作简单 GC，而是和 entry 生命周期绑定：

- `refcnt == 1` 表示只有 sentinel reference。
- active 请求会增加 normal ref 和 active ref。
- entry 在 CIH 中可达时，sentinel ref 保证不会被释放。
- 回收前必须先从 CIH 移除，使 entry 对新 lookup 不可达。
- 有 open/lock/state 的对象通常不能普通回收，避免协议正确性问题。

### 4.3 L1/L2/ACTIVE 迁移

主要规则：

- 新 entry 创建后插入 `ACTIVE`。
- 获取 `LRU_ACTIVE_REF` 时进入 `ACTIVE` 或移动到 `ACTIVE` MRU。
- 释放最后一个 active ref 时，entry 从 `ACTIVE` 移到：
  - `L1`: 如果曾经被 `LRU_PROMOTE` 标记过。
  - `L2`: 如果仅由目录扫描使用，未被提升。
- 周期性 reaper 扫描各 lane 的 `L1`，把满足条件的冷 entry 移到 `L2`。
- 回收时优先尝试从 `L2` 取，再尝试 `L1`。

这个策略的意图：

- 热对象经由主动访问留在 L1。
- 大目录扫描产生的一次性对象更容易落入 L2，降低 scan pollution。
- reaper 只检查有限数量对象，避免请求线程承担大块清理延迟。

### 4.4 高水位与后台 reaper

关键配置来自 `MDCACHE {}`：

- `NParts`: CIH partition 数，默认 7。
- `Cache_Size`: 每 partition direct cache 大小，默认 32633。
- `Entries_HWMark`: entry 高水位，默认 100000。
- `Entries_Release_Size`: 超过高水位后每轮尝试释放 entry 数，默认 100。
- `Chunks_HWMark`: dirent chunk 高水位，默认 1000。
- `Chunks_LWMark`: dirent chunk 低水位，默认 1000。
- `LRU_Run_Interval`: LRU cleaner 基础周期，默认 90 秒。
- `Reaper_Work_Per_Lane`: 每个 lane 每轮扫描数量，默认 50。
- `Cache_FDs`、`Close_Fast`、`FD_*`: FD 缓存和 FD 水位控制。

`mdcache_lru_pkginit()` 会：

1. 根据配置设置 `lru_state.entries_hiwat`、`entries_release_size`、`chunks_hiwat`、`chunks_lowat`。
2. 初始化 17 个 entry LRU lane 和 17 个 chunk LRU lane。
3. 启动两个 fridge looper 线程：entry LRU 和 chunk LRU。
4. 初始化 FD LRU。

entry reaper 逻辑：

```text
每轮唤醒
  -> 每个 lane 扫描 L1，最多 Reaper_Work_Per_Lane 个
  -> refcnt == sentinel 时从 L1 demote 到 L2
  -> 如果 entries_used > Entries_HWMark
       -> mdcache_lru_release_entries(Entries_Release_Size)
       -> 仍高于水位则缩短下一轮等待
```

chunk reaper 逻辑：

```text
每轮唤醒
  -> 每个 lane demote L1 chunk 到 L2
  -> chunks_used > Chunks_HWMark 时释放约 1%
  -> entries_used > Entries_HWMark 时额外释放约 1%
  -> chunks_used > Chunks_LWMark 时继续向低水位回收
  -> 根据接近高水位程度动态调整下一轮等待
```

### 4.5 目录 chunk 算法

目录 chunk 默认大小为 128。`mdcache_readdir_chunked()` 在 READDIR 时按 FSAL cookie 填充 chunk，并把 chunk 挂入目录的 `chunks` 列表和 chunk LRU。

关键点：

- `Dir_Chunk_Enable` 控制是否启用。
- `get_readdir_mode()` 会综合 FSAL 模式、配置和 export 的 `NO_DIR_CACHING` 选项。
- chunk 保存 `reload_ck` 和 `next_ck`，用于从某个 cookie 重新加载或跳转到下一个 chunk。
- 当 chunk 达到 `avl_chunk_split = Dir_Chunk * 3 / 2` 时，会拆成两个 chunk。
- 新创建的 dirent 如果能通过 `compute_readdir_cookie()` 定位，会尽量插入现有 chunk；否则作为 detached dirent，并可能让 chunk trust 标志失效。
- `Chunks_LWMark * Dir_Chunk` 可能长期保留大量 entry，官方配置文档也提醒这会影响 `Entries_HWMark` 的实际效果。

## 5. 缓存命中/未命中埋点口径

当前源码里有两套相关统计。

### 5.1 `mdcache_stats` / DBus 口径

`mdcache_int.h` 定义：

```c
struct mdcache_stats {
    uint64_t inode_req;
    uint64_t inode_hit;
    uint64_t inode_miss;
    uint64_t inode_conf;
    uint64_t inode_added;
    uint64_t inode_mapping;
};
```

`mdcache_main.c` 的 DBus show 会展示这些字段。

但当前源码中实际找到的增量点只有：

- `inode_hit`: `mdcache_find_keyed_reason()` 在 CIH key lookup 成功后递增。
- `inode_added`: `mdcache_new_entry()` 成功创建并插入新 entry 后递增。
- `inode_mapping`: `mdc_check_mapping()` 检查/维护 entry-export 映射时递增。

没有找到 `inode_req`、`inode_miss`、`inode_conf` 的递增点。因此，不能直接用 DBus 的 `Cache Hits / Cache Misses` 做可信实时命中率，除非先补齐这些计数。

### 5.2 Prometheus 动态指标口径

动态指标定义在 `src/monitoring/dynamic_metrics.cc`：

- `mdcache_cache_hits_total{operation=...}`
- `mdcache_cache_misses_total{operation=...}`
- `mdcache_cache_hits_by_export_total{export=...,operation=...}`
- `mdcache_cache_misses_by_export_total{export=...,operation=...}`

它们由以下函数递增：

- `dynamic_metrics__mdcache_cache_hit(operation, export_id)`
- `dynamic_metrics__mdcache_cache_miss(operation, export_id)`

调用点目前集中在：

- `mdcache_getattrs()`，`operation="getattr"`。
- `mdc_lookup()`，`operation="lookup"`。

因此 Prometheus 口径是“被埋点的 MDCACHE lookup/getattr hit/miss”，不是所有内部 CIH 访问、READDIR chunk 访问、FD LRU 或属性刷新路径的全量统计。

启用条件：

- 编译期开启 `USE_MONITORING`，当前 `src/CMakeLists.txt` 默认 `USE_MONITORING` 为 ON。
- 配置 `Enable_Metrics = true`，否则 exporter 不启动。
- 配置 `Enable_Dynamic_Metrics = true`，否则动态指标对象不初始化，hit/miss 函数直接无效。
- 默认监听端口来自 `Monitoring_Port`，文档默认 9587。

## 6. 实时命中率计算

源码没有直接计算“实时命中率”这个 gauge，而是暴露 counter。实时命中率应在 Prometheus/Grafana 查询层用 `rate()` 或 `increase()` 计算。

### 6.1 全局实时命中率

1 分钟滑动窗口：

```promql
sum(rate(mdcache_cache_hits_total[1m]))
/
(
  sum(rate(mdcache_cache_hits_total[1m]))
  +
  sum(rate(mdcache_cache_misses_total[1m]))
)
```

5 分钟滑动窗口更平滑：

```promql
sum(rate(mdcache_cache_hits_total[5m]))
/
(
  sum(rate(mdcache_cache_hits_total[5m]))
  +
  sum(rate(mdcache_cache_misses_total[5m]))
)
```

### 6.2 按操作计算

```promql
sum by (operation) (rate(mdcache_cache_hits_total[1m]))
/
(
  sum by (operation) (rate(mdcache_cache_hits_total[1m]))
  +
  sum by (operation) (rate(mdcache_cache_misses_total[1m]))
)
```

当前有效 operation 主要是：

- `lookup`
- `getattr`

### 6.3 按 export 和操作计算

```promql
sum by (export, operation) (rate(mdcache_cache_hits_by_export_total[1m]))
/
(
  sum by (export, operation) (rate(mdcache_cache_hits_by_export_total[1m]))
  +
  sum by (export, operation) (rate(mdcache_cache_misses_by_export_total[1m]))
)
```

### 6.4 累计命中率

进程启动以来累计命中率：

```promql
sum(mdcache_cache_hits_total)
/
(
  sum(mdcache_cache_hits_total)
  +
  sum(mdcache_cache_misses_total)
)
```

某个时间段内累计命中率：

```promql
sum(increase(mdcache_cache_hits_total[30m]))
/
(
  sum(increase(mdcache_cache_hits_total[30m]))
  +
  sum(increase(mdcache_cache_misses_total[30m]))
)
```

### 6.5 分母为 0 的处理

窗口内没有 lookup/getattr 请求时，分母为 0。Grafana 面板建议处理为空值，或使用：

```promql
(
  sum(rate(mdcache_cache_hits_total[1m]))
  /
  clamp_min(
    sum(rate(mdcache_cache_hits_total[1m]))
    +
    sum(rate(mdcache_cache_misses_total[1m])),
    1
  )
)
```

这个写法会在无请求时显示 0，但从语义上“无请求”更适合显示为 N/A。

## 7. 关键操作命中判定

### 7.1 GETATTR

`mdcache_getattrs()` 的判定：

```text
Attr_Expiration_Time == 0
  -> miss，直接调用 sub-FSAL getattrs

读 attr_lock
  -> mdcache_is_attrs_valid() true
       -> hit，复制 entry->attrs 到 attrs_out

升级写锁后再检查
  -> 仍 valid
       -> hit
  -> invalid
       -> miss，调用 mdcache_refresh_attrs()
```

GETATTR hit 表示本次请求不需要访问下层 FSAL 刷新属性。

GETATTR miss 包括：

- 属性缓存被配置禁用。
- 请求属性 mask 不满足。
- trust 标志缺失。
- 属性过期。
- 目录启用了 getattr invalidation。
- ACL 或 fs_locations 等特殊属性未缓存或过期。

### 7.2 LOOKUP

`mdc_lookup()` 的判定：

```text
name == ".."
  -> miss 口径

未启用 dirent caching
  -> miss，调用 mdc_lookup_uncached()

父目录内容不可信
  -> miss，必要时 invalidate dirents

父目录 name AVL 找到 dirent 且 child key 在 CIH 中有效
  -> hit

否则
  -> miss，调用 sub-FSAL lookup，再创建/复用 MDCACHE entry
```

LOOKUP hit 表示“目录 name cache + child file-handle cache”都成功。

LOOKUP miss 不一定表示最终找到了对象；也可能是 negative lookup、stale、父目录缓存失效或禁用目录缓存。

## 8. 失效与一致性

MDCACHE 使用多种机制控制一致性：

- 属性超时：`Attr_Expiration_Time`。
- trust flags：`MDCACHE_TRUST_ATTRS`、`MDCACHE_TRUST_CONTENT`、`MDCACHE_TRUST_DIR_CHUNKS` 等。
- FSAL upcall：`mdcache_up.c` 可处理下层 FSAL 通知并更新/失效 entry。
- 目录 mtime 变化：属性刷新可通过 `invalidate` 标志触发 `mdcache_dirent_invalidate_all()`。
- 显式操作：create/remove/rename/link 等修改目录或对象时，会更新或失效相关 dirent。
- stale handle：下层返回 `ERR_FSAL_STALE` 时，可能 kill entry。
- export unexport：通过 `mdc_check_mapping()` 和 `cleanup_export/cleanup_pending` 防止 unexport 期间继续使用错误 export context。

锁粒度：

- CIH partition mutex 保护 file-handle AVL 和 direct cache slot。
- entry `attr_lock` 保护属性、export 映射和属性时间。
- entry `content_lock` 保护目录 AVL、dirent、chunk 和 symlink 内容。
- LRU lane mutex 保护每个 lane 的队列移动。

## 9. 监控建议

最小配置示例：

```conf
NFS_CORE_PARAM {
    Enable_Metrics = true;
    Enable_Dynamic_Metrics = true;
    Monitoring_Addr = 0.0.0.0;
    Monitoring_Port = 9587;
}
```

建议 Grafana 面板：

- 全局 MDCACHE hit rate，窗口 1m/5m。
- 按 `operation` 展示 lookup/getattr hit rate。
- 按 `export` 展示 hit rate，定位热点 export。
- 同时展示 hit/s 和 miss/s，避免只看比率导致误判。
- 同时展示 `nfs_requests_total` 或 `nfs_requests_by_export_total`，确认命中率窗口内有足够流量。

建议告警只在有足够请求量时触发，例如：

```promql
(
  sum(rate(mdcache_cache_hits_total[5m]))
  /
  (
    sum(rate(mdcache_cache_hits_total[5m]))
    +
    sum(rate(mdcache_cache_misses_total[5m]))
  )
) < 0.7
and
(
  sum(rate(mdcache_cache_hits_total[5m]))
  +
  sum(rate(mdcache_cache_misses_total[5m]))
) > 100
```

## 10. 源码层面的注意点

1. DBus `mdcache_stats` 不是完整命中率来源。当前树里 `inode_miss` 和 `inode_req` 没有递增点，不能用它们直接算 hit rate。
2. Prometheus hit/miss 只覆盖已埋点操作，当前主要是 `lookup` 和 `getattr`。
3. CIH direct cache 的 debug 日志 `cih cache hit` 是 file-handle 查找内部 fast path，不等同于业务层 MDCACHE hit rate。
4. READDIR chunk cache 没有独立 hit/miss counter。如果要衡量 READDIR chunk 命中率，需要新增指标。
5. negative lookup 当前会按 lookup miss 口径计数，即使它可能由可信目录缓存直接返回 `NOENT`。
6. `Attr_Expiration_Time = 0` 会导致 getattr 直接走 miss；调优命中率时应先确认 export/client 的最终 permission 里该值不是 0。
7. `Chunks_LWMark * Dir_Chunk` 会影响可长期保留的对象数量，可能让 entry 数持续高于 `Entries_HWMark`。

## 11. 可扩展的命中率补点建议

如果需要更完整的缓存观测，建议新增以下指标：

- `mdcache_fh_lookup_hits_total` / `mdcache_fh_lookup_misses_total`: CIH key lookup 层。
- `mdcache_dirent_lookup_hits_total` / `mdcache_dirent_lookup_misses_total`: 父目录 name AVL 层。
- `mdcache_attr_hits_total` / `mdcache_attr_misses_total`: 属性缓存层，可细分 miss reason。
- `mdcache_readdir_chunk_hits_total` / `mdcache_readdir_chunk_misses_total`: READDIR chunk 层。
- `mdcache_negative_lookup_hits_total`: 可信 negative cache 直接返回 `NOENT` 的次数。

建议在新增时明确 label 上限，避免动态 label 过多影响性能。已有 `dynamic_metrics.h` 也提示动态标签会影响性能。

## 12. 参考源码清单

- `README.md`
- `src/MainNFSD/nfs_main.c`
- `src/MainNFSD/nfs_init.c`
- `src/FSAL/fsal_manager.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_int.h`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_ext.h`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_main.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_hash.h`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_hash.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.h`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_lru.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_avl.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_helpers.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_handle.c`
- `src/FSAL/Stackable_FSALs/FSAL_MDCACHE/mdcache_read_conf.c`
- `src/monitoring/dynamic_metrics.cc`
- `src/monitoring/include/dynamic_metrics.h`
- `src/monitoring/prometheus_exposer.cc`
- `src/doc/man/ganesha-cache-config.rst`
- `src/doc/man/ganesha-core-config.rst`
- `src/doc/man/ganesha-export-config.rst`
