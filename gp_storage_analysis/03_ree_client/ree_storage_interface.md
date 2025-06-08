---
title: 'REE 侧存储接口深度分析'
---

# OP-TEE REE侧存储接口深度分析

本文档深入分析OP-TEE在REE侧的存储接口实现，包括TEE Supplicant文件系统服务、RPC通信协议、安全机制和性能优化等核心技术。

## REE侧存储架构深度解析

### 架构层次和数据流

```
┌────────────────────────────────────────────────────────────┐
│                          TEE World                           │
│    tee_ree_fs.c --> tee_fs_rpc.c --> RPC Call                │
└─────────────────────────────┬─────────────────────────────┘
                              │ Secure Monitor / ARM TrustZone
┌─────────────────────────────┴─────────────────────────────┐
│                         REE World                            │
│  ┌────────────────────────────────────────────────────────┐  │
│  │             TEE Supplicant Daemon                     │  │  ← 用户空间守护进程
│  │  tee_supp_fs.c - 文件系统服务实现                │  │
│  │  handle_db.c - 句柄管理和资源追踪              │  │
│  │  rpmb.c - RPMB硬件接口实现                      │  │
│  └────────────────────────────────────────────────────────┘  │
│              │ /dev/tee0 ioctl、shared memory              │
│  ┌────────────▼────────────────────────────────────────────┐  │
│  │               OP-TEE Driver                           │  │  ← 内核驱动
│  │  optee_rpc.c - RPC消息处理和路由                   │  │
│  │  optee_smc.c - SMC调用和上下文切换                │  │
│  └────────────────────────────────────────────────────────┘  │
│              │ SMC (Secure Monitor Call)                 │
│  ┌────────────▼────────────────────────────────────────────┐  │
│  │            Linux Filesystem Layer                     │  │  ← 文件系统层
│  │  VFS -> ext4/f2fs -> Block Layer -> Storage Device    │  │
│  └────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 1. TEE Supplicant高级文件系统服务架构

#### 服务组件和职责分工
- **核心服务**: `optee_client/tee-supplicant/src/tee_supp_fs.c`
- **服务模式**: 守护进程，为TEE提供可信的REE存储服务
- **通信机制**: 基于RPC的异步调用模型，支持零拷贝和大数据块传输

#### 高级服务初始化和配置
```c
// TEE文件系统服务的全局配置结构
struct tee_fs_service_config {
    char root_path[PATH_MAX];            // 存储根目录
    mode_t dir_mode;                     // 目录权限模式
    mode_t file_mode;                    // 文件权限模式
    bool sync_on_write;                  // 写入时立即同步
    size_t max_file_size;                // 最大文件大小限制
    size_t io_buffer_size;               // I/O缓冲区大小
    uint32_t security_flags;             // 安全特性标志
    struct {
        size_t max_open_files;           // 最大同时打开文件数
        size_t max_open_dirs;            // 最大同时打开目录数
        uint32_t handle_timeout_ms;      // 句柄超时时间
    } limits;
};

static struct tee_fs_service_config g_fs_config;
static bool g_fs_service_initialized = false;
static pthread_mutex_t g_fs_init_mutex = PTHREAD_MUTEX_INITIALIZER;

