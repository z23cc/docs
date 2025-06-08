# OP-TEE存储配额和资源管理深度分析

## 存储资源管理概述

OP-TEE存储系统实现了多层次的资源管理和配额控制机制，从GP API层面的限制到底层存储后端的资源约束，确保系统稳定性和安全性。

## GP API层面的存储限制

### 1. 基本存储限制常量

```c
// 位置: /lib/libutee/include/tee_api_defines.h

// 对象标识符最大长度
#define TEE_OBJECT_ID_MAX_LEN              64          // 64字节

// 数据流最大位置
#define TEE_DATA_MAX_POSITION              0xFFFFFFFF  // 4GB-1

// 文件名最大长度 (文件系统层)
#define TEE_FS_NAME_MAX                    U(350)      // 350字符

// RPMB文件名最大长度
#define TEE_RPMB_FS_FILENAME_LENGTH        224         // 224字节
```

### 2. 对象大小管理

#### TEE_ObjectInfo结构的大小字段
```c
typedef struct {
    uint32_t objectType;            // 对象类型
    uint32_t objectSize;            // 当前对象大小(位)
    uint32_t maxObjectSize;         // 最大对象大小(位)
    uint32_t objectUsage;           // 对象用途标志
    uint32_t dataSize;              // 数据大小(字节)
    uint32_t dataPosition;          // 当前数据位置(字节)
    uint32_t handleFlags;           // 句柄标志
} TEE_ObjectInfo;
```

#### 对象大小验证机制
```c
// 对象创建时的大小验证
static TEE_Result validate_object_size(uint32_t objectType, 
                                       uint32_t maxObjectSize,
                                       size_t initialDataLen)
{
    // 1. 检查最大对象大小限制
    if (maxObjectSize > get_max_object_size_for_type(objectType)) {
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    // 2. 检查初始数据大小
    if (initialDataLen > (maxObjectSize / 8)) {  // 位转字节
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    // 3. 检查数据流位置限制
    if (maxObjectSize > TEE_DATA_MAX_POSITION * 8) {
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    return TEE_SUCCESS;
}
```

### 3. 对象ID长度验证

```c
// 对象ID长度检查
static TEE_Result validate_object_id(const void *objectID, size_t objectIDLen)
{
    if (!objectID && objectIDLen != 0) {
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    if (objectIDLen > TEE_OBJECT_ID_MAX_LEN) {
        EMSG("Object ID length %zu exceeds maximum %d", 
             objectIDLen, TEE_OBJECT_ID_MAX_LEN);
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    // 检查对象ID有效性
    if (objectIDLen > 0 && !objectID) {
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    return TEE_SUCCESS;
}
```

## 存储后端资源限制

### 1. REE文件系统后端限制

#### 块大小和内存管理
```c
// 位置: /core/tee/tee_ree_fs.c

#define BLOCK_SHIFT                    12              // 4KB块
#define BLOCK_SIZE                     (1 << BLOCK_SHIFT)  // 4096字节

// 内存池管理
static void *get_tmp_block(void)
{
    return mempool_alloc(mempool_default, BLOCK_SIZE);
}

static void put_tmp_block(void *tmp_block)
{
    mempool_free(mempool_default, tmp_block);
}
```

#### 文件路径长度限制
```c
// REE端supplicant文件路径限制
#ifndef PATH_MAX
#define PATH_MAX 255                    // 255字符路径长度
#endif

// 绝对路径构建检查
static size_t tee_fs_get_absolute_filename(char *file, char *out, size_t out_size)
{
    int s = 0;
    
    if (!file || !out || (out_size <= strlen(tee_fs_root) + 1))
        return 0;
    
    s = snprintf(out, out_size, "%s%s", tee_fs_root, file);
    if (s < 0 || (size_t)s >= out_size)
        return 0;
    
    return (size_t)s;
}
```

