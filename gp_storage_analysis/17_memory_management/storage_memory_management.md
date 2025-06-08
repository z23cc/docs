# OP-TEE GP存储内存管理机制分析

## 内存管理架构概览

OP-TEE存储系统采用多层次的内存管理策略，包括对象缓存、块缓存、内存池和垃圾回收机制，以优化存储性能和内存使用效率。

```
┌─────────────────────────────────────────────────────────────┐
│                Object Cache Layer                          │  ← 对象级缓存
├─────────────────────────────────────────────────────────────┤
│                 Block Cache Layer                          │  ← 块级缓存
├─────────────────────────────────────────────────────────────┤
│                Memory Pool Layer                           │  ← 内存池管理
├─────────────────────────────────────────────────────────────┤
│              Hash Tree Cache Layer                         │  ← 哈希树缓存
├─────────────────────────────────────────────────────────────┤
│                Buffer Management                           │  ← 缓冲区管理
├─────────────────────────────────────────────────────────────┤
│              Garbage Collection                            │  ← 垃圾回收
└─────────────────────────────────────────────────────────────┘
```

## 核心内存管理组件

### 1. 对象缓存管理

#### 持久化对象缓存
位置：`optee_os/core/tee/tee_pobj.c`

```c
// 对象缓存结构
struct pobj_cache {
    struct mutex cache_mutex;       // 缓存锁
    TAILQ_HEAD(pobj_head, tee_pobj) objects; // 对象链表
    size_t max_objects;             // 最大对象数
    size_t current_objects;         // 当前对象数
    uint64_t hit_count;             // 命中计数
    uint64_t miss_count;            // 未命中计数
};

static struct pobj_cache global_pobj_cache = {
    .cache_mutex = MUTEX_INITIALIZER,
    .objects = TAILQ_HEAD_INITIALIZER(global_pobj_cache.objects),
    .max_objects = CFG_TEE_POBJ_CACHE_SIZE,
    .current_objects = 0,
    .hit_count = 0,
    .miss_count = 0
};

// 对象缓存查找
static struct tee_pobj *pobj_cache_find(const TEE_UUID *uuid,
                                       void *obj_id, uint32_t obj_id_len)
{
    struct tee_pobj *po;
    
    mutex_lock(&global_pobj_cache.cache_mutex);
    
    TAILQ_FOREACH(po, &global_pobj_cache.objects, link) {
        if (uuid_equal(&po->uuid, uuid) &&
            obj_id_len == po->objectID_len &&
            !memcmp(obj_id, po->objectID, obj_id_len)) {
            
            // 移到链表头部（LRU策略）
            TAILQ_REMOVE(&global_pobj_cache.objects, po, link);
            TAILQ_INSERT_HEAD(&global_pobj_cache.objects, po, link);
            
            global_pobj_cache.hit_count++;
            mutex_unlock(&global_pobj_cache.cache_mutex);
            return po;
        }
    }
    
    global_pobj_cache.miss_count++;
    mutex_unlock(&global_pobj_cache.cache_mutex);
    return NULL;
}

// 对象缓存淘汰策略（LRU）
static void pobj_cache_evict_lru(void)
{
    struct tee_pobj *po;
    
    mutex_lock(&global_pobj_cache.cache_mutex);
    
    while (global_pobj_cache.current_objects >= global_pobj_cache.max_objects) {
        // 从尾部移除（最久未使用）
        po = TAILQ_LAST(&global_pobj_cache.objects, pobj_head);
        if (!po)
            break;
            
        // 检查引用计数
        if (po->refcount == 0) {
            TAILQ_REMOVE(&global_pobj_cache.objects, po, link);
            global_pobj_cache.current_objects--;
            
            // 释放对象资源
            tee_pobj_release(po);
        } else {
            // 有引用的对象不能淘汰
            break;
        }
    }
    
    mutex_unlock(&global_pobj_cache.cache_mutex);
}
```

### 2. 块级缓存管理

#### REE文件系统块缓存
位置：`optee_os/core/tee/tee_ree_fs.c`