// 高级初始化函数
static int tee_supp_fs_advanced_init(const char *parent_path)
{
    int ret = 0;
    struct stat st;
    
    pthread_mutex_lock(&g_fs_init_mutex);
    
    if (g_fs_service_initialized) {
        pthread_mutex_unlock(&g_fs_init_mutex);
        return 0;
    }
    
    // 1. 初始化配置结构
    memset(&g_fs_config, 0, sizeof(g_fs_config));
    snprintf(g_fs_config.root_path, sizeof(g_fs_config.root_path), 
             "%s/", parent_path ? parent_path : "/data/tee");
    
    g_fs_config.dir_mode = 0700;        // 仅所有者可访问
    g_fs_config.file_mode = 0600;       // 仅所有者可读写
    g_fs_config.sync_on_write = true;   // 安全模式：立即同步
    g_fs_config.max_file_size = 256 * 1024 * 1024; // 256MB限制
    g_fs_config.io_buffer_size = 64 * 1024;        // 64KB I/O缓冲
    g_fs_config.limits.max_open_files = 256;
    g_fs_config.limits.max_open_dirs = 64;
    g_fs_config.limits.handle_timeout_ms = 30000;  // 30秒超时
    
    // 2. 创建根目录结构
    ret = mkpath(g_fs_config.root_path, g_fs_config.dir_mode);
    if (ret != 0) {
        EMSG("Failed to create root directory: %s", g_fs_config.root_path);
        goto exit;
    }
    
    // 3. 验证目录权限和所有权
    if (stat(g_fs_config.root_path, &st) != 0) {
        EMSG("Cannot stat root directory: %s", strerror(errno));
        ret = -1;
        goto exit;
    }
    
    if (!S_ISDIR(st.st_mode)) {
        EMSG("Root path is not a directory: %s", g_fs_config.root_path);
        ret = -1;
        goto exit;
    }
    
    // 4. 检查读写权限
    if (access(g_fs_config.root_path, R_OK | W_OK | X_OK) != 0) {
        EMSG("Insufficient permissions for root directory: %s", 
             strerror(errno));
        ret = -1;
        goto exit;
    }
    
    // 5. 初始化句柄管理器
    ret = init_handle_databases();
    if (ret != 0) {
        EMSG("Failed to initialize handle databases");
        goto exit;
    }
    
    // 6. 初始化统计和监控
    init_fs_statistics();
    
    g_fs_service_initialized = true;
    IMSG("TEE filesystem service initialized: %s", g_fs_config.root_path);
    
exit:
    pthread_mutex_unlock(&g_fs_init_mutex);
    return ret;
}

#### 复杂的资源管理和句柄系统

```c
// 全局句柄管理器结构
struct tee_fs_handle_manager {
    struct handle_db file_handles;       // 文件句柄数据库
    struct handle_db dir_handles;        // 目录句柄数据库
    pthread_mutex_t global_mutex;       // 全局互斥锁
    
    // 资源统计和限制
    struct {
        atomic_uint open_files;          // 当前打开文件数
        atomic_uint open_dirs;           // 当前打开目录数
        uint64_t total_bytes_read;       // 累计读取字节数
        uint64_t total_bytes_written;    // 累计写入字节数
        uint64_t total_operations;       // 累计操作次数
    } stats;
    
    // 句柄生命周期管理
    struct {
        time_t *creation_times;          // 句柄创建时间数组
        time_t *last_access_times;       // 最后访问时间数组
        size_t max_handles;              // 最大句柄数
        pthread_t cleanup_thread;       // 清理线程
        bool cleanup_running;            // 清理线程运行标志
    } lifecycle;
};

static struct tee_fs_handle_manager g_handle_mgr;

// 高级句柄分配函数
static int allocate_file_handle(int fd, const char *path)
{
    int handle_id;
    time_t now = time(NULL);
    
    pthread_mutex_lock(&g_handle_mgr.global_mutex);
    
    // 1. 检查资源限制
    if (atomic_load(&g_handle_mgr.stats.open_files) >= 
        g_fs_config.limits.max_open_files) {
        EMSG("Too many open files: %u >= %zu",
             atomic_load(&g_handle_mgr.stats.open_files),
             g_fs_config.limits.max_open_files);
        pthread_mutex_unlock(&g_handle_mgr.global_mutex);
        return -EMFILE;
    }
    
    // 2. 分配句柄ID
    handle_id = handle_get(&g_handle_mgr.file_handles, (void *)(uintptr_t)fd);
    if (handle_id < 0) {
        EMSG("Failed to allocate handle for fd %d", fd);
        pthread_mutex_unlock(&g_handle_mgr.global_mutex);
        return handle_id;
    }
    
    // 3. 记录生命周期信息
    if (handle_id < g_handle_mgr.lifecycle.max_handles) {
        g_handle_mgr.lifecycle.creation_times[handle_id] = now;
        g_handle_mgr.lifecycle.last_access_times[handle_id] = now;
    }
    
    // 4. 更新统计信息
    atomic_fetch_add(&g_handle_mgr.stats.open_files, 1);
    atomic_fetch_add(&g_handle_mgr.stats.total_operations, 1);
    
    DMSG("Allocated file handle %d for fd %d, path: %s", 
         handle_id, fd, path);
    
    pthread_mutex_unlock(&g_handle_mgr.global_mutex);
    return handle_id;
}

// 句柄超时清理线程
static void *handle_cleanup_thread(void *arg)
{
    time_t now, threshold;
    int handle_id;
    
    while (g_handle_mgr.lifecycle.cleanup_running) {
        sleep(30); // 每30秒检查一次
        
        now = time(NULL);
        threshold = now - (g_fs_config.limits.handle_timeout_ms / 1000);
        
        pthread_mutex_lock(&g_handle_mgr.global_mutex);
        
        // 清理超时的文件句柄
        for (handle_id = 0; handle_id < g_handle_mgr.lifecycle.max_handles; handle_id++) {
            if (g_handle_mgr.lifecycle.last_access_times[handle_id] > 0 &&
                g_handle_mgr.lifecycle.last_access_times[handle_id] < threshold) {
                
                void *ptr = handle_lookup(&g_handle_mgr.file_handles, handle_id);
                if (ptr) {
                    int fd = (int)(uintptr_t)ptr;
                    DMSG("Cleaning up stale file handle %d (fd %d)", handle_id, fd);
                    
                    close(fd);
                    handle_put(&g_handle_mgr.file_handles, handle_id);
                    g_handle_mgr.lifecycle.last_access_times[handle_id] = 0;
                    atomic_fetch_sub(&g_handle_mgr.stats.open_files, 1);
                }
            }
        }
        
        pthread_mutex_unlock(&g_handle_mgr.global_mutex);
    }
    
    return NULL;
}
```

