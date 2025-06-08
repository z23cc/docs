# OP-TEE 存储系统架构决策记录 (ADRs)

## 概述

本文档记录了 OP-TEE 存储系统的重要架构决策，包括决策背景、考虑的备选方案、选择的原因、实施后果以及从实践中获得的经验教训。通过 ADR 格式来确保架构决策的透明性和可追溯性。

## ADR-001: 四层密钥层次结构

### 状态
已接受并实施

### 背景
TEE 环境需要安全的存储加密方案，需要在硬件信任根和实际数据加密之间建立适当的密钥层次。

### 决策
采用四层密钥层次结构：HUK → SSK → TSK → FEK → 数据

```c
// 实施的四层结构
HUK (Hardware Unique Key) - 硬件唯一密钥
 ↓ HMAC-SHA256
SSK (Secure Storage Key) - 安全存储密钥  
 ↓ HMAC-SHA256 + TA_UUID
TSK (Trusted App Storage Key) - TA存储密钥
 ↓ AES-ECB
FEK (File Encryption Key) - 文件加密密钥
 ↓ AES-CBC + ESSIV
数据块
```

### 考虑的备选方案

#### 备选方案 1: 三层结构 (HUK → TSK → FEK)
```c
HUK → (直接推导) → TSK → FEK → 数据
```
**优势:** 更简单，性能更好
**劣势:** 
- 硬件密钥直接用于应用推导，泄露风险高
- 缺乏平台级别的密钥隔离
- 难以支持非 TA 存储（如系统配置）

#### 备选方案 2: 五层结构 (HUK → PSK → SSK → TSK → FEK)  
```c
HUK → PSK → SSK → TSK → FEK → 数据
```
**优势:** 更细粒度的隔离
**劣势:**
- 增加计算开销
- 复杂性过高
- 额外层次没有明确的安全边界

#### 备选方案 3: 直接硬件加密 (HUK → 数据)
```c
HUK → (直接加密) → 数据
```
**优势:** 最高性能
**劣势:**
- 无法支持多 TA 隔离
- 硬件密钥过度暴露
- 无法支持密钥轮换

### 决策原因

1. **安全隔离**: SSK 层提供平台级隔离，TSK 层提供应用级隔离
2. **性能平衡**: 四层在安全性和性能间取得最佳平衡
3. **灵活性**: 支持 TA 隔离、非 TA 存储、密钥轮换
4. **标准兼容**: 符合业界密钥管理最佳实践

### 实施细节
```c
// SSK 推导 - 平台级密钥
static TEE_Result derive_ssk(void)
{
    return huk_subkey_derive(HUK_SUBKEY_SSK, 
                           "STORAGE", 7,
                           tee_fs_ssk.key, sizeof(tee_fs_ssk.key));
}

// TSK 推导 - TA 级密钥
static TEE_Result derive_tsk(const TEE_UUID *uuid, uint8_t *tsk)
{
    if (uuid) {
        return do_hmac(tsk, TEE_FS_KM_TSK_SIZE,
                      tee_fs_ssk.key, TEE_FS_KM_SSK_SIZE,
                      uuid, sizeof(*uuid));
    } else {
        // 非 TA 存储使用特殊盐值
        uint8_t dummy[1] = { 0 };
        return do_hmac(tsk, TEE_FS_KM_TSK_SIZE,
                      tee_fs_ssk.key, TEE_FS_KM_SSK_SIZE,
                      dummy, sizeof(dummy));
    }
}
```

### 后果
**正面:**
- 强有力的 TA 隔离
- 支持密钥轮换和前向安全
- 符合安全最佳实践
- 良好的性能表现

**负面:**
- 增加了一定的计算开销
- 密钥管理逻辑相对复杂
- 需要安全的密钥推导实现

### 经验教训
- 四层结构证明是安全性和性能的最佳平衡点
- HMAC-SHA256 推导提供了良好的安全性
- 需要特别注意密钥清理和错误处理路径

---

## ADR-002: 双存储后端架构

### 状态
已接受并实施