```c
// 块缓存结构
struct block_cache {
    struct mutex cache_mutex;       // 缓存锁
    struct block_entry *entries;    // 缓存条目数组
    size_t num_entries;             // 条目数量
    size_t entry_size;              // 条目大小
    struct block_allocator *allocator; // 内存分配器
    struct cache_stats stats;       // 缓存统计
};

struct block_entry {
    struct mutex entry_mutex;       // 条目锁
    bool valid;                     // 有效标志
    bool dirty;                     // 脏数据标志
    uint64_t block_id;              // 块ID
    uint64_t access_time;           // 访问时间
    void *data;                     // 数据指针
    struct condvar write_complete;  // 写完成条件变量
};

// 块缓存初始化
static TEE_Result block_cache_init(struct block_cache **cache,
                                  size_t num_entries, size_t entry_size)
{
    struct block_cache *c;
    
    c = calloc(1, sizeof(*c));
    if (!c)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 分配缓存条目
    c->entries = calloc(num_entries, sizeof(struct block_entry));
    if (!c->entries) {
        free(c);
        return TEE_ERROR_OUT_OF_MEMORY;
    }
    
    // 初始化内存分配器
    c->allocator = block_allocator_create(entry_size, num_entries);
    if (!c->allocator) {
        free(c->entries);
        free(c);
        return TEE_ERROR_OUT_OF_MEMORY;
    }
    
    mutex_init(&c->cache_mutex);
    c->num_entries = num_entries;
    c->entry_size = entry_size;
    
    // 初始化所有条目
    for (size_t i = 0; i < num_entries; i++) {
        struct block_entry *e = &c->entries[i];
        mutex_init(&e->entry_mutex);
        condvar_init(&e->write_complete);
        e->data = block_allocator_alloc(c->allocator);
    }
    
    *cache = c;
    return TEE_SUCCESS;
}

// 块缓存查找和获取
static TEE_Result block_cache_get(struct block_cache *cache,
                                 uint64_t block_id,
                                 struct block_entry **entry)
{
    struct block_entry *e;
    struct block_entry *lru_entry = NULL;
    uint64_t oldest_time = UINT64_MAX;
    
    mutex_lock(&cache->cache_mutex);
    
    // 查找现有条目
    for (size_t i = 0; i < cache->num_entries; i++) {
        e = &cache->entries[i];
        
        if (e->valid && e->block_id == block_id) {
            // 命中缓存
            e->access_time = get_current_time();
            cache->stats.hit_count++;
            
            mutex_lock(&e->entry_mutex);
            mutex_unlock(&cache->cache_mutex);
            
            *entry = e;
            return TEE_SUCCESS;
        }
        
        // 记录LRU条目
        if (!e->valid || e->access_time < oldest_time) {
            oldest_time = e->access_time;
            lru_entry = e;
        }
    }
    
    // 缓存未命中，使用LRU条目
    if (lru_entry) {
        cache->stats.miss_count++;
        
        mutex_lock(&lru_entry->entry_mutex);
        
        // 如果条目脏，先写回
        if (lru_entry->dirty) {
            TEE_Result res = write_back_block(lru_entry);
            if (res != TEE_SUCCESS) {
                mutex_unlock(&lru_entry->entry_mutex);
                mutex_unlock(&cache->cache_mutex);
                return res;
            }
        }
        
        // 更新条目信息
        lru_entry->block_id = block_id;
        lru_entry->access_time = get_current_time();
        lru_entry->valid = true;
        lru_entry->dirty = false;
        
        mutex_unlock(&cache->cache_mutex);
        
        *entry = lru_entry;
        return TEE_SUCCESS;
    }
    
    mutex_unlock(&cache->cache_mutex);
    return TEE_ERROR_OUT_OF_MEMORY;
}
```

#### RPMB缓存管理
位置：`optee_os/core/tee/tee_rpmb_fs.c`

