# OP-TEE 存储管理接口和 PTA 实现分析

## 概述

本文档分析 OP-TEE 中的存储管理接口和 Pseudo Trusted Application (PTA) 实现，重点关注系统级存储操作、管理功能、配额管理、诊断接口和安全控制。

## 1. 存储管理 PTA 架构

### 1.1 核心存储管理 PTA

#### 安全存储 TA 管理 PTA (secstor_ta_mgmt.c)
```c
// UUID: 6e256cba-fc4d-4941-ad09-2ca18603-42dd
#define PTA_SECSTOR_TA_MGMT_UUID { 0x6e256cba, 0xfc4d, 0x4941, { \
                                   0xad, 0x09, 0x2c, 0xa1, 0x86, 0x03, 0x42, \
                                   0xdd } }

// 主要功能：
#define PTA_SECSTOR_TA_MGMT_BOOTSTRAP    0  // 引导安装 TA

// TA 安装冲突检查
static TEE_Result check_install_conflict(const struct shdr_bootstrap_ta *bs_ta)
{
    TEE_Result res;
    struct tee_tadb_ta_read *ta;
    TEE_UUID uuid;
    const struct tee_tadb_property *prop;

    tee_uuid_from_octets(&uuid, bs_ta->uuid);
    res = tee_tadb_ta_open(&uuid, &ta);
    if (res == TEE_ERROR_ITEM_NOT_FOUND)
        return TEE_SUCCESS;
    if (res)
        return res;

    prop = tee_tadb_ta_get_property(ta);
    if (prop->version > bs_ta->ta_version)
        res = TEE_ERROR_ACCESS_CONFLICT;

    tee_tadb_ta_close(ta);
    return res;
}

// TA 安装实现
static TEE_Result install_ta(struct shdr *shdr, const uint8_t *nw, size_t nw_size)
{
    // 验证签名头
    // 检查版本冲突
    // 创建 TA 数据库条目
    // 写入 TA 二进制
    // 验证哈希
    // 提交到存储
}
```

#### 系统管理 PTA (system.c)
```c
// UUID: 3a2f8978-5dc0-11e8-9c2d-fa7ae01bbebc
#define PTA_SYSTEM_UUID { 0x3a2f8978, 0x5dc0, 0x11e8, { \
                         0x9c, 0x2d, 0xfa, 0x7a, 0xe0, 0x1b, 0xbe, 0xbc } }

// 主要功能：
#define PTA_SYSTEM_ADD_RNG_ENTROPY       0   // 添加随机熵
#define PTA_SYSTEM_DERIVE_TA_UNIQUE_KEY  1   // 派生 TA 唯一密钥
#define PTA_SYSTEM_MAP_ZI               2   // 映射零初始化内存
#define PTA_SYSTEM_UNMAP                3   // 解除内存映射
#define PTA_SYSTEM_DLOPEN               10  // 动态库加载
#define PTA_SYSTEM_DLSYM                11  // 符号解析
#define PTA_SYSTEM_GET_TPM_EVENT_LOG    12  // 获取 TPM 事件日志
#define PTA_SYSTEM_SUPP_PLUGIN_INVOKE   13  // 调用插件

// TA 唯一密钥派生
static TEE_Result system_derive_ta_unique_key(struct user_mode_ctx *uctx,
                                             uint32_t param_types,
                                             TEE_Param params[TEE_NUM_PARAMS])
{
    // 安全检查
    access_flags = TEE_MEMORY_ACCESS_WRITE | TEE_MEMORY_ACCESS_ANY_OWNER |
                   TEE_MEMORY_ACCESS_SECURE;
    res = vm_check_access_rights(uctx, access_flags,
                                (uaddr_t)params[1].memref.buffer,
                                params[1].memref.size);
    
    // 构建派生数据
    memcpy(data, &uctx->ts_ctx->uuid, sizeof(TEE_UUID));
    res = copy_from_user(data + sizeof(TEE_UUID), params[0].memref.buffer,
                        params[0].memref.size);
    
    // 派生密钥
    res = huk_subkey_derive(HUK_SUBKEY_UNIQUE_TA, data, data_len,
                           subkey_bbuf, params[1].memref.size);
}
```

### 1.2 统计和监控 PTA (stats.c)