### 背景
TEE 存储需要支持不同的安全级别和性能要求。单一存储后端无法同时满足高性能和高安全性的需求。

### 决策
采用 REE 文件系统 + RPMB 的双后端架构，根据数据特性智能选择后端。

### 考虑的备选方案

#### 备选方案 1: 仅 REE 文件系统
```c
// 单一 REE FS 后端
struct single_ree_backend {
    char *base_path;
    bool encryption_enabled;
    bool integrity_enabled;
};
```
**优势:** 简单实现，高性能，大容量
**劣势:** 
- 依赖 REE 安全性
- 容易受到物理攻击
- 无硬件防回滚保护

#### 备选方案 2: 仅 RPMB
```c
// 单一 RPMB 后端  
struct single_rpmb_backend {
    uint8_t rpmb_key[32];
    uint32_t current_counter;
};
```
**优势:** 硬件保护，防篡改，防回滚
**劣势:**
- 容量限制（通常 < 16MB）
- 性能限制
- 硬件依赖性

#### 备选方案 3: 软件 RAID 合并
```c
// RAID-like 合并方案
struct raid_storage {
    struct ree_backend primary;
    struct rpmb_backend mirror;
    enum raid_mode mode;  // RAID0, RAID1
};
```
**优势:** 高可用性
**劣势:**
- 复杂性高
- 性能开销大
- 不适合容量差异大的后端

### 决策原因

1. **互补优势**: REE FS 高性能大容量，RPMB 高安全防篡改
2. **灵活选择**: 根据数据重要性和大小选择合适后端
3. **渐进部署**: 可以在不同硬件平台上渐进部署
4. **成本效益**: 在安全性和成本间取得平衡

### 实施细节
```c
// 后端选择策略
static const struct tee_fs_htree_storage *
select_storage_backend(const TEE_UUID *uuid, void *obj_id, uint32_t flags)
{
    // 安全关键数据优先使用 RPMB
    if (is_security_critical(uuid, obj_id) && rpmb_available()) {
        return &rpmb_storage_ops;
    }
    
    // 大文件使用 REE FS
    size_t estimated_size = estimate_object_size(uuid, obj_id);
    if (estimated_size > RPMB_EFFICIENT_THRESHOLD) {
        return &ree_fs_storage_ops;
    }
    
    // 默认选择最安全的可用后端
    return rpmb_available() ? &rpmb_storage_ops : &ree_fs_storage_ops;
}

// 统一的存储接口
struct tee_fs_htree_storage {
    size_t block_size;
    TEE_Result (*rpc_read_init)(...);
    TEE_Result (*rpc_write_init)(...);
    TEE_Result (*rpc_read_final)(...);
    TEE_Result (*rpc_write_final)(...);
};
```

### 后果
**正面:**
- 满足不同安全和性能需求
- 硬件平台适应性强
- 支持渐进式部署
- 良好的成本效益比

**负面:**
- 增加代码复杂性
- 需要后端选择逻辑
- 测试覆盖面更广

### 经验教训
- 双后端策略证明非常成功
- 后端抽象层设计良好，易于扩展
- 需要仔细测试不同后端的故障情况

---

## ADR-003: 哈希树完整性保护

### 状态
已接受并实施

### 背景
存储数据需要强完整性保护，能够检测任何篡改并支持高效的部分更新。

### 决策
采用二叉哈希树 (Binary Hash Tree) 提供分块数据的完整性保护。

### 考虑的备选方案

#### 备选方案 1: 全文件 HMAC
```c
struct file_hmac {
    uint8_t hmac[32];
    uint32_t file_size;
    uint8_t data[];
};
```
**优势:** 实现简单，计算开销小
**劣势:**
- 修改任何字节都需要重新计算整个 HMAC
- 无法支持大文件的高效更新
- 无法并行验证

#### 备选方案 2: 分块 HMAC
```c
struct block_hmac {
    uint32_t block_count;
    uint8_t hmac_array[];  // 每块一个 HMAC
    uint8_t data[];
};
```
**优势:** 支持部分更新
**劣势:**
- HMAC 数组本身需要完整性保护
- 存储开销线性增长
- 验证所有块仍然是 O(n)