### 2. 高级RPC通信协议设计

#### 完整的文件系统RPC命令集
位置: `/tee-supplicant/src/optee_msg_supplicant.h`

```c
#define OPTEE_MSG_RPC_CMD_FS    2   // 文件系统RPC命令

// 核心文件操作命令
#define OPTEE_MRF_OPEN          0   // 打开文件
#define OPTEE_MRF_CREATE        1   // 创建文件  
#define OPTEE_MRF_CLOSE         2   // 关闭文件
#define OPTEE_MRF_READ          3   // 读文件
#define OPTEE_MRF_WRITE         4   // 写文件
#define OPTEE_MRF_TRUNCATE      5   // 截断文件
#define OPTEE_MRF_REMOVE        6   // 删除文件
#define OPTEE_MRF_RENAME        7   // 重命名文件

// 目录操作命令
#define OPTEE_MRF_OPENDIR       8   // 打开目录
#define OPTEE_MRF_CLOSEDIR      9   // 关闭目录
#define OPTEE_MRF_READDIR      10   // 读目录

// 扩展操作命令
#define OPTEE_MRF_SYNC         11   // 同步数据到存储
#define OPTEE_MRF_GET_INFO     12   // 获取文件信息
#define OPTEE_MRF_SET_INFO     13   // 设置文件信息
```

#### 高性能RPC消息处理架构