```c
// RPMB缓存配置
#define RPMB_CACHE_SIZE         CFG_RPMB_FS_CACHE_ENTRIES
#define RPMB_BLOCK_SIZE         256

struct rpmb_cache {
    struct mutex cache_mutex;
    struct rpmb_cache_entry entries[RPMB_CACHE_SIZE];
    uint32_t next_evict_idx;    // 下一个淘汰索引（轮转策略）
    struct cache_stats stats;
};

struct rpmb_cache_entry {
    struct mutex entry_mutex;
    bool valid;
    bool dirty;
    uint16_t block_addr;
    uint8_t data[RPMB_BLOCK_SIZE];
    uint64_t last_access;
};

static struct rpmb_cache g_rpmb_cache = {
    .cache_mutex = MUTEX_INITIALIZER,
    .next_evict_idx = 0
};

// RPMB缓存获取
static TEE_Result rpmb_cache_get_block(uint16_t block_addr,
                                      struct rpmb_cache_entry **entry)
{
    struct rpmb_cache_entry *e;
    
    mutex_lock(&g_rpmb_cache.cache_mutex);
    
    // 查找现有缓存
    for (int i = 0; i < RPMB_CACHE_SIZE; i++) {
        e = &g_rpmb_cache.entries[i];
        
        if (e->valid && e->block_addr == block_addr) {
            e->last_access = get_current_time();
            g_rpmb_cache.stats.hit_count++;
            
            mutex_lock(&e->entry_mutex);
            mutex_unlock(&g_rpmb_cache.cache_mutex);
            
            *entry = e;
            return TEE_SUCCESS;
        }
    }
    
    // 缓存未命中，选择淘汰条目
    uint32_t evict_idx = g_rpmb_cache.next_evict_idx;
    e = &g_rpmb_cache.entries[evict_idx];
    
    g_rpmb_cache.next_evict_idx = (evict_idx + 1) % RPMB_CACHE_SIZE;
    g_rpmb_cache.stats.miss_count++;
    
    mutex_lock(&e->entry_mutex);
    
    // 如果条目脏，写回RPMB
    if (e->dirty) {
        TEE_Result res = rpmb_write_block(e->block_addr, e->data);
        if (res != TEE_SUCCESS) {
            mutex_unlock(&e->entry_mutex);
            mutex_unlock(&g_rpmb_cache.cache_mutex);
            return res;
        }
        e->dirty = false;
    }
    
    // 从RPMB读取新数据
    TEE_Result res = rpmb_read_block(block_addr, e->data);
    if (res != TEE_SUCCESS) {
        e->valid = false;
        mutex_unlock(&e->entry_mutex);
        mutex_unlock(&g_rpmb_cache.cache_mutex);
        return res;
    }
    
    // 更新缓存条目
    e->block_addr = block_addr;
    e->valid = true;
    e->last_access = get_current_time();
    
    mutex_unlock(&g_rpmb_cache.cache_mutex);
    
    *entry = e;
    return TEE_SUCCESS;
}
```

### 3. 内存池管理