```c
// UUID: d96a5b40-e2c7-b1af-8794-1002a5d5c61b
#define STATS_UUID { 0xd96a5b40, 0xe2c7, 0xb1af, \
                    { 0x87, 0x94, 0x10, 0x02, 0xa5, 0xd5, 0xc6, 0x1b } }

// 监控命令
#define STATS_CMD_PAGER_STATS      0  // 分页器统计
#define STATS_CMD_ALLOC_STATS      1  // 分配器统计
#define STATS_CMD_MEMLEAK_STATS    2  // 内存泄漏统计
#define STATS_CMD_TA_STATS         3  // TA 统计
#define STATS_CMD_GET_TIME         4  // 获取时间
#define STATS_CMD_PRINT_DRIVER_INFO 5 // 驱动信息

// 分配器类型
#define ALLOC_ID_ALL            0  // 所有分配器
#define ALLOC_ID_HEAP           1  // 核心堆分配器
#define ALLOC_ID_PUBLIC_DDR     2  // 公共 DDR 分配器（已弃用）
#define ALLOC_ID_TA_RAM         3  // TA_RAM 分配器
#define ALLOC_ID_NEXUS_HEAP     4  // Nexus 堆分配器
#define ALLOC_ID_RPMB           5  // RPMB 安全存储

// 分配统计结构
struct pta_stats_alloc {
    char desc[TEE_ALLOCATOR_DESC_LENGTH];
    uint64_t free2_sum;               // 空闲块大小平方和
    uint32_t allocated;               // 当前已分配字节
    uint32_t max_allocated;           // 最大分配字节
    uint32_t size;                    // 分配器总大小
    uint32_t num_alloc_fail;          // 分配失败次数
    uint32_t biggest_alloc_fail;      // 最大失败分配大小
    uint32_t biggest_alloc_fail_used; // 失败时已用字节
};

// TA 统计结构
struct pta_stats_ta {
    TEE_UUID uuid;
    uint32_t panicked;      // TA 是否崩溃
    uint32_t sess_num;      // 打开的会话数
    struct pta_stats_alloc heap; // 堆统计
};
```

## 2. 存储系统架构

### 2.1 文件系统操作接口

```c
// 文件操作接口
struct tee_file_operations {
    TEE_Result (*open)(struct tee_pobj *po, size_t *size,
                      struct tee_file_handle **fh);
    TEE_Result (*create)(struct tee_pobj *po, bool overwrite,
                        const void *head, size_t head_size,
                        const void *attr, size_t attr_size,
                        const void *data_core, const void *data_user,
                        size_t data_size, struct tee_file_handle **fh);
    void (*close)(struct tee_file_handle **fh);
    TEE_Result (*read)(struct tee_file_handle *fh, size_t pos,
                      void *buf_core, void *buf_user, size_t *len);
    TEE_Result (*write)(struct tee_file_handle *fh, size_t pos,
                       const void *buf_core, const void *buf_user,
                       size_t len);
    TEE_Result (*rename)(struct tee_pobj *old_po, struct tee_pobj *new_po,
                        bool overwrite);
    TEE_Result (*remove)(struct tee_pobj *po);
    TEE_Result (*truncate)(struct tee_file_handle *fh, size_t size);
    
    // 目录操作
    TEE_Result (*opendir)(const TEE_UUID *uuid, struct tee_fs_dir **d);
    TEE_Result (*readdir)(struct tee_fs_dir *d, struct tee_fs_dirent **ent);
    void (*closedir)(struct tee_fs_dir *d);
};

// 存储类型选择
static inline const struct tee_file_operations *
tee_svc_storage_file_ops(uint32_t storage_id)
{
    switch (storage_id) {
    case TEE_STORAGE_PRIVATE:
#if defined(CFG_REE_FS)
        return &ree_fs_ops;
#elif defined(CFG_RPMB_FS)
        return &rpmb_fs_ops;
#else
        return NULL;
#endif
#ifdef CFG_REE_FS
    case TEE_STORAGE_PRIVATE_REE:
        return &ree_fs_ops;
#endif
#ifdef CFG_RPMB_FS
    case TEE_STORAGE_PRIVATE_RPMB:
        return &rpmb_fs_ops;
#endif
    default:
        return NULL;
    }
}
```

### 2.2 持久化对象管理