#### 备选方案 3: 单根哈希链
```c
struct hash_chain {
    uint8_t root_hash[32];
    uint8_t block_hashes[];  // 线性哈希链
};
```
**优势:** 单一根哈希验证
**劣势:**
- 更新传播是线性的
- 无法并行计算
- 不适合随机访问模式

### 决策原因

1. **对数复杂度**: 更新和验证都是 O(log n)
2. **并行友好**: 不同子树可以并行处理
3. **增量同步**: 只需同步变化的节点
4. **内存效率**: 可以按需加载树节点

### 实施细节
```c
// 哈希树节点结构
struct tee_fs_htree_node_image {
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];    // 32 字节 SHA-256
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];        // 16 字节 IV
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];      // 16 字节 GCM tag
    uint16_t flags;                          // 节点标志
};

// 高效的部分更新
static TEE_Result update_hash_path(struct tee_fs_htree *ht, 
                                  size_t block_idx)
{
    struct htree_node *node = get_leaf_node(ht, block_idx);
    
    // 只更新从叶子到根的路径
    while (node) {
        res = calc_node_hash(node, NULL, ctx, node->node.hash);
        if (res) return res;
        
        node->dirty = true;
        node = node->parent;  // 向上传播
    }
    
    return TEE_SUCCESS;
}

// 并行验证支持
static TEE_Result verify_subtree_parallel(struct htree_node *root)
{
    if (!root) return TEE_SUCCESS;
    
    // 递归验证子树（可以并行化）
    TEE_Result left_res = verify_subtree_parallel(root->child[0]);
    TEE_Result right_res = verify_subtree_parallel(root->child[1]);
    
    if (left_res || right_res) 
        return TEE_ERROR_CORRUPT_OBJECT;
        
    // 验证当前节点
    return verify_node_hash(root);
}
```

### 后果
**正面:**
- 高效的部分更新 (O(log n))
- 支持并行验证和计算
- 增量同步友好
- 强完整性保护

**负面:**
- 实现复杂度较高
- 小文件有额外开销
- 需要careful的内存管理

### 经验教训
- 哈希树是大文件完整性保护的最佳选择
- 双版本机制对原子更新至关重要
- 需要仔细处理树的平衡和深度

---

## ADR-004: 写时复制 (Copy-on-Write) 更新策略

### 状态
已接受并实施

### 背景
存储更新需要保证原子性和一致性，系统崩溃时不能产生部分更新或损坏状态。

### 决策
采用写时复制 (COW) 策略，所有更新操作都写入新位置，完成后原子性地切换引用。

### 考虑的备选方案

#### 备选方案 1: 就地更新 (In-place Update)
```c
static TEE_Result inplace_update(void *block, size_t offset, 
                                const void *data, size_t len)
{
    // 直接修改原始位置
    memcpy((char*)block + offset, data, len);
    sync_to_storage(block);
    return TEE_SUCCESS;
}
```
**优势:** 性能最高，实现简单
**劣势:**
- 无原子性保证
- 系统崩溃可能导致损坏
- 并发访问不安全
- 无法回滚

#### 备选方案 2: 双缓冲 (Double Buffering)
```c
struct double_buffer {
    void *buffer_a;
    void *buffer_b;
    bool active_is_a;
};

static TEE_Result double_buffer_update(struct double_buffer *db,
                                      const void *new_data)
{
    void *inactive = db->active_is_a ? db->buffer_b : db->buffer_a;
    memcpy(inactive, new_data, BUFFER_SIZE);
    sync_to_storage(inactive);
    
    // 原子切换
    db->active_is_a = !db->active_is_a;
    return TEE_SUCCESS;
}
```
**优势:** 原子性，实现较简单
**劣势:**
- 内存开销翻倍
- 只适用于固定大小数据
- 无法支持部分更新