#### 哈希树资源管理
```c
// 哈希树节点数量限制
struct tee_fs_htree_meta {
    uint64_t length;                    // 文件长度
};

struct tee_fs_htree_imeta {
    struct tee_fs_htree_meta meta;
    uint32_t max_node_id;               // 最大节点ID
};

// 节点ID到块号转换
#define BLOCK_NUM_TO_NODE_ID(num)       ((num) + 1)
#define NODE_ID_TO_BLOCK_NUM(id)        ((id) - 1)

// 哈希树深度计算和限制
static uint32_t calculate_tree_depth(uint64_t file_size)
{
    uint32_t blocks = (file_size + BLOCK_SIZE - 1) / BLOCK_SIZE;
    uint32_t depth = 0;
    uint32_t nodes = blocks;
    
    while (nodes > 1) {
        nodes = (nodes + 1) / 2;  // 二叉树每层节点数
        depth++;
    }
    
    return depth;
}
```

### 2. RPMB文件系统后端限制

#### RPMB物理限制
```c
// 位置: /core/tee/tee_rpmb_fs.c

#define RPMB_BLOCK_SIZE_SHIFT           8               // 256字节块
#define RPMB_DATA_SIZE                  256             // 256字节数据区
#define RPMB_SIZE_SINGLE                (128 * 1024)    // 128KB单个RPMB分区

// RPMB设备信息和容量
struct rpmb_dev_info {
    uint8_t cid[RPMB_EMMC_CID_SIZE];        // eMMC CID
    uint8_t rpmb_size_mult;                 // RPMB大小倍数
    uint8_t rel_wr_sec_c;                   // 可靠写扇区数
    uint8_t ret_code;                       // 返回码
};

// RPMB容量计算
static uint32_t calculate_rpmb_capacity(uint8_t rpmb_size_mult)
{
    // RPMB容量 = rpmb_size_mult * 128KB
    return rpmb_size_mult * RPMB_SIZE_SINGLE;
}
```

#### RPMB缓存配置
```c
// RPMB缓存配置选项
#ifndef CFG_RPMB_FS_CACHE_ENTRIES
#define CFG_RPMB_FS_CACHE_ENTRIES       0              // 默认无缓存
#endif

#ifndef CFG_RPMB_FS_RD_ENTRIES  
#define CFG_RPMB_FS_RD_ENTRIES          1              // 默认单次读1个条目
#endif

// 缓存总大小
#define RPMB_BUF_MAX_ENTRIES (CFG_RPMB_FS_CACHE_ENTRIES + CFG_RPMB_FS_RD_ENTRIES)

// FAT条目缓存管理
struct rpmb_fat_entry_dir {
    struct rpmb_fat_entry *rpmb_fat_entry_buf;          // 缓冲区
    uint32_t idx_curr;                                  // 当前索引
    uint32_t num_buffered;                              // 缓冲条目数
    uint32_t num_total_read;                            // 总读取数
    bool last_reached;                                  // 末尾标志
};

// 缓存初始化
static TEE_Result init_fat_cache(void)
{
    size_t cache_size = RPMB_BUF_MAX_ENTRIES * sizeof(struct rpmb_fat_entry);
    
    fat_entry_dir = calloc(1, sizeof(struct rpmb_fat_entry_dir));
    if (!fat_entry_dir)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    if (RPMB_BUF_MAX_ENTRIES > 0) {
        fat_entry_dir->rpmb_fat_entry_buf = malloc(cache_size);
        if (!fat_entry_dir->rpmb_fat_entry_buf) {
            free(fat_entry_dir);
            return TEE_ERROR_OUT_OF_MEMORY;
        }
    }
    
    return TEE_SUCCESS;
}
```

#### RPMB写计数器限制
```c
// RPMB写计数器管理
struct tee_rpmb_ctx {
    uint32_t wr_cnt;                        // 当前写计数器
    uint16_t max_blk_idx;                   // 最大块索引
    uint16_t rel_wr_blkcnt;                 // 可靠写块数限制
    // ... 其他字段
};

// 写计数器耗尽检查
static TEE_Result check_write_counter_limit(void)
{
    // 检查写计数器是否接近上限
    if (rpmb_ctx->wr_cnt >= 0xFFFFFFF0) {  // 接近32位上限
        EMSG("RPMB write counter approaching limit: %u", rpmb_ctx->wr_cnt);
        return TEE_ERROR_STORAGE_NOT_AVAILABLE;
    }
    
    return TEE_SUCCESS;
}
```

## 内存资源管理