#### 块分配器实现
```c
// 内存池结构
struct block_allocator {
    struct mutex alloc_mutex;       // 分配锁
    void *memory_pool;              // 内存池基址
    size_t pool_size;               // 池大小
    size_t block_size;              // 块大小
    size_t num_blocks;              // 块数量
    uint8_t *free_bitmap;           // 空闲位图
    size_t free_count;              // 空闲块数
    struct block_list free_list;    // 空闲块链表
};

// 内存池初始化
static struct block_allocator *block_allocator_create(size_t block_size,
                                                     size_t num_blocks)
{
    struct block_allocator *alloc;
    size_t pool_size = block_size * num_blocks;
    size_t bitmap_size = (num_blocks + 7) / 8;
    
    alloc = calloc(1, sizeof(*alloc));
    if (!alloc)
        return NULL;
    
    // 分配内存池
    alloc->memory_pool = malloc(pool_size);
    if (!alloc->memory_pool) {
        free(alloc);
        return NULL;
    }
    
    // 分配位图
    alloc->free_bitmap = calloc(1, bitmap_size);
    if (!alloc->free_bitmap) {
        free(alloc->memory_pool);
        free(alloc);
        return NULL;
    }
    
    mutex_init(&alloc->alloc_mutex);
    alloc->pool_size = pool_size;
    alloc->block_size = block_size;
    alloc->num_blocks = num_blocks;
    alloc->free_count = num_blocks;
    
    // 初始化空闲链表
    TAILQ_INIT(&alloc->free_list);
    for (size_t i = 0; i < num_blocks; i++) {
        struct block_node *node = (struct block_node *)
            ((uint8_t *)alloc->memory_pool + i * block_size);
        TAILQ_INSERT_TAIL(&alloc->free_list, node, link);
    }
    
    return alloc;
}

// 内存块分配
static void *block_allocator_alloc(struct block_allocator *alloc)
{
    struct block_node *node;
    
    mutex_lock(&alloc->alloc_mutex);
    
    // 从空闲链表获取块
    node = TAILQ_FIRST(&alloc->free_list);
    if (node) {
        TAILQ_REMOVE(&alloc->free_list, node, link);
        alloc->free_count--;
        
        // 更新位图
        size_t block_idx = ((uint8_t *)node - (uint8_t *)alloc->memory_pool) 
                          / alloc->block_size;
        alloc->free_bitmap[block_idx / 8] |= (1 << (block_idx % 8));
    }
    
    mutex_unlock(&alloc->alloc_mutex);
    
    return node;
}

// 内存块释放
static void block_allocator_free(struct block_allocator *alloc, void *ptr)
{
    if (!ptr)
        return;
    
    mutex_lock(&alloc->alloc_mutex);
    
    // 计算块索引
    size_t block_idx = ((uint8_t *)ptr - (uint8_t *)alloc->memory_pool) 
                      / alloc->block_size;
    
    // 检查是否已经空闲
    if (!(alloc->free_bitmap[block_idx / 8] & (1 << (block_idx % 8)))) {
        // 已经空闲，可能是双重释放
        mutex_unlock(&alloc->alloc_mutex);
        return;
    }
    
    // 清除位图标记
    alloc->free_bitmap[block_idx / 8] &= ~(1 << (block_idx % 8));
    
    // 添加到空闲链表
    struct block_node *node = (struct block_node *)ptr;
    TAILQ_INSERT_HEAD(&alloc->free_list, node, link);
    alloc->free_count++;
    
    mutex_unlock(&alloc->alloc_mutex);
}
```

### 4. 哈希树内存管理

#### 位置：`optee_os/core/tee/fs_htree.c`

```c
// 哈希树节点缓存
struct htree_node_cache {
    struct mutex cache_mutex;
    struct htree_cache_entry *entries;
    size_t max_entries;
    size_t current_entries;
    struct mem_pool *node_pool;     // 节点内存池
};

struct htree_cache_entry {
    struct mutex entry_mutex;
    bool valid;
    bool dirty;
    uint32_t node_id;
    struct tee_fs_htree_node_image *node;
    uint64_t last_access;
    struct condvar flush_complete;
};

// 哈希树节点分配
static struct tee_fs_htree_node_image *htree_alloc_node(void)
{
    struct tee_fs_htree_node_image *node;
    
    // 从内存池分配
    node = mem_pool_alloc(htree_node_pool, sizeof(*node));
    if (!node) {
        // 触发垃圾回收
        htree_garbage_collect();
        
        // 再次尝试分配
        node = mem_pool_alloc(htree_node_pool, sizeof(*node));
    }
    
    if (node) {
        memset(node, 0, sizeof(*node));
    }
    
    return node;
}

// 哈希树节点释放
static void htree_free_node(struct tee_fs_htree_node_image *node)
{
    if (node) {
        // 清理节点数据
        memset(node, 0, sizeof(*node));
        
        // 返回内存池
        mem_pool_free(htree_node_pool, node);
    }
}

// 哈希树缓存垃圾回收
static void htree_garbage_collect(void)
{
    struct htree_node_cache *cache = &global_htree_cache;
    uint64_t current_time = get_current_time();
    uint64_t evict_threshold = current_time - HTREE_CACHE_TIMEOUT;
    
    mutex_lock(&cache->cache_mutex);
    
    for (size_t i = 0; i < cache->max_entries; i++) {
        struct htree_cache_entry *entry = &cache->entries[i];
        
        if (!entry->valid)
            continue;
        
        // 检查是否超时
        if (entry->last_access < evict_threshold) {
            mutex_lock(&entry->entry_mutex);
            
            // 如果脏，先刷新
            if (entry->dirty) {
                flush_htree_node(entry);
            }
            
            // 释放节点
            htree_free_node(entry->node);
            entry->valid = false;
            cache->current_entries--;
            
            mutex_unlock(&entry->entry_mutex);
        }
    }
    
    mutex_unlock(&cache->cache_mutex);
}
```

### 5. 缓冲区管理