```c
// RPC参数结构定义
struct optee_msg_param {
    uint64_t attr;              // 参数属性和类型
    union {
        struct {
            uint64_t buf_ptr;   // 缓冲区指针
            uint64_t size;      // 缓冲区大小
            uint64_t shm_ref;   // 共享内存引用
        } memref;
        
        struct {
            uint64_t a;         // 64位值A
            uint64_t b;         // 64位值B
            uint64_t c;         // 64位值C
        } value;
    } u;
};

// 高级RPC消息分发器
static const struct fs_rpc_operation {
    const char *name;
    TEE_Result (*handler)(size_t num_params, struct tee_ioctl_param *params);
    size_t min_params;
    uint32_t required_caps;     // 所需的安全能力
} fs_ops[] = {
    {"open",     ree_fs_new_open,     3, FS_CAP_READ},
    {"create",   ree_fs_new_create,   3, FS_CAP_WRITE | FS_CAP_CREATE},
    {"close",    ree_fs_new_close,    1, FS_CAP_BASIC},
    {"read",     ree_fs_new_read,     2, FS_CAP_READ},
    {"write",    ree_fs_new_write,    2, FS_CAP_WRITE},
    {"truncate", ree_fs_new_truncate, 2, FS_CAP_WRITE | FS_CAP_TRUNCATE},
    {"remove",   ree_fs_new_remove,   2, FS_CAP_DELETE},
    {"rename",   ree_fs_new_rename,   3, FS_CAP_RENAME},
    {"opendir",  ree_fs_new_opendir,  3, FS_CAP_READ},
    {"closedir", ree_fs_new_closedir, 1, FS_CAP_BASIC},
    {"readdir",  ree_fs_new_readdir,  3, FS_CAP_READ},
    {"sync",     ree_fs_new_sync,     1, FS_CAP_SYNC},
};

// 智能RPC处理器实现
TEEC_Result tee_supp_fs_process(size_t num_params, struct tee_ioctl_param *params)
{
    TEEC_Result res;
    uint32_t cmdid;
    struct timespec start_time, end_time;
    bool timing_enabled = false;
    
    // 1. 基本参数验证
    if (num_params < 1 || 
        (params[0].attr & TEE_IOCTL_PARAM_ATTR_TYPE_MASK) != TEE_IOCTL_PARAM_ATTR_TYPE_VALUE_INPUT) {
        EMSG("Invalid RPC parameters");
        return TEEC_ERROR_BAD_PARAMETERS;
    }
    
    cmdid = params[0].a;
    
    // 2. 操作范围检查
    if (cmdid >= ARRAY_SIZE(fs_ops)) {
        EMSG("Invalid filesystem command ID: %u", cmdid);
        return TEEC_ERROR_NOT_SUPPORTED;
    }
    
    const struct fs_rpc_operation *op = &fs_ops[cmdid];
    
    // 3. 参数数量验证
    if (num_params < op->min_params) {
        EMSG("Insufficient parameters for %s: got %zu, need %zu", 
             op->name, num_params, op->min_params);
        return TEEC_ERROR_BAD_PARAMETERS;
    }
    
    // 4. 安全能力检查
    if (!check_fs_capabilities(op->required_caps)) {
        EMSG("Insufficient capabilities for %s operation", op->name);
        return TEEC_ERROR_ACCESS_DENIED;
    }
    
    // 5. 性能监控开始
    if (should_monitor_performance(cmdid)) {
        clock_gettime(CLOCK_MONOTONIC, &start_time);
        timing_enabled = true;
    }
    
    // 6. 调用具体的处理函数
    DMSG("Executing %s operation", op->name);
    res = op->handler(num_params, params);
    
    // 7. 性能统计更新
    if (timing_enabled) {
        clock_gettime(CLOCK_MONOTONIC, &end_time);
        update_operation_stats(cmdid, &start_time, &end_time, res);
    }
    
    // 8. 错误处理和日志记录
    if (res != TEEC_SUCCESS) {
        EMSG("Operation %s failed with error 0x%x", op->name, res);
        log_fs_error(cmdid, res, params);
    } else {
        DMSG("Operation %s completed successfully", op->name);
    }
    
    return res;
}
```

#### 智能错误恢复和重试机制