#### 备选方案 3: 事务日志 (Transaction Log)
```c
struct transaction_log {
    uint32_t transaction_id;
    uint32_t operation_count;
    struct log_entry entries[];
};

static TEE_Result log_based_update(struct transaction_log *log,
                                  struct update_operation *ops, size_t count)
{
    // 先记录所有操作到日志
    for (size_t i = 0; i < count; i++) {
        append_log_entry(log, &ops[i]);
    }
    sync_log(log);
    
    // 应用操作
    for (size_t i = 0; i < count; i++) {
        apply_operation(&ops[i]);
    }
    
    // 清理日志
    clear_log(log);
    return TEE_SUCCESS;
}
```
**优势:** 支持事务，可以批量操作
**劣势:**
- 实现复杂
- 性能开销大
- 需要恢复逻辑

### 决策原因

1. **原子性**: COW 天然提供原子更新
2. **并发安全**: 读者可以继续访问旧版本
3. **错误恢复**: 失败时旧数据保持完整
4. **快照支持**: 自然支持版本控制
5. **简单性**: 相比事务日志更简单

### 实施细节
```c
// COW 块更新实现
static TEE_Result out_of_place_write(struct tee_fs_fd *fdp, size_t pos,
                                     const void *buf_core, const void *buf_user,
                                     size_t len)
{
    uint8_t *block = get_tmp_block();
    
    // 1. 读取原始块到临时缓冲区
    if (start_block_num * BLOCK_SIZE < ROUNDUP(meta->length, BLOCK_SIZE)) {
        res = tee_fs_htree_read_block(&fdp->ht, start_block_num, block);
        if (res) goto exit;
    } else {
        memset(block, 0, BLOCK_SIZE);  // 新块初始化为零
    }
    
    // 2. 在临时缓冲区中修改数据
    if (data_core_ptr) {
        memcpy(block + offset, data_core_ptr, size_to_write);
    }
    if (data_user_ptr) {
        res = copy_from_user(block + offset, data_user_ptr, size_to_write);
        if (res) goto exit;
    }
    
    // 3. 写入新位置（原子操作）
    res = tee_fs_htree_write_block(&fdp->ht, start_block_num, block);
    
exit:
    put_tmp_block(block);
    return res;
}

// 双版本机制支持原子提交
struct htree_node {
    size_t id;
    bool dirty;
    bool block_updated;
    struct tee_fs_htree_node_image node;
    struct htree_node *parent;
    struct htree_node *child[2];
};

// 原子提交机制
static TEE_Result commit_tree_changes(struct tee_fs_htree *ht)
{
    // 所有脏节点已经写入存储的备用位置
    // 现在原子性地更新头部计数器
    ht->head.counter++;
    
    res = tee_fs_htree_sync_to_storage(&ht, hash, &ht->head.counter);
    if (res == TEE_SUCCESS) {
        // 成功：标记所有节点为干净
        clear_dirty_flags(ht);
    } else {
        // 失败：回滚到原始状态
        rollback_dirty_changes(ht);
    }
    
    return res;
}
```

### 后果
**正面:**
- 强原子性保证
- 优秀的并发性能
- 简单的错误恢复
- 自然支持快照
- 防止数据损坏

**负面:**
- 增加存储空间需求
- 垃圾回收复杂性
- 写放大效应
- 临时内存使用

### 经验教训
- COW 策略在 TEE 环境中表现优异
- 双版本机制对原子性至关重要
- 需要仔细设计垃圾回收策略
- 临时块管理需要优化

---

## ADR-005: GlobalPlatform API 兼容性

### 状态
已接受并实施

### 背景
TEE 存储需要提供标准化的 API 以确保应用程序的可移植性和生态系统兼容性。

### 决策
严格遵循 GlobalPlatform TEE Internal Core API 规范，提供标准兼容的存储接口。

### 考虑的备选方案