#### 动态缓冲区分配
```c
// 缓冲区管理器
struct buffer_manager {
    struct mutex buffer_mutex;
    struct buffer_pool small_pool;    // 小缓冲区池 (≤1KB)
    struct buffer_pool medium_pool;   // 中缓冲区池 (≤4KB)
    struct buffer_pool large_pool;    // 大缓冲区池 (≤16KB)
    struct buffer_stats stats;
};

struct buffer_pool {
    size_t buffer_size;
    size_t num_buffers;
    void **free_buffers;
    size_t free_count;
    size_t total_allocated;
};

// 智能缓冲区分配
static void *buffer_alloc(size_t size)
{
    struct buffer_manager *mgr = &global_buffer_manager;
    struct buffer_pool *pool;
    void *buffer = NULL;
    
    mutex_lock(&mgr->buffer_mutex);
    
    // 选择合适的缓冲区池
    if (size <= 1024) {
        pool = &mgr->small_pool;
    } else if (size <= 4096) {
        pool = &mgr->medium_pool;
    } else if (size <= 16384) {
        pool = &mgr->large_pool;
    } else {
        // 大于16KB，直接分配
        buffer = malloc(size);
        mgr->stats.large_alloc_count++;
        goto out;
    }
    
    // 从池中获取缓冲区
    if (pool->free_count > 0) {
        buffer = pool->free_buffers[--pool->free_count];
        mgr->stats.pool_hit_count++;
    } else {
        // 池中无可用缓冲区，直接分配
        buffer = malloc(pool->buffer_size);
        pool->total_allocated++;
        mgr->stats.pool_miss_count++;
    }
    
out:
    mutex_unlock(&mgr->buffer_mutex);
    return buffer;
}

// 缓冲区释放
static void buffer_free(void *buffer, size_t size)
{
    struct buffer_manager *mgr = &global_buffer_manager;
    struct buffer_pool *pool;
    
    if (!buffer)
        return;
    
    mutex_lock(&mgr->buffer_mutex);
    
    // 选择对应的缓冲区池
    if (size <= 1024) {
        pool = &mgr->small_pool;
    } else if (size <= 4096) {
        pool = &mgr->medium_pool;
    } else if (size <= 16384) {
        pool = &mgr->large_pool;
    } else {
        // 大缓冲区直接释放
        free(buffer);
        goto out;
    }
    
    // 如果池未满，返回池中
    if (pool->free_count < pool->num_buffers) {
        pool->free_buffers[pool->free_count++] = buffer;
    } else {
        // 池已满，直接释放
        free(buffer);
        pool->total_allocated--;
    }
    
out:
    mutex_unlock(&mgr->buffer_mutex);
}
```

### 6. 垃圾回收机制

#### 定期垃圾回收
```c
// 垃圾回收器
struct garbage_collector {
    struct thread *gc_thread;       // GC线程
    struct condvar gc_trigger;      // GC触发条件变量
    struct mutex gc_mutex;          // GC锁
    bool gc_running;                // GC运行标志
    uint32_t gc_interval_ms;        // GC间隔
    struct gc_stats stats;          // GC统计
};

static struct garbage_collector global_gc = {
    .gc_mutex = MUTEX_INITIALIZER,
    .gc_interval_ms = CFG_GC_INTERVAL_MS,
    .gc_running = false
};

// GC线程主循环
static void garbage_collector_thread(void *arg)
{
    struct garbage_collector *gc = (struct garbage_collector *)arg;
    
    while (gc->gc_running) {
        mutex_lock(&gc->gc_mutex);
        
        // 等待GC触发或超时
        condvar_wait_timeout(&gc->gc_trigger, &gc->gc_mutex, 
                           gc->gc_interval_ms);
        
        mutex_unlock(&gc->gc_mutex);
        
        if (!gc->gc_running)
            break;
        
        // 执行垃圾回收
        gc_collect_garbage();
    }
}

// 垃圾回收实现
static void gc_collect_garbage(void)
{
    uint64_t start_time = get_current_time();
    size_t freed_bytes = 0;
    
    // 1. 回收对象缓存
    freed_bytes += gc_collect_object_cache();
    
    // 2. 回收块缓存
    freed_bytes += gc_collect_block_cache();
    
    // 3. 回收哈希树缓存
    freed_bytes += gc_collect_htree_cache();
    
    // 4. 回收缓冲区池
    freed_bytes += gc_collect_buffer_pools();
    
    // 5. 合并内存池碎片
    gc_defragment_memory_pools();
    
    uint64_t gc_time = get_current_time() - start_time;
    
    // 更新统计信息
    global_gc.stats.total_collections++;
    global_gc.stats.total_freed_bytes += freed_bytes;
    global_gc.stats.total_gc_time += gc_time;
    
    DMSG("GC completed: freed %zu bytes in %llu ms", 
         freed_bytes, gc_time);
}

// 内存压力触发GC
static void trigger_gc_on_memory_pressure(void)
{
    size_t free_memory = get_free_memory_size();
    size_t total_memory = get_total_memory_size();
    
    // 如果可用内存低于阈值，触发GC
    if (free_memory < total_memory * CFG_GC_MEMORY_THRESHOLD / 100) {
        mutex_lock(&global_gc.gc_mutex);
        condvar_signal(&global_gc.gc_trigger);
        mutex_unlock(&global_gc.gc_mutex);
    }
}
```