```c
// 持久化对象结构
struct tee_pobj {
    TAILQ_ENTRY(tee_pobj) link;
    uint32_t refcnt;              // 引用计数
    TEE_UUID uuid;                // TA UUID
    void *obj_id;                 // 对象 ID
    uint32_t obj_id_len;          // 对象 ID 长度
    uint32_t flags;               // 标志
    uint32_t obj_info_usage;      // 对象信息使用
    bool temporary;               // 临时对象
    bool creating;                // 正在创建
    const struct tee_file_operations *fops; // 文件系统操作
};

// 对象使用类型
enum tee_pobj_usage {
    TEE_POBJ_USAGE_OPEN,    // 打开
    TEE_POBJ_USAGE_RENAME,  // 重命名
    TEE_POBJ_USAGE_CREATE,  // 创建
    TEE_POBJ_USAGE_ENUM,    // 枚举
};
```

## 3. TA 数据库管理

### 3.1 TADB 结构

```c
// TADB 配置
#define TADB_MAX_BUFFER_SIZE     (64U * 1024)
#define TADB_AUTH_ENC_ALG        TEE_ALG_AES_GCM
#define TADB_IV_SIZE             TEE_AES_BLOCK_SIZE
#define TADB_TAG_SIZE            TEE_AES_BLOCK_SIZE
#define TADB_KEY_SIZE            TEE_AES_MAX_KEY_SIZE

// TADB 目录结构
struct tee_tadb_dir {
    const struct tee_file_operations *ops;
    struct tee_file_handle *fh;
    int nbits;
    bitstr_t *files;
};

// TADB 条目
struct tadb_entry {
    struct tee_tadb_property prop;
    uint32_t file_number;                    // 文件编号
    uint8_t iv[TADB_IV_SIZE];               // 初始化向量
    uint8_t tag[TADB_TAG_SIZE];             // 认证标签
    uint8_t key[TADB_KEY_SIZE];             // 加密密钥
};

// TA 写入结构
struct tee_tadb_ta_write {
    struct tee_tadb_dir *db;
    int fd;
    struct tadb_entry entry;
    size_t pos;
    void *ctx;
};

// TA 读取结构
struct tee_tadb_ta_read {
    struct tee_tadb_dir *db;
    int fd;
    struct tadb_entry entry;
    size_t pos;
    void *ctx;
    struct mobj *ta_mobj;
    uint8_t *ta_buf;
};
```

### 3.2 文件命名和操作

```c
// 文件编号到字符串转换
static void file_num_to_str(char *buf, size_t blen, uint32_t file_number)
{
    int rc __maybe_unused = 0;
    
    rc = snprintf(buf, blen, "%" PRIu32 ".ta", file_number);
    assert(rc >= 0);
}

// TA 操作
static TEE_Result ta_operation_open(unsigned int cmd, uint32_t file_number,
                                   int *fd)
{
    struct thread_param params[3] = { };
    struct mobj *mobj = NULL;
    TEE_Result res = TEE_SUCCESS;
    void *va = NULL;

    va = thread_rpc_shm_cache_alloc(THREAD_SHM_CACHE_USER_FS,
                                   THREAD_SHM_TYPE_APPLICATION,
                                   TEE_FS_NAME_MAX, &mobj);
    if (!va)
        return TEE_ERROR_OUT_OF_MEMORY;

    file_num_to_str(va, TEE_FS_NAME_MAX, file_number);

    params[0] = THREAD_PARAM_VALUE(IN, cmd, 0, 0);
    params[1] = THREAD_PARAM_MEMREF(IN, mobj, 0, TEE_FS_NAME_MAX);
    params[2] = THREAD_PARAM_VALUE(OUT, 0, 0, 0);

    res = thread_rpc_cmd(OPTEE_RPC_CMD_FS, ARRAY_SIZE(params), params);
    if (!res)
        *fd = params[2].u.value.a;

    return res;
}
```

## 4. 存储加密和密钥管理

### 4.1 文件系统密钥管理器

```c
// 密钥类型和大小
#define TEE_FS_KM_FEK_SIZE    16    // 文件加密密钥大小
#define TEE_FS_KM_SSK_SIZE    16    // 安全存储密钥大小
#define TEE_FS_KM_TSK_SIZE    32    // 可信应用存储密钥大小
#define TEE_FS_KM_HMAC_ALG    TEE_ALG_HMAC_SHA256

// 安全存储密钥结构
struct tee_fs_ssk {
    bool is_init;
    uint8_t key[TEE_FS_KM_SSK_SIZE];
};

// FEK 加密/解密
TEE_Result tee_fs_fek_crypt(const TEE_UUID *uuid, TEE_OperationMode mode,
                           const uint8_t *in_key, size_t size,
                           uint8_t *out_key)
{
    TEE_Result res;
    void *ctx = NULL;
    uint8_t tsk[TEE_FS_KM_TSK_SIZE];
    uint8_t dst_key[size];

    if (!in_key || !out_key)
        return TEE_ERROR_BAD_PARAMETERS;

    if (size != TEE_FS_KM_FEK_SIZE)
        return TEE_ERROR_BAD_PARAMETERS;

    if (tee_fs_ssk.is_init == 0)
        return TEE_ERROR_GENERIC;

    if (uuid) {
        // 基于 UUID 派生 TSK
        res = do_hmac(tsk, sizeof(tsk), tee_fs_ssk.key,
                     TEE_FS_KM_SSK_SIZE, uuid, sizeof(*uuid));
        if (res != TEE_SUCCESS)
            return res;
    }
    // 继续处理加密/解密...
}
```

### 4.2 HMAC 实现

```c
static TEE_Result do_hmac(void *out_key, size_t out_key_size,
                         const void *in_key, size_t in_key_size,
                         const void *message, size_t message_size)
{
    TEE_Result res;
    void *ctx = NULL;

    if (!out_key || !in_key || !message)
        return TEE_ERROR_BAD_PARAMETERS;

    res = crypto_mac_alloc_ctx(&ctx, TEE_FS_KM_HMAC_ALG);
    if (res != TEE_SUCCESS)
        return res;

    res = crypto_mac_init(ctx, in_key, in_key_size);
    if (res != TEE_SUCCESS)
        goto exit;

    res = crypto_mac_update(ctx, message, message_size);
    if (res != TEE_SUCCESS)
        goto exit;

    res = crypto_mac_final(ctx, out_key, out_key_size);
    if (res != TEE_SUCCESS)
        goto exit;

    res = TEE_SUCCESS;

exit:
    crypto_mac_free_ctx(ctx);
    return res;
}
```

## 5. RPMB 文件系统

### 5.1 RPMB 配置

```c
// RPMB 文件系统配置
#define RPMB_STORAGE_START_ADDRESS      0
#define RPMB_FS_FAT_START_ADDRESS       512
#define RPMB_BLOCK_SIZE_SHIFT           8
#define RPMB_FS_MAGIC                   0x52504D42
#define FS_VERSION                      2

#define FILE_IS_ACTIVE                  (1u << 0)
#define FILE_IS_LAST_ENTRY              (1u << 1)
#define TEE_RPMB_FS_FILENAME_LENGTH     224
#define TMP_BLOCK_SIZE                  4096U
#define RPMB_MAX_RETRIES                10

// 缓存配置
#define RPMB_BUF_MAX_ENTRIES (CFG_RPMB_FS_CACHE_ENTRIES + \
                             CFG_RPMB_FS_RD_ENTRIES)
```

### 5.2 RPMB 文件系统结构

```c
// FS 参数
struct rpmb_fs_parameters {
    uint32_t fat_start_address;
    uint32_t max_rpmb_address;
};

// FAT 条目
struct rpmb_fat_entry {
    uint32_t start_address;                          // 起始地址
    uint32_t data_size;                             // 数据大小
    uint32_t flags;                                 // 标志
    uint32_t unused;                                // 未使用
    uint8_t fek[TEE_FS_KM_FEK_SIZE];               // 文件加密密钥
    char filename[TEE_RPMB_FS_FILENAME_LENGTH];     // 文件名
};

// FAT 条目目录缓存
struct rpmb_fat_entry_dir {
    struct rpmb_fat_entry *rpmb_fat_entry_buf;      // 缓冲区
    uint32_t idx_curr;                              // 当前索引
    uint32_t num_buffered;                          // 缓冲条目数
    uint32_t num_total_read;                        // 总读取条目数
};
```

## 6. REE 文件系统

### 6.1 REE FS 结构

```c
#define BLOCK_SHIFT    12
#define BLOCK_SIZE     (1 << BLOCK_SHIFT)

// 文件描述符
struct tee_fs_fd {
    struct tee_fs_htree *ht;              // 哈希树
    int fd;                               // 文件描述符
    struct tee_fs_dirfile_fileh dfh;      // 目录文件句柄
    const TEE_UUID *uuid;                 // UUID
};

// 目录结构
struct tee_fs_dir {
    struct tee_fs_dirfile_dirh *dirh;     // 目录句柄
    int idx;                              // 索引
    struct tee_fs_dirent d;               // 目录条目
    const TEE_UUID *uuid;                 // UUID
};

// 块编号计算
static int pos_to_block_num(int position)
{
    return position >> BLOCK_SHIFT;
}
```

### 6.2 哈希树存储

```c
// 哈希树测试接口
static const struct tee_fs_htree_storage test_htree_ops = {
    .block_size = TEST_BLOCK_SIZE,
    .rpc_read_init = test_read_init,
    .rpc_read_final = test_read_final,
    .rpc_write_init = test_write_init,
    .rpc_write_final = test_write_final,
};

// 读写操作
static TEE_Result out_of_place_write(struct tee_fs_fd *fdp, size_t pos,
                                     const void *buf_core,
                                     const void *buf_user, size_t len)
{
    TEE_Result res;
    size_t start_block_num = pos_to_block_num(pos);
    size_t end_block_num = pos_to_block_num(pos + len - 1);
    size_t remain_bytes = len;
    uint8_t *block;
    struct tee_fs_htree_meta *meta = tee_fs_htree_get_meta(fdp->ht);

    // 获取临时块
    block = get_tmp_block();
    if (!block)
        return TEE_ERROR_OUT_OF_MEMORY;

    // 分块写入
    while (start_block_num <= end_block_num) {
        size_t offset = pos % BLOCK_SIZE;
        size_t size_to_write = MIN(remain_bytes, (size_t)BLOCK_SIZE);

        // 读取现有数据
        if (start_block_num * BLOCK_SIZE < ROUNDUP(meta->length, BLOCK_SIZE)) {
            res = tee_fs_htree_read_block(&fdp->ht, start_block_num, block);
            if (res != TEE_SUCCESS)
                goto exit;
        }
        // 写入新数据...
    }
}
```

## 7. 存储服务接口

### 7.1 存储服务头结构

```c
// GP 格式安全存储文件头
struct tee_svc_storage_head {
    uint32_t attr_size;        // 属性大小
    uint32_t objectSize;       // 对象大小
    uint32_t maxObjectSize;    // 最大对象大小
    uint32_t objectUsage;      // 对象使用
    uint32_t objectType;       // 对象类型
    uint32_t have_attrs;       // 是否有属性
};

// 存储枚举
struct tee_storage_enum {
    TAILQ_ENTRY(tee_storage_enum) link;
    struct tee_fs_dir *dir;
    const struct tee_file_operations *fops;
};
```

### 7.2 存储枚举管理

```c
// 获取存储枚举
static TEE_Result tee_svc_storage_get_enum(struct user_ta_ctx *utc,
                                          vaddr_t enum_id,
                                          struct tee_storage_enum **e_out)
{
    struct tee_storage_enum *e;

    TAILQ_FOREACH(e, &utc->storage_enums, link) {
        if (enum_id == (vaddr_t)e) {
            *e_out = e;
            return TEE_SUCCESS;
        }
    }
    return TEE_ERROR_BAD_PARAMETERS;
}

// 关闭枚举
static TEE_Result tee_svc_close_enum(struct user_ta_ctx *utc,
                                    struct tee_storage_enum *e)
{
    if (e == NULL || utc == NULL)
        return TEE_ERROR_BAD_PARAMETERS;

    TAILQ_REMOVE(&utc->storage_enums, e, link);

    if (e->fops)
        e->fops->closedir(e->dir);

    e->dir = NULL;
    e->fops = NULL;
    free(e);

    return TEE_SUCCESS;
}
```

## 8. 诊断和调试接口

### 8.1 文件系统哈希树测试

```c
// 测试辅助结构
struct test_aux {
    uint8_t *data;         // 数据缓冲区
    size_t data_len;       // 数据长度
    size_t data_alloced;   // 已分配大小
    uint8_t *block;        // 块缓冲区
};

// 哈希树完整性测试
TEE_Result core_fs_htree_tests(uint32_t nParamTypes,
                              TEE_Param pParams[TEE_NUM_PARAMS] __unused)
{
    TEE_Result res = TEE_SUCCESS;

    if (nParamTypes)
        return TEE_ERROR_BAD_PARAMETERS;

    // 写读测试
    res = test_write_read(10);
    if (res)
        return res;

    // 损坏检测测试
    return test_corrupt(5);
}

// 数据损坏测试
static TEE_Result test_corrupt_type(const TEE_UUID *uuid, uint8_t *hash,
                                   size_t num_blocks, struct test_aux *aux,
                                   enum tee_fs_htree_type type, size_t idx)
{
    // 故意损坏数据
    // 尝试重新打开
    // 验证是否检测到损坏
    // 确保没有静默数据损坏
}
```

### 8.2 性能监控

```c
// 分页器统计
static TEE_Result get_pager_stats(uint32_t type, TEE_Param p[TEE_NUM_PARAMS])
{
    struct tee_pager_stats stats = { };

    tee_pager_get_stats(&stats);
    p[0].value.a = stats.npages;        // 页数
    p[0].value.b = stats.npages_all;    // 总页数
    p[1].value.a = stats.ro_hits;       // 只读命中
    p[1].value.b = stats.rw_hits;       // 读写命中
    p[2].value.a = stats.hidden_hits;   // 隐藏命中
    p[2].value.b = stats.zi_released;   // 零初始化页释放

    return TEE_SUCCESS;
}

// 分配器统计获取
static TEE_Result get_alloc_stats(uint32_t type, TEE_Param p[TEE_NUM_PARAMS])
{
    struct pta_stats_alloc *stats = NULL;
    uint32_t pool_id = p[0].value.a;

    for (i = ALLOC_ID_HEAP; i <= STATS_NB_POOLS; i++) {
        switch (i) {
        case ALLOC_ID_HEAP:
            malloc_get_stats(stats);
            strlcpy(stats->desc, "Heap", sizeof(stats->desc));
            break;
        case ALLOC_ID_TA_RAM:
            phys_mem_stats(stats, p[0].value.b);
            strlcpy(stats->desc, "Physical TA memory", sizeof(stats->desc));
            break;
        case ALLOC_ID_RPMB:
            if (rpmb_mem_stats(stats, p[0].value.b))
                strlcpy(stats->desc, "RPMB secure storage (no data)",
                       sizeof(stats->desc));
            else
                strlcpy(stats->desc, "RPMB secure storage",
                       sizeof(stats->desc));
            break;
        }
        stats++;
    }
}
```

## 9. 安全控制和权限管理

### 9.1 会话验证

```c
// 系统 PTA 会话开启验证
static TEE_Result open_session(uint32_t param_types __unused,
                              TEE_Param params[TEE_NUM_PARAMS] __unused,
                              void **sess_ctx __unused)
{
    struct ts_session *s = NULL;

    // 检查调用者是否为用户 TA
    s = ts_get_calling_session();
    if (!s)
        return TEE_ERROR_ACCESS_DENIED;
    if (!is_user_ta_ctx(s->ctx))
        return TEE_ERROR_ACCESS_DENIED;

    return TEE_SUCCESS;
}
```

### 9.2 内存访问控制

```c
// 派生密钥时的内存访问验证
static TEE_Result system_derive_ta_unique_key(struct user_mode_ctx *uctx,
                                             uint32_t param_types,
                                             TEE_Param params[TEE_NUM_PARAMS])
{
    uint32_t access_flags = 0;

    // 确保派生密钥不会意外进入非安全内存
    access_flags = TEE_MEMORY_ACCESS_WRITE | TEE_MEMORY_ACCESS_ANY_OWNER |
                   TEE_MEMORY_ACCESS_SECURE;
    res = vm_check_access_rights(uctx, access_flags,
                                (uaddr_t)params[1].memref.buffer,
                                params[1].memref.size);
    if (res != TEE_SUCCESS)
        return TEE_ERROR_SECURITY;
}
```

### 9.3 对象完整性保护

```c
// 损坏对象移除
static void remove_corrupt_obj(struct user_ta_ctx *utc, struct tee_obj *o)
{
    o->pobj->fops->remove(o->pobj);
    if (!(utc->ta_ctx.flags & TA_FLAG_DONT_CLOSE_HANDLE_ON_CORRUPT_OBJECT))
        tee_obj_close(utc, o);
}
```

## 10. 总结

OP-TEE 的存储管理接口提供了：

1. **多层次管理**：从 PTA 到文件系统操作的完整层次
2. **安全保障**：多重加密、完整性检查、访问控制
3. **性能监控**：全面的统计和诊断接口
4. **灵活配置**：支持 REE FS 和 RPMB FS 等多种后端
5. **调试支持**：丰富的测试和验证机制

这些接口为安全存储提供了强大的管理和监控能力，确保了 TEE 环境中存储操作的安全性和可靠性。