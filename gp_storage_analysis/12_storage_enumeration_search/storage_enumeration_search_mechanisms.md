---
title: '存储枚举和搜索机制深度分析'
---

# OP-TEE存储枚举和搜索机制深度分析

## 存储枚举架构概述

OP-TEE实现了一个完整的存储对象枚举系统，支持对持久化对象的发现、遍历和搜索。该系统基于GlobalPlatform TEE Internal Core API规范，提供标准化的存储目录遍历接口。

## GP API层枚举接口

### 1. 对象枚举器数据结构

#### 枚举器句柄定义
```c
// GP API中的枚举器句柄类型
typedef uintptr_t TEE_ObjectEnumHandle;

// 内核中的存储枚举器结构
struct tee_storage_enum {
    TAILQ_ENTRY(tee_storage_enum) link;     // 链表节点
    struct tee_fs_dir *dir;                 // 目录句柄
    const struct tee_file_operations *fops; // 文件操作接口
};

// TA上下文中的枚举器管理
struct user_ta_ctx {
    TAILQ_HEAD(tee_storage_enum_head, tee_storage_enum) storage_enums;
    // ... 其他字段
};
```

### 2. 枚举器生命周期管理

#### 枚举器分配
```c
TEE_Result TEE_AllocatePersistentObjectEnumerator(TEE_ObjectEnumHandle *objectEnumerator)
{
    TEE_Result res;
    uint32_t oe;
    
    __utee_check_out_annotation(objectEnumerator, sizeof(*objectEnumerator));
    
    // 调用系统调用分配枚举器
    res = _utee_storage_alloc_enum(&oe);
    
    if (res != TEE_SUCCESS)
        oe = TEE_HANDLE_NULL;
    
    *objectEnumerator = (TEE_ObjectEnumHandle)(uintptr_t)oe;
    
    if (res != TEE_SUCCESS && res != TEE_ERROR_ACCESS_CONFLICT)
        TEE_Panic(res);
    
    return res;
}

// 内核侧枚举器分配实现
TEE_Result syscall_storage_alloc_enum(uint32_t *obj_enum)
{
    struct ts_session *sess = ts_get_current_session();
    struct user_ta_ctx *utc = to_user_ta_ctx(sess->ctx);
    struct tee_storage_enum *e = NULL;
    
    if (obj_enum == NULL)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 分配枚举器结构
    e = malloc(sizeof(struct tee_storage_enum));
    if (e == NULL)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 初始化枚举器
    e->dir = NULL;
    e->fops = NULL;
    TAILQ_INSERT_TAIL(&utc->storage_enums, e, link);
    
    // 返回枚举器句柄
    return copy_kaddr_to_uref(obj_enum, e);
}
```

#### 枚举器释放
```c
void TEE_FreePersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator)
{
    TEE_Result res;
    
    if (objectEnumerator == TEE_HANDLE_NULL)
        return;
    
    res = _utee_storage_free_enum((unsigned long)objectEnumerator);
    
    if (res != TEE_SUCCESS)
        TEE_Panic(res);
}

// 内核侧释放实现
TEE_Result syscall_storage_free_enum(unsigned long obj_enum)
{
    struct ts_session *sess = ts_get_current_session();
    struct user_ta_ctx *utc = to_user_ta_ctx(sess->ctx);
    struct tee_storage_enum *e = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    res = tee_svc_storage_get_enum(utc, uref_to_vaddr(obj_enum), &e);
    if (res != TEE_SUCCESS)
        return res;
    
    return tee_svc_close_enum(utc, e);
}

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

#### 枚举器重置
```c
void TEE_ResetPersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator)
{
    TEE_Result res;
    
    if (objectEnumerator == TEE_HANDLE_NULL)
        return;
    
    res = _utee_storage_reset_enum((unsigned long)objectEnumerator);
    
    if (res != TEE_SUCCESS)
        TEE_Panic(res);
}