## 内存使用优化策略

### 1. 预分配策略
```c
// 预分配常用大小的内存块
static void preallocate_memory_pools(void)
{
    // 预分配对象池
    pobj_pool = create_memory_pool(sizeof(struct tee_pobj), 
                                  CFG_MAX_POBJ_COUNT);
    
    // 预分配块缓存池
    block_cache_pool = create_memory_pool(BLOCK_CACHE_ENTRY_SIZE,
                                         CFG_BLOCK_CACHE_SIZE);
    
    // 预分配哈希树节点池
    htree_node_pool = create_memory_pool(sizeof(struct tee_fs_htree_node_image),
                                        CFG_HTREE_NODE_COUNT);
    
    // 预分配RPC缓冲区池
    rpc_buffer_pool = create_memory_pool(RPC_BUFFER_SIZE,
                                        CFG_RPC_BUFFER_COUNT);
}
```

### 2. 内存对齐优化
```c
// 内存对齐分配
static void *aligned_alloc(size_t size, size_t alignment)
{
    void *ptr;
    size_t aligned_size = ROUNDUP(size, alignment);
    
    // 确保分配地址对齐到指定边界
    if (posix_memalign(&ptr, alignment, aligned_size) != 0) {
        return NULL;
    }
    
    return ptr;
}

// 缓存行对齐的分配（提高缓存性能）
static void *cache_aligned_alloc(size_t size)
{
    return aligned_alloc(size, CPU_CACHE_LINE_SIZE);
}
```

### 3. 零拷贝优化
```c
// 零拷贝缓冲区共享
struct shared_buffer {
    void *data;
    size_t size;
    atomic_t refcount;
    void (*free_func)(void *data);
};

static struct shared_buffer *create_shared_buffer(void *data, size_t size,
                                                 void (*free_func)(void *))
{
    struct shared_buffer *buf = malloc(sizeof(*buf));
    if (!buf)
        return NULL;
    
    buf->data = data;
    buf->size = size;
    atomic_set(&buf->refcount, 1);
    buf->free_func = free_func;
    
    return buf;
}

static void shared_buffer_get(struct shared_buffer *buf)
{
    atomic_inc(&buf->refcount);
}

static void shared_buffer_put(struct shared_buffer *buf)
{
    if (atomic_dec_and_test(&buf->refcount)) {
        if (buf->free_func) {
            buf->free_func(buf->data);
        }
        free(buf);
    }
}
```

## 内存监控和调试