```c
// 错误分类和恢复策略
enum fs_error_class {
    FS_ERROR_TRANSIENT,     // 临时错误，可重试
    FS_ERROR_RESOURCE,      // 资源错误，需要等待
    FS_ERROR_PERMANENT,     // 永久错误，不可恢复
    FS_ERROR_SECURITY,      // 安全错误，记录并拒绝
};

struct fs_error_info {
    enum fs_error_class class;
    uint32_t max_retries;
    uint32_t delay_ms;
    const char *description;
};

static const struct fs_error_info error_recovery_table[] = {
    {FS_ERROR_TRANSIENT, 3,  100,  "Temporary I/O error"},     // EAGAIN
    {FS_ERROR_RESOURCE,  5,  500,  "Resource temporarily unavailable"}, // ENOMEM
    {FS_ERROR_TRANSIENT, 2,  200,  "Device busy"},             // EBUSY
    {FS_ERROR_PERMANENT, 0,  0,    "File not found"},          // ENOENT
    {FS_ERROR_PERMANENT, 0,  0,    "Permission denied"},       // EACCES
    {FS_ERROR_SECURITY,  0,  0,    "Security violation"},      // EPERM
};

// 智能重试执行器
static TEEC_Result execute_with_retry(fs_operation_func_t func,
                                     struct fs_operation_context *ctx)
{
    TEEC_Result res;
    uint32_t attempt = 0;
    const struct fs_error_info *error_info;
    
    do {
        res = func(ctx);
        
        if (res == TEEC_SUCCESS)
            break;
            
        // 获取错误信息
        error_info = classify_error(res);
        if (!error_info || error_info->class == FS_ERROR_PERMANENT ||
            error_info->class == FS_ERROR_SECURITY) {
            break;  // 不可恢复的错误
        }
        
        attempt++;
        if (attempt >= error_info->max_retries) {
            EMSG("Max retries (%u) exceeded for operation", error_info->max_retries);
            break;
        }
        
        DMSG("Retry attempt %u after %u ms for error: %s", 
             attempt, error_info->delay_ms, error_info->description);
        
        // 等待后重试
        usleep(error_info->delay_ms * 1000);
        
    } while (true);
    
    // 记录最终结果
    if (res != TEEC_SUCCESS && attempt > 0) {
        EMSG("Operation failed after %u retries: 0x%x", attempt, res);
    }
    
    return res;
}
```

### 3. 文件系统操作实现

#### 存储路径管理
```c
// 存储根目录初始化
static int tee_supp_fs_init(void)
{
    size_t n = 0;
    mode_t mode = 0700;  // 仅所有者可访问
    
    // 设置存储根路径: ${fs_parent_path}/
    n = snprintf(tee_fs_root, sizeof(tee_fs_root), "%s/", 
                 supplicant_params.fs_parent_path);
    
    // 创建目录结构
    if (mkpath(tee_fs_root, mode) != 0)
        return -1;
        
    return 0;
}

// 生成绝对路径
static size_t tee_fs_get_absolute_filename(char *file, char *out, size_t out_size)
{
    // 拼接: tee_fs_root + file
    int s = snprintf(out, out_size, "%s%s", tee_fs_root, file);
    if (s < 0 || (size_t)s >= out_size)
        return 0;
    return (size_t)s;
}
```

#### 文件创建操作
```c
static TEEC_Result ree_fs_new_create(size_t num_params, 
                                     struct tee_ioctl_param *params)
{
    char abs_filename[PATH_MAX] = { 0 };
    char abs_dir[PATH_MAX] = { 0 };
    const int flags = O_RDWR | O_CREAT | O_TRUNC;
    int fd = 0;
    
    // 1. 参数验证
    if (num_params != 3 || /* 验证参数类型 */)
        return TEEC_ERROR_BAD_PARAMETERS;
    
    // 2. 获取文件名并生成绝对路径
    char *fname = tee_supp_param_to_va(params + 1);
    if (!tee_fs_get_absolute_filename(fname, abs_filename, sizeof(abs_filename)))
        return TEEC_ERROR_BAD_PARAMETERS;
    
    // 3. 尝试创建文件
    fd = open_wrapper(abs_filename, flags);
    if (fd >= 0)
        goto out;
    
    // 4. 如果目录不存在,创建目录层次结构
    if (errno == ENOENT) {
        // 创建父目录
        char *d = dirname(abs_dir);
        if (!mkdir(d, 0700)) {
            fd = open_wrapper(abs_filename, flags);
            if (fd >= 0)
                goto out;
        }
        // 处理多层目录创建...
    }
    
out:
    fs_fsync();  // 确保数据同步到磁盘
    params[2].a = fd;  // 返回文件描述符
    return TEEC_SUCCESS;
}
```