// 内核侧重置实现
TEE_Result syscall_storage_reset_enum(unsigned long obj_enum)
{
    struct ts_session *sess = ts_get_current_session();
    struct tee_storage_enum *e = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    res = tee_svc_storage_get_enum(to_user_ta_ctx(sess->ctx),
                                  uref_to_vaddr(obj_enum), &e);
    if (res != TEE_SUCCESS)
        return res;
    
    // 关闭当前目录句柄
    if (e->fops) {
        e->fops->closedir(e->dir);
        e->fops = NULL;
        e->dir = NULL;
    }
    assert(!e->dir);
    
    return TEE_SUCCESS;
}
```

### 3. 枚举操作

#### 开始枚举
```c
TEE_Result TEE_StartPersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator,
                                              uint32_t storageID)
{
    TEE_Result res;
    
    res = _utee_storage_start_enum((unsigned long)objectEnumerator, storageID);
    
    if (res != TEE_SUCCESS &&
        res != TEE_ERROR_ITEM_NOT_FOUND &&
        res != TEE_ERROR_CORRUPT_OBJECT &&
        res != TEE_ERROR_STORAGE_NOT_AVAILABLE)
        TEE_Panic(res);
    
    return res;
}

// 内核侧开始枚举实现
TEE_Result syscall_storage_start_enum(unsigned long obj_enum,
                                     unsigned long storage_id)
{
    struct ts_session *sess = ts_get_current_session();
    struct tee_storage_enum *e = NULL;
    TEE_Result res = TEE_SUCCESS;
    const struct tee_file_operations *fops = tee_svc_storage_file_ops(storage_id);
    
    res = tee_svc_storage_get_enum(to_user_ta_ctx(sess->ctx),
                                  uref_to_vaddr(obj_enum), &e);
    if (res != TEE_SUCCESS)
        return res;
    
    // 关闭之前的目录句柄
    if (e->dir) {
        e->fops->closedir(e->dir);
        e->dir = NULL;
    }
    
    if (!fops)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    e->fops = fops;
    
    // 打开TA特定的目录
    return fops->opendir(&sess->ctx->uuid, &e->dir);
}
```

#### 获取下一个对象
```c
TEE_Result TEE_GetNextPersistentObject(TEE_ObjectEnumHandle objectEnumerator,
                                      TEE_ObjectInfo *objectInfo,
                                      void *objectID, size_t *objectIDLen)
{
    struct utee_object_info info = { };
    TEE_Result res = TEE_SUCCESS;
    uint64_t len = 0;
    
    if (objectInfo)
        __utee_check_out_annotation(objectInfo, sizeof(*objectInfo));
    __utee_check_out_annotation(objectIDLen, sizeof(*objectIDLen));
    
    if (!objectID) {
        res = TEE_ERROR_BAD_PARAMETERS;
        goto out;
    }
    
    len = *objectIDLen;
    res = _utee_storage_next_enum((unsigned long)objectEnumerator,
                                 &info, objectID, &len);
    if (objectInfo) {
        objectInfo->objectType = info.obj_type;
        objectInfo->objectSize = info.obj_size;
        objectInfo->maxObjectSize = info.max_obj_size;
        objectInfo->objectUsage = info.obj_usage;
        objectInfo->dataSize = info.data_size;
        objectInfo->dataPosition = info.data_pos;
        objectInfo->handleFlags = info.handle_flags;
    }
    *objectIDLen = len;
    
out:
    if (res != TEE_SUCCESS &&
        res != TEE_ERROR_ITEM_NOT_FOUND &&
        res != TEE_ERROR_CORRUPT_OBJECT &&
        res != TEE_ERROR_STORAGE_NOT_AVAILABLE)
        TEE_Panic(res);
    
    return res;
}
```

## 文件系统后端枚举实现

### 1. 抽象接口定义

#### 目录操作接口
```c
// 位置: /core/include/tee/tee_fs.h

struct tee_fs_dirent {
    uint8_t oid[TEE_OBJECT_ID_MAX_LEN];     // 对象ID
    size_t oidlen;                          // 对象ID长度
};

struct tee_fs_dir;  // 不透明的目录句柄

struct tee_file_operations {
    // ... 文件操作函数指针
    