#### 备选方案 1: 完全自定义 API
```c
// 假设的 OP-TEE 优化 API
TEE_Result optee_storage_open_advanced(
    const char *path,
    uint32_t flags,
    struct storage_hints *hints,    // 性能提示
    struct encryption_params *enc,  // 加密参数
    struct file_handle **fh
);

TEE_Result optee_storage_batch_write(
    struct file_handle *fh,
    struct write_operation *ops,
    size_t op_count
);
```
**优势:** 
- 性能优化空间大
- 可以暴露 OP-TEE 特定功能
- API 设计自由度高

**劣势:**
- 应用程序不可移植
- 开发者学习成本高
- 无标准测试套件
- 生态系统割裂

#### 备选方案 2: GP API + 扩展 API
```c
// GP 标准 API
TEE_Result TEE_OpenPersistentObject(...);

// OP-TEE 扩展 API
TEE_Result TEE_OpenPersistentObjectExt(
    uint32_t storageID,
    void* objectID, size_t objectIDLen,
    uint32_t flags,
    TEE_ObjectHandle* object,
    struct optee_storage_hints *hints  // 扩展参数
);
```
**优势:**
- 兼容性和性能兼顾
- 渐进式功能增强

**劣势:**
- API 表面增大
- 维护复杂性增加
- 标准兼容性可能受影响

#### 备选方案 3: 最小 GP 子集
```c
// 只实现 GP API 的核心子集
TEE_Result TEE_OpenPersistentObject(...);   // 支持
TEE_Result TEE_ReadObjectData(...);         // 支持
TEE_Result TEE_WriteObjectData(...);        // 支持
// TEE_SeekObjectData(...);                 // 不支持
// TEE_TruncateObjectData(...);             // 不支持
```
**优势:**
- 实现简单
- 核心功能足够

**劣势:**
- 功能不完整
- 应用程序兼容性问题
- 不符合标准要求

### 决策原因

1. **生态系统兼容**: 支持现有 GP 兼容应用
2. **开发者体验**: 降低学习成本
3. **标准合规**: 符合行业标准要求
4. **测试覆盖**: 利用标准测试套件
5. **长期维护**: 标准演进跟踪

### 实施细节
```c
// 严格的 GP API 实现
TEE_Result TEE_OpenPersistentObject(uint32_t storageID,
                                   void* objectID, size_t objectIDLen,
                                   uint32_t flags,
                                   TEE_ObjectHandle* object)
{
    struct tee_obj *o = NULL;
    struct tee_pobj *po = NULL;
    struct tee_file_handle *fh = NULL;
    TEE_Result res;
    
    // 严格的参数验证（按 GP 规范）
    if (objectIDLen > TEE_OBJECT_ID_MAX_LEN)
        return TEE_ERROR_BAD_PARAMETERS;
        
    if (!objectID && objectIDLen)
        return TEE_ERROR_BAD_PARAMETERS;
        
    if (!object)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 存储 ID 验证（GP 规范要求）
    if (storageID != TEE_STORAGE_PRIVATE &&
        storageID != TEE_STORAGE_PRIVATE_REE)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    // 标准的对象创建流程
    res = tee_obj_alloc(&o);
    if (res) return res;
    
    // 内部实现可以优化，但 API 严格遵循 GP
    res = tee_pobj_get(&session_ctx->uuid, objectID, objectIDLen, flags, &po);
    if (res) goto err;
    
    res = tee_file_ops_open(po, &fh);
    if (res) goto err;
    
    o->info.objectType = TEE_TYPE_DATA;
    o->pobj = po;
    o->fh = fh;
    
    *object = (TEE_ObjectHandle)o;
    return TEE_SUCCESS;

err:
    tee_obj_free(o);
    tee_pobj_put(po);
    return res;
}

// 内部优化不影响 API 兼容性
static TEE_Result optimized_read_implementation(struct tee_file_handle *fh,
                                               void *buffer, size_t *size)
{
    // 可以使用 OP-TEE 特定的优化技术
    // 如块缓存、预读、批量 I/O 等
    // 但不暴露给应用程序
    return internal_optimized_read(fh, buffer, size);
}
```

### 后果
**正面:**
- 优秀的应用程序兼容性
- 标准化开发体验
- 丰富的测试覆盖
- 生态系统支持
- 规范合规性