#### 文件打开操作
```c
static TEEC_Result ree_fs_new_open(size_t num_params,
                                   struct tee_ioctl_param *params)
{
    char abs_filename[PATH_MAX] = { 0 };
    int fd = 0;
    
    // 1. 获取绝对路径
    char *fname = tee_supp_param_to_va(params + 1);
    if (!tee_fs_get_absolute_filename(fname, abs_filename, sizeof(abs_filename)))
        return TEEC_ERROR_BAD_PARAMETERS;
    
    // 2. 尝试以读写模式打开
    fd = open_wrapper(abs_filename, O_RDWR);
    if (fd < 0) {
        // 3. 失败则尝试只读模式(处理只读文件系统)
        fd = open_wrapper(abs_filename, O_RDONLY);
        if (fd < 0)
            return TEEC_ERROR_ITEM_NOT_FOUND;
    }
    
    params[2].a = fd;
    return TEEC_SUCCESS;
}
```

#### 文件I/O操作
```c
// 读操作
static TEEC_Result ree_fs_new_read(size_t num_params,
                                   struct tee_ioctl_param *params)
{
    int fd = params[0].b;           // 文件描述符
    off_t offset = params[0].c;     // 偏移量
    void *buf = tee_supp_param_to_va(params + 1);  // 缓冲区
    size_t len = MEMREF_SIZE(params + 1);          // 长度
    
    // 定位并读取
    if (lseek(fd, offset, SEEK_SET) < 0)
        return errno_to_teec(errno);
    
    ssize_t bytes = read(fd, buf, len);
    if (bytes < 0)
        return errno_to_teec(errno);
    
    MEMREF_SIZE(params + 1) = bytes;  // 返回实际读取字节数
    return TEEC_SUCCESS;
}

// 写操作
static TEEC_Result ree_fs_new_write(size_t num_params,
                                    struct tee_ioctl_param *params)
{
    int fd = params[0].b;
    off_t offset = params[0].c;
    void *buf = tee_supp_param_to_va(params + 1);
    size_t len = MEMREF_SIZE(params + 1);
    
    // 定位并写入
    if (lseek(fd, offset, SEEK_SET) < 0)
        return errno_to_teec(errno);
    
    ssize_t bytes = write(fd, buf, len);
    if (bytes < 0)
        return errno_to_teec(errno);
    
    if ((size_t)bytes != len)
        return TEEC_ERROR_STORAGE_NO_SPACE;
    
    return TEEC_SUCCESS;
}
```

### 4. 目录操作支持

#### 目录遍历
```c
// 打开目录
static TEEC_Result ree_fs_new_opendir(size_t num_params,
                                      struct tee_ioctl_param *params)
{
    char abs_path[PATH_MAX] = { 0 };
    DIR *dir = NULL;
    int handle = 0;
    
    // 生成绝对路径
    char *path = tee_supp_param_to_va(params + 1);
    if (!tee_fs_get_absolute_filename(path, abs_path, sizeof(abs_path)))
        return TEEC_ERROR_BAD_PARAMETERS;
    
    // 打开目录
    dir = opendir(abs_path);
    if (!dir)
        return errno_to_teec(errno);
    
    // 分配句柄
    handle = handle_get(&dir_handle_db, dir);
    if (handle < 0) {
        closedir(dir);
        return TEEC_ERROR_OUT_OF_MEMORY;
    }
    
    params[2].a = handle;
    return TEEC_SUCCESS;
}

// 读目录项
static TEEC_Result ree_fs_new_readdir(size_t num_params,
                                      struct tee_ioctl_param *params)
{
    int handle = params[0].b;
    DIR *dir = handle_lookup(&dir_handle_db, handle);
    struct dirent *dirent = NULL;
    
    if (!dir)
        return TEEC_ERROR_BAD_PARAMETERS;
    
    // 跳过隐藏文件(以'.'开头)
    while (true) {
        dirent = readdir(dir);
        if (!dirent)
            return TEEC_ERROR_ITEM_NOT_FOUND;
        if (dirent->d_name[0] != '.')
            break;
    }
    
    // 复制文件名到输出缓冲区
    void *buf = tee_supp_param_to_va(params + 1);
    size_t fname_len = strlen(dirent->d_name) + 1;
    
    MEMREF_SIZE(params + 1) = fname_len;
    if (fname_len > len)
        return TEEC_ERROR_SHORT_BUFFER;
    
    memcpy(buf, dirent->d_name, fname_len);
    return TEEC_SUCCESS;
}
```

### 5. 错误处理和安全机制

#### 错误码转换
```c
static TEEC_Result errno_to_teec(int err)
{
    switch (err) {
        case ENOSPC:
            return TEEC_ERROR_STORAGE_NO_SPACE;  // 存储空间不足
        case ENOENT:
            return TEEC_ERROR_ITEM_NOT_FOUND;    // 文件不存在
        default:
            return TEEC_ERROR_GENERIC;           // 通用错误
    }
}
```

#### 安全考虑
1. **路径验证**: 所有文件路径都在tee_fs_root下,防止目录遍历攻击
2. **权限控制**: 目录权限设置为0700(仅所有者访问)
3. **同步操作**: 使用O_SYNC标志和fsync()确保数据持久化
4. **句柄管理**: 使用句柄数据库管理目录句柄,防止泄露

### 6. RPC调用路径

#### TEE到REE的调用流程
```
1. TEE侧文件操作 (ree_fs_ops)
     ↓
2. RPC调用 (OPTEE_MSG_RPC_CMD_FS)
     ↓
3. tee-supplicant接收RPC
     ↓
4. tee_supp_fs_process()分发
     ↓
5. 具体操作函数 (ree_fs_new_*)
     ↓
6. Linux文件系统调用
     ↓
7. 结果返回给TEE
```

#### 参数传递
- **输入参数**: 通过共享内存传递文件名、数据等
- **输出参数**: 文件描述符、读取的数据、错误码等
- **缓冲区管理**: 使用tee_supp_param_to_va()访问共享内存

### 7. 存储目录结构

#### 实际文件系统布局
```
${fs_parent_path}/                    ← 存储根目录
├── TA_UUID_1/                        ← TA1的存储目录
│   ├── dirf.db                       ← 目录文件数据库
│   ├── dirh.db                       ← 目录哈希数据库
│   └── 50A7B331-D26A-...-F76C3421/    ← 具体对象目录
│       ├── O                         ← 对象数据文件
│       └── .info                     ← 对象信息文件
├── TA_UUID_2/                        ← TA2的存储目录
│   └── ...
└── ...
```

#### 文件命名规则
- **TA目录**: 以TA的UUID命名
- **对象目录**: 以对象ID的十六进制表示命名
- **数据文件**: 'O'表示对象数据
- **元数据文件**: '.info'包含对象属性和元数据

### 8. 配置和初始化

#### Supplicant启动配置
```c
// 存储路径配置 (通常为 /data/tee/)
supplicant_params.fs_parent_path = "/data/tee/";

// 初始化时调用
if (tee_supp_fs_init() != 0) {
    EMSG("Failed to initialize TEE filesystem");
    return TEEC_ERROR_STORAGE_NOT_AVAILABLE;
}
```

#### 运行时特性
- **延迟初始化**: 第一次文件操作时才初始化存储根目录
- **目录自动创建**: 根据需要自动创建TA存储目录
- **错误恢复**: 创建失败时自动清理半创建的目录结构