    // 目录操作接口
    TEE_Result (*opendir)(const TEE_UUID *uuid, struct tee_fs_dir **d);
    TEE_Result (*readdir)(struct tee_fs_dir *d, struct tee_fs_dirent **ent);
    void (*closedir)(struct tee_fs_dir *d);
};
```

### 2. REE FS枚举实现

#### REE FS目录结构
```c
// 位置: /core/tee/tee_ree_fs.c

struct tee_fs_dir {
    struct tee_fs_dirfile_dirh *dirh;       // 目录文件句柄
    int idx;                                // 当前索引
    struct tee_fs_dirent d;                 // 当前目录项
    const TEE_UUID *uuid;                   // TA UUID
};
```

#### REE FS目录操作实现
```c
static TEE_Result ree_fs_opendir_rpc(const TEE_UUID *uuid,
                                     struct tee_fs_dir **dir)
{
    TEE_Result res = TEE_SUCCESS;
    struct tee_fs_dirfile_dirh *dirh = NULL;
    struct tee_fs_dir *d = calloc(1, sizeof(*d));
    
    if (!d)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    d->uuid = uuid;
    
    mutex_lock(&ree_fs_mutex);
    
    // 获取目录文件句柄
    res = get_dirh(&dirh);
    if (res)
        goto out;
    
    /* 检查是否至少有一个文件 */
    d->idx = -1;
    d->d.oidlen = sizeof(d->d.oid);
    res = tee_fs_dirfile_get_next(dirh, d->uuid, &d->idx, d->d.oid,
                                 &d->d.oidlen);
    d->idx = -1;  // 重置索引
    
out:
    if (!res) {
        *dir = d;
    } else {
        if (d)
            put_dirh(dirh, false);
        free(d);
    }
    mutex_unlock(&ree_fs_mutex);
    
    return res;
}

static TEE_Result ree_fs_readdir_rpc(struct tee_fs_dir *d,
                                     struct tee_fs_dirent **ent)
{
    struct tee_fs_dirfile_dirh *dirh = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    mutex_lock(&ree_fs_mutex);
    
    res = get_dirh(&dirh);
    if (res)
        goto out;
    
    d->d.oidlen = sizeof(d->d.oid);
    res = tee_fs_dirfile_get_next(dirh, d->uuid, &d->idx, d->d.oid,
                                 &d->d.oidlen);
    if (res == TEE_SUCCESS)
        *ent = &d->d;
    
    put_dirh(dirh, res);
out:
    mutex_unlock(&ree_fs_mutex);
    
    return res;
}

static void ree_fs_closedir_rpc(struct tee_fs_dir *d)
{
    if (d) {
        mutex_lock(&ree_fs_mutex);
        
        put_dirh(ree_fs_dirh, false);
        free(d);
        
        mutex_unlock(&ree_fs_mutex);
    }
}
```

### 3. RPMB FS枚举实现

#### RPMB FS目录结构
```c
// 位置: /core/tee/tee_rpmb_fs.c

struct tee_rpmb_fs_dirent {
    struct tee_fs_dirent entry;                         // 目录项
    SIMPLEQ_ENTRY(tee_rpmb_fs_dirent) link;            // 链表节点
};

struct tee_fs_dir {
    struct tee_rpmb_fs_dirent *current;                 // 当前项
    SIMPLEQ_HEAD(next_head, tee_rpmb_fs_dirent) next;  // 目录项队列
};
```

#### RPMB FS目录操作实现
```c
static TEE_Result rpmb_fs_opendir(const TEE_UUID *uuid, struct tee_fs_dir **dir)
{
    uint32_t len;
    char path_local[TEE_RPMB_FS_FILENAME_LENGTH];
    TEE_Result res = TEE_ERROR_GENERIC;
    struct tee_fs_dir *rpmb_dir = NULL;
    
    if (!uuid || !dir) {
        res = TEE_ERROR_BAD_PARAMETERS;
        goto out;
    }
    
    memset(path_local, 0, sizeof(path_local));
    if (create_dirname(path_local, sizeof(path_local) - 1, uuid)) {
        res = TEE_ERROR_BAD_PARAMETERS;
        goto out;
    }
    len = strlen(path_local);
    
    /* 添加斜杠以正确匹配完整目录名 */
    if (path_local[len - 1] != '/')
        path_local[len] = '/';
    
    rpmb_dir = calloc(1, sizeof(*rpmb_dir));
    if (!rpmb_dir) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto out;
    }
    SIMPLEQ_INIT(&rpmb_dir->next);
    
    // 填充目录内容
    res = rpmb_fs_dir_populate(path_local, rpmb_dir);
    if (res != TEE_SUCCESS) {
        free(rpmb_dir);
        rpmb_dir = NULL;
        goto out;
    }
    
    *dir = rpmb_dir;
    