### 1. 对象句柄管理

#### 每TA的对象句柄限制
```c
// 对象句柄池管理
struct tee_obj {
    TAILQ_ENTRY(tee_obj) link;              // 链表节点
    TEE_ObjectInfo info;                    // 对象信息
    bool busy;                              // 忙标志
    uint32_t have_attrs;                    // 属性掩码
    void *attr;                             // 属性数据
    size_t ds_pos;                          // 数据流位置
    struct tee_pobj *pobj;                  // 持久化对象
    struct tee_file_handle *fh;             // 文件句柄
};

// TA对象管理
struct user_ta_ctx {
    TAILQ_HEAD(tee_obj_head, tee_obj) objects; // 对象链表
    TAILQ_HEAD(tee_storage_enum_head, tee_storage_enum) storage_enums; // 枚举器链表
    // ... 其他字段
};

// 对象数量统计和限制
static size_t count_ta_objects(struct user_ta_ctx *utc)
{
    struct tee_obj *o;
    size_t count = 0;
    
    TAILQ_FOREACH(o, &utc->objects, link) {
        count++;
    }
    
    return count;
}

#define MAX_OBJECTS_PER_TA              64              // 每TA最大对象数

static TEE_Result check_object_limit(struct user_ta_ctx *utc)
{
    if (count_ta_objects(utc) >= MAX_OBJECTS_PER_TA) {
        EMSG("TA object limit reached: %d", MAX_OBJECTS_PER_TA);
        return TEE_ERROR_OUT_OF_MEMORY;
    }
    
    return TEE_SUCCESS;
}
```

### 2. 存储枚举器资源管理

#### 枚举器限制
```c
// 存储枚举器结构
struct tee_storage_enum {
    TAILQ_ENTRY(tee_storage_enum) link;     // 链表节点
    struct tee_fs_dir *dir;                 // 目录句柄
    const struct tee_file_operations *fops; // 文件操作接口
};

#define MAX_ENUMERATORS_PER_TA          8               // 每TA最大枚举器数

static size_t count_storage_enumerators(struct user_ta_ctx *utc)
{
    struct tee_storage_enum *e;
    size_t count = 0;
    
    TAILQ_FOREACH(e, &utc->storage_enums, link) {
        count++;
    }
    
    return count;
}

static TEE_Result check_enumerator_limit(struct user_ta_ctx *utc)
{
    if (count_storage_enumerators(utc) >= MAX_ENUMERATORS_PER_TA) {
        EMSG("TA enumerator limit reached: %d", MAX_ENUMERATORS_PER_TA);
        return TEE_ERROR_OUT_OF_MEMORY;
    }
    
    return TEE_SUCCESS;
}
```

### 3. 内存池和临时缓冲区管理

#### 块缓冲区池
```c
// REE FS临时块分配
static void *get_tmp_block(void)
{
    void *block = mempool_alloc(mempool_default, BLOCK_SIZE);
    if (!block) {
        EMSG("Failed to allocate temporary block of size %d", BLOCK_SIZE);
    }
    return block;
}

static void put_tmp_block(void *tmp_block)
{
    if (tmp_block) {
        mempool_free(mempool_default, tmp_block);
    }
}

// 内存池统计
struct mempool_stats {
    size_t allocated_blocks;
    size_t free_blocks;
    size_t max_allocated;
    size_t allocation_failures;
};

static struct mempool_stats block_pool_stats = { 0 };

static void update_pool_stats(bool allocation, bool success)
{
    if (allocation) {
        if (success) {
            block_pool_stats.allocated_blocks++;
            if (block_pool_stats.allocated_blocks > block_pool_stats.max_allocated) {
                block_pool_stats.max_allocated = block_pool_stats.allocated_blocks;
            }
        } else {
            block_pool_stats.allocation_failures++;
        }
    } else {
        if (block_pool_stats.allocated_blocks > 0) {
            block_pool_stats.allocated_blocks--;
            block_pool_stats.free_blocks++;
        }
    }
}
```

## 存储空间配额管理

### 1. 文件系统级别配额

#### REE FS存储空间检查
```c
// REE端存储空间检查
static TEE_Result check_storage_space(const char *path, size_t required_size)
{
    struct statvfs vfs_stat;
    
    if (statvfs(path, &vfs_stat) != 0) {
        return TEE_ERROR_STORAGE_NOT_AVAILABLE;
    }
    
    // 计算可用空间
    uint64_t available_bytes = vfs_stat.f_bavail * vfs_stat.f_bsize;
    
    if (available_bytes < required_size) {
        EMSG("Insufficient storage space: need %zu, available %"PRIu64, 
             required_size, available_bytes);
        return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    return TEE_SUCCESS;
}
```

#### RPMB空间管理
```c
// RPMB空间分配跟踪
struct rpmb_space_manager {
    uint32_t total_blocks;              // 总块数
    uint32_t used_blocks;               // 已用块数
    uint32_t fat_blocks;                // FAT占用块数
    uint32_t free_blocks;               // 空闲块数
};

static struct rpmb_space_manager rpmb_space = { 0 };

// RPMB空间初始化
static TEE_Result init_rpmb_space_tracking(void)
{
    rpmb_space.total_blocks = rpmb_ctx->max_blk_idx + 1;
    rpmb_space.fat_blocks = calculate_fat_blocks_needed();
    rpmb_space.used_blocks = count_used_blocks_in_fat();
    rpmb_space.free_blocks = rpmb_space.total_blocks - 
                            rpmb_space.fat_blocks - 
                            rpmb_space.used_blocks;
    
    return TEE_SUCCESS;
}

// RPMB空间分配检查
static TEE_Result check_rpmb_space(uint32_t blocks_needed)
{
    if (blocks_needed > rpmb_space.free_blocks) {
        EMSG("RPMB space exhausted: need %u, available %u", 
             blocks_needed, rpmb_space.free_blocks);
        return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    return TEE_SUCCESS;
}
```

### 2. 每TA存储配额

#### TA级别存储限制
```c
// TA存储配额结构
struct ta_storage_quota {
    TEE_UUID ta_uuid;                   // TA标识
    uint64_t max_storage_bytes;         // 最大存储字节数
    uint64_t used_storage_bytes;        // 已用存储字节数
    uint32_t max_objects;               // 最大对象数
    uint32_t used_objects;              // 已用对象数
    bool quota_enabled;                 // 配额启用标志
};

// 默认配额设置
#define DEFAULT_TA_STORAGE_QUOTA        (16 * 1024 * 1024)  // 16MB
#define DEFAULT_TA_MAX_OBJECTS          128                  // 128个对象

// 配额检查
static TEE_Result check_ta_storage_quota(const TEE_UUID *uuid, 
                                        size_t additional_bytes)
{
    struct ta_storage_quota *quota = find_ta_quota(uuid);
    
    if (!quota || !quota->quota_enabled) {
        return TEE_SUCCESS;  // 无配额限制
    }
    
    if (quota->used_storage_bytes + additional_bytes > quota->max_storage_bytes) {
        EMSG("TA storage quota exceeded for %pUl: %"PRIu64" + %zu > %"PRIu64,
             (void *)uuid, quota->used_storage_bytes, additional_bytes, 
             quota->max_storage_bytes);
        return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    return TEE_SUCCESS;
}

// 配额更新
static void update_ta_storage_usage(const TEE_UUID *uuid, 
                                   int64_t size_delta,
                                   int32_t object_delta)
{
    struct ta_storage_quota *quota = find_ta_quota(uuid);
    
    if (!quota) {
        return;  // 无配额跟踪
    }
    
    // 更新存储使用量
    if (size_delta > 0) {
        quota->used_storage_bytes += size_delta;
    } else if (size_delta < 0) {
        if (quota->used_storage_bytes >= (uint64_t)(-size_delta)) {
            quota->used_storage_bytes -= (uint64_t)(-size_delta);
        } else {
            quota->used_storage_bytes = 0;
        }
    }
    
    // 更新对象计数
    if (object_delta > 0) {
        quota->used_objects += object_delta;
    } else if (object_delta < 0) {
        if (quota->used_objects >= (uint32_t)(-object_delta)) {
            quota->used_objects -= (uint32_t)(-object_delta);
        } else {
            quota->used_objects = 0;
        }
    }
}
```

## 资源清理和垃圾回收

### 1. 对象生命周期管理

#### 自动资源清理
```c
// TA会话结束时的资源清理
void tee_obj_close_all(struct user_ta_ctx *utc)
{
    struct tee_obj *o;
    struct tee_obj *next_o;
    
    TAILQ_FOREACH_SAFE(o, &utc->objects, link, next_o) {
        tee_obj_close(utc, o);
    }
}

// 枚举器清理
void tee_svc_storage_close_all_enum(struct user_ta_ctx *utc)
{
    struct tee_storage_enum *e;
    struct tee_storage_enum *next_e;
    
    TAILQ_FOREACH_SAFE(e, &utc->storage_enums, link, next_e) {
        tee_svc_close_enum(utc, e);
    }
}
```

### 2. 死锁检测和清理

#### 资源死锁预防
```c
// 资源获取超时机制
#define RESOURCE_ACQUIRE_TIMEOUT_MS     5000    // 5秒超时

static TEE_Result acquire_resource_with_timeout(struct resource *res)
{
    uint64_t start_time = get_time_stamp();
    uint64_t timeout = start_time + RESOURCE_ACQUIRE_TIMEOUT_MS;
    
    while (!try_acquire_resource(res)) {
        if (get_time_stamp() > timeout) {
            EMSG("Resource acquisition timeout");
            return TEE_ERROR_TIMEOUT;
        }
        
        // 短暂等待
        thread_yield();
    }
    
    return TEE_SUCCESS;
}
```

### 3. 存储碎片整理

#### RPMB空间整理
```c
// RPMB空间碎片整理
static TEE_Result rpmb_defragment(void)
{
    TEE_Result res;
    struct rpmb_fat_entry *entries = NULL;
    uint32_t entry_count = 0;
    uint32_t compacted_count = 0;
    
    // 1. 读取所有活跃的FAT条目
    res = read_all_active_fat_entries(&entries, &entry_count);
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 压缩FAT表，移除无效条目
    res = compact_fat_entries(entries, entry_count, &compacted_count);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 3. 重新写入压缩后的FAT表
    res = write_compacted_fat_table(entries, compacted_count);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 4. 更新空间统计
    update_rpmb_space_stats();
    
    IMSG("RPMB defragmentation completed: %u -> %u entries", 
         entry_count, compacted_count);
    
cleanup:
    free(entries);
    return res;
}
```

## 配置管理和调优

### 1. 编译时配置选项

#### 存储相关配置
```makefile
# REE文件系统配置
CFG_REE_FS ?= y                         # 启用REE FS
CFG_REE_FS_HTREE_HASH_SIZE_COMPAT ?= y  # 哈希树兼容性

# RPMB文件系统配置
CFG_RPMB_FS ?= n                        # RPMB FS (默认禁用)
CFG_RPMB_FS_CACHE_ENTRIES ?= 0          # RPMB缓存条目数
CFG_RPMB_FS_RD_ENTRIES ?= 1             # RPMB批量读条目数
CFG_RPMB_TESTKEY ?= n                   # 使用测试密钥

# 内存配置
CFG_TEE_CORE_DEBUG ?= n                 # 调试模式
CFG_WITH_PAGER ?= y                     # 分页器支持
```

### 2. 运行时调优参数

#### 性能调优配置
```c
// 存储性能参数
struct storage_performance_config {
    uint32_t block_cache_size;          // 块缓存大小
    uint32_t write_batch_size;          // 写入批处理大小
    uint32_t read_ahead_blocks;         // 预读块数
    bool enable_compression;            // 启用压缩
    uint32_t compression_threshold;     // 压缩阈值
};

// 默认性能配置
static struct storage_performance_config perf_config = {
    .block_cache_size = 16,             // 16个块缓存
    .write_batch_size = 4,              // 4块批处理
    .read_ahead_blocks = 2,             // 2块预读
    .enable_compression = false,        // 禁用压缩
    .compression_threshold = 1024,      // 1KB压缩阈值
};
```

这个存储配额和资源管理系统确保了OP-TEE存储系统在各种负载条件下的稳定性和可预测性，通过多层次的限制和监控机制，防止资源耗尽和系统过载。