**负面:**
- 某些性能优化受限
- API 设计灵活性受约束
- 需要跟踪标准演进
- 扩展功能实现复杂

### 经验教训
- GP API 兼容性是正确的选择
- 内部实现可以高度优化而不影响兼容性
- 标准兼容带来的生态系统价值远超性能损失
- 需要持续跟踪 GP 标准的演进

---

## ADR-006: ESSIV 加密模式选择

### 状态
已接受并实施

### 背景
块级加密需要选择适当的 IV 生成策略，防止模式分析攻击并确保相同明文块产生不同密文。

### 决策
采用 ESSIV (Encrypted Salt-Sector IV) 模式，使用 AES-ECB(SHA256(FEK), block_index) 生成唯一 IV。

### 考虑的备选方案

#### 备选方案 1: 简单计数器 IV
```c
static void generate_counter_iv(uint8_t iv[16], uint16_t block_idx)
{
    memset(iv, 0, 16);
    iv[0] = block_idx & 0xFF;
    iv[1] = (block_idx >> 8) & 0xFF;
    // 简单但可预测的 IV
}
```
**优势:** 实现简单，性能高
**劣势:**
- IV 可预测，容易遭受攻击
- 相同明文块可能产生相同密文（如果使用相同密钥）
- 不抵抗模式分析

#### 备选方案 2: 随机 IV
```c
static TEE_Result generate_random_iv(uint8_t iv[16])
{
    return crypto_rng_read(iv, 16);  // 完全随机
}
```
**优势:** 最高的不可预测性
**劣势:**
- 需要存储 IV（额外存储开销）
- 随机数生成器可能成为性能瓶颈
- 对于同一块的重复写入产生不同密文（影响去重）

#### 备选方案 3: 基于文件密钥的简单 IV
```c
static void generate_fek_iv(uint8_t iv[16], const uint8_t fek[16], 
                           uint16_t block_idx)
{
    memcpy(iv, fek, 16);
    iv[0] ^= block_idx & 0xFF;
    iv[1] ^= (block_idx >> 8) & 0xFF;
}
```
**优势:** 与文件绑定，简单高效
**劣势:**
- 安全性不如 ESSIV
- 可能存在相关密钥攻击风险

#### 备选方案 4: ESSIV (选择的方案)
```c
static TEE_Result essiv(uint8_t iv[TEE_AES_BLOCK_SIZE],
                       const uint8_t fek[TEE_FS_KM_FEK_SIZE],
                       uint16_t blk_idx)
{
    uint8_t sha[TEE_SHA256_HASH_SIZE];
    uint8_t pad_blkid[TEE_AES_BLOCK_SIZE] = { 0, };
    
    // 哈希文件密钥创建 IV 加密密钥
    res = sha256(sha, sizeof(sha), fek, TEE_FS_KM_FEK_SIZE);
    if (res) return res;
    
    // 块索引填充到 AES 块大小
    pad_blkid[0] = (blk_idx & 0xFF);
    pad_blkid[1] = (blk_idx & 0xFF00) >> 8;
    
    // 使用哈希后的密钥加密块索引得到 IV
    res = aes_ecb(iv, pad_blkid, sha, 16);
    
    memzero_explicit(sha, sizeof(sha));
    return res;
}
```

### 决策原因

1. **安全性**: ESSIV 提供密码学强度的 IV 唯一性
2. **确定性**: 相同块在相同文件中产生相同 IV（支持去重）
3. **不可预测性**: 不知道 FEK 就无法预测 IV
4. **标准化**: ESSIV 是业界认可的安全模式
5. **性能**: 相比随机 IV 不需要额外存储

### 实施细节
```c
// 完整的 ESSIV 实现
TEE_Result tee_fs_crypt_block(const TEE_UUID *uuid, uint8_t *out,
                             const uint8_t *in, size_t size,
                             uint16_t blk_idx, const uint8_t *encrypted_fek,
                             TEE_OperationMode mode)
{
    uint8_t fek[TEE_FS_KM_FEK_SIZE];
    uint8_t iv[TEE_AES_BLOCK_SIZE];
    void *ctx = NULL;
    
    // 解密 FEK
    res = tee_fs_fek_crypt(uuid, TEE_MODE_DECRYPT, encrypted_fek,
                          sizeof(fek), fek);
    if (res) goto wipe;
    
    // 生成 ESSIV
    res = essiv(iv, fek, blk_idx);
    if (res) goto wipe;
    
    // 使用 FEK 和 ESSIV 进行 AES-CBC 加密/解密
    res = crypto_cipher_alloc_ctx(&ctx, TEE_ALG_AES_CBC_NOPAD);
    if (res) goto wipe;
    
    res = crypto_cipher_init(ctx, mode, fek, sizeof(fek), NULL, 0, iv, sizeof(iv));
    if (res) goto wipe;
    
    res = crypto_cipher_update(ctx, mode, true, in, size, out, &size);

wipe:
    crypto_cipher_free_ctx(ctx);
    memzero_explicit(fek, sizeof(fek));     // 清理 FEK
    memzero_explicit(iv, sizeof(iv));       // 清理 IV
    return res;
}

// IV 生成的安全属性验证
static void test_essiv_properties(void)
{
    uint8_t fek1[16] = { /* 测试密钥1 */ };
    uint8_t fek2[16] = { /* 测试密钥2 */ };
    uint8_t iv1[16], iv2[16], iv3[16];
    
    // 属性1: 相同 FEK 和块索引产生相同 IV
    assert(essiv(iv1, fek1, 0) == TEE_SUCCESS);
    assert(essiv(iv2, fek1, 0) == TEE_SUCCESS);
    assert(memcmp(iv1, iv2, 16) == 0);
    
    // 属性2: 不同块索引产生不同 IV
    assert(essiv(iv1, fek1, 0) == TEE_SUCCESS);
    assert(essiv(iv2, fek1, 1) == TEE_SUCCESS);
    assert(memcmp(iv1, iv2, 16) != 0);
    
    // 属性3: 不同 FEK 产生不同 IV
    assert(essiv(iv1, fek1, 0) == TEE_SUCCESS);
    assert(essiv(iv2, fek2, 0) == TEE_SUCCESS);
    assert(memcmp(iv1, iv2, 16) != 0);
}
```

### 后果
**正面:**
- 强密码学安全性
- 防止模式分析攻击
- 支持确定性去重
- 符合安全最佳实践
- 无需额外存储空间

**负面:**
- 比简单 IV 计算开销大
- 实现复杂度增加
- 需要正确的密钥清理

### 经验教训
- ESSIV 是块加密的最佳实践
- 必须正确处理中间密钥的清理
- 性能开销是可接受的
- 测试需要验证 IV 的安全属性

---

## 决策总结和经验教训

### 架构决策的一致性原则

通过分析这些 ADR，我们可以看到 OP-TEE 存储系统的架构决策遵循一致的原则：

1. **安全优先**: 每个决策都将安全性作为首要考虑
2. **标准兼容**: 优先选择行业标准和最佳实践
3. **性能平衡**: 在安全约束下追求最佳性能
4. **简单性**: 避免过度工程化，选择简单有效的方案
5. **可扩展性**: 为未来演进留出空间

### 跨 ADR 的设计哲学

- **分层设计**: 每个 ADR 都体现了清晰的分层和抽象
- **正交性**: 不同决策相对独立，可以单独演进
- **一致性**: 所有决策都支持整体的安全和性能目标
- **可测试性**: 每个决策都考虑了测试和验证需求

### 未来决策指导

这些 ADR 为未来的架构决策提供了重要指导：

1. **新功能必须符合既定的安全原则**
2. **性能优化不能牺牲安全性**
3. **标准兼容性优于自定义优化**
4. **简单性优于复杂的理论最优解**

通过记录和分析这些架构决策，我们不仅理解了当前系统的设计原理，也为未来的演进提供了清晰的指导框架。