out:
    return res;
}

static TEE_Result rpmb_fs_readdir(struct tee_fs_dir *dir,
                                 struct tee_fs_dirent **ent)
{
    if (!dir)
        return TEE_ERROR_GENERIC;
    
    free(dir->current);
    
    dir->current = SIMPLEQ_FIRST(&dir->next);
    if (!dir->current)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    SIMPLEQ_REMOVE_HEAD(&dir->next, link);
    
    *ent = &dir->current->entry;
    return TEE_SUCCESS;
}

static void rpmb_fs_closedir(struct tee_fs_dir *dir)
{
    if (dir) {
        rpmb_fs_dir_free(dir);
        free(dir);
    }
}

static void rpmb_fs_dir_free(struct tee_fs_dir *dir)
{
    struct tee_rpmb_fs_dirent *e;
    
    if (!dir)
        return;
    
    free(dir->current);
    
    while (!SIMPLEQ_EMPTY(&dir->next)) {
        e = SIMPLEQ_FIRST(&dir->next);
        SIMPLEQ_REMOVE_HEAD(&dir->next, link);
        free(e);
    }
}
```

#### RPMB目录填充机制
```c
static TEE_Result rpmb_fs_dir_populate(const char *path,
                                      struct tee_fs_dir *dir)
{
    struct rpmb_fat_entry *fat_entries = NULL;
    uint32_t num_entries = 0;
    TEE_Result res;
    uint32_t i;
    
    // 读取FAT表条目
    res = read_fat(&fat_entries, &num_entries);
    if (res != TEE_SUCCESS)
        return res;
    
    // 遍历FAT表，查找匹配的文件
    for (i = 0; i < num_entries; i++) {
        if (fat_entries[i].flags & FILE_IS_ACTIVE) {
            // 检查文件名是否匹配目录路径
            if (strncmp(fat_entries[i].filename, path, strlen(path)) == 0) {
                struct tee_rpmb_fs_dirent *entry;
                const char *filename = &fat_entries[i].filename[strlen(path)];
                
                // 跳过子目录
                if (strchr(filename, '/'))
                    continue;
                
                entry = malloc(sizeof(*entry));
                if (!entry) {
                    res = TEE_ERROR_OUT_OF_MEMORY;
                    break;
                }
                
                // 解析对象ID
                res = extract_object_id_from_filename(filename,
                                                     entry->entry.oid,
                                                     &entry->entry.oidlen);
                if (res != TEE_SUCCESS) {
                    free(entry);
                    continue;
                }
                
                // 添加到目录列表
                SIMPLEQ_INSERT_TAIL(&dir->next, entry, link);
            }
        }
    }
    