### 内存使用统计
```c
// 内存使用统计结构
struct memory_stats {
    size_t total_allocated;         // 总分配内存
    size_t total_freed;             // 总释放内存
    size_t current_usage;           // 当前使用量
    size_t peak_usage;              // 峰值使用量
    size_t allocation_count;        // 分配次数
    size_t free_count;              // 释放次数
    size_t cache_hit_rate;          // 缓存命中率
    size_t gc_freed_bytes;          // GC释放字节数
};

// 获取内存统计信息
static void get_memory_stats(struct memory_stats *stats)
{
    memset(stats, 0, sizeof(*stats));
    
    // 收集各组件的内存统计
    stats->total_allocated += get_pobj_memory_usage();
    stats->total_allocated += get_block_cache_memory_usage();
    stats->total_allocated += get_htree_memory_usage();
    stats->total_allocated += get_buffer_pool_memory_usage();
    
    // 计算缓存命中率
    uint64_t total_hits = global_pobj_cache.hit_count + 
                         g_rpmb_cache.stats.hit_count;
    uint64_t total_requests = total_hits + 
                             global_pobj_cache.miss_count + 
                             g_rpmb_cache.stats.miss_count;
    
    if (total_requests > 0) {
        stats->cache_hit_rate = (total_hits * 100) / total_requests;
    }
    
    stats->gc_freed_bytes = global_gc.stats.total_freed_bytes;
}
```

### 内存泄漏检测
```c
#ifdef CFG_MEMORY_LEAK_DETECTION

struct alloc_record {
    void *ptr;
    size_t size;
    const char *file;
    int line;
    uint64_t timestamp;
    struct alloc_record *next;
};

static struct alloc_record *alloc_records = NULL;
static struct mutex alloc_records_mutex = MUTEX_INITIALIZER;

// 记录内存分配
void record_allocation(void *ptr, size_t size, const char *file, int line)
{
    struct alloc_record *record = malloc(sizeof(*record));
    if (!record)
        return;
    
    record->ptr = ptr;
    record->size = size;
    record->file = file;
    record->line = line;
    record->timestamp = get_current_time();
    
    mutex_lock(&alloc_records_mutex);
    record->next = alloc_records;
    alloc_records = record;
    mutex_unlock(&alloc_records_mutex);
}

// 移除分配记录
void remove_allocation_record(void *ptr)
{
    struct alloc_record **current;
    
    mutex_lock(&alloc_records_mutex);
    
    for (current = &alloc_records; *current; current = &(*current)->next) {
        if ((*current)->ptr == ptr) {
            struct alloc_record *record = *current;
            *current = record->next;
            free(record);
            break;
        }
    }
    
    mutex_unlock(&alloc_records_mutex);
}

// 检查内存泄漏
void check_memory_leaks(void)
{
    struct alloc_record *record;
    size_t leak_count = 0;
    size_t leak_bytes = 0;
    
    mutex_lock(&alloc_records_mutex);
    
    for (record = alloc_records; record; record = record->next) {
        EMSG("Memory leak: %p (%zu bytes) allocated at %s:%d",
             record->ptr, record->size, record->file, record->line);
        leak_count++;
        leak_bytes += record->size;
    }
    
    mutex_unlock(&alloc_records_mutex);
    
    if (leak_count > 0) {
        EMSG("Total memory leaks: %zu allocations, %zu bytes",
             leak_count, leak_bytes);
    }
}

#define malloc(size) debug_malloc(size, __FILE__, __LINE__)
#define free(ptr) debug_free(ptr)

#endif /* CFG_MEMORY_LEAK_DETECTION */
```

## 性能调优建议

### 1. 缓存大小调优
- **对象缓存**: 根据并发TA数量调整`CFG_TEE_POBJ_CACHE_SIZE`
- **块缓存**: 根据I/O模式调整块缓存大小
- **RPMB缓存**: 平衡内存使用和RPMB访问性能

### 2. 内存池配置
- **预分配**: 根据预期负载预分配常用对象
- **池大小**: 避免过大的内存池造成内存浪费
- **对齐**: 使用适当的内存对齐提高访问性能

### 3. 垃圾回收优化
- **触发条件**: 调整GC触发的内存阈值
- **回收频率**: 平衡GC开销和内存使用
- **增量回收**: 实现增量GC减少延迟

## 总结

OP-TEE的GP存储内存管理机制具有以下特点：

1. **多层次缓存**: 对象、块、哈希树的分层缓存策略
2. **内存池管理**: 高效的内存分配和回收机制
3. **垃圾回收**: 自动内存回收和碎片整理
4. **性能优化**: 零拷贝、预分配、对齐等优化技术
5. **监控调试**: 完整的内存使用监控和泄漏检测

这个内存管理框架为GP存储系统提供了高效、可靠的内存服务，确保了系统的性能和稳定性。