    free(fat_entries);
    return res;
}
```

## 内核侧枚举处理

### 1. 系统调用处理

#### 下一个对象获取的详细实现
```c
TEE_Result syscall_storage_next_enum(unsigned long obj_enum,
                                    struct utee_object_info *info,
                                    void *obj_id, uint64_t *len)
{
    struct ts_session *sess = ts_get_current_session();
    struct user_ta_ctx *utc = to_user_ta_ctx(sess->ctx);
    struct tee_storage_enum *e = NULL;
    struct tee_fs_dirent *d = NULL;
    TEE_Result res = TEE_SUCCESS;
    struct tee_obj *o = NULL;
    uint64_t l = 0;
    struct utee_object_info bbuf = { };
    
    res = tee_svc_storage_get_enum(utc, uref_to_vaddr(obj_enum), &e);
    if (res != TEE_SUCCESS)
        goto exit;
    
    info = memtag_strip_tag(info);
    obj_id = memtag_strip_tag(obj_id);
    
    /* 检查提供缓冲区的权限 */
    res = vm_check_access_rights(&utc->uctx, TEE_MEMORY_ACCESS_WRITE,
                                (uaddr_t)info, sizeof(*info));
    if (res != TEE_SUCCESS)
        goto exit;
    
    res = vm_check_access_rights(&utc->uctx, TEE_MEMORY_ACCESS_WRITE,
                                (uaddr_t)obj_id, TEE_OBJECT_ID_MAX_LEN);
    if (res != TEE_SUCCESS)
        goto exit;
    
    if (!e->fops) {
        res = TEE_ERROR_ITEM_NOT_FOUND;
        goto exit;
    }
    
    // 读取下一个目录项
    res = e->fops->readdir(e->dir, &d);
    if (res != TEE_SUCCESS)
        goto exit;
    
    // 创建临时对象以读取元数据
    o = tee_obj_alloc();
    if (o == NULL) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto exit;
    }
    
    res = tee_pobj_get(&sess->ctx->uuid, d->oid, d->oidlen, 0,
                      TEE_POBJ_USAGE_ENUM, e->fops, &o->pobj);
    if (res)
        goto exit;
    
    o->info.handleFlags = o->pobj->flags | TEE_HANDLE_FLAG_PERSISTENT |
                         TEE_HANDLE_FLAG_INITIALIZED;
    
    // 读取对象头部信息
    tee_pobj_lock_usage(o->pobj);
    res = tee_svc_storage_read_head(o);
    bbuf = (struct utee_object_info){
        .obj_type = o->info.objectType,
        .obj_size = o->info.objectSize,
        .max_obj_size = o->info.maxObjectSize,
        .obj_usage = o->pobj->obj_info_usage,
        .data_size = o->info.dataSize,
        .data_pos = o->info.dataPosition,
        .handle_flags = o->info.handleFlags,
    };
    tee_pobj_unlock_usage(o->pobj);
    if (res != TEE_SUCCESS)
        goto exit;
    
    // 复制信息到用户空间
    res = copy_to_user(info, &bbuf, sizeof(bbuf));
    if (res)
        goto exit;
    
    res = copy_to_user(obj_id, o->pobj->obj_id, o->pobj->obj_id_len);
    if (res)
        goto exit;
    
    l = o->pobj->obj_id_len;
    res = copy_to_user_private(len, &l, sizeof(*len));
    
exit:
    if (o) {
        if (o->pobj) {
            o->pobj->fops->close(&o->fh);
            tee_pobj_release(o->pobj);
        }
        tee_obj_free(o);
    }
    
    return res;
}
```

### 2. 枚举器查找和验证

#### 枚举器查找机制
```c
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
```

## 对象搜索和过滤机制

### 1. 基于对象类型的过滤

#### 对象类型匹配
```c
// 枚举时的对象类型过滤示例
TEE_Result filter_objects_by_type(TEE_ObjectEnumHandle enumerator,
                                 uint32_t object_type,
                                 TEE_ObjectInfo *found_objects,
                                 uint32_t *count)
{
    TEE_Result res;
    TEE_ObjectInfo info;
    void *obj_id = malloc(TEE_OBJECT_ID_MAX_LEN);
    size_t obj_id_len;
    uint32_t found_count = 0;
    uint32_t max_count = *count;
    
    if (!obj_id)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    while (found_count < max_count) {
        obj_id_len = TEE_OBJECT_ID_MAX_LEN;
        res = TEE_GetNextPersistentObject(enumerator, &info, 
                                         obj_id, &obj_id_len);
        if (res == TEE_ERROR_ITEM_NOT_FOUND)
            break;
        if (res != TEE_SUCCESS)
            goto cleanup;
        
        // 检查对象类型匹配
        if (info.objectType == object_type) {
            found_objects[found_count] = info;
            found_count++;
        }
    }
    
    *count = found_count;
    res = TEE_SUCCESS;
    
cleanup:
    free(obj_id);
    return res;
}
```

### 2. 基于对象ID模式的搜索

#### 对象ID前缀匹配
```c
TEE_Result search_objects_by_id_prefix(TEE_ObjectEnumHandle enumerator,
                                      const void *prefix, size_t prefix_len,
                                      void **found_ids, size_t *found_lens,
                                      uint32_t *count)
{
    TEE_Result res;
    TEE_ObjectInfo info;
    void *obj_id = malloc(TEE_OBJECT_ID_MAX_LEN);
    size_t obj_id_len;
    uint32_t found_count = 0;
    uint32_t max_count = *count;
    
    if (!obj_id)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    while (found_count < max_count) {
        obj_id_len = TEE_OBJECT_ID_MAX_LEN;
        res = TEE_GetNextPersistentObject(enumerator, &info,
                                         obj_id, &obj_id_len);
        if (res == TEE_ERROR_ITEM_NOT_FOUND)
            break;
        if (res != TEE_SUCCESS)
            goto cleanup;
        
        // 检查ID前缀匹配
        if (obj_id_len >= prefix_len &&
            memcmp(obj_id, prefix, prefix_len) == 0) {
            
            found_ids[found_count] = malloc(obj_id_len);
            if (!found_ids[found_count]) {
                res = TEE_ERROR_OUT_OF_MEMORY;
                goto cleanup;
            }
            
            memcpy(found_ids[found_count], obj_id, obj_id_len);
            found_lens[found_count] = obj_id_len;
            found_count++;
        }
    }
    
    *count = found_count;
    res = TEE_SUCCESS;
    
cleanup:
    free(obj_id);
    return res;
}
```

### 3. 基于对象大小的过滤

#### 大小范围过滤
```c
TEE_Result filter_objects_by_size_range(TEE_ObjectEnumHandle enumerator,
                                       uint32_t min_size, uint32_t max_size,
                                       TEE_ObjectInfo *filtered_objects,
                                       uint32_t *count)
{
    TEE_Result res;
    TEE_ObjectInfo info;
    void *obj_id = malloc(TEE_OBJECT_ID_MAX_LEN);
    size_t obj_id_len;
    uint32_t filtered_count = 0;
    uint32_t max_count = *count;
    
    if (!obj_id)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    while (filtered_count < max_count) {
        obj_id_len = TEE_OBJECT_ID_MAX_LEN;
        res = TEE_GetNextPersistentObject(enumerator, &info,
                                         obj_id, &obj_id_len);
        if (res == TEE_ERROR_ITEM_NOT_FOUND)
            break;
        if (res != TEE_SUCCESS)
            goto cleanup;
        
        // 检查数据大小范围
        if (info.dataSize >= min_size && info.dataSize <= max_size) {
            filtered_objects[filtered_count] = info;
            filtered_count++;
        }
    }
    
    *count = filtered_count;
    res = TEE_SUCCESS;
    
cleanup:
    free(obj_id);
    return res;
}
```

## 枚举性能优化

### 1. 批量枚举

#### 批量获取对象信息
```c
TEE_Result batch_enumerate_objects(TEE_ObjectEnumHandle enumerator,
                                  TEE_ObjectInfo *batch_info,
                                  void **batch_ids, size_t *batch_id_lens,
                                  uint32_t batch_size, uint32_t *actual_count)
{
    TEE_Result res;
    uint32_t count = 0;
    
    for (count = 0; count < batch_size; count++) {
        batch_id_lens[count] = TEE_OBJECT_ID_MAX_LEN;
        
        res = TEE_GetNextPersistentObject(enumerator, 
                                         &batch_info[count],
                                         batch_ids[count], 
                                         &batch_id_lens[count]);
        if (res == TEE_ERROR_ITEM_NOT_FOUND) {
            break;  // 已到末尾
        }
        if (res != TEE_SUCCESS) {
            *actual_count = count;
            return res;
        }
    }
    
    *actual_count = count;
    return TEE_SUCCESS;
}
```

### 2. 缓存机制

#### 目录内容缓存
```c
struct dir_cache_entry {
    TEE_UUID ta_uuid;
    uint32_t storage_id;
    uint64_t cache_time;
    uint32_t entry_count;
    struct tee_fs_dirent *entries;
    bool valid;
};

static struct dir_cache_entry dir_cache[MAX_CACHED_DIRS];

static TEE_Result get_cached_dir_entries(const TEE_UUID *uuid,
                                        uint32_t storage_id,
                                        struct tee_fs_dirent **entries,
                                        uint32_t *count)
{
    struct dir_cache_entry *cache = find_cache_entry(uuid, storage_id);
    uint64_t current_time = get_time_stamp();
    
    if (cache && cache->valid && 
        (current_time - cache->cache_time) < CACHE_VALIDITY_MS) {
        *entries = cache->entries;
        *count = cache->entry_count;
        return TEE_SUCCESS;
    }
    
    return TEE_ERROR_ITEM_NOT_FOUND;
}

static void update_dir_cache(const TEE_UUID *uuid, uint32_t storage_id,
                            struct tee_fs_dirent *entries, uint32_t count)
{
    struct dir_cache_entry *cache = get_free_cache_entry();
    
    if (!cache)
        return;  // 缓存满
    
    cache->ta_uuid = *uuid;
    cache->storage_id = storage_id;
    cache->cache_time = get_time_stamp();
    cache->entry_count = count;
    cache->entries = malloc(count * sizeof(struct tee_fs_dirent));
    
    if (cache->entries) {
        memcpy(cache->entries, entries, count * sizeof(struct tee_fs_dirent));
        cache->valid = true;
    }
}
```

## 错误处理和边界情况

### 1. 枚举器状态验证

#### 状态一致性检查
```c
static TEE_Result validate_enumerator_state(struct tee_storage_enum *e)
{
    // 检查枚举器基本状态
    if (!e)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 检查文件操作接口一致性
    if (e->fops && !e->dir)
        return TEE_ERROR_BAD_STATE;
    
    if (!e->fops && e->dir)
        return TEE_ERROR_BAD_STATE;
    
    return TEE_SUCCESS;
}
```

### 2. 并发访问处理

#### 枚举器并发保护
```c
static struct mutex enum_global_mutex = MUTEX_INITIALIZER;

static TEE_Result safe_enum_operation(struct tee_storage_enum *e,
                                     enum_operation_t op, void *data)
{
    TEE_Result res;
    
    mutex_lock(&enum_global_mutex);
    
    res = validate_enumerator_state(e);
    if (res != TEE_SUCCESS)
        goto out;
    
    switch (op) {
        case ENUM_OP_START:
            res = perform_start_operation(e, data);
            break;
        case ENUM_OP_NEXT:
            res = perform_next_operation(e, data);
            break;
        case ENUM_OP_RESET:
            res = perform_reset_operation(e, data);
            break;
        default:
            res = TEE_ERROR_BAD_PARAMETERS;
    }
    
out:
    mutex_unlock(&enum_global_mutex);
    return res;
}
```

### 3. 资源清理

#### 会话结束时的枚举器清理
```c
void tee_svc_storage_close_all_enum(struct user_ta_ctx *utc)
{
    struct tee_storage_enum_head *eh = &utc->storage_enums;
    
    while (!TAILQ_EMPTY(eh))
        tee_svc_close_enum(utc, TAILQ_FIRST(eh));
}

// 在TA会话结束时调用
static void cleanup_ta_session(struct user_ta_ctx *utc)
{
    // 清理所有打开的存储枚举器
    tee_svc_storage_close_all_enum(utc);
    
    // 清理其他资源
    tee_obj_close_all(utc);
}
```

这个存储枚举和搜索机制为OP-TEE提供了完整的对象发现和遍历功能，支持高效的存储空间管理和对象查找操作，同时确保了系统的安全性和稳定性。