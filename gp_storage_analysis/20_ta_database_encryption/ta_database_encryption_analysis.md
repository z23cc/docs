# Trusted Application Database and Encryption Analysis

## Overview

OP-TEE 的 TADB (Trusted Application Database) 系统提供了完整的可信应用程序加密存储、管理和加载功能。该系统采用 AES-GCM 认证加密、位图文件分配和分层安全架构，为 TA 的安全存储和运行提供了强大的保护机制。

## TADB Architecture Overview

### 系统架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                 TA Management Layer                        │
│    • TA Installation  • Version Control  • Lifecycle      │
├─────────────────────────────────────────────────────────────┤
│              TADB Abstraction Layer                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │        tadb_open() • tadb_insert() • tadb_lookup()    ││
│  │      tadb_delete() • tadb_close() • tadb_iterate()    ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│           Encryption and Authentication Layer             │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐│
│  │  AES-GCM        │  │    Signature Verification         ││
│  │ • 256-bit keys  │  │ • PKCS#1 v1.5  • X.509 certs    ││
│  │ • 96-bit IV     │  │ • RSA/ECC      • Trust chains    ││
│  │ • 128-bit tags  │  │ • Image hash   • Policy check    ││
│  └─────────────────┘  └─────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│              File Management Layer                        │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐│
│  │ Bitmap Allocator│  │        Transaction System          ││
│  │ • File numbering│  │ • Atomic operations • Rollback    ││
│  │ • Conflict res. │  │ • Consistency      • Recovery     ││
│  │ • Growth mgmt   │  │ • Write ordering   • Validation   ││
│  └─────────────────┘  └─────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│           Storage Backend Selection Layer                 │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐│
│  │  Secure Storage │  │       REE File System             ││
│  │ • TEE_STORAGE_* │  │ • Standard file operations         ││
│  │ • Hardware prot │  │ • High performance                 ││
│  │ • RPMB/encrypted│  │ • Large capacity                   ││
│  └─────────────────┘  └─────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                Hardware Security Layer                    │
│    • HUK derivation • Key isolation • Secure boot        │
└─────────────────────────────────────────────────────────────┘
```

## TADB Core Implementation

### 核心数据结构

#### TADB 条目结构

```c
struct tadb_entry {
    struct tee_tadb_property prop;     // TA 属性信息
    uint32_t file_number;              // 文件编号
    uint8_t iv[TADB_IV_SIZE];         // AES-GCM 初始化向量 (12字节)
    uint8_t tag[TADB_TAG_SIZE];       // AES-GCM 认证标签 (16字节)
    uint8_t key[TADB_KEY_SIZE];       // 加密密钥 (32字节)
};

// TADB 常量定义
#define TADB_IV_SIZE    12             // AES-GCM IV 大小
#define TADB_TAG_SIZE   16             // AES-GCM 认证标签大小
#define TADB_KEY_SIZE   32             // AES-256 密钥大小
#define TADB_MAX_ENTRIES 512           // 最大 TA 条目数
```

#### TA 属性结构

```c
struct tee_tadb_property {
    TEE_UUID uuid;                     // TA 唯一标识符
    uint32_t version;                  // TA 版本号
    uint32_t flags;                    // TA 标志位
    
    // 安全属性
    struct {
        bool single_instance;          // 单实例标志
        bool multi_session;            // 多会话支持
        bool keep_alive;               // 保持活跃状态
        bool instance_keep_alive;      // 实例保持活跃
        uint32_t data_size;           // 数据段大小
        uint32_t stack_size;          // 栈大小
        uint32_t heap_size;           // 堆大小
    } ta_props;
    
    // 签名和认证信息
    struct {
        uint32_t sig_type;             // 签名类型
        uint32_t hash_algo;            // 哈希算法
        size_t sig_len;                // 签名长度
        uint8_t *signature;            // 数字签名
        uint8_t image_hash[TEE_SHA256_HASH_SIZE]; // 镜像哈希
    } auth_info;
};
```

### TADB 操作接口

#### 1. TADB 打开和初始化

```c
TEE_Result tadb_open(const char *db_path, struct tadb_handle **handle)
{
    struct tadb_handle *db = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    db = calloc(1, sizeof(*db));
    if (!db)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 初始化互斥锁
    mutex_init(&db->mutex);
    
    // 设置数据库路径
    db->db_path = strdup(db_path);
    if (!db->db_path) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto error;
    }
    
    // 初始化文件分配位图
    res = tadb_init_file_bitmap(db);
    if (res != TEE_SUCCESS)
        goto error;
    
    // 加载现有的 TADB 条目
    res = tadb_load_entries(db);
    if (res != TEE_SUCCESS)
        goto error;
    
    *handle = db;
    return TEE_SUCCESS;
    
error:
    tadb_close(db);
    return res;
}
```

#### 2. TA 插入操作

```c
TEE_Result tadb_insert(struct tadb_handle *db, const struct tee_tadb_property *prop,
                      const uint8_t *ta_data, size_t ta_size)
{
    struct tadb_entry entry = { 0 };
    uint32_t file_num = 0;
    uint8_t *encrypted_data = NULL;
    size_t encrypted_size = 0;
    TEE_Result res = TEE_SUCCESS;
    
    if (!db || !prop || !ta_data || ta_size == 0)
        return TEE_ERROR_BAD_PARAMETERS;
    
    mutex_lock(&db->mutex);
    
    // 检查是否已存在相同 UUID 的 TA
    if (tadb_lookup_by_uuid(db, &prop->uuid) == TEE_SUCCESS) {
        EMSG("TA with UUID already exists");
        res = TEE_ERROR_ACCESS_CONFLICT;
        goto out;
    }
    
    // 分配文件编号
    res = tadb_allocate_file_number(db, &file_num);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 生成随机加密密钥和 IV
    res = crypto_rng_read(entry.key, sizeof(entry.key));
    if (res != TEE_SUCCESS)
        goto out;
        
    res = crypto_rng_read(entry.iv, sizeof(entry.iv));
    if (res != TEE_SUCCESS)
        goto out;
    
    // 加密 TA 数据
    res = tadb_encrypt_ta_data(ta_data, ta_size, entry.key, entry.iv,
                              &encrypted_data, &encrypted_size, entry.tag);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 填充条目信息
    entry.prop = *prop;
    entry.file_number = file_num;
    
    // 写入加密的 TA 数据到存储
    res = tadb_write_ta_file(db, file_num, encrypted_data, encrypted_size);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 添加条目到数据库
    res = tadb_add_entry(db, &entry);
    if (res != TEE_SUCCESS) {
        // 回滚：删除已写入的文件
        tadb_delete_ta_file(db, file_num);
        goto out;
    }
    
    // 持久化数据库状态
    res = tadb_save_metadata(db);
    
out:
    free(encrypted_data);
    mutex_unlock(&db->mutex);
    return res;
}
```

#### 3. TA 查找和加载

```c
TEE_Result tadb_lookup_and_load(struct tadb_handle *db, const TEE_UUID *uuid,
                               uint8_t **ta_data, size_t *ta_size)
{
    struct tadb_entry *entry = NULL;
    uint8_t *encrypted_data = NULL;
    size_t encrypted_size = 0;
    uint8_t *decrypted_data = NULL;
    size_t decrypted_size = 0;
    TEE_Result res = TEE_SUCCESS;
    
    if (!db || !uuid || !ta_data || !ta_size)
        return TEE_ERROR_BAD_PARAMETERS;
    
    mutex_lock(&db->mutex);
    
    // 根据 UUID 查找条目
    entry = tadb_find_entry_by_uuid(db, uuid);
    if (!entry) {
        res = TEE_ERROR_ITEM_NOT_FOUND;
        goto out;
    }
    
    // 读取加密的 TA 数据
    res = tadb_read_ta_file(db, entry->file_number, &encrypted_data, &encrypted_size);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 解密 TA 数据
    res = tadb_decrypt_ta_data(encrypted_data, encrypted_size,
                              entry->key, entry->iv, entry->tag,
                              &decrypted_data, &decrypted_size);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to decrypt TA data - integrity check failed");
        goto out;
    }
    
    // 验证 TA 签名和完整性
    res = tadb_verify_ta_integrity(entry, decrypted_data, decrypted_size);
    if (res != TEE_SUCCESS) {
        EMSG("TA integrity verification failed");
        goto out;
    }
    
    *ta_data = decrypted_data;
    *ta_size = decrypted_size;
    decrypted_data = NULL;  // 避免释放
    
out:
    free(encrypted_data);
    free(decrypted_data);
    mutex_unlock(&db->mutex);
    return res;
}
```

## Encryption and Authentication

### AES-GCM 认证加密实现

#### 加密函数

```c
static TEE_Result tadb_encrypt_ta_data(const uint8_t *plaintext, size_t plaintext_len,
                                      const uint8_t *key, const uint8_t *iv,
                                      uint8_t **ciphertext, size_t *ciphertext_len,
                                      uint8_t *tag)
{
    void *ctx = NULL;
    uint8_t *encrypted = NULL;
    size_t enc_len = plaintext_len;
    size_t tag_len = TADB_TAG_SIZE;
    TEE_Result res = TEE_SUCCESS;
    
    // 分配加密数据缓冲区
    encrypted = malloc(plaintext_len);
    if (!encrypted)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 创建 AES-GCM 认证加密上下文
    res = crypto_authenc_alloc_ctx(&ctx, TEE_ALG_AES_GCM);
    if (res != TEE_SUCCESS)
        goto error;
    
    // 初始化加密操作
    res = crypto_authenc_init(ctx, TEE_MODE_ENCRYPT, key, TADB_KEY_SIZE,
                            iv, TADB_IV_SIZE, tag_len, 0, plaintext_len);
    if (res != TEE_SUCCESS)
        goto error;
    
    // 执行认证加密
    res = crypto_authenc_enc_final(ctx, plaintext, plaintext_len,
                                 encrypted, &enc_len, tag, &tag_len);
    if (res != TEE_SUCCESS)
        goto error;
    
    *ciphertext = encrypted;
    *ciphertext_len = enc_len;
    
    crypto_authenc_free_ctx(ctx);
    return TEE_SUCCESS;
    
error:
    free(encrypted);
    crypto_authenc_free_ctx(ctx);
    return res;
}
```

#### 解密函数

```c
static TEE_Result tadb_decrypt_ta_data(const uint8_t *ciphertext, size_t ciphertext_len,
                                      const uint8_t *key, const uint8_t *iv,
                                      const uint8_t *tag,
                                      uint8_t **plaintext, size_t *plaintext_len)
{
    void *ctx = NULL;
    uint8_t *decrypted = NULL;
    size_t dec_len = ciphertext_len;
    TEE_Result res = TEE_SUCCESS;
    
    // 分配解密数据缓冲区
    decrypted = malloc(ciphertext_len);
    if (!decrypted)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 创建 AES-GCM 认证解密上下文
    res = crypto_authenc_alloc_ctx(&ctx, TEE_ALG_AES_GCM);
    if (res != TEE_SUCCESS)
        goto error;
    
    // 初始化解密操作
    res = crypto_authenc_init(ctx, TEE_MODE_DECRYPT, key, TADB_KEY_SIZE,
                            iv, TADB_IV_SIZE, TADB_TAG_SIZE, 0, ciphertext_len);
    if (res != TEE_SUCCESS)
        goto error;
    
    // 执行认证解密
    res = crypto_authenc_dec_final(ctx, ciphertext, ciphertext_len,
                                 decrypted, &dec_len, tag, TADB_TAG_SIZE);
    if (res != TEE_SUCCESS) {
        EMSG("AES-GCM authentication failed - data may be corrupted");
        goto error;
    }
    
    *plaintext = decrypted;
    *plaintext_len = dec_len;
    
    crypto_authenc_free_ctx(ctx);
    return TEE_SUCCESS;
    
error:
    free(decrypted);
    crypto_authenc_free_ctx(ctx);
    return res;
}
```

### 密钥管理和派生

#### TA 加密密钥派生

```c
static TEE_Result tadb_derive_ta_encryption_key(const TEE_UUID *ta_uuid,
                                               uint8_t *key, size_t key_len)
{
    TEE_Result res = TEE_SUCCESS;
    const uint8_t label[] = "TA_ENCRYPTION_KEY";
    uint8_t uuid_bytes[sizeof(TEE_UUID)];
    
    // 将 UUID 转换为字节数组
    memcpy(uuid_bytes, ta_uuid, sizeof(*ta_uuid));
    
    // 使用 HUK 子密钥派生
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, uuid_bytes, sizeof(uuid_bytes),
                          key, key_len);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to derive TA encryption key");
        return res;
    }
    
    return TEE_SUCCESS;
}
```

#### 数据库主密钥管理

```c
struct tadb_master_key {
    uint8_t encryption_key[32];        // 数据库加密密钥
    uint8_t integrity_key[32];         // 完整性保护密钥
    uint32_t version;                  // 密钥版本号
    uint64_t creation_time;            // 创建时间戳
};

static TEE_Result tadb_derive_master_keys(struct tadb_master_key *master_key)
{
    const uint8_t enc_label[] = "TADB_ENCRYPTION_KEY";
    const uint8_t int_label[] = "TADB_INTEGRITY_KEY";
    TEE_Result res = TEE_SUCCESS;
    
    // 派生加密密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, enc_label, sizeof(enc_label) - 1,
                          master_key->encryption_key, sizeof(master_key->encryption_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 派生完整性密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, int_label, sizeof(int_label) - 1,
                          master_key->integrity_key, sizeof(master_key->integrity_key));
    if (res != TEE_SUCCESS)
        return res;
    
    master_key->version = TADB_KEY_VERSION_CURRENT;
    master_key->creation_time = get_time_stamp();
    
    return TEE_SUCCESS;
}
```

## File Management and Allocation

### 位图文件分配器

#### 位图数据结构

```c
#define TADB_MAX_FILES        2048     // 最大文件数
#define TADB_BITMAP_WORDS     (TADB_MAX_FILES / 32)

struct tadb_file_bitmap {
    uint32_t bitmap[TADB_BITMAP_WORDS]; // 文件分配位图
    uint32_t next_free_hint;            // 下次分配提示
    uint32_t allocated_count;           // 已分配文件数
    uint32_t max_file_number;           // 最大文件编号
    struct mutex allocation_mutex;      // 分配互斥锁
};
```

#### 文件编号分配

```c
static TEE_Result tadb_allocate_file_number(struct tadb_handle *db, uint32_t *file_num)
{
    struct tadb_file_bitmap *bitmap = &db->file_bitmap;
    uint32_t word_idx, bit_idx;
    TEE_Result res = TEE_ERROR_ITEM_NOT_FOUND;
    
    mutex_lock(&bitmap->allocation_mutex);
    
    // 检查是否已达到最大文件数
    if (bitmap->allocated_count >= TADB_MAX_FILES) {
        EMSG("TADB file limit reached");
        res = TEE_ERROR_STORAGE_NO_SPACE;
        goto out;
    }
    
    // 从提示位置开始搜索空闲位
    for (uint32_t i = bitmap->next_free_hint; i < TADB_MAX_FILES; i++) {
        word_idx = i / 32;
        bit_idx = i % 32;
        
        if (!(bitmap->bitmap[word_idx] & (1U << bit_idx))) {
            // 找到空闲位，标记为已分配
            bitmap->bitmap[word_idx] |= (1U << bit_idx);
            bitmap->allocated_count++;
            bitmap->next_free_hint = i + 1;
            
            if (i > bitmap->max_file_number)
                bitmap->max_file_number = i;
            
            *file_num = i;
            res = TEE_SUCCESS;
            goto out;
        }
    }
    
    // 如果从提示位置开始没找到，从头开始搜索
    for (uint32_t i = 0; i < bitmap->next_free_hint; i++) {
        word_idx = i / 32;
        bit_idx = i % 32;
        
        if (!(bitmap->bitmap[word_idx] & (1U << bit_idx))) {
            bitmap->bitmap[word_idx] |= (1U << bit_idx);
            bitmap->allocated_count++;
            bitmap->next_free_hint = i + 1;
            
            *file_num = i;
            res = TEE_SUCCESS;
            goto out;
        }
    }
    
out:
    mutex_unlock(&bitmap->allocation_mutex);
    return res;
}
```

#### 文件编号释放

```c
static TEE_Result tadb_free_file_number(struct tadb_handle *db, uint32_t file_num)
{
    struct tadb_file_bitmap *bitmap = &db->file_bitmap;
    uint32_t word_idx, bit_idx;
    
    if (file_num >= TADB_MAX_FILES)
        return TEE_ERROR_BAD_PARAMETERS;
    
    mutex_lock(&bitmap->allocation_mutex);
    
    word_idx = file_num / 32;
    bit_idx = file_num % 32;
    
    // 检查文件是否已分配
    if (!(bitmap->bitmap[word_idx] & (1U << bit_idx))) {
        EMSG("Attempting to free unallocated file number %u", file_num);
        mutex_unlock(&bitmap->allocation_mutex);
        return TEE_ERROR_BAD_STATE;
    }
    
    // 清除分配位
    bitmap->bitmap[word_idx] &= ~(1U << bit_idx);
    bitmap->allocated_count--;
    
    // 更新提示位置
    if (file_num < bitmap->next_free_hint)
        bitmap->next_free_hint = file_num;
    
    mutex_unlock(&bitmap->allocation_mutex);
    return TEE_SUCCESS;
}
```

### 事务处理系统

#### 事务状态管理

```c
enum tadb_transaction_state {
    TADB_TXN_INACTIVE = 0,
    TADB_TXN_ACTIVE,
    TADB_TXN_PREPARING,
    TADB_TXN_COMMITTED,
    TADB_TXN_ABORTED
};

struct tadb_transaction {
    enum tadb_transaction_state state;
    uint64_t txn_id;                   // 事务标识符
    uint32_t *allocated_files;         // 分配的文件编号数组
    size_t num_allocated;              // 分配的文件数量
    struct tadb_entry *entries_backup; // 条目备份
    size_t backup_count;               // 备份条目数
    uint64_t start_time;               // 事务开始时间
};
```

#### 事务开始和提交

```c
static TEE_Result tadb_transaction_begin(struct tadb_handle *db)
{
    struct tadb_transaction *txn = &db->current_txn;
    
    if (txn->state != TADB_TXN_INACTIVE) {
        EMSG("Transaction already active");
        return TEE_ERROR_BAD_STATE;
    }
    
    // 初始化事务
    txn->state = TADB_TXN_ACTIVE;
    txn->txn_id = get_time_stamp();
    txn->num_allocated = 0;
    txn->backup_count = 0;
    txn->start_time = txn->txn_id;
    
    // 创建当前状态的备份
    return tadb_create_state_backup(db, txn);
}

static TEE_Result tadb_transaction_commit(struct tadb_handle *db)
{
    struct tadb_transaction *txn = &db->current_txn;
    TEE_Result res = TEE_SUCCESS;
    
    if (txn->state != TADB_TXN_ACTIVE) {
        EMSG("No active transaction to commit");
        return TEE_ERROR_BAD_STATE;
    }
    
    txn->state = TADB_TXN_PREPARING;
    
    // 持久化所有更改
    res = tadb_save_metadata(db);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to persist transaction changes");
        return tadb_transaction_rollback(db);
    }
    
    // 同步到存储
    res = tadb_sync_storage(db);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to sync transaction to storage");
        return tadb_transaction_rollback(db);
    }
    
    // 清理事务状态
    txn->state = TADB_TXN_COMMITTED;
    tadb_cleanup_transaction(db);
    txn->state = TADB_TXN_INACTIVE;
    
    return TEE_SUCCESS;
}
```

#### 事务回滚

```c
static TEE_Result tadb_transaction_rollback(struct tadb_handle *db)
{
    struct tadb_transaction *txn = &db->current_txn;
    TEE_Result res = TEE_SUCCESS;
    
    txn->state = TADB_TXN_ABORTED;
    
    // 释放分配的文件编号
    for (size_t i = 0; i < txn->num_allocated; i++) {
        tadb_free_file_number(db, txn->allocated_files[i]);
        tadb_delete_ta_file(db, txn->allocated_files[i]);
    }
    
    // 恢复条目备份
    if (txn->entries_backup && txn->backup_count > 0) {
        memcpy(db->entries, txn->entries_backup, 
               txn->backup_count * sizeof(struct tadb_entry));
        db->num_entries = txn->backup_count;
    }
    
    // 清理事务状态
    tadb_cleanup_transaction(db);
    txn->state = TADB_TXN_INACTIVE;
    
    return res;
}
```

## TA Signature Verification

### 数字签名验证框架

#### 支持的签名算法

```c
enum tadb_signature_type {
    TADB_SIG_TYPE_NONE = 0,
    TADB_SIG_TYPE_RSA_PKCS1_SHA256,    // RSA PKCS#1 v1.5 with SHA-256
    TADB_SIG_TYPE_RSA_PSS_SHA256,      // RSA PSS with SHA-256
    TADB_SIG_TYPE_ECDSA_SHA256,        // ECDSA with SHA-256
    TADB_SIG_TYPE_EDDSA,               // EdDSA (Ed25519/Ed448)
};

struct tadb_signature_info {
    enum tadb_signature_type type;
    uint32_t key_size_bits;            // 密钥大小 (位)
    uint32_t hash_algo;                // 哈希算法
    size_t signature_len;              // 签名长度
    uint8_t *signature_data;           // 签名数据
    uint8_t *public_key;               // 公钥数据
    size_t public_key_len;             // 公钥长度
};
```

#### RSA 签名验证

```c
static TEE_Result tadb_verify_rsa_signature(const struct tadb_signature_info *sig_info,
                                           const uint8_t *data, size_t data_len)
{
    TEE_ObjectHandle key_handle = TEE_HANDLE_NULL;
    TEE_OperationHandle op_handle = TEE_HANDLE_NULL;
    TEE_Attribute key_attrs[2];
    uint8_t hash[TEE_SHA256_HASH_SIZE];
    size_t hash_len = sizeof(hash);
    TEE_Result res = TEE_SUCCESS;
    
    // 计算数据哈希
    res = crypto_hash_sha256(data, data_len, hash, &hash_len);
    if (res != TEE_SUCCESS)
        return res;
    
    // 创建 RSA 公钥对象
    res = TEE_AllocateTransientObject(TEE_TYPE_RSA_PUBLIC_KEY,
                                     sig_info->key_size_bits, &key_handle);
    if (res != TEE_SUCCESS)
        return res;
    
    // 设置公钥属性
    TEE_InitRefAttribute(&key_attrs[0], TEE_ATTR_RSA_MODULUS,
                        sig_info->public_key, sig_info->public_key_len);
    TEE_InitValueAttribute(&key_attrs[1], TEE_ATTR_RSA_PUBLIC_EXPONENT,
                          0x10001, 0);  // 标准公钥指数 65537
    
    res = TEE_PopulateTransientObject(key_handle, key_attrs, 2);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 创建签名验证操作
    res = TEE_AllocateOperation(&op_handle,
                               (sig_info->type == TADB_SIG_TYPE_RSA_PKCS1_SHA256) ?
                               TEE_ALG_RSASSA_PKCS1_V1_5_SHA256 :
                               TEE_ALG_RSASSA_PKCS1_PSS_MGF1_SHA256,
                               TEE_MODE_VERIFY, sig_info->key_size_bits);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    res = TEE_SetOperationKey(op_handle, key_handle);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 执行签名验证
    res = TEE_AsymmetricVerifyDigest(op_handle, NULL, 0,
                                   hash, hash_len,
                                   sig_info->signature_data, sig_info->signature_len);
    
cleanup:
    TEE_FreeOperation(op_handle);
    TEE_FreeTransientObject(key_handle);
    return res;
}
```

#### ECDSA 签名验证

```c
static TEE_Result tadb_verify_ecdsa_signature(const struct tadb_signature_info *sig_info,
                                             const uint8_t *data, size_t data_len)
{
    TEE_ObjectHandle key_handle = TEE_HANDLE_NULL;
    TEE_OperationHandle op_handle = TEE_HANDLE_NULL;
    TEE_Attribute key_attrs[3];
    uint8_t hash[TEE_SHA256_HASH_SIZE];
    size_t hash_len = sizeof(hash);
    TEE_Result res = TEE_SUCCESS;
    
    // 计算数据哈希
    res = crypto_hash_sha256(data, data_len, hash, &hash_len);
    if (res != TEE_SUCCESS)
        return res;
    
    // 创建 ECDSA 公钥对象 (P-256 曲线)
    res = TEE_AllocateTransientObject(TEE_TYPE_ECDSA_PUBLIC_KEY, 256, &key_handle);
    if (res != TEE_SUCCESS)
        return res;
    
    // 设置 ECC 公钥属性
    TEE_InitValueAttribute(&key_attrs[0], TEE_ATTR_ECC_CURVE,
                          TEE_ECC_CURVE_NIST_P256, 0);
    TEE_InitRefAttribute(&key_attrs[1], TEE_ATTR_ECC_PUBLIC_VALUE_X,
                        sig_info->public_key, 32);  // X 坐标
    TEE_InitRefAttribute(&key_attrs[2], TEE_ATTR_ECC_PUBLIC_VALUE_Y,
                        sig_info->public_key + 32, 32);  // Y 坐标
    
    res = TEE_PopulateTransientObject(key_handle, key_attrs, 3);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 创建 ECDSA 验证操作
    res = TEE_AllocateOperation(&op_handle, TEE_ALG_ECDSA_SHA256,
                               TEE_MODE_VERIFY, 256);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    res = TEE_SetOperationKey(op_handle, key_handle);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 执行签名验证
    res = TEE_AsymmetricVerifyDigest(op_handle, NULL, 0,
                                   hash, hash_len,
                                   sig_info->signature_data, sig_info->signature_len);
    
cleanup:
    TEE_FreeOperation(op_handle);
    TEE_FreeTransientObject(key_handle);
    return res;
}
```

### 证书链验证

#### X.509 证书解析

```c
struct tadb_x509_certificate {
    uint8_t *der_data;                 // DER 编码的证书数据
    size_t der_len;                    // DER 数据长度
    
    // 证书字段
    struct {
        uint8_t serial_number[20];     // 序列号
        TEE_Time not_before;           // 有效期开始
        TEE_Time not_after;            // 有效期结束
        char subject_dn[256];          // 主题可分辨名称
        char issuer_dn[256];           // 颁发者可分辨名称
    } fields;
    
    // 公钥信息
    struct tadb_signature_info public_key_info;
    
    // 签名信息
    struct tadb_signature_info signature_info;
};

static TEE_Result tadb_parse_x509_certificate(const uint8_t *cert_data, size_t cert_len,
                                             struct tadb_x509_certificate *cert)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 分配 DER 数据缓冲区
    cert->der_data = malloc(cert_len);
    if (!cert->der_data)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    memcpy(cert->der_data, cert_data, cert_len);
    cert->der_len = cert_len;
    
    // 解析证书结构 (简化的 ASN.1 解析)
    res = asn1_parse_certificate(cert_data, cert_len, cert);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to parse X.509 certificate");
        goto error;
    }
    
    // 验证证书有效期
    TEE_Time current_time;
    TEE_GetSystemTime(&current_time);
    
    if (TEE_CompareTime(&current_time, &cert->fields.not_before) < 0 ||
        TEE_CompareTime(&current_time, &cert->fields.not_after) > 0) {
        EMSG("Certificate is not within valid time range");
        res = TEE_ERROR_TIME_NOT_SET;
        goto error;
    }
    
    return TEE_SUCCESS;
    
error:
    free(cert->der_data);
    return res;
}
```

#### 证书链验证

```c
static TEE_Result tadb_verify_certificate_chain(const uint8_t **certs, size_t *cert_lens,
                                               size_t num_certs,
                                               const struct tadb_x509_certificate *trust_anchor)
{
    struct tadb_x509_certificate *chain = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    if (num_certs == 0)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 解析证书链
    chain = calloc(num_certs, sizeof(*chain));
    if (!chain)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    for (size_t i = 0; i < num_certs; i++) {
        res = tadb_parse_x509_certificate(certs[i], cert_lens[i], &chain[i]);
        if (res != TEE_SUCCESS)
            goto cleanup;
    }
    
    // 验证链中的每个证书
    for (size_t i = 0; i < num_certs; i++) {
        const struct tadb_x509_certificate *issuer = NULL;
        
        if (i == num_certs - 1) {
            // 最后一个证书应该由信任锚点签名
            issuer = trust_anchor;
        } else {
            // 由链中的下一个证书签名
            issuer = &chain[i + 1];
        }
        
        // 验证证书签名
        res = tadb_verify_certificate_signature(&chain[i], issuer);
        if (res != TEE_SUCCESS) {
            EMSG("Certificate signature verification failed at index %zu", i);
            goto cleanup;
        }
        
        // 验证证书路径约束
        res = tadb_verify_certificate_constraints(&chain[i], issuer);
        if (res != TEE_SUCCESS) {
            EMSG("Certificate path constraints violated at index %zu", i);
            goto cleanup;
        }
    }
    
cleanup:
    if (chain) {
        for (size_t i = 0; i < num_certs; i++)
            free(chain[i].der_data);
        free(chain);
    }
    return res;
}
```

## Storage Backend Selection

### 多后端支持架构

#### 存储后端定义

```c
enum tadb_storage_backend {
    TADB_STORAGE_AUTO = 0,             // 自动选择
    TADB_STORAGE_SECURE,               // 安全存储 (RPMB/encrypted)
    TADB_STORAGE_REE_FS,               // REE 文件系统
    TADB_STORAGE_MEMORY,               // 内存存储 (测试用)
};

struct tadb_storage_ops {
    TEE_Result (*open)(const char *path, void **handle);
    TEE_Result (*close)(void *handle);
    TEE_Result (*read)(void *handle, uint32_t file_num, uint8_t **data, size_t *len);
    TEE_Result (*write)(void *handle, uint32_t file_num, const uint8_t *data, size_t len);
    TEE_Result (*delete)(void *handle, uint32_t file_num);
    TEE_Result (*sync)(void *handle);
    TEE_Result (*get_info)(void *handle, struct tadb_storage_info *info);
};
```

#### 自动后端选择

```c
static enum tadb_storage_backend tadb_select_backend(void)
{
    struct tadb_storage_info info;
    enum tadb_storage_backend backend = TADB_STORAGE_REE_FS;
    
    // 检查安全存储可用性
    if (tee_fs_get_capabilities(TEE_STORAGE_PRIVATE_RPMB, &info) == TEE_SUCCESS) {
        if (info.available_space > TADB_MIN_SPACE_REQUIRED) {
            DMSG("Using RPMB secure storage for TADB");
            return TADB_STORAGE_SECURE;
        }
    }
    
    // 检查 REE 加密文件系统
    if (tee_fs_get_capabilities(TEE_STORAGE_PRIVATE_REE, &info) == TEE_SUCCESS) {
        if (info.available_space > TADB_MIN_SPACE_REQUIRED) {
            DMSG("Using REE encrypted filesystem for TADB");
            return TADB_STORAGE_REE_FS;
        }
    }
    
    EMSG("No suitable storage backend available for TADB");
    return TADB_STORAGE_AUTO;  // 错误情况
}
```

#### 安全存储后端实现

```c
static TEE_Result tadb_secure_storage_write(void *handle, uint32_t file_num,
                                           const uint8_t *data, size_t len)
{
    TEE_ObjectHandle obj_handle = TEE_HANDLE_NULL;
    char filename[64];
    TEE_Result res = TEE_SUCCESS;
    
    // 构造文件名
    snprintf(filename, sizeof(filename), "tadb_file_%08x", file_num);
    
    // 创建持久化对象
    res = TEE_CreatePersistentObject(TEE_STORAGE_PRIVATE_RPMB,
                                   filename, strlen(filename),
                                   TEE_DATA_FLAG_ACCESS_WRITE |
                                   TEE_DATA_FLAG_ACCESS_READ |
                                   TEE_DATA_FLAG_OVERWRITE,
                                   TEE_HANDLE_NULL, NULL, 0,
                                   &obj_handle);
    if (res != TEE_SUCCESS)
        return res;
    
    // 写入数据
    res = TEE_WriteObjectData(obj_handle, data, len);
    if (res != TEE_SUCCESS) {
        TEE_CloseAndDeletePersistentObject1(obj_handle);
        return res;
    }
    
    TEE_CloseObject(obj_handle);
    return TEE_SUCCESS;
}

static TEE_Result tadb_secure_storage_read(void *handle, uint32_t file_num,
                                          uint8_t **data, size_t *len)
{
    TEE_ObjectHandle obj_handle = TEE_HANDLE_NULL;
    char filename[64];
    TEE_ObjectInfo obj_info;
    uint8_t *buffer = NULL;
    uint32_t read_bytes = 0;
    TEE_Result res = TEE_SUCCESS;
    
    // 构造文件名
    snprintf(filename, sizeof(filename), "tadb_file_%08x", file_num);
    
    // 打开持久化对象
    res = TEE_OpenPersistentObject(TEE_STORAGE_PRIVATE_RPMB,
                                 filename, strlen(filename),
                                 TEE_DATA_FLAG_ACCESS_READ,
                                 &obj_handle);
    if (res != TEE_SUCCESS)
        return res;
    
    // 获取对象信息
    res = TEE_GetObjectInfo1(obj_handle, &obj_info);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 分配读取缓冲区
    buffer = malloc(obj_info.dataSize);
    if (!buffer) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto cleanup;
    }
    
    // 读取数据
    res = TEE_ReadObjectData(obj_handle, buffer, obj_info.dataSize, &read_bytes);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    *data = buffer;
    *len = read_bytes;
    buffer = NULL;  // 避免释放
    
cleanup:
    free(buffer);
    TEE_CloseObject(obj_handle);
    return res;
}
```

## TA Installation and Loading Process

### TA 安装流程

#### 安装步骤序列

```c
struct tadb_installation_context {
    const uint8_t *ta_image;           // TA 镜像数据
    size_t ta_image_size;              // 镜像大小
    const struct tadb_signature_info *signature; // 签名信息
    const uint8_t **cert_chain;        // 证书链
    size_t *cert_chain_lens;           // 证书长度数组
    size_t num_certs;                  // 证书数量
    struct tee_tadb_property props;    // TA 属性
    enum tadb_installation_flags {
        TADB_INSTALL_FLAG_FORCE = BIT(0),      // 强制安装
        TADB_INSTALL_FLAG_VERIFY_ONLY = BIT(1), // 仅验证
        TADB_INSTALL_FLAG_REPLACE = BIT(2),     // 替换现有
    } flags;
};

TEE_Result tadb_install_ta(struct tadb_handle *db,
                          const struct tadb_installation_context *install_ctx)
{
    TEE_Result res = TEE_SUCCESS;
    struct tadb_entry *existing_entry = NULL;
    
    // 步骤 1: 预验证
    res = tadb_pre_install_validation(install_ctx);
    if (res != TEE_SUCCESS) {
        EMSG("Pre-installation validation failed");
        return res;
    }
    
    // 步骤 2: 签名和证书链验证
    res = tadb_verify_ta_signature_and_certs(install_ctx);
    if (res != TEE_SUCCESS) {
        EMSG("TA signature or certificate verification failed");
        return res;
    }
    
    // 步骤 3: TA 镜像完整性验证
    res = tadb_verify_ta_image_integrity(install_ctx);
    if (res != TEE_SUCCESS) {
        EMSG("TA image integrity verification failed");
        return res;
    }
    
    // 步骤 4: 检查冲突和版本
    existing_entry = tadb_find_entry_by_uuid(db, &install_ctx->props.uuid);
    if (existing_entry) {
        res = tadb_handle_version_conflict(db, existing_entry, install_ctx);
        if (res != TEE_SUCCESS)
            return res;
    }
    
    // 步骤 5: 开始事务
    res = tadb_transaction_begin(db);
    if (res != TEE_SUCCESS)
        return res;
    
    // 步骤 6: 执行安装
    res = tadb_insert(db, &install_ctx->props, 
                     install_ctx->ta_image, install_ctx->ta_image_size);
    if (res != TEE_SUCCESS) {
        tadb_transaction_rollback(db);
        return res;
    }
    
    // 步骤 7: 提交事务
    res = tadb_transaction_commit(db);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to commit TA installation transaction");
        return res;
    }
    
    DMSG("TA installation completed successfully");
    return TEE_SUCCESS;
}
```

#### 版本冲突处理

```c
static TEE_Result tadb_handle_version_conflict(struct tadb_handle *db,
                                              struct tadb_entry *existing,
                                              const struct tadb_installation_context *install_ctx)
{
    uint32_t existing_version = existing->prop.version;
    uint32_t new_version = install_ctx->props.version;
    
    DMSG("Version conflict: existing=%u, new=%u", existing_version, new_version);
    
    // 检查版本号
    if (new_version < existing_version) {
        EMSG("Attempting to downgrade TA (existing: %u, new: %u)", 
             existing_version, new_version);
        if (!(install_ctx->flags & TADB_INSTALL_FLAG_FORCE))
            return TEE_ERROR_ACCESS_DENIED;
    }
    
    if (new_version == existing_version) {
        if (!(install_ctx->flags & TADB_INSTALL_FLAG_REPLACE)) {
            EMSG("TA with same version already exists");
            return TEE_ERROR_ACCESS_CONFLICT;
        }
    }
    
    // 检查运行中的实例
    if (ta_is_running(&existing->prop.uuid)) {
        EMSG("Cannot update TA - instances are currently running");
        return TEE_ERROR_BUSY;
    }
    
    // 标记现有 TA 为待删除
    existing->prop.flags |= TADB_FLAG_PENDING_DELETE;
    
    return TEE_SUCCESS;
}
```

### TA 加载和验证

#### 动态加载流程

```c
TEE_Result tadb_load_ta(struct tadb_handle *db, const TEE_UUID *uuid,
                       void **ta_entry_point, void **ta_context)
{
    uint8_t *ta_data = NULL;
    size_t ta_size = 0;
    struct tadb_entry *entry = NULL;
    void *loaded_image = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 查找并加载 TA 数据
    res = tadb_lookup_and_load(db, uuid, &ta_data, &ta_size);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to load TA data from TADB");
        return res;
    }
    
    entry = tadb_find_entry_by_uuid(db, uuid);
    if (!entry) {
        res = TEE_ERROR_ITEM_NOT_FOUND;
        goto cleanup;
    }
    
    // 验证加载时的完整性
    res = tadb_verify_runtime_integrity(entry, ta_data, ta_size);
    if (res != TEE_SUCCESS) {
        EMSG("Runtime integrity verification failed");
        goto cleanup;
    }
    
    // 加载 ELF 镜像到内存
    res = elf_load_ta_image(ta_data, ta_size, &loaded_image);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to load TA ELF image");
        goto cleanup;
    }
    
    // 解析入口点
    res = elf_get_entry_point(loaded_image, ta_entry_point);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to get TA entry point");
        goto cleanup;
    }
    
    // 创建 TA 运行时上下文
    res = ta_create_runtime_context(&entry->prop, loaded_image, ta_context);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to create TA runtime context");
        goto cleanup;
    }
    
    DMSG("TA loaded successfully: UUID=%pUl", uuid);
    
cleanup:
    free(ta_data);
    if (res != TEE_SUCCESS && loaded_image)
        elf_unload_image(loaded_image);
    return res;
}
```

#### 运行时完整性验证

```c
static TEE_Result tadb_verify_runtime_integrity(const struct tadb_entry *entry,
                                               const uint8_t *ta_data, size_t ta_size)
{
    uint8_t computed_hash[TEE_SHA256_HASH_SIZE];
    size_t hash_len = sizeof(computed_hash);
    TEE_Result res = TEE_SUCCESS;
    
    // 计算运行时镜像哈希
    res = crypto_hash_sha256(ta_data, ta_size, computed_hash, &hash_len);
    if (res != TEE_SUCCESS)
        return res;
    
    // 与存储的哈希比较
    if (memcmp(computed_hash, entry->prop.auth_info.image_hash, 
               sizeof(computed_hash)) != 0) {
        EMSG("Runtime TA image hash mismatch - possible tampering");
        return TEE_ERROR_SECURITY;
    }
    
    // 验证 ELF 头部完整性
    res = elf_verify_header_integrity(ta_data, ta_size);
    if (res != TEE_SUCCESS) {
        EMSG("ELF header integrity check failed");
        return res;
    }
    
    // 验证代码段完整性
    res = elf_verify_code_section_integrity(ta_data, ta_size);
    if (res != TEE_SUCCESS) {
        EMSG("Code section integrity check failed");
        return res;
    }
    
    return TEE_SUCCESS;
}
```

## Security Mechanisms

### 硬件根信任

#### 密钥隔离架构

```c
struct tadb_key_hierarchy {
    // 第一层：硬件唯一密钥 (HUK)
    uint8_t huk[HW_UNIQUE_KEY_LENGTH];
    
    // 第二层：TADB 主密钥
    struct {
        uint8_t encryption_key[32];    // AES-256 主加密密钥
        uint8_t integrity_key[32];     // HMAC-SHA256 完整性密钥
        uint8_t derivation_key[32];    // 密钥派生密钥
    } master_keys;
    
    // 第三层：TA 特定密钥
    struct {
        uint8_t ta_encryption_key[32]; // TA 加密密钥
        uint8_t ta_auth_key[32];       // TA 认证密钥
    } ta_specific_keys;
    
    // 密钥版本和时间戳
    uint32_t key_version;
    uint64_t generation_time;
};

static TEE_Result tadb_derive_key_hierarchy(const TEE_UUID *ta_uuid,
                                           struct tadb_key_hierarchy *hierarchy)
{
    TEE_Result res = TEE_SUCCESS;
    const uint8_t master_label[] = "TADB_MASTER_KEY";
    const uint8_t ta_label[] = "TA_SPECIFIC_KEY";
    
    // 获取硬件唯一密钥
    res = tee_otp_get_hw_unique_key((struct tee_hw_unique_key *)hierarchy->huk);
    if (res != TEE_SUCCESS)
        return res;
    
    // 派生 TADB 主密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, master_label, sizeof(master_label) - 1,
                          hierarchy->master_keys.encryption_key, 
                          sizeof(hierarchy->master_keys.encryption_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 派生完整性密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, "TADB_INTEGRITY", 14,
                          hierarchy->master_keys.integrity_key,
                          sizeof(hierarchy->master_keys.integrity_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 派生 TA 特定密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, (const uint8_t *)ta_uuid, sizeof(*ta_uuid),
                          hierarchy->ta_specific_keys.ta_encryption_key,
                          sizeof(hierarchy->ta_specific_keys.ta_encryption_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 设置版本和时间戳
    hierarchy->key_version = TADB_KEY_VERSION_CURRENT;
    hierarchy->generation_time = get_time_stamp();
    
    return TEE_SUCCESS;
}
```

### 命名空间保护

#### TA 隔离机制

```c
struct tadb_namespace {
    TEE_UUID owner_uuid;               // 命名空间所有者
    uint32_t access_flags;             // 访问标志
    uint8_t isolation_key[32];         // 隔离密钥
    struct {
        uint32_t max_files;            // 最大文件数限制
        uint32_t max_storage_size;     // 最大存储大小限制
        uint32_t current_files;        // 当前文件数
        uint32_t current_storage_size; // 当前存储大小
    } quotas;
};

static TEE_Result tadb_create_namespace(const TEE_UUID *ta_uuid,
                                       struct tadb_namespace *ns)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 设置命名空间所有者
    ns->owner_uuid = *ta_uuid;
    
    // 派生隔离密钥
    res = huk_subkey_derive(HUK_SUBKEY_UNIQUE_TA, 
                          (const uint8_t *)ta_uuid, sizeof(*ta_uuid),
                          ns->isolation_key, sizeof(ns->isolation_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 设置默认配额
    ns->quotas.max_files = TADB_DEFAULT_MAX_FILES_PER_TA;
    ns->quotas.max_storage_size = TADB_DEFAULT_MAX_STORAGE_PER_TA;
    ns->quotas.current_files = 0;
    ns->quotas.current_storage_size = 0;
    
    // 设置访问标志
    ns->access_flags = TADB_ACCESS_READ | TADB_ACCESS_WRITE | TADB_ACCESS_DELETE;
    
    return TEE_SUCCESS;
}
```

#### 跨命名空间访问控制

```c
static TEE_Result tadb_check_namespace_access(const struct tadb_namespace *ns,
                                             const TEE_UUID *requesting_ta,
                                             uint32_t requested_access)
{
    // 检查是否为命名空间所有者
    if (memcmp(&ns->owner_uuid, requesting_ta, sizeof(TEE_UUID)) == 0) {
        return TEE_SUCCESS;  // 所有者拥有完全访问权限
    }
    
    // 检查请求的访问权限
    if ((requested_access & ns->access_flags) != requested_access) {
        EMSG("Access denied: requested=0x%x, allowed=0x%x", 
             requested_access, ns->access_flags);
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    // 特殊权限检查 (例如管理权限)
    if (requested_access & TADB_ACCESS_ADMIN) {
        if (!tadb_is_privileged_ta(requesting_ta)) {
            EMSG("Admin access denied for non-privileged TA");
            return TEE_ERROR_ACCESS_DENIED;
        }
    }
    
    return TEE_SUCCESS;
}
```

## Performance Optimization

### 缓存系统

#### 多级缓存架构

```c
#define TADB_L1_CACHE_SIZE 8           // L1 缓存条目数
#define TADB_L2_CACHE_SIZE 32          // L2 缓存条目数

struct tadb_cache_entry {
    TEE_UUID ta_uuid;                  // TA UUID (缓存键)
    uint8_t *ta_data;                  // TA 数据
    size_t ta_size;                    // 数据大小
    uint64_t access_time;              // 最后访问时间
    uint32_t access_count;             // 访问计数
    bool dirty;                        // 脏标志
    bool valid;                        // 有效标志
};

struct tadb_cache_system {
    // L1 缓存 (热点数据)
    struct {
        struct tadb_cache_entry entries[TADB_L1_CACHE_SIZE];
        struct mutex l1_mutex;
        uint64_t hit_count;
        uint64_t miss_count;
    } l1;
    
    // L2 缓存 (温数据)
    struct {
        struct tadb_cache_entry entries[TADB_L2_CACHE_SIZE];
        struct mutex l2_mutex;
        uint64_t hit_count;
        uint64_t miss_count;
    } l2;
    
    uint64_t lru_counter;              // LRU 计数器
};

static struct tadb_cache_system tadb_cache = {
    .l1.l1_mutex = MUTEX_INITIALIZER,
    .l2.l2_mutex = MUTEX_INITIALIZER,
    .lru_counter = 0
};
```

#### 缓存查找和更新

```c
static TEE_Result tadb_cache_lookup(const TEE_UUID *ta_uuid,
                                   uint8_t **ta_data, size_t *ta_size)
{
    struct tadb_cache_entry *entry = NULL;
    
    // L1 缓存查找
    mutex_lock(&tadb_cache.l1.l1_mutex);
    for (size_t i = 0; i < TADB_L1_CACHE_SIZE; i++) {
        entry = &tadb_cache.l1.entries[i];
        if (entry->valid && 
            memcmp(&entry->ta_uuid, ta_uuid, sizeof(TEE_UUID)) == 0) {
            
            // L1 缓存命中
            *ta_data = malloc(entry->ta_size);
            if (*ta_data) {
                memcpy(*ta_data, entry->ta_data, entry->ta_size);
                *ta_size = entry->ta_size;
                entry->access_time = ++tadb_cache.lru_counter;
                entry->access_count++;
                tadb_cache.l1.hit_count++;
                mutex_unlock(&tadb_cache.l1.l1_mutex);
                return TEE_SUCCESS;
            }
        }
    }
    tadb_cache.l1.miss_count++;
    mutex_unlock(&tadb_cache.l1.l1_mutex);
    
    // L2 缓存查找
    mutex_lock(&tadb_cache.l2.l2_mutex);
    for (size_t i = 0; i < TADB_L2_CACHE_SIZE; i++) {
        entry = &tadb_cache.l2.entries[i];
        if (entry->valid && 
            memcmp(&entry->ta_uuid, ta_uuid, sizeof(TEE_UUID)) == 0) {
            
            // L2 缓存命中，提升到 L1
            *ta_data = malloc(entry->ta_size);
            if (*ta_data) {
                memcpy(*ta_data, entry->ta_data, entry->ta_size);
                *ta_size = entry->ta_size;
                entry->access_time = ++tadb_cache.lru_counter;
                entry->access_count++;
                tadb_cache.l2.hit_count++;
                
                // 提升到 L1 缓存
                tadb_cache_promote_to_l1(entry);
                
                mutex_unlock(&tadb_cache.l2.l2_mutex);
                return TEE_SUCCESS;
            }
        }
    }
    tadb_cache.l2.miss_count++;
    mutex_unlock(&tadb_cache.l2.l2_mutex);
    
    return TEE_ERROR_ITEM_NOT_FOUND;
}
```

### 批量操作优化

#### 批量插入接口

```c
struct tadb_batch_insert {
    const struct tee_tadb_property *props;  // TA 属性数组
    const uint8_t **ta_data;                // TA 数据数组
    const size_t *ta_sizes;                 // TA 大小数组
    size_t count;                           // 批量数量
};

TEE_Result tadb_batch_insert(struct tadb_handle *db,
                             const struct tadb_batch_insert *batch)
{
    uint32_t *file_numbers = NULL;
    uint8_t **encrypted_data = NULL;
    size_t *encrypted_sizes = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    if (!db || !batch || batch->count == 0)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 分配临时数组
    file_numbers = calloc(batch->count, sizeof(uint32_t));
    encrypted_data = calloc(batch->count, sizeof(uint8_t *));
    encrypted_sizes = calloc(batch->count, sizeof(size_t));
    
    if (!file_numbers || !encrypted_data || !encrypted_sizes) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto cleanup;
    }
    
    // 开始批量事务
    res = tadb_transaction_begin(db);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 批量预分配文件编号
    for (size_t i = 0; i < batch->count; i++) {
        res = tadb_allocate_file_number(db, &file_numbers[i]);
        if (res != TEE_SUCCESS) {
            tadb_transaction_rollback(db);
            goto cleanup;
        }
    }
    
    // 批量加密数据
    for (size_t i = 0; i < batch->count; i++) {
        uint8_t key[TADB_KEY_SIZE], iv[TADB_IV_SIZE], tag[TADB_TAG_SIZE];
        
        // 生成随机密钥和 IV
        crypto_rng_read(key, sizeof(key));
        crypto_rng_read(iv, sizeof(iv));
        
        res = tadb_encrypt_ta_data(batch->ta_data[i], batch->ta_sizes[i],
                                  key, iv, &encrypted_data[i], 
                                  &encrypted_sizes[i], tag);
        if (res != TEE_SUCCESS) {
            tadb_transaction_rollback(db);
            goto cleanup;
        }
    }
    
    // 批量写入文件
    for (size_t i = 0; i < batch->count; i++) {
        res = tadb_write_ta_file(db, file_numbers[i], 
                                encrypted_data[i], encrypted_sizes[i]);
        if (res != TEE_SUCCESS) {
            tadb_transaction_rollback(db);
            goto cleanup;
        }
    }
    
    // 提交事务
    res = tadb_transaction_commit(db);
    
cleanup:
    free(file_numbers);
    if (encrypted_data) {
        for (size_t i = 0; i < batch->count; i++)
            free(encrypted_data[i]);
        free(encrypted_data);
    }
    free(encrypted_sizes);
    
    return res;
}
```

## Summary

OP-TEE 的 TADB 系统提供了一个全面、安全且高效的 Trusted Application 管理解决方案。该系统的主要特点包括：

### 核心优势

1. **强安全性**: AES-GCM 认证加密 + 数字签名验证 + 硬件根信任
2. **高可靠性**: 事务处理 + 原子操作 + 完整性验证
3. **多后端支持**: 自动选择最佳存储后端
4. **高性能**: 多级缓存 + 批量操作 + 优化的数据结构
5. **完整生命周期**: 安装、验证、加载、更新、删除的完整管理

### 安全特性

1. **密钥隔离**: 基于 HUK 的分层密钥派生
2. **命名空间保护**: TA 间完全隔离
3. **完整性保护**: 多层完整性验证机制
4. **访问控制**: 细粒度权限管理

### 适用场景

- 安全 TA 部署和管理
- 企业级 TA 分发
- 设备固件更新
- 安全应用商店
- IoT 设备管理

TADB 系统是 OP-TEE 安全架构的重要组成部分，为 Trusted Application 的安全管理提供了工业级的解决方案。

## 实际实现分析

### 核心实现文件

基于 OP-TEE 实际代码库分析，TADB 的核心实现分布在以下关键文件中：

- **`optee_os/core/include/tee/tadb.h`**: TADB 核心接口定义
- **`optee_os/core/tee/tadb.c`**: TADB 主要实现逻辑
- **`optee_os/core/include/tee/tee_ta_enc_manager.h`**: TA 加密管理接口
- **`optee_os/core/tee/tee_ta_enc_manager.c`**: TA 加密/解密实现
- **`optee_os/core/include/signed_hdr.h`**: 签名头部结构定义
- **`optee_os/core/crypto/signed_hdr.c`**: 签名验证实现
- **`optee_os/core/kernel/secstor_ta.c`**: 安全存储后端实现

### 实际数据结构定义

#### 1. 实际 TADB 条目结构

```c
/*
 * struct tadb_entry - TA database entry
 * @prop:        properties of TA
 * @file_number: encrypted TA is stored in <file_number>.ta
 * @iv:          Initialization vector of the authentication crypto
 * @tag:         Tag used to validate the authentication encrypted TA
 * @key:         Key used to decrypt the TA
 */
struct tadb_entry {
    struct tee_tadb_property prop;
    uint32_t file_number;
    uint8_t iv[TADB_IV_SIZE];       // 16 bytes (TEE_AES_BLOCK_SIZE)
    uint8_t tag[TADB_TAG_SIZE];     // 16 bytes (TEE_AES_BLOCK_SIZE)
    uint8_t key[TADB_KEY_SIZE];     // 32 bytes (TEE_AES_MAX_KEY_SIZE)
};

// 实际常量定义
#define TADB_MAX_BUFFER_SIZE    (64U * 1024)
#define TADB_AUTH_ENC_ALG       TEE_ALG_AES_GCM
#define TADB_IV_SIZE            TEE_AES_BLOCK_SIZE      // 16 bytes
#define TADB_TAG_SIZE           TEE_AES_BLOCK_SIZE      // 16 bytes  
#define TADB_KEY_SIZE           TEE_AES_MAX_KEY_SIZE    // 32 bytes
```

#### 2. TA 属性结构

```c
/*
 * struct tee_tadb_property
 * @uuid:        UUID of Trusted Application (TA) or Security Domain (SD)
 * @version:     Version of TA or SD
 * @custom_size: Size of customized properties, prepended to the encrypted TA binary
 * @bin_size:    Size of the binary TA
 */
struct tee_tadb_property {
    TEE_UUID uuid;
    uint32_t version;
    uint32_t custom_size;
    uint32_t bin_size;
};
```

#### 3. 签名头部结构

```c
/**
 * struct shdr - signed header
 * @magic:       magic number must match SHDR_MAGIC (0x4f545348)
 * @img_type:    image type (SHDR_TA, SHDR_BOOTSTRAP_TA, SHDR_ENCRYPTED_TA, SHDR_SUBKEY)
 * @img_size:    image size in bytes
 * @algo:        algorithm, defined by public key algorithms TEE_ALG_*
 * @hash_size:   size of the signed hash
 * @sig_size:    size of the signature
 * @hash:        hash of an image
 * @sig:         signature of @hash
 */
struct shdr {
    uint32_t magic;
    uint32_t img_type;
    uint32_t img_size;
    uint32_t algo;
    uint16_t hash_size;
    uint16_t sig_size;
    /* Dynamic parts: hash[hash_size] + sig[sig_size] */
};

/**
 * struct shdr_encrypted_ta - encrypted TA header
 * @enc_algo:    authenticated encryption algorithm (TEE_ALG_AES_GCM)
 * @flags:       encryption flags (device-specific vs class-wide key)
 * @iv_size:     size of the initialization vector
 * @tag_size:    size of the authentication tag
 * @iv:          initialization vector
 * @tag:         authentication tag
 */
struct shdr_encrypted_ta {
    uint32_t enc_algo;
    uint32_t flags;
    uint16_t iv_size;
    uint16_t tag_size;
    /* Dynamic parts: iv[iv_size] + tag[tag_size] */
};
```

### 实际加密实现

#### 1. TA 解密管理器

```c
// 实际加密密钥大小定义
#define TEE_TA_ENC_KEY_SIZE    TEE_SHA256_HASH_SIZE  // 32 bytes

// TA 解密初始化
TEE_Result tee_ta_decrypt_init(void **enc_ctx, struct shdr_encrypted_ta *ehdr,
                               size_t len)
{
    TEE_Result res = TEE_SUCCESS;
    uint8_t key[TEE_TA_ENC_KEY_SIZE] = {0};

    // 分配认证加密上下文
    res = crypto_authenc_alloc_ctx(enc_ctx, ehdr->enc_algo);
    if (res != TEE_SUCCESS)
        return res;

    // 获取 TA 加密密钥 (从 OTP/HUK 派生)
    res = tee_otp_get_ta_enc_key(ehdr->flags & SHDR_ENC_KEY_TYPE_MASK,
                                 key, sizeof(key));
    if (res != TEE_SUCCESS)
        goto out_init;

    // 初始化解密操作
    res = crypto_authenc_init(*enc_ctx, TEE_MODE_DECRYPT, key, sizeof(key),
                              SHDR_ENC_GET_IV(ehdr), ehdr->iv_size,
                              ehdr->tag_size, 0, len);

out_init:
    if (res != TEE_SUCCESS)
        crypto_authenc_free_ctx(*enc_ctx);

    memzero_explicit(key, sizeof(key));  // 安全清零密钥
    return res;
}

// TA 解密更新
TEE_Result tee_ta_decrypt_update(void *enc_ctx, uint8_t *dst, uint8_t *src,
                                 size_t len)
{
    TEE_Result res = TEE_SUCCESS;
    size_t dlen = len;

    res = crypto_authenc_update_payload(enc_ctx, TEE_MODE_DECRYPT, src, len,
                                        dst, &dlen);
    if (res != TEE_SUCCESS)
        crypto_authenc_free_ctx(enc_ctx);

    return res;
}

// TA 解密完成和认证
TEE_Result tee_ta_decrypt_final(void *enc_ctx, struct shdr_encrypted_ta *ehdr,
                                uint8_t *dst, uint8_t *src, size_t len)
{
    TEE_Result res = TEE_SUCCESS;
    size_t dlen = len;

    // 执行解密并验证认证标签
    res = crypto_authenc_dec_final(enc_ctx, src, len, dst, &dlen,
                                   SHDR_ENC_GET_TAG(ehdr), ehdr->tag_size);

    crypto_authenc_free_ctx(enc_ctx);
    return res;
}
```

#### 2. 密钥类型和派生

```c
// 加密密钥类型
enum shdr_enc_key_type {
    SHDR_ENC_KEY_DEV_SPECIFIC = 0,    // 设备特定密钥
    SHDR_ENC_KEY_CLASS_WIDE = 1,      // 类别通用密钥
};

#define SHDR_ENC_KEY_TYPE_MASK    0x1

// 密钥派生通过 tee_otp_get_ta_enc_key() 实现
// 该函数基于 HUK (Hardware Unique Key) 派生出 TA 特定的加密密钥
```

### 实际存储后端实现

#### 1. 安全存储 TA 后端

```c
static TEE_Result secstor_ta_open(const TEE_UUID *uuid,
                                  struct ts_store_handle **handle)
{
    TEE_Result res;
    struct tee_tadb_ta_read *ta;
    size_t l;
    const struct tee_tadb_property *prop;

    // 通过 TADB 打开 TA
    res = tee_tadb_ta_open(uuid, &ta);
    if (res)
        return res;
    prop = tee_tadb_ta_get_property(ta);

    // 验证自定义数据大小
    l = prop->custom_size;
    res = tee_tadb_ta_read(ta, NULL, NULL, &l);
    if (res)
        goto err;
    if (l != prop->custom_size) {
        res = TEE_ERROR_CORRUPT_OBJECT;
        goto err;
    }

    *handle = (struct ts_store_handle *)ta;
    return TEE_SUCCESS;
err:
    tee_tadb_ta_close(ta);
    return res;
}

static TEE_Result secstor_ta_get_size(const struct ts_store_handle *h,
                                      size_t *size)
{
    struct tee_tadb_ta_read *ta = (struct tee_tadb_ta_read *)h;
    const struct tee_tadb_property *prop = tee_tadb_ta_get_property(ta);

    *size = prop->bin_size;
    return TEE_SUCCESS;
}

static TEE_Result secstor_ta_get_tag(const struct ts_store_handle *h,
                                     uint8_t *tag, unsigned int *tag_len)
{
    return tee_tadb_get_tag((struct tee_tadb_ta_read *)h, tag, tag_len);
}
```

### 实际 TADB 目录管理

#### 1. TADB 目录结构

```c
struct tee_tadb_dir {
    const struct tee_file_operations *ops;    // 文件操作接口
    struct tee_file_handle *fh;              // 文件句柄
    int nbits;                               // 位图位数
    bitstr_t *files;                         // 文件分配位图
};

// TADB 数据库文件名
static const char tadb_obj_id[] = "ta.db";
static struct tee_tadb_dir *tadb_db;
static unsigned int tadb_db_refc;           // 引用计数
static struct mutex tadb_mutex = MUTEX_INITIALIZER;
```

#### 2. 文件编号管理

```c
static void file_num_to_str(char *buf, size_t blen, uint32_t file_number)
{
    int rc __maybe_unused = 0;
    
    // 文件名格式：<file_number>.ta
    rc = snprintf(buf, blen, "%" PRIu32 ".ta", file_number);
    assert(rc >= 0);
}

static bool is_null_uuid(const TEE_UUID *uuid)
{
    const TEE_UUID null_uuid = { 0 };
    
    return !memcmp(uuid, &null_uuid, sizeof(*uuid));
}
```

### 实际读写操作结构

#### 1. TA 写入操作结构

```c
struct tee_tadb_ta_write {
    struct tee_tadb_dir *db;        // 数据库引用
    int fd;                         // 文件描述符
    struct tadb_entry entry;       // TADB 条目
    size_t pos;                     // 当前位置
    void *ctx;                      // 上下文
};
```

#### 2. TA 读取操作结构

```c
struct tee_tadb_ta_read {
    struct tee_tadb_dir *db;        // 数据库引用
    int fd;                         // 文件描述符
    struct tadb_entry entry;       // TADB 条目
    size_t pos;                     // 当前位置
    void *ctx;                      // 上下文
    struct mobj *ta_mobj;           // 内存对象
    uint8_t *ta_buf;                // TA 缓冲区
};
```

### 核心 API 接口

#### 1. TA 创建和写入

```c
// 创建新的 TA 写入句柄
TEE_Result tee_tadb_ta_create(const struct tee_tadb_property *property,
                              struct tee_tadb_ta_write **ta);

// 写入 TA 数据
TEE_Result tee_tadb_ta_write(struct tee_tadb_ta_write *ta, const void *buf,
                             size_t len);

// 关闭并删除 TA
void tee_tadb_ta_close_and_delete(struct tee_tadb_ta_write *ta);

// 关闭并提交 TA
TEE_Result tee_tadb_ta_close_and_commit(struct tee_tadb_ta_write *ta);
```

#### 2. TA 删除和读取

```c
// 删除指定 UUID 的 TA
TEE_Result tee_tadb_ta_delete(const TEE_UUID *uuid);

// 打开 TA 进行读取
TEE_Result tee_tadb_ta_open(const TEE_UUID *uuid, struct tee_tadb_ta_read **ta);

// 获取 TA 属性
const struct tee_tadb_property *
tee_tadb_ta_get_property(struct tee_tadb_ta_read *ta);

// 获取认证标签
TEE_Result tee_tadb_get_tag(struct tee_tadb_ta_read *ta, uint8_t *tag,
                            unsigned int *tag_len);

// 读取 TA 数据
TEE_Result tee_tadb_ta_read(struct tee_tadb_ta_read *ta, void *buf_core,
                            void *buf_user, size_t *len);

// 关闭 TA 读取句柄
void tee_tadb_ta_close(struct tee_tadb_ta_read *ta);
```

### 实际安全特性

#### 1. 密钥隔离
- 使用 **HUK (Hardware Unique Key)** 作为根密钥
- 支持 **设备特定** 和 **类别通用** 两种密钥类型
- 通过 `tee_otp_get_ta_enc_key()` 实现密钥派生

#### 2. 认证加密
- 使用 **AES-GCM** 算法提供认证加密
- **16 字节 IV** 和 **16 字节认证标签**
- **32 字节 AES-256 密钥**

#### 3. 完整性保护
- 数字签名验证 (支持 RSA、ECDSA 等)
- 认证标签验证防止篡改
- 版本回滚保护

#### 4. 存储隔离
- 每个 TA 使用唯一的文件编号存储
- 位图管理文件分配
- 支持多种存储后端 (安全存储、REE 文件系统)

这个实际实现展示了 OP-TEE TADB 系统的工业级安全设计，通过硬件根信任、多层加密、完整性验证和访问控制，为 Trusted Application 提供了强大的安全保护机制。

## 存储架构设计深度分析

### 分层存储架构

#### 1. 存储后端层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    GlobalPlatform TEE API Layer                │
│  TEE_OpenPersistentObject() • TEE_CreatePersistentObject()     │
├─────────────────────────────────────────────────────────────────┤
│                     TEE Service Layer                          │
│    tee_svc_storage.c • tee_svc_storage_file_ops()             │
├─────────────────────────────────────────────────────────────────┤
│                   File System Abstraction                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │    TADB FS      │  │    REE FS       │  │   RPMB FS       │ │
│  │ • Encrypted TA  │  │ • Normal World  │  │ • Hardware      │ │
│  │ • Database      │  │ • High Speed    │  │ • Secure        │ │
│  │ • Priority 4    │  │ • Software Enc  │  │ • Replay Prot   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                    Storage Device Layer                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Memory Objects  │  │   RPC Interface │  │  eMMC RPMB      │ │
│  │ • MOBJ System   │  │ • Thread Local  │  │ • 256B blocks   │ │
│  │ • Buffer Pool   │  │ • Shared Memory │  │ • Authenticated │ │
│  │ • Reference Cnt │  │ • Cache Layer   │  │ • Counter Based │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. 存储后端选择机制

```c
// 实际的存储后端选择逻辑
static const struct tee_file_operations *
tee_svc_storage_file_ops(uint32_t storage_id)
{
    switch (storage_id) {
    case TEE_STORAGE_PRIVATE:
        // 自动选择最佳后端：REE FS → RPMB FS
        return tee_storage_get_ops();
        
    case TEE_STORAGE_PRIVATE_REE:
        // 强制使用 REE 文件系统
        return &ree_fs_ops;
        
    case TEE_STORAGE_PRIVATE_RPMB:
        // 强制使用 RPMB 安全存储
        return &rpmb_fs_ops;
        
    default:
        return NULL;
    }
}

// 动态后端选择算法
const struct tee_file_operations *tee_storage_get_ops(void)
{
    // 优先级顺序：性能 vs 安全性平衡
    if (IS_ENABLED(CFG_REE_FS) && ree_fs_available())
        return &ree_fs_ops;       // 高性能选择
    
    if (IS_ENABLED(CFG_RPMB_FS) && rpmb_fs_available())
        return &rpmb_fs_ops;      // 高安全选择
    
    return NULL;  // 无可用后端
}
```

### TADB 存储子系统设计

#### 1. TADB 目录管理架构

```c
// TADB 目录结构实现
struct tee_tadb_dir {
    const struct tee_file_operations *ops;    // 文件操作接口指针
    struct tee_file_handle *fh;              // 主数据库文件句柄
    int nbits;                               // 文件位图位数
    bitstr_t *files;                         // 文件分配位图
    
    // 扩展字段（推断）
    struct {
        uint32_t max_file_number;            // 最大文件编号
        uint32_t allocated_count;            // 已分配文件数
        uint32_t free_hint;                  // 分配提示位置
    } allocation_info;
    
    struct {
        uint64_t total_size;                 // 总存储大小
        uint64_t used_size;                  // 已使用大小
        uint32_t entry_count;                // 条目数量
    } statistics;
};

// TADB 全局管理状态
static const char tadb_obj_id[] = "ta.db";        // 数据库文件名
static struct tee_tadb_dir *tadb_db;               // 全局数据库实例
static unsigned int tadb_db_refc;                 // 引用计数
static struct mutex tadb_mutex = MUTEX_INITIALIZER; // 全局互斥锁
```

#### 2. 文件分配和管理策略

```c
// 文件编号分配算法
static TEE_Result tadb_allocate_file_number(struct tee_tadb_dir *db, uint32_t *file_num)
{
    int free_bit = -1;
    
    // 在位图中查找空闲位
    bit_ffc(db->files, db->nbits, &free_bit);
    if (free_bit < 0) {
        // 扩展位图大小
        TEE_Result res = maybe_grow_files(db);
        if (res != TEE_SUCCESS)
            return res;
        
        bit_ffc(db->files, db->nbits, &free_bit);
        if (free_bit < 0)
            return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    // 标记位为已使用
    bit_set(db->files, free_bit);
    *file_num = free_bit;
    
    return TEE_SUCCESS;
}

// 动态位图扩展
static TEE_Result maybe_grow_files(struct tee_tadb_dir *db)
{
    int new_nbits = db->nbits * 2;  // 双倍扩展
    bitstr_t *new_files = calloc(1, bitstr_size(new_nbits));
    
    if (!new_files)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 复制现有位图
    memcpy(new_files, db->files, bitstr_size(db->nbits));
    
    free(db->files);
    db->files = new_files;
    db->nbits = new_nbits;
    
    return TEE_SUCCESS;
}

// 文件名生成策略
static void file_num_to_str(char *buf, size_t blen, uint32_t file_number)
{
    // 文件命名格式：<file_number>.ta
    snprintf(buf, blen, "%" PRIu32 ".ta", file_number);
}
```

### 存储设备驱动架构

#### 1. REE 文件系统实现

```c
// REE FS 配置参数
#define REE_FS_BLOCK_SHIFT        12          // 4KB 块大小
#define REE_FS_BLOCK_SIZE         (1 << REE_FS_BLOCK_SHIFT)
#define REE_FS_MAX_FILE_SIZE      0x40000000  // 1GB 最大文件

// REE FS 操作结构
struct ree_fs_file {
    struct tee_file_handle base;
    int fd;                                   // 文件描述符
    bool dirty;                               // 脏标志
    uint8_t *tmp_block;                       // 临时块缓冲区
    size_t tmp_block_size;                    // 临时块大小
};

// REE FS 核心操作
static TEE_Result ree_fs_open(struct tee_pobj *po, size_t *size,
                              struct tee_file_handle **fh)
{
    struct ree_fs_file *f = calloc(1, sizeof(*f));
    if (!f)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 打开文件操作通过 RPC 调用
    TEE_Result res = tee_fs_rpc_open(OPTEE_RPC_CMD_FS_OPEN, po, &f->fd);
    if (res != TEE_SUCCESS) {
        free(f);
        return res;
    }
    
    *fh = &f->base;
    return TEE_SUCCESS;
}

// 临时块管理
static uint8_t *get_tmp_block(struct ree_fs_file *f)
{
    if (!f->tmp_block) {
        f->tmp_block = malloc(REE_FS_BLOCK_SIZE);
        f->tmp_block_size = REE_FS_BLOCK_SIZE;
    }
    return f->tmp_block;
}

static void put_tmp_block(struct ree_fs_file *f, uint8_t *block)
{
    if (block == f->tmp_block) {
        // 保留缓冲区以供重用
        return;
    }
    free(block);
}
```

#### 2. RPMB 存储实现

```c
// RPMB 配置参数
#define RPMB_BLOCK_SIZE_SHIFT     8           // 256 字节块
#define RPMB_BLOCK_SIZE           (1 << RPMB_BLOCK_SIZE_SHIFT)
#define RPMB_MAX_ENTRIES          256         // 最大条目数

// RPMB 条目结构
struct rpmb_fs_entry {
    uint32_t start_block;                     // 起始块号
    uint32_t num_blocks;                      // 块数量
    uint32_t write_counter;                   // 写计数器
    uint8_t fek[TEE_FS_KM_FEK_SIZE];         // 文件加密密钥
    char filename[TEE_FS_NAME_MAX];           // 文件名
    uint8_t file_cbc_cts_ctx[TEE_SHA256_HASH_SIZE];  // CBC-CTS 上下文
};

// RPMB 目录结构
struct rpmb_fs_dir {
    struct rpmb_fs_entry entries[RPMB_MAX_ENTRIES];
    uint32_t num_entries;                     // 当前条目数
    uint32_t write_counter;                   // 全局写计数器
    uint8_t hash[TEE_SHA256_HASH_SIZE];      // 目录哈希
};

// RPMB 原子写操作
static TEE_Result rpmb_fs_write_blocks(uint32_t start_block,
                                       const uint8_t *data, uint32_t num_blocks)
{
    struct rpmb_req req = {
        .cmd = RPMB_CMD_WRITE_DATA,
        .dev_id = 0,
        .block_count = num_blocks,
        .addr = start_block,
        .data = (void *)data,
    };
    
    // 硬件认证写操作
    return tee_rpmb_write(&req);
}
```

### 内存管理和缓冲策略

#### 1. 内存对象 (MOBJ) 系统

```c
// 内存对象类型定义
enum mobj_type {
    MOBJ_TYPE_PHYS,                          // 物理内存映射
    MOBJ_TYPE_REG_SHM,                       // 注册共享内存
    MOBJ_TYPE_FFA,                           // FFA 共享内存
    MOBJ_TYPE_DYN_SHM,                       // 动态共享内存
    MOBJ_TYPE_SECSTOR,                       // 安全存储内存
};

// 内存对象基础结构
struct mobj {
    enum mobj_type type;                     // 对象类型
    const struct mobj_ops *ops;              // 操作接口
    size_t size;                             // 大小
    uint32_t refc;                           // 引用计数
    bool shared;                             // 共享标志
};

// 内存对象操作接口
struct mobj_ops {
    void *(*get_va)(struct mobj *mobj, size_t offset);
    TEE_Result (*get_pa)(struct mobj *mobj, size_t offs, size_t granule, paddr_t *pa);
    bool (*matches)(struct mobj *mobj, enum buf_is_attr attr);
    void (*free)(struct mobj *mobj);
    uint64_t (*get_cookie)(struct mobj *mobj);
};

// TADB 使用的内存对象分配
static struct mobj *tadb_alloc_mobj(size_t size)
{
    struct mobj *mobj = NULL;
    
    // 优先使用安全内存
    mobj = mobj_sec_ddr_alloc(size);
    if (mobj)
        return mobj;
    
    // 回退到普通内存
    return mobj_phys_alloc(size, 0, 0);
}
```

#### 2. 缓冲区管理策略

```c
// TADB 缓冲区管理
#define TADB_MAX_BUFFER_SIZE      (64U * 1024)   // 64KB 最大缓冲区

struct tadb_buffer_pool {
    uint8_t *buffers[8];                     // 缓冲区池
    size_t buffer_sizes[8];                  // 对应大小
    bool in_use[8];                          // 使用标志
    struct mutex pool_mutex;                 // 池互斥锁
};

// 缓冲区分配算法
static uint8_t *tadb_get_buffer(size_t size)
{
    if (size > TADB_MAX_BUFFER_SIZE)
        return NULL;
    
    // 查找合适的缓冲区
    for (int i = 0; i < 8; i++) {
        if (!pool.in_use[i] && pool.buffer_sizes[i] >= size) {
            pool.in_use[i] = true;
            return pool.buffers[i];
        }
    }
    
    // 分配新缓冲区
    return malloc(size);
}

// RPC 缓冲区管理
struct thread_rpc_cache {
    uint64_t cookie;                         // 缓冲区标识
    void *va;                                // 虚拟地址
    paddr_t pa;                              // 物理地址
    size_t size;                             // 大小
    bool allocated;                          // 分配标志
};

// 线程局部 RPC 缓存
static struct thread_rpc_cache *get_rpc_cache(void)
{
    return &thread_get_tsd()->rpc_cache;
}
```

### 事务和原子性机制

#### 1. TADB 事务管理

```c
// TADB 事务状态
enum tadb_transaction_state {
    TADB_TXN_IDLE,                           // 空闲状态
    TADB_TXN_WRITING,                        // 写入状态
    TADB_TXN_COMMITTING,                     // 提交状态
    TADB_TXN_ABORTING,                       // 中止状态
};

// TADB 写操作管理
struct tee_tadb_ta_write {
    struct tee_tadb_dir *db;                 // 数据库引用
    int fd;                                  // 文件描述符
    struct tadb_entry entry;                // 条目信息
    size_t pos;                              // 当前位置
    void *ctx;                               // 加密上下文
    enum tadb_transaction_state state;      // 事务状态
    
    // 事务相关
    uint32_t allocated_file_num;             // 分配的文件号
    bool entry_committed;                    // 条目是否已提交
    uint8_t *temp_buffer;                    // 临时缓冲区
    size_t temp_buffer_size;                 // 临时缓冲区大小
};

// 原子提交操作
TEE_Result tee_tadb_ta_close_and_commit(struct tee_tadb_ta_write *ta)
{
    TEE_Result res = TEE_SUCCESS;
    
    ta->state = TADB_TXN_COMMITTING;
    
    // 1. 完成加密并获取最终标签
    res = tadb_finalize_encryption(ta);
    if (res != TEE_SUCCESS)
        goto abort;
    
    // 2. 将条目写入数据库文件
    res = tadb_write_entry_to_db(ta->db, &ta->entry);
    if (res != TEE_SUCCESS)
        goto abort;
    
    // 3. 同步文件系统
    res = tadb_sync_database(ta->db);
    if (res != TEE_SUCCESS)
        goto abort;
    
    // 4. 提交完成，清理资源
    ta->entry_committed = true;
    ta->state = TADB_TXN_IDLE;
    
    return TEE_SUCCESS;
    
abort:
    ta->state = TADB_TXN_ABORTING;
    tee_tadb_ta_close_and_delete(ta);
    return res;
}
```

#### 2. 层次化事务支持

```c
// 文件系统层事务支持
struct fs_htree_meta {
    uint64_t counter;                        // 版本计数器
    struct tee_fs_htree_node_image node;    // 节点镜像
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];   // 节点哈希
    bool dirty;                              // 脏标志
    bool committed;                          // 提交标志
};

// 原子同步操作
TEE_Result tee_fs_htree_sync_to_storage(struct tee_fs_htree_meta **meta)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 1. 计算新的版本计数器
    uint64_t new_counter = (*meta)->counter + 1;
    
    // 2. 更新所有脏节点
    res = update_dirty_nodes(*meta, new_counter);
    if (res != TEE_SUCCESS)
        return res;
    
    // 3. 原子更新根节点
    res = atomic_update_root_node(*meta, new_counter);
    if (res != TEE_SUCCESS)
        return res;
    
    // 4. 标记为已提交
    (*meta)->committed = true;
    (*meta)->counter = new_counter;
    
    return TEE_SUCCESS;
}
```

### 存储配额和资源管理

#### 1. 动态资源分配

```c
// 存储配额管理
struct tadb_quota_manager {
    uint64_t max_total_size;                 // 最大总大小
    uint64_t current_total_size;             // 当前总大小
    uint32_t max_file_count;                 // 最大文件数
    uint32_t current_file_count;             // 当前文件数
    
    // 每 TA 配额
    struct {
        uint32_t max_files_per_ta;           // 每 TA 最大文件数
        uint64_t max_size_per_ta;            // 每 TA 最大大小
    } per_ta_limits;
    
    struct mutex quota_mutex;                // 配额互斥锁
};

// 配额检查函数
static TEE_Result check_quota_before_allocation(struct tadb_quota_manager *qm,
                                               const TEE_UUID *ta_uuid,
                                               size_t requested_size)
{
    mutex_lock(&qm->quota_mutex);
    
    // 检查全局限制
    if (qm->current_total_size + requested_size > qm->max_total_size ||
        qm->current_file_count >= qm->max_file_count) {
        mutex_unlock(&qm->quota_mutex);
        return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    // 检查 TA 特定限制
    uint32_t ta_file_count = get_ta_file_count(ta_uuid);
    uint64_t ta_total_size = get_ta_total_size(ta_uuid);
    
    if (ta_file_count >= qm->per_ta_limits.max_files_per_ta ||
        ta_total_size + requested_size > qm->per_ta_limits.max_size_per_ta) {
        mutex_unlock(&qm->quota_mutex);
        return TEE_ERROR_STORAGE_NO_SPACE;
    }
    
    mutex_unlock(&qm->quota_mutex);
    return TEE_SUCCESS;
}
```

#### 2. 垃圾回收机制

```c
// TADB 垃圾回收
struct tadb_gc_context {
    struct tee_tadb_dir *db;                 // 数据库引用
    uint32_t *dead_files;                    // 待删除文件列表
    size_t num_dead_files;                   // 待删除文件数
    uint64_t reclaimed_space;                // 回收空间
    bool gc_in_progress;                     // GC 进行中标志
};

// 垃圾回收算法
static TEE_Result tadb_garbage_collect(struct tee_tadb_dir *db)
{
    struct tadb_gc_context gc = { .db = db };
    TEE_Result res = TEE_SUCCESS;
    
    gc.gc_in_progress = true;
    
    // 1. 扫描数据库，标记无效条目
    res = mark_invalid_entries(db, &gc);
    if (res != TEE_SUCCESS)
        goto cleanup;
    
    // 2. 删除无效文件
    for (size_t i = 0; i < gc.num_dead_files; i++) {
        res = delete_ta_file(db, gc.dead_files[i]);
        if (res != TEE_SUCCESS)
            continue;  // 继续删除其他文件
        
        // 释放文件编号
        bit_clear(db->files, gc.dead_files[i]);
    }
    
    // 3. 压缩数据库
    res = compact_database(db);
    
cleanup:
    free(gc.dead_files);
    gc.gc_in_progress = false;
    return res;
}
```

### 存储同步和一致性

#### 1. 并发控制机制

```c
// 读写锁实现
struct tadb_rwlock {
    struct mutex mutex;                      // 基础互斥锁
    struct condvar readers_cv;               // 读者条件变量
    struct condvar writers_cv;               // 写者条件变量
    int active_readers;                      // 活跃读者数
    bool active_writer;                      // 活跃写者标志
    int waiting_writers;                     // 等待写者数
};

// 读锁获取
static void tadb_read_lock(struct tadb_rwlock *rwlock)
{
    mutex_lock(&rwlock->mutex);
    
    // 等待写者完成
    while (rwlock->active_writer || rwlock->waiting_writers > 0) {
        condvar_wait(&rwlock->readers_cv, &rwlock->mutex);
    }
    
    rwlock->active_readers++;
    mutex_unlock(&rwlock->mutex);
}

// 写锁获取
static void tadb_write_lock(struct tadb_rwlock *rwlock)
{
    mutex_lock(&rwlock->mutex);
    rwlock->waiting_writers++;
    
    // 等待所有读者和写者完成
    while (rwlock->active_readers > 0 || rwlock->active_writer) {
        condvar_wait(&rwlock->writers_cv, &rwlock->mutex);
    }
    
    rwlock->waiting_writers--;
    rwlock->active_writer = true;
    mutex_unlock(&rwlock->mutex);
}
```

#### 2. 版本一致性管理

```c
// 版本化数据结构
struct versioned_tadb_entry {
    struct tadb_entry entry;                // 实际条目
    uint64_t version;                        // 版本号
    uint64_t timestamp;                      // 时间戳
    enum entry_state {
        ENTRY_VALID,                         // 有效
        ENTRY_UPDATING,                      // 更新中
        ENTRY_DELETED,                       // 已删除
    } state;
    struct versioned_tadb_entry *next_version; // 下个版本链
};

// 版本选择算法
static struct tadb_entry *get_consistent_entry(struct versioned_tadb_entry *v_entry,
                                               uint64_t read_timestamp)
{
    struct versioned_tadb_entry *current = v_entry;
    
    // 找到合适的版本
    while (current) {
        if (current->timestamp <= read_timestamp && 
            current->state == ENTRY_VALID) {
            return &current->entry;
        }
        current = current->next_version;
    }
    
    return NULL;  // 无有效版本
}
```

### 存储错误处理和恢复

#### 1. 错误分类和处理

```c
// 存储错误类型
enum tadb_error_type {
    TADB_ERROR_CORRUPTION,                   // 数据损坏
    TADB_ERROR_IO_FAILURE,                   // I/O 失败
    TADB_ERROR_AUTH_FAILURE,                 // 认证失败
    TADB_ERROR_QUOTA_EXCEEDED,               // 配额超限
    TADB_ERROR_INCONSISTENCY,                // 数据不一致
};

// 错误恢复策略
struct tadb_recovery_policy {
    enum tadb_error_type error_type;
    bool auto_recover;                       // 自动恢复
    bool notify_user;                        // 通知用户
    bool quarantine_data;                    // 隔离数据
    TEE_Result (*recovery_handler)(void *ctx); // 恢复处理器
};

// 损坏检测和恢复
static TEE_Result handle_corruption_error(struct tee_tadb_dir *db,
                                          uint32_t file_number)
{
    // 1. 隔离损坏的文件
    TEE_Result res = quarantine_file(db, file_number);
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 尝试从备份恢复
    res = restore_from_backup(db, file_number);
    if (res == TEE_SUCCESS)
        return res;
    
    // 3. 删除损坏的条目
    res = remove_corrupted_entry(db, file_number);
    if (res != TEE_SUCCESS)
        return res;
    
    // 4. 记录错误日志
    log_corruption_event(file_number);
    
    return TEE_SUCCESS;
}
```

#### 2. 数据完整性验证

```c
// 完整性检查框架
struct integrity_checker {
    bool (*check_header)(const void *header, size_t size);
    bool (*check_entry)(const struct tadb_entry *entry);
    bool (*check_file_consistency)(struct tee_tadb_dir *db, uint32_t file_num);
    TEE_Result (*repair_corruption)(struct tee_tadb_dir *db, uint32_t file_num);
};

// 数据库完整性验证
static TEE_Result verify_database_integrity(struct tee_tadb_dir *db)
{
    TEE_Result res = TEE_SUCCESS;
    uint32_t corrupted_files[16];
    size_t num_corrupted = 0;
    
    // 遍历所有分配的文件
    for (int file_num = 0; file_num < db->nbits; file_num++) {
        if (!bit_test(db->files, file_num))
            continue;
        
        // 检查文件完整性
        if (!check_file_integrity(db, file_num)) {
            if (num_corrupted < ARRAY_SIZE(corrupted_files)) {
                corrupted_files[num_corrupted++] = file_num;
            }
        }
    }
    
    // 处理发现的损坏
    for (size_t i = 0; i < num_corrupted; i++) {
        res = handle_corruption_error(db, corrupted_files[i]);
        if (res != TEE_SUCCESS)
            EMSG("Failed to handle corruption in file %u", corrupted_files[i]);
    }
    
    return (num_corrupted == 0) ? TEE_SUCCESS : TEE_ERROR_CORRUPT_OBJECT;
}
```

### 存储性能优化

#### 1. 缓存和预取策略

```c
// TADB 缓存系统
#define TADB_CACHE_SIZE           32          // 缓存条目数

struct tadb_cache_entry {
    TEE_UUID uuid;                           // TA UUID
    struct tadb_entry entry;                // 缓存条目
    uint64_t access_time;                    // 访问时间
    uint32_t access_count;                   // 访问计数
    bool dirty;                              // 脏标志
    bool valid;                              // 有效标志
};

struct tadb_cache {
    struct tadb_cache_entry entries[TADB_CACHE_SIZE];
    struct mutex cache_mutex;               // 缓存互斥锁
    uint64_t lru_counter;                   // LRU 计数器
    
    // 统计信息
    uint64_t hit_count;                     // 命中次数
    uint64_t miss_count;                    // 未命中次数
    uint64_t eviction_count;                // 驱逐次数
};

// LRU 缓存替换算法
static int find_lru_entry(struct tadb_cache *cache)
{
    int lru_index = 0;
    uint64_t min_access_time = cache->entries[0].access_time;
    
    for (int i = 1; i < TADB_CACHE_SIZE; i++) {
        if (!cache->entries[i].valid) {
            return i;  // 找到无效条目
        }
        
        if (cache->entries[i].access_time < min_access_time) {
            min_access_time = cache->entries[i].access_time;
            lru_index = i;
        }
    }
    
    return lru_index;
}

// 预取策略
static TEE_Result prefetch_related_entries(struct tee_tadb_dir *db,
                                          const TEE_UUID *uuid)
{
    // 基于访问模式的预取
    // 实现策略：预取相邻文件编号的条目
    
    struct tadb_entry *entry = find_cached_entry(uuid);
    if (!entry)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    uint32_t base_file_num = entry->file_number;
    
    // 预取前后各2个文件
    for (int i = -2; i <= 2; i++) {
        if (i == 0) continue;  // 跳过当前文件
        
        uint32_t target_file = base_file_num + i;
        if (target_file < db->nbits && bit_test(db->files, target_file)) {
            // 异步预取
            prefetch_file_async(db, target_file);
        }
    }
    
    return TEE_SUCCESS;
}
```

#### 2. I/O 优化策略

```c
// 批量 I/O 操作
struct tadb_batch_io {
    struct {
        uint32_t file_number;
        void *buffer;
        size_t size;
        bool is_write;
    } operations[16];
    size_t count;
    struct mutex batch_mutex;
};

// 批量读操作
static TEE_Result tadb_batch_read(struct tee_tadb_dir *db,
                                 struct tadb_batch_io *batch)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 按文件编号排序以优化访问模式
    sort_operations_by_file_number(batch);
    
    for (size_t i = 0; i < batch->count; i++) {
        if (!batch->operations[i].is_write) {
            res = read_ta_file(db, batch->operations[i].file_number,
                              batch->operations[i].buffer,
                              batch->operations[i].size);
            if (res != TEE_SUCCESS)
                break;
        }
    }
    
    return res;
}

// 写入合并优化
static TEE_Result optimize_write_pattern(struct tee_tadb_dir *db)
{
    // 合并小的写操作以减少I/O次数
    // 延迟写入以实现批量提交
    
    if (db->pending_writes_count > WRITE_BATCH_THRESHOLD ||
        time_since_last_sync() > MAX_SYNC_DELAY) {
        return flush_pending_writes(db);
    }
    
    return TEE_SUCCESS;
}
```

### 存储安全隔离

#### 1. TA 间完全隔离

```c
// TA 命名空间隔离
struct ta_storage_namespace {
    TEE_UUID owner_uuid;                     // 所有者 UUID
    uint8_t namespace_key[32];               // 命名空间密钥
    
    // 访问控制
    struct {
        bool allow_read;
        bool allow_write;
        bool allow_delete;
        bool allow_admin;
    } permissions;
    
    // 资源限制
    struct {
        uint64_t max_storage_size;
        uint32_t max_file_count;
        uint32_t max_concurrent_handles;
    } limits;
    
    // 统计信息
    struct {
        uint64_t current_storage_size;
        uint32_t current_file_count;
        uint32_t active_handles;
        uint64_t total_operations;
    } stats;
};

// 命名空间密钥派生
static TEE_Result derive_namespace_key(const TEE_UUID *ta_uuid,
                                      uint8_t *namespace_key)
{
    const uint8_t label[] = "TA_NAMESPACE_KEY";
    
    // 使用 HUK 派生 TA 特定的命名空间密钥
    return huk_subkey_derive(HUK_SUBKEY_TA_ENC,
                           (const uint8_t *)ta_uuid, sizeof(*ta_uuid),
                           namespace_key, 32);
}

// 访问权限检查
static TEE_Result check_namespace_access(const struct ta_storage_namespace *ns,
                                        const TEE_UUID *requesting_ta,
                                        uint32_t operation_type)
{
    // 1. 检查是否为命名空间所有者
    if (memcmp(&ns->owner_uuid, requesting_ta, sizeof(TEE_UUID)) == 0) {
        return TEE_SUCCESS;  // 所有者拥有完全权限
    }
    
    // 2. 检查特定操作权限
    switch (operation_type) {
    case TADB_OP_READ:
        if (!ns->permissions.allow_read)
            return TEE_ERROR_ACCESS_DENIED;
        break;
        
    case TADB_OP_WRITE:
        if (!ns->permissions.allow_write)
            return TEE_ERROR_ACCESS_DENIED;
        break;
        
    case TADB_OP_DELETE:
        if (!ns->permissions.allow_delete)
            return TEE_ERROR_ACCESS_DENIED;
        break;
        
    case TADB_OP_ADMIN:
        if (!ns->permissions.allow_admin)
            return TEE_ERROR_ACCESS_DENIED;
        break;
        
    default:
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    return TEE_SUCCESS;
}
```

#### 2. 多层密钥隔离

```c
// 分层密钥管理
struct tadb_key_hierarchy {
    // 第1层：硬件根密钥 (HUK)
    uint8_t huk[HW_UNIQUE_KEY_LENGTH];
    
    // 第2层：TADB 主密钥
    struct {
        uint8_t master_enc_key[32];          // 主加密密钥
        uint8_t master_auth_key[32];         // 主认证密钥
        uint8_t master_kdf_key[32];          // 主密钥派生密钥
    } master_keys;
    
    // 第3层：TA 特定密钥
    struct {
        uint8_t ta_storage_key[32];          // TA 存储密钥
        uint8_t ta_namespace_key[32];        // TA 命名空间密钥
        uint8_t ta_integrity_key[32];        // TA 完整性密钥
    } ta_keys;
    
    // 第4层：文件特定密钥
    struct {
        uint8_t file_enc_key[32];            // 文件加密密钥
        uint8_t file_auth_key[32];           // 文件认证密钥
    } file_keys;
    
    // 密钥元数据
    uint32_t key_version;                    // 密钥版本
    uint64_t derivation_timestamp;           // 派生时间戳
    uint32_t usage_counter;                  // 使用计数器
};

// 完整密钥派生链
static TEE_Result derive_complete_key_hierarchy(const TEE_UUID *ta_uuid,
                                               uint32_t file_number,
                                               struct tadb_key_hierarchy *keys)
{
    TEE_Result res = TEE_SUCCESS;
    uint8_t context[64];
    size_t context_len = 0;
    
    // 1. 获取硬件唯一密钥
    res = tee_otp_get_hw_unique_key((struct tee_hw_unique_key *)keys->huk);
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 派生 TADB 主密钥
    res = huk_subkey_derive(HUK_SUBKEY_TA_ENC, "TADB_MASTER", 11,
                          keys->master_keys.master_enc_key, 32);
    if (res != TEE_SUCCESS)
        return res;
    
    // 3. 派生 TA 特定密钥
    memcpy(context, ta_uuid, sizeof(*ta_uuid));
    context_len = sizeof(*ta_uuid);
    
    res = kdf_derive_key(keys->master_keys.master_kdf_key, 32,
                        context, context_len,
                        "TA_STORAGE_KEY", 14,
                        keys->ta_keys.ta_storage_key, 32);
    if (res != TEE_SUCCESS)
        return res;
    
    // 4. 派生文件特定密钥
    memcpy(context, ta_uuid, sizeof(*ta_uuid));
    memcpy(context + sizeof(*ta_uuid), &file_number, sizeof(file_number));
    context_len = sizeof(*ta_uuid) + sizeof(file_number);
    
    res = kdf_derive_key(keys->ta_keys.ta_storage_key, 32,
                        context, context_len,
                        "FILE_ENC_KEY", 12,
                        keys->file_keys.file_enc_key, 32);
    
    return res;
}
```

这个详细的存储架构分析展示了 OP-TEE TADB 系统的复杂性和完整性，包括分层设计、事务支持、性能优化、错误恢复和安全隔离等多个关键方面，为理解和实现类似的安全存储系统提供了全面的技术参考。

## 高级存储特性和实现细节

### 层次化加密和密钥管理深度分析

#### 1. 文件系统层次化树 (FS Htree) 实现

```c
// FS Htree 节点结构
#define TEE_FS_HTREE_TYPE_BLOCK    0    // 数据块类型
#define TEE_FS_HTREE_TYPE_NODE     1    // 内部节点类型

struct tee_fs_htree_node_image {
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];      // 32字节 SHA256 哈希
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];          // 16字节 AES IV
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];        // 16字节 AES-GCM 标签
    uint16_t flags;                             // 节点标志
    uint16_t counter;                           // 版本计数器
};

// Htree 元数据管理
struct tee_fs_htree_meta {
    struct tee_fs_htree_node_image *nodes;     // 节点数组
    size_t max_node_count;                     // 最大节点数
    size_t node_count;                         // 当前节点数
    uint64_t counter;                          // 全局计数器
    bool dirty;                                // 脏标志
};

// Htree 加密上下文
struct tee_fs_htree_storage {
    struct tee_file_handle *fh;                // 文件句柄
    uint8_t fek[TEE_FS_KM_FEK_SIZE];          // 文件加密密钥
    struct tee_fs_htree_meta *meta;           // 元数据
    size_t block_size;                         // 块大小
    size_t max_file_size;                      // 最大文件大小
};
```

#### 2. ESSIV (Encrypted Salt-Sector IV) 实现

```c
// ESSIV 密钥派生
struct essiv_context {
    uint8_t sector_key[32];                    // 扇区密钥
    uint8_t hash_key[32];                      // 哈希密钥
    struct crypto_cipher_ctx *cipher_ctx;     // 加密上下文
};

// ESSIV IV 生成算法
static TEE_Result generate_essiv(struct essiv_context *ctx,
                                 uint64_t sector_number,
                                 uint8_t *iv, size_t iv_len)
{
    uint8_t sector_hash[TEE_SHA256_HASH_SIZE];
    TEE_Result res = TEE_SUCCESS;
    
    // 1. 计算扇区号的哈希
    res = crypto_hash_sha256((uint8_t *)&sector_number, sizeof(sector_number),
                            sector_hash, sizeof(sector_hash));
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 使用哈希密钥加密扇区哈希
    res = crypto_cipher_encrypt(ctx->cipher_ctx, sector_hash, iv, iv_len);
    if (res != TEE_SUCCESS)
        return res;
    
    return TEE_SUCCESS;
}

// 文件级 ESSIV 初始化
static TEE_Result init_file_essiv(const uint8_t *fek, size_t fek_len,
                                  struct essiv_context *ctx)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 派生 ESSIV 密钥
    res = hkdf_extract_expand(fek, fek_len,
                             "FILE_ESSIV_KEY", 15,
                             NULL, 0,
                             ctx->sector_key, sizeof(ctx->sector_key));
    if (res != TEE_SUCCESS)
        return res;
    
    // 初始化密码上下文
    res = crypto_cipher_alloc_ctx(&ctx->cipher_ctx, TEE_ALG_AES_ECB_NOPAD);
    if (res != TEE_SUCCESS)
        return res;
    
    res = crypto_cipher_init(ctx->cipher_ctx, TEE_MODE_ENCRYPT,
                            ctx->sector_key, sizeof(ctx->sector_key));
    
    return res;
}
```

#### 3. 存储密钥分级管理系统

```c
// 存储密钥层次结构
enum storage_key_level {
    STORAGE_KEY_LEVEL_HUK = 0,         // 硬件唯一密钥
    STORAGE_KEY_LEVEL_SSK = 1,         // 存储密封密钥
    STORAGE_KEY_LEVEL_TSK = 2,         // TA 存储密钥
    STORAGE_KEY_LEVEL_FEK = 3,         // 文件加密密钥
    STORAGE_KEY_LEVEL_DEK = 4,         // 数据加密密钥
};

// 密钥管理器状态
struct tee_fs_key_manager {
    uint8_t ssk[TEE_FS_KM_SSK_SIZE];          // 存储密封密钥
    bool ssk_initialized;                      // SSK 初始化标志
    struct mutex key_mutex;                   // 密钥操作互斥锁
    
    // 密钥缓存
    struct {
        TEE_UUID ta_uuid;                      // TA UUID
        uint8_t tsk[TEE_FS_KM_TSK_SIZE];      // 缓存的 TSK
        uint64_t timestamp;                    // 缓存时间戳
        bool valid;                            // 缓存有效性
    } tsk_cache[8];                           // TSK 缓存数组
    
    // 密钥使用统计
    struct {
        uint64_t ssk_derivations;             // SSK 派生次数
        uint64_t tsk_derivations;             // TSK 派生次数
        uint64_t fek_generations;             // FEK 生成次数
        uint64_t key_cache_hits;              // 缓存命中次数
        uint64_t key_cache_misses;            // 缓存未命中次数
    } stats;
};

// SSK (Storage Seal Key) 派生
static TEE_Result derive_ssk(uint8_t *ssk, size_t ssk_len)
{
    const uint8_t label[] = "STORAGE_SEAL_KEY";
    struct tee_hw_unique_key huk;
    TEE_Result res = TEE_SUCCESS;
    
    // 获取硬件唯一密钥
    res = tee_otp_get_hw_unique_key(&huk);
    if (res != TEE_SUCCESS)
        return res;
    
    // 使用 HKDF 派生 SSK
    res = hkdf_extract_expand(huk.data, sizeof(huk.data),
                             label, sizeof(label) - 1,
                             NULL, 0,
                             ssk, ssk_len);
    
    // 安全清零 HUK
    memzero_explicit(&huk, sizeof(huk));
    
    return res;
}

// TSK (TA Storage Key) 派生
static TEE_Result derive_tsk(const TEE_UUID *ta_uuid,
                             uint8_t *tsk, size_t tsk_len)
{
    struct tee_fs_key_manager *km = get_key_manager();
    const uint8_t label[] = "TA_STORAGE_KEY";
    TEE_Result res = TEE_SUCCESS;
    
    mutex_lock(&km->key_mutex);
    
    // 检查 TSK 缓存
    for (int i = 0; i < ARRAY_SIZE(km->tsk_cache); i++) {
        if (km->tsk_cache[i].valid &&
            memcmp(&km->tsk_cache[i].ta_uuid, ta_uuid, sizeof(TEE_UUID)) == 0) {
            memcpy(tsk, km->tsk_cache[i].tsk, tsk_len);
            km->stats.key_cache_hits++;
            mutex_unlock(&km->key_mutex);
            return TEE_SUCCESS;
        }
    }
    
    km->stats.key_cache_misses++;
    
    // 确保 SSK 已初始化
    if (!km->ssk_initialized) {
        res = derive_ssk(km->ssk, sizeof(km->ssk));
        if (res != TEE_SUCCESS)
            goto out;
        km->ssk_initialized = true;
    }
    
    // 使用 SSK 和 TA UUID 派生 TSK
    res = hkdf_extract_expand(km->ssk, sizeof(km->ssk),
                             label, sizeof(label) - 1,
                             (const uint8_t *)ta_uuid, sizeof(*ta_uuid),
                             tsk, tsk_len);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 更新 TSK 缓存 (LRU 替换)
    int lru_index = find_lru_tsk_cache_entry(km);
    memcpy(&km->tsk_cache[lru_index].ta_uuid, ta_uuid, sizeof(TEE_UUID));
    memcpy(km->tsk_cache[lru_index].tsk, tsk, tsk_len);
    km->tsk_cache[lru_index].timestamp = get_time_stamp();
    km->tsk_cache[lru_index].valid = true;
    
    km->stats.tsk_derivations++;
    
out:
    mutex_unlock(&km->key_mutex);
    return res;
}
```

### 存储文件系统抽象层

#### 1. 目录文件 (DirFile) 管理系统

```c
// 目录文件结构
#define TEE_FS_DIRFILE_MAGIC         0x64666964  // "dfid"
#define TEE_FS_DIRFILE_MAX_ENTRIES   1024

struct tee_fs_dirfile_header {
    uint32_t magic;                            // 魔数
    uint32_t version;                          // 版本号
    uint32_t num_entries;                      // 条目数量
    uint32_t max_entries;                      // 最大条目数
    uint64_t generation;                       // 生成号
    uint8_t hash[TEE_SHA256_HASH_SIZE];       // 目录哈希
};

struct tee_fs_dirfile_entry {
    TEE_UUID uuid;                             // 对象 UUID
    char filename[TEE_FS_NAME_MAX];           // 文件名
    uint32_t file_number;                      // 文件编号
    uint32_t flags;                            // 标志位
    uint64_t file_size;                        // 文件大小
    uint64_t creation_time;                    // 创建时间
    uint64_t modification_time;                // 修改时间
    uint8_t file_hash[TEE_SHA256_HASH_SIZE];  // 文件哈希
};

// 目录文件管理上下文
struct tee_fs_dirfile {
    struct tee_fs_dirfile_header header;      // 头部
    struct tee_fs_dirfile_entry *entries;    // 条目数组
    bool dirty;                               // 脏标志
    struct mutex dirfile_mutex;              // 目录互斥锁
    
    // 索引优化
    struct {
        struct tee_fs_dirfile_entry **uuid_index;  // UUID 索引
        struct tee_fs_dirfile_entry **name_index;  // 名称索引
        bool indices_valid;                    // 索引有效性
    } indices;
    
    // 变更跟踪
    struct {
        uint32_t *added_entries;              // 新增条目
        uint32_t *modified_entries;           // 修改条目
        uint32_t *deleted_entries;            // 删除条目
        size_t num_added;                     // 新增数量
        size_t num_modified;                  // 修改数量
        size_t num_deleted;                   // 删除数量
    } changes;
};

// 目录文件操作接口
static TEE_Result dirfile_create_entry(struct tee_fs_dirfile *df,
                                       const TEE_UUID *uuid,
                                       const char *filename,
                                       uint32_t *file_number)
{
    struct tee_fs_dirfile_entry *entry = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    mutex_lock(&df->dirfile_mutex);
    
    // 检查容量
    if (df->header.num_entries >= df->header.max_entries) {
        res = expand_dirfile(df);
        if (res != TEE_SUCCESS)
            goto out;
    }
    
    // 分配新条目
    entry = &df->entries[df->header.num_entries];
    memset(entry, 0, sizeof(*entry));
    
    // 填充条目信息
    memcpy(&entry->uuid, uuid, sizeof(TEE_UUID));
    strlcpy(entry->filename, filename, sizeof(entry->filename));
    entry->file_number = allocate_file_number(df);
    entry->flags = 0;
    entry->creation_time = get_time_stamp();
    entry->modification_time = entry->creation_time;
    
    df->header.num_entries++;
    df->header.generation++;
    df->dirty = true;
    
    // 失效索引
    df->indices.indices_valid = false;
    
    // 记录变更
    add_to_changes(&df->changes, CHANGE_TYPE_ADD, df->header.num_entries - 1);
    
    *file_number = entry->file_number;
    
out:
    mutex_unlock(&df->dirfile_mutex);
    return res;
}
```

#### 2. 持久化对象生命周期管理

```c
// 持久化对象状态
enum tee_pobj_state {
    TEE_POBJ_STATE_CREATED,                   // 已创建
    TEE_POBJ_STATE_OPENED,                    // 已打开
    TEE_POBJ_STATE_MODIFIED,                  // 已修改
    TEE_POBJ_STATE_COMMITTED,                 // 已提交
    TEE_POBJ_STATE_DELETED,                   // 已删除
};

// 持久化对象结构
struct tee_pobj {
    uint32_t obj_id;                          // 对象 ID
    TEE_UUID uuid;                            // 对象 UUID
    uint32_t storage_id;                      // 存储 ID
    char obj_name[TEE_FS_NAME_MAX];          // 对象名称
    uint32_t flags;                           // 标志位
    enum tee_pobj_state state;               // 对象状态
    uint32_t refcnt;                         // 引用计数
    
    // 文件系统信息
    uint32_t file_number;                     // 文件编号
    size_t file_size;                         // 文件大小
    uint8_t file_hash[TEE_SHA256_HASH_SIZE]; // 文件哈希
    
    // 访问控制
    struct {
        bool read_access;                     // 读权限
        bool write_access;                    // 写权限
        bool delete_access;                   // 删除权限
    } access;
    
    // 生命周期钩子
    struct {
        TEE_Result (*on_create)(struct tee_pobj *po);
        TEE_Result (*on_open)(struct tee_pobj *po);
        TEE_Result (*on_close)(struct tee_pobj *po);
        TEE_Result (*on_delete)(struct tee_pobj *po);
    } hooks;
    
    struct tee_pobj *next;                   // 链表指针
};

// 对象管理器
struct tee_pobj_manager {
    struct tee_pobj *objects;                // 对象链表
    struct mutex obj_mutex;                  // 对象互斥锁
    uint32_t next_obj_id;                    // 下一个对象 ID
    
    // 对象池
    struct {
        struct tee_pobj *free_objects;       // 空闲对象池
        size_t pool_size;                    // 池大小
        size_t free_count;                   // 空闲数量
    } object_pool;
    
    // 统计信息
    struct {
        uint32_t total_objects;              // 总对象数
        uint32_t active_objects;             // 活跃对象数
        uint32_t peak_objects;               // 峰值对象数
        uint64_t total_operations;           // 总操作数
    } stats;
};

// 对象生命周期管理
static TEE_Result manage_object_lifecycle(struct tee_pobj *po,
                                         enum tee_pobj_state new_state)
{
    TEE_Result res = TEE_SUCCESS;
    enum tee_pobj_state old_state = po->state;
    
    // 状态转换验证
    if (!is_valid_state_transition(old_state, new_state)) {
        return TEE_ERROR_BAD_STATE;
    }
    
    // 执行状态转换钩子
    switch (new_state) {
    case TEE_POBJ_STATE_CREATED:
        if (po->hooks.on_create) {
            res = po->hooks.on_create(po);
        }
        break;
        
    case TEE_POBJ_STATE_OPENED:
        if (po->hooks.on_open) {
            res = po->hooks.on_open(po);
        }
        po->refcnt++;
        break;
        
    case TEE_POBJ_STATE_DELETED:
        if (po->hooks.on_delete) {
            res = po->hooks.on_delete(po);
        }
        break;
        
    default:
        break;
    }
    
    if (res == TEE_SUCCESS) {
        po->state = new_state;
        log_state_transition(po, old_state, new_state);
    }
    
    return res;
}
```

### 存储同步和一致性协议

#### 1. 双版本存储协议

```c
// 双版本存储头部
struct dual_version_header {
    uint32_t magic;                           // 魔数
    uint32_t version_a;                       // 版本 A
    uint32_t version_b;                       // 版本 B
    uint64_t counter_a;                       // 计数器 A
    uint64_t counter_b;                       // 计数器 B
    uint8_t hash_a[TEE_SHA256_HASH_SIZE];    // 哈希 A
    uint8_t hash_b[TEE_SHA256_HASH_SIZE];    // 哈希 B
    uint32_t flags;                           // 标志位
    uint32_t reserved;                        // 保留字段
};

// 版本选择算法
static uint32_t select_current_version(const struct dual_version_header *hdr)
{
    // 选择更高计数器的版本
    if (hdr->counter_a > hdr->counter_b) {
        return 0;  // 选择版本 A
    } else if (hdr->counter_b > hdr->counter_a) {
        return 1;  // 选择版本 B
    }
    
    // 计数器相等时的处理
    if (hdr->version_a > hdr->version_b) {
        return 0;
    } else {
        return 1;
    }
}

// 原子提交协议
static TEE_Result atomic_commit_dual_version(struct tee_file_handle *fh,
                                            const void *data, size_t size)
{
    struct dual_version_header hdr;
    uint32_t inactive_version;
    uint64_t new_counter;
    TEE_Result res = TEE_SUCCESS;
    
    // 1. 读取当前头部
    res = read_dual_version_header(fh, &hdr);
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 确定非活跃版本
    uint32_t active_version = select_current_version(&hdr);
    inactive_version = 1 - active_version;
    
    // 3. 计算新计数器
    new_counter = MAX(hdr.counter_a, hdr.counter_b) + 1;
    
    // 4. 写入非活跃版本
    res = write_version_data(fh, inactive_version, data, size);
    if (res != TEE_SUCCESS)
        return res;
    
    // 5. 计算数据哈希
    uint8_t data_hash[TEE_SHA256_HASH_SIZE];
    res = crypto_hash_sha256(data, size, data_hash, sizeof(data_hash));
    if (res != TEE_SUCCESS)
        return res;
    
    // 6. 原子更新头部
    if (inactive_version == 0) {
        hdr.counter_a = new_counter;
        memcpy(hdr.hash_a, data_hash, sizeof(hdr.hash_a));
    } else {
        hdr.counter_b = new_counter;
        memcpy(hdr.hash_b, data_hash, sizeof(hdr.hash_b));
    }
    
    // 7. 写入头部 (原子操作)
    res = write_dual_version_header_atomic(fh, &hdr);
    
    return res;
}
```

#### 2. 写前日志 (WAL) 实现

```c
// WAL 记录类型
enum wal_record_type {
    WAL_RECORD_START_TXN = 1,                // 事务开始
    WAL_RECORD_WRITE_DATA = 2,               // 数据写入
    WAL_RECORD_COMMIT_TXN = 3,               // 事务提交
    WAL_RECORD_ABORT_TXN = 4,                // 事务中止
    WAL_RECORD_CHECKPOINT = 5,               // 检查点
};

// WAL 记录头部
struct wal_record_header {
    uint32_t magic;                          // 魔数
    uint16_t type;                           // 记录类型
    uint16_t flags;                          // 标志位
    uint32_t length;                         // 记录长度
    uint64_t lsn;                           // 日志序列号
    uint64_t prev_lsn;                      // 前一个 LSN
    uint64_t txn_id;                        // 事务 ID
    uint32_t checksum;                      // 校验和
    uint32_t reserved;                      // 保留字段
};

// WAL 写入记录
struct wal_write_record {
    struct wal_record_header header;        // 记录头部
    uint32_t file_number;                   // 文件编号
    uint64_t offset;                        // 偏移量
    uint32_t data_length;                   // 数据长度
    uint8_t old_hash[TEE_SHA256_HASH_SIZE]; // 旧数据哈希
    uint8_t new_hash[TEE_SHA256_HASH_SIZE]; // 新数据哈希
    uint8_t data[];                         // 变长数据
};

// WAL 管理器
struct wal_manager {
    struct tee_file_handle *wal_file;       // WAL 文件句柄
    uint64_t next_lsn;                      // 下一个 LSN
    uint64_t checkpoint_lsn;                // 检查点 LSN
    struct mutex wal_mutex;                 // WAL 互斥锁
    
    // WAL 缓冲区
    struct {
        uint8_t *buffer;                    // 缓冲区
        size_t buffer_size;                 // 缓冲区大小
        size_t used_size;                   // 已使用大小
        bool dirty;                         // 脏标志
    } wal_buffer;
    
    // 恢复信息
    struct {
        uint64_t recovery_lsn;              // 恢复起始 LSN
        bool recovery_needed;               // 需要恢复
        uint32_t incomplete_txns;           // 未完成事务数
    } recovery_info;
};

// WAL 写入函数
static TEE_Result wal_write_record(struct wal_manager *wm,
                                  const struct wal_record_header *hdr,
                                  const void *data, size_t data_len)
{
    TEE_Result res = TEE_SUCCESS;
    size_t total_len = sizeof(*hdr) + data_len;
    uint32_t checksum;
    
    mutex_lock(&wm->wal_mutex);
    
    // 检查缓冲区空间
    if (wm->wal_buffer.used_size + total_len > wm->wal_buffer.buffer_size) {
        res = flush_wal_buffer(wm);
        if (res != TEE_SUCCESS)
            goto out;
    }
    
    // 计算校验和
    checksum = calculate_wal_checksum(hdr, data, data_len);
    
    // 写入记录头部
    struct wal_record_header mutable_hdr = *hdr;
    mutable_hdr.checksum = checksum;
    
    memcpy(wm->wal_buffer.buffer + wm->wal_buffer.used_size,
           &mutable_hdr, sizeof(mutable_hdr));
    wm->wal_buffer.used_size += sizeof(mutable_hdr);
    
    // 写入数据
    if (data && data_len > 0) {
        memcpy(wm->wal_buffer.buffer + wm->wal_buffer.used_size,
               data, data_len);
        wm->wal_buffer.used_size += data_len;
    }
    
    wm->wal_buffer.dirty = true;
    wm->next_lsn++;
    
out:
    mutex_unlock(&wm->wal_mutex);
    return res;
}
```

### 存储监控和诊断系统

#### 1. 存储健康监控

```c
// 存储健康状态
enum storage_health_status {
    STORAGE_HEALTH_GOOD = 0,                 // 良好
    STORAGE_HEALTH_WARNING = 1,              // 警告
    STORAGE_HEALTH_CRITICAL = 2,             // 严重
    STORAGE_HEALTH_FAILED = 3,               // 失败
};

// 存储健康指标
struct storage_health_metrics {
    // I/O 统计
    struct {
        uint64_t read_operations;            // 读操作数
        uint64_t write_operations;           // 写操作数
        uint64_t read_errors;                // 读错误数
        uint64_t write_errors;               // 写错误数
        uint64_t bytes_read;                 // 读取字节数
        uint64_t bytes_written;              // 写入字节数
    } io_stats;
    
    // 性能指标
    struct {
        uint64_t avg_read_latency_us;        // 平均读延迟(微秒)
        uint64_t avg_write_latency_us;       // 平均写延迟(微秒)
        uint64_t max_read_latency_us;        // 最大读延迟
        uint64_t max_write_latency_us;       // 最大写延迟
        uint32_t iops;                       // 每秒I/O操作数
    } performance;
    
    // 错误统计
    struct {
        uint32_t corruption_events;         // 损坏事件数
        uint32_t authentication_failures;   // 认证失败数
        uint32_t key_derivation_errors;     // 密钥派生错误
        uint32_t memory_allocation_failures; // 内存分配失败
        uint32_t storage_quota_violations;  // 存储配额违规
    } error_counts;
    
    // 资源使用
    struct {
        uint64_t total_storage_used;         // 总存储使用量
        uint64_t total_storage_available;    // 总可用存储
        uint32_t active_file_handles;        // 活跃文件句柄数
        uint32_t cached_objects;             // 缓存对象数
        uint32_t pending_transactions;       // 待处理事务数
    } resource_usage;
    
    enum storage_health_status overall_status; // 整体健康状态
    uint64_t last_updated;                   // 最后更新时间
};

// 健康监控器
struct storage_health_monitor {
    struct storage_health_metrics metrics; // 健康指标
    struct mutex monitor_mutex;            // 监控互斥锁
    
    // 阈值配置
    struct {
        uint32_t max_error_rate;            // 最大错误率
        uint64_t max_latency_threshold_us;  // 最大延迟阈值
        uint32_t max_corruption_events;     // 最大损坏事件数
        uint64_t min_free_space_bytes;      // 最小空闲空间
    } thresholds;
    
    // 历史数据
    struct {
        struct storage_health_metrics *history; // 历史指标数组
        size_t history_size;                // 历史大小
        size_t current_index;               // 当前索引
        bool history_full;                  // 历史已满标志
    } history;
    
    // 告警回调
    void (*health_alert_callback)(enum storage_health_status status,
                                  const char *message);
};

// 健康状态评估
static enum storage_health_status assess_storage_health(
    const struct storage_health_metrics *metrics,
    const struct storage_health_monitor *monitor)
{
    enum storage_health_status status = STORAGE_HEALTH_GOOD;
    
    // 检查错误率
    uint64_t total_ops = metrics->io_stats.read_operations + 
                        metrics->io_stats.write_operations;
    uint64_t total_errors = metrics->io_stats.read_errors + 
                           metrics->io_stats.write_errors;
    
    if (total_ops > 0) {
        uint32_t error_rate = (total_errors * 100) / total_ops;
        if (error_rate > monitor->thresholds.max_error_rate) {
            status = STORAGE_HEALTH_CRITICAL;
        }
    }
    
    // 检查延迟
    if (metrics->performance.avg_read_latency_us > monitor->thresholds.max_latency_threshold_us ||
        metrics->performance.avg_write_latency_us > monitor->thresholds.max_latency_threshold_us) {
        if (status < STORAGE_HEALTH_WARNING)
            status = STORAGE_HEALTH_WARNING;
    }
    
    // 检查损坏事件
    if (metrics->error_counts.corruption_events > monitor->thresholds.max_corruption_events) {
        status = STORAGE_HEALTH_CRITICAL;
    }
    
    // 检查存储空间
    uint64_t free_space = metrics->resource_usage.total_storage_available - 
                         metrics->resource_usage.total_storage_used;
    if (free_space < monitor->thresholds.min_free_space_bytes) {
        if (status < STORAGE_HEALTH_WARNING)
            status = STORAGE_HEALTH_WARNING;
    }
    
    return status;
}
```

#### 2. 存储调试和跟踪系统

```c
// 存储调试级别
enum storage_debug_level {
    STORAGE_DEBUG_NONE = 0,                  // 无调试
    STORAGE_DEBUG_ERROR = 1,                 // 错误级别
    STORAGE_DEBUG_WARNING = 2,               // 警告级别
    STORAGE_DEBUG_INFO = 3,                  // 信息级别
    STORAGE_DEBUG_VERBOSE = 4,               // 详细级别
    STORAGE_DEBUG_TRACE = 5,                 // 跟踪级别
};

// 存储事件类型
enum storage_event_type {
    STORAGE_EVENT_FILE_CREATE = 1,          // 文件创建
    STORAGE_EVENT_FILE_OPEN = 2,            // 文件打开
    STORAGE_EVENT_FILE_READ = 3,            // 文件读取
    STORAGE_EVENT_FILE_WRITE = 4,           // 文件写入
    STORAGE_EVENT_FILE_DELETE = 5,          // 文件删除
    STORAGE_EVENT_TXN_BEGIN = 6,            // 事务开始
    STORAGE_EVENT_TXN_COMMIT = 7,           // 事务提交
    STORAGE_EVENT_TXN_ABORT = 8,            // 事务中止
    STORAGE_EVENT_KEY_DERIVE = 9,           // 密钥派生
    STORAGE_EVENT_ENCRYPT = 10,             // 加密操作
    STORAGE_EVENT_DECRYPT = 11,             // 解密操作
    STORAGE_EVENT_ERROR = 12,               // 错误事件
};

// 存储跟踪记录
struct storage_trace_record {
    uint64_t timestamp;                     // 时间戳
    enum storage_event_type event_type;     // 事件类型
    TEE_UUID ta_uuid;                       // TA UUID
    uint32_t file_number;                   // 文件编号
    size_t data_size;                       // 数据大小
    TEE_Result result;                      // 操作结果
    char message[128];                      // 消息字符串
    
    // 性能数据
    uint64_t start_time_us;                 // 开始时间(微秒)
    uint64_t end_time_us;                   // 结束时间(微秒)
    uint64_t duration_us;                   // 持续时间(微秒)
};

// 存储调试系统
struct storage_debug_system {
    enum storage_debug_level debug_level;  // 调试级别
    struct mutex debug_mutex;              // 调试互斥锁
    
    // 跟踪缓冲区
    struct {
        struct storage_trace_record *records; // 跟踪记录数组
        size_t buffer_size;                 // 缓冲区大小
        size_t write_index;                 // 写入索引
        size_t read_index;                  // 读取索引
        bool buffer_full;                   // 缓冲区已满
    } trace_buffer;
    
    // 过滤器
    struct {
        bool filter_by_ta;                  // 按 TA 过滤
        TEE_UUID filter_ta_uuid;           // 过滤的 TA UUID
        bool filter_by_event_type;         // 按事件类型过滤
        uint32_t filter_event_mask;        // 事件类型掩码
        bool filter_by_result;              // 按结果过滤
        TEE_Result filter_result;          // 过滤的结果
    } filters;
    
    // 统计信息
    struct {
        uint64_t total_events;              // 总事件数
        uint64_t filtered_events;           // 过滤的事件数
        uint64_t buffer_overruns;           // 缓冲区溢出次数
    } stats;
};

// 跟踪记录函数
static void storage_trace_event(enum storage_event_type event_type,
                               const TEE_UUID *ta_uuid,
                               uint32_t file_number,
                               size_t data_size,
                               TEE_Result result,
                               const char *message)
{
    struct storage_debug_system *debug = get_storage_debug_system();
    struct storage_trace_record *record;
    
    if (debug->debug_level < STORAGE_DEBUG_TRACE)
        return;
    
    mutex_lock(&debug->debug_mutex);
    
    // 应用过滤器
    if (!should_trace_event(debug, event_type, ta_uuid, result)) {
        debug->stats.filtered_events++;
        goto out;
    }
    
    // 获取跟踪记录槽位
    record = &debug->trace_buffer.records[debug->trace_buffer.write_index];
    
    // 填充跟踪记录
    record->timestamp = get_time_stamp();
    record->event_type = event_type;
    if (ta_uuid)
        memcpy(&record->ta_uuid, ta_uuid, sizeof(TEE_UUID));
    else
        memset(&record->ta_uuid, 0, sizeof(TEE_UUID));
    record->file_number = file_number;
    record->data_size = data_size;
    record->result = result;
    strlcpy(record->message, message ? message : "", sizeof(record->message));
    
    // 更新写入索引
    debug->trace_buffer.write_index = 
        (debug->trace_buffer.write_index + 1) % debug->trace_buffer.buffer_size;
    
    // 检查缓冲区溢出
    if (debug->trace_buffer.write_index == debug->trace_buffer.read_index) {
        debug->trace_buffer.buffer_full = true;
        debug->stats.buffer_overruns++;
        // 移动读取索引以避免覆盖
        debug->trace_buffer.read_index = 
            (debug->trace_buffer.read_index + 1) % debug->trace_buffer.buffer_size;
    }
    
    debug->stats.total_events++;
    
out:
    mutex_unlock(&debug->debug_mutex);
}
```

### 高级存储安全特性

#### 1. 反回滚机制

```c
// 反回滚计数器结构
struct anti_rollback_counter {
    uint64_t counter_value;                 // 计数器值
    uint64_t backup_counter;                // 备份计数器
    uint32_t write_count;                   // 写入次数
    uint32_t checksum;                      // 校验和
    uint8_t signature[64];                  // 数字签名
    uint64_t timestamp;                     // 时间戳
};

// 反回滚管理器
struct anti_rollback_manager {
    struct anti_rollback_counter counters[16]; // 计数器数组
    struct mutex arb_mutex;                 // 反回滚互斥锁
    uint32_t active_counters;               // 活跃计数器数
    
    // 策略配置
    struct {
        bool enforce_monotonic;             // 强制单调递增
        bool require_signature;             // 需要数字签名
        uint32_t max_counter_skew;          // 最大计数器偏差
        uint64_t max_time_skew_ms;          // 最大时间偏差
    } policy;
    
    // 违规检测
    struct {
        uint32_t rollback_attempts;         // 回滚尝试次数
        uint32_t counter_violations;        // 计数器违规次数
        uint32_t signature_failures;        // 签名失败次数
        uint64_t last_violation_time;       // 最后违规时间
    } violations;
};

// 反回滚检查函数
static TEE_Result check_anti_rollback(struct anti_rollback_manager *arm,
                                     uint32_t counter_id,
                                     uint64_t new_value)
{
    struct anti_rollback_counter *counter;
    TEE_Result res = TEE_SUCCESS;
    
    if (counter_id >= ARRAY_SIZE(arm->counters))
        return TEE_ERROR_BAD_PARAMETERS;
    
    mutex_lock(&arm->arb_mutex);
    
    counter = &arm->counters[counter_id];
    
    // 检查单调性
    if (arm->policy.enforce_monotonic && new_value <= counter->counter_value) {
        EMSG("Anti-rollback violation: new_value=%"PRIu64" <= current=%"PRIu64,
             new_value, counter->counter_value);
        arm->violations.rollback_attempts++;
        res = TEE_ERROR_SECURITY;
        goto out;
    }
    
    // 检查计数器偏差
    uint64_t skew = new_value - counter->counter_value;
    if (skew > arm->policy.max_counter_skew) {
        EMSG("Counter skew too large: %"PRIu64" > %u",
             skew, arm->policy.max_counter_skew);
        arm->violations.counter_violations++;
        res = TEE_ERROR_SECURITY;
        goto out;
    }
    
    // 检查时间偏差
    uint64_t current_time = get_time_stamp();
    uint64_t time_diff = current_time - counter->timestamp;
    if (time_diff > arm->policy.max_time_skew_ms) {
        EMSG("Time skew too large: %"PRIu64" > %"PRIu64,
             time_diff, arm->policy.max_time_skew_ms);
        res = TEE_ERROR_SECURITY;
        goto out;
    }
    
out:
    if (res != TEE_SUCCESS) {
        arm->violations.last_violation_time = get_time_stamp();
        log_security_violation("ANTI_ROLLBACK_VIOLATION", counter_id, new_value);
    }
    
    mutex_unlock(&arm->arb_mutex);
    return res;
}
```

#### 2. 存储篡改检测

```c
// 篡改检测策略
enum tamper_detection_policy {
    TAMPER_DETECT_HASH_MISMATCH = BIT(0),   // 哈希不匹配
    TAMPER_DETECT_SIZE_CHANGE = BIT(1),     // 大小变化
    TAMPER_DETECT_TIMESTAMP_ANOMALY = BIT(2), // 时间戳异常
    TAMPER_DETECT_METADATA_CORRUPTION = BIT(3), // 元数据损坏
    TAMPER_DETECT_SIGNATURE_INVALID = BIT(4), // 签名无效
};

// 篡改检测上下文
struct tamper_detection_context {
    uint32_t policy_mask;                   // 策略掩码
    struct mutex tamper_mutex;              // 篡改检测互斥锁
    
    // 基线数据
    struct {
        uint8_t *baseline_hashes;           // 基线哈希数组
        uint64_t *baseline_sizes;           // 基线大小数组
        uint64_t *baseline_timestamps;      // 基线时间戳数组
        uint32_t num_files;                 // 文件数量
        uint64_t baseline_generation;       // 基线生成号
    } baseline;
    
    // 检测统计
    struct {
        uint32_t tamper_events;             // 篡改事件数
        uint32_t false_positives;           // 误报数
        uint32_t hash_mismatches;           // 哈希不匹配数
        uint32_t size_anomalies;            // 大小异常数
        uint32_t timestamp_anomalies;       // 时间戳异常数
    } stats;
    
    // 响应策略
    struct {
        bool quarantine_on_tamper;          // 篡改时隔离
        bool alert_on_tamper;               // 篡改时告警
        bool restore_from_backup;           // 从备份恢复
        bool deny_access;                   // 拒绝访问
    } response;
};

// 篡改检测函数
static TEE_Result detect_storage_tamper(struct tamper_detection_context *tdc,
                                       uint32_t file_number,
                                       const uint8_t *current_hash,
                                       uint64_t current_size,
                                       uint64_t current_timestamp)
{
    TEE_Result res = TEE_SUCCESS;
    bool tamper_detected = false;
    uint32_t tamper_reasons = 0;
    
    mutex_lock(&tdc->tamper_mutex);
    
    if (file_number >= tdc->baseline.num_files) {
        res = TEE_ERROR_BAD_PARAMETERS;
        goto out;
    }
    
    // 检查哈希匹配
    if (tdc->policy_mask & TAMPER_DETECT_HASH_MISMATCH) {
        if (memcmp(current_hash, 
                   &tdc->baseline.baseline_hashes[file_number * TEE_SHA256_HASH_SIZE],
                   TEE_SHA256_HASH_SIZE) != 0) {
            tamper_detected = true;
            tamper_reasons |= TAMPER_DETECT_HASH_MISMATCH;
            tdc->stats.hash_mismatches++;
        }
    }
    
    // 检查大小变化
    if (tdc->policy_mask & TAMPER_DETECT_SIZE_CHANGE) {
        if (current_size != tdc->baseline.baseline_sizes[file_number]) {
            tamper_detected = true;
            tamper_reasons |= TAMPER_DETECT_SIZE_CHANGE;
            tdc->stats.size_anomalies++;
        }
    }
    
    // 检查时间戳异常
    if (tdc->policy_mask & TAMPER_DETECT_TIMESTAMP_ANOMALY) {
        uint64_t baseline_ts = tdc->baseline.baseline_timestamps[file_number];
        if (current_timestamp < baseline_ts) {  // 时间倒退
            tamper_detected = true;
            tamper_reasons |= TAMPER_DETECT_TIMESTAMP_ANOMALY;
            tdc->stats.timestamp_anomalies++;
        }
    }
    
    if (tamper_detected) {
        tdc->stats.tamper_events++;
        
        EMSG("Storage tamper detected for file %u, reasons=0x%x",
             file_number, tamper_reasons);
        
        // 执行响应策略
        res = handle_tamper_detection(tdc, file_number, tamper_reasons);
        
        // 记录安全事件
        log_security_event("STORAGE_TAMPER_DETECTED", file_number, tamper_reasons);
    }
    
out:
    mutex_unlock(&tdc->tamper_mutex);
    return tamper_detected ? TEE_ERROR_SECURITY : res;
}
```

### 存储压缩和优化技术

#### 1. 存储压缩框架

```c
// 压缩算法类型
enum storage_compression_algo {
    STORAGE_COMPRESS_NONE = 0,              // 无压缩
    STORAGE_COMPRESS_LZ4 = 1,               // LZ4 压缩
    STORAGE_COMPRESS_ZSTD = 2,              // ZSTD 压缩
    STORAGE_COMPRESS_LZMA = 3,              // LZMA 压缩
};

// 压缩上下文
struct storage_compression_context {
    enum storage_compression_algo algo;     // 压缩算法
    uint32_t compression_level;             // 压缩级别
    size_t block_size;                      // 块大小
    
    // 压缩缓冲区
    struct {
        uint8_t *input_buffer;              // 输入缓冲区
        uint8_t *output_buffer;             // 输出缓冲区
        size_t input_size;                  // 输入大小
        size_t output_size;                 // 输出大小
        size_t max_buffer_size;             // 最大缓冲区大小
    } buffers;
    
    // 压缩统计
    struct {
        uint64_t total_input_bytes;         // 总输入字节数
        uint64_t total_output_bytes;        // 总输出字节数
        uint64_t compression_operations;    // 压缩操作数
        uint32_t average_ratio;             // 平均压缩比
    } stats;
    
    // 算法特定上下文
    union {
        void *lz4_ctx;                      // LZ4 上下文
        void *zstd_ctx;                     // ZSTD 上下文
        void *lzma_ctx;                     // LZMA 上下文
    } algo_ctx;
};

// 自适应压缩策略
static enum storage_compression_algo select_compression_algo(
    const uint8_t *data, size_t size)
{
    // 数据类型检测
    if (is_already_compressed(data, size)) {
        return STORAGE_COMPRESS_NONE;  // 已压缩数据不再压缩
    }
    
    if (size < 1024) {
        return STORAGE_COMPRESS_NONE;  // 小文件不压缩
    }
    
    // 计算数据熵
    double entropy = calculate_entropy(data, size);
    if (entropy > 7.5) {
        return STORAGE_COMPRESS_NONE;  // 高熵数据(加密/随机)不压缩
    }
    
    // 基于大小选择算法
    if (size < 64 * 1024) {
        return STORAGE_COMPRESS_LZ4;   // 小到中等文件使用 LZ4
    } else {
        return STORAGE_COMPRESS_ZSTD;  // 大文件使用 ZSTD
    }
}

// 压缩数据函数
static TEE_Result compress_storage_data(struct storage_compression_context *ctx,
                                       const uint8_t *input, size_t input_len,
                                       uint8_t **output, size_t *output_len)
{
    TEE_Result res = TEE_SUCCESS;
    size_t compressed_size = 0;
    
    // 选择压缩算法
    if (ctx->algo == STORAGE_COMPRESS_NONE) {
        ctx->algo = select_compression_algo(input, input_len);
    }
    
    switch (ctx->algo) {
    case STORAGE_COMPRESS_LZ4:
        compressed_size = lz4_compress(input, input_len,
                                      ctx->buffers.output_buffer,
                                      ctx->buffers.max_buffer_size);
        if (compressed_size == 0) {
            res = TEE_ERROR_GENERIC;
        }
        break;
        
    case STORAGE_COMPRESS_ZSTD:
        compressed_size = zstd_compress(ctx->algo_ctx.zstd_ctx,
                                       ctx->buffers.output_buffer,
                                       ctx->buffers.max_buffer_size,
                                       input, input_len,
                                       ctx->compression_level);
        if (compressed_size == 0) {
            res = TEE_ERROR_GENERIC;
        }
        break;
        
    case STORAGE_COMPRESS_NONE:
    default:
        // 无压缩，直接复制
        if (input_len <= ctx->buffers.max_buffer_size) {
            memcpy(ctx->buffers.output_buffer, input, input_len);
            compressed_size = input_len;
        } else {
            res = TEE_ERROR_SHORT_BUFFER;
        }
        break;
    }
    
    if (res == TEE_SUCCESS) {
        // 检查压缩效果
        if (compressed_size >= input_len * 0.95) {
            // 压缩效果不佳，使用原始数据
            memcpy(ctx->buffers.output_buffer, input, input_len);
            compressed_size = input_len;
            ctx->algo = STORAGE_COMPRESS_NONE;
        }
        
        *output = ctx->buffers.output_buffer;
        *output_len = compressed_size;
        
        // 更新统计
        ctx->stats.total_input_bytes += input_len;
        ctx->stats.total_output_bytes += compressed_size;
        ctx->stats.compression_operations++;
        ctx->stats.average_ratio = 
            (ctx->stats.total_output_bytes * 100) / ctx->stats.total_input_bytes;
    }
    
    return res;
}
```

#### 2. 存储去重技术

```c
// 内容哈希结构
struct content_hash_entry {
    uint8_t hash[TEE_SHA256_HASH_SIZE];     // 内容哈希
    uint32_t reference_count;               // 引用计数
    uint32_t block_number;                  // 块编号
    size_t block_size;                      // 块大小
    uint64_t first_seen;                    // 首次出现时间
    uint64_t last_access;                   // 最后访问时间
};

// 去重管理器
struct storage_deduplication_manager {
    struct content_hash_entry *hash_table;  // 哈希表
    size_t hash_table_size;                 // 哈希表大小
    size_t num_entries;                     // 条目数量
    struct mutex dedup_mutex;               // 去重互斥锁
    
    // 去重策略
    struct {
        size_t min_block_size;              // 最小块大小
        size_t max_block_size;              // 最大块大小
        bool aggressive_dedup;              // 激进去重
        uint32_t hash_collision_limit;      // 哈希冲突限制
    } policy;
    
    // 去重统计
    struct {
        uint64_t total_blocks_processed;    // 处理的总块数
        uint64_t duplicate_blocks_found;    // 发现的重复块数
        uint64_t space_saved_bytes;         // 节省的空间字节数
        uint32_t hash_collisions;           // 哈希冲突数
        double deduplication_ratio;         // 去重比率
    } stats;
    
    // 块存储池
    struct {
        uint8_t **blocks;                   // 块存储数组
        uint32_t *reference_counts;        // 引用计数数组
        size_t pool_size;                   // 池大小
        uint32_t next_block_id;             // 下一个块 ID
    } block_pool;
};

// 内容去重函数
static TEE_Result deduplicate_content(struct storage_deduplication_manager *ddm,
                                     const uint8_t *data, size_t size,
                                     uint32_t *block_id, bool *is_duplicate)
{
    uint8_t content_hash[TEE_SHA256_HASH_SIZE];
    struct content_hash_entry *entry = NULL;
    TEE_Result res = TEE_SUCCESS;
    uint32_t hash_index;
    
    // 计算内容哈希
    res = crypto_hash_sha256(data, size, content_hash, sizeof(content_hash));
    if (res != TEE_SUCCESS)
        return res;
    
    mutex_lock(&ddm->dedup_mutex);
    
    // 在哈希表中查找
    hash_index = hash_to_index(content_hash, ddm->hash_table_size);
    entry = find_hash_entry(ddm, content_hash, hash_index);
    
    if (entry) {
        // 找到重复内容
        entry->reference_count++;
        entry->last_access = get_time_stamp();
        *block_id = entry->block_number;
        *is_duplicate = true;
        
        ddm->stats.duplicate_blocks_found++;
        ddm->stats.space_saved_bytes += size;
        
        DMSG("Deduplication hit: block_id=%u, ref_count=%u",
             entry->block_number, entry->reference_count);
    } else {
        // 新内容，存储到块池
        if (ddm->block_pool.next_block_id >= ddm->block_pool.pool_size) {
            res = expand_block_pool(ddm);
            if (res != TEE_SUCCESS)
                goto out;
        }
        
        uint32_t new_block_id = ddm->block_pool.next_block_id++;
        
        // 分配块存储
        ddm->block_pool.blocks[new_block_id] = malloc(size);
        if (!ddm->block_pool.blocks[new_block_id]) {
            res = TEE_ERROR_OUT_OF_MEMORY;
            goto out;
        }
        
        memcpy(ddm->block_pool.blocks[new_block_id], data, size);
        ddm->block_pool.reference_counts[new_block_id] = 1;
        
        // 添加哈希表条目
        res = add_hash_entry(ddm, content_hash, new_block_id, size);
        if (res != TEE_SUCCESS) {
            free(ddm->block_pool.blocks[new_block_id]);
            ddm->block_pool.blocks[new_block_id] = NULL;
            goto out;
        }
        
        *block_id = new_block_id;
        *is_duplicate = false;
    }
    
    ddm->stats.total_blocks_processed++;
    
    // 更新去重比率
    if (ddm->stats.total_blocks_processed > 0) {
        ddm->stats.deduplication_ratio = 
            (double)ddm->stats.duplicate_blocks_found / 
            ddm->stats.total_blocks_processed;
    }
    
out:
    mutex_unlock(&ddm->dedup_mutex);
    return res;
}
```

这些高级存储特性的补充进一步完善了 OP-TEE TADB 存储架构分析，涵盖了从底层加密密钥管理到高级优化技术的完整技术栈，为实现企业级安全存储系统提供了全面的技术参考和实现指导。

## 高级并发控制和存储一致性

### 多版本并发控制 (MVCC)

#### 1. 时间戳排序并发控制

```c
// 时间戳版本控制
struct tadb_timestamp_manager {
    uint64_t global_timestamp;              // 全局时间戳
    struct mutex ts_mutex;                  // 时间戳互斥锁
    
    // 事务时间戳
    struct {
        uint64_t start_ts;                  // 事务开始时间戳
        uint64_t commit_ts;                 // 事务提交时间戳
        bool read_only;                     // 只读事务标志
    } transaction_info[MAX_CONCURRENT_TXN];
    
    // 版本清理策略
    struct {
        uint64_t min_active_ts;             // 最小活跃时间戳
        uint64_t gc_threshold_ts;           // GC 阈值时间戳
        uint32_t version_retention_count;   // 版本保留数量
    } gc_policy;
};

// 版本化存储条目
struct versioned_tadb_entry {
    struct tadb_entry base_entry;          // 基础条目
    uint64_t create_timestamp;             // 创建时间戳
    uint64_t delete_timestamp;             // 删除时间戳 (MAX_UINT64 = 未删除)
    struct versioned_tadb_entry *prev_version; // 前一个版本
    struct versioned_tadb_entry *next_version; // 下一个版本
    
    // 版本元数据
    struct {
        uint32_t version_id;               // 版本 ID
        uint64_t txn_id;                   // 创建事务 ID
        bool committed;                    // 提交状态
        uint32_t read_count;               // 当前读取计数
    } version_meta;
};

// MVCC 读取操作
static TEE_Result mvcc_read_entry(struct tadb_timestamp_manager *tsm,
                                 const TEE_UUID *uuid,
                                 uint64_t read_timestamp,
                                 struct tadb_entry **entry)
{
    struct versioned_tadb_entry *v_entry = find_versioned_entry(uuid);
    struct versioned_tadb_entry *best_version = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    if (!v_entry)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    // 查找适合读取时间戳的版本
    struct versioned_tadb_entry *current = v_entry;
    while (current) {
        // 检查版本可见性
        if (current->create_timestamp <= read_timestamp &&
            current->delete_timestamp > read_timestamp &&
            current->version_meta.committed) {
            
            best_version = current;
            break;
        }
        current = current->prev_version;
    }
    
    if (!best_version) {
        return TEE_ERROR_ITEM_NOT_FOUND;
    }
    
    // 增加读取计数
    __atomic_add_fetch(&best_version->version_meta.read_count, 1, __ATOMIC_SEQ_CST);
    
    *entry = &best_version->base_entry;
    return TEE_SUCCESS;
}

// MVCC 写入操作
static TEE_Result mvcc_write_entry(struct tadb_timestamp_manager *tsm,
                                  const TEE_UUID *uuid,
                                  const struct tadb_entry *new_entry,
                                  uint64_t write_timestamp)
{
    struct versioned_tadb_entry *v_entry = find_versioned_entry(uuid);
    struct versioned_tadb_entry *new_version;
    TEE_Result res = TEE_SUCCESS;
    
    // 创建新版本
    new_version = calloc(1, sizeof(*new_version));
    if (!new_version)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 复制数据
    memcpy(&new_version->base_entry, new_entry, sizeof(*new_entry));
    new_version->create_timestamp = write_timestamp;
    new_version->delete_timestamp = UINT64_MAX;
    new_version->version_meta.version_id = get_next_version_id();
    new_version->version_meta.committed = false;
    
    // 链接版本
    if (v_entry) {
        new_version->prev_version = v_entry;
        v_entry->next_version = new_version;
    }
    
    // 更新版本链头
    update_version_chain_head(uuid, new_version);
    
    return TEE_SUCCESS;
}

// 版本清理机制
static TEE_Result mvcc_garbage_collect(struct tadb_timestamp_manager *tsm)
{
    uint64_t min_active_ts = get_min_active_timestamp(tsm);
    struct versioned_tadb_entry *entry, *temp;
    uint32_t cleaned_versions = 0;
    
    // 遍历所有版本链
    for (entry = get_first_version_entry(); entry; entry = get_next_version_entry(entry)) {
        struct versioned_tadb_entry *current = entry;
        
        while (current && current->prev_version) {
            struct versioned_tadb_entry *prev = current->prev_version;
            
            // 检查是否可以清理
            if (prev->delete_timestamp < min_active_ts &&
                prev->version_meta.read_count == 0) {
                
                // 解除链接
                current->prev_version = prev->prev_version;
                if (prev->prev_version) {
                    prev->prev_version->next_version = current;
                }
                
                // 释放版本
                free(prev);
                cleaned_versions++;
            } else {
                current = prev;
            }
        }
    }
    
    DMSG("MVCC GC: cleaned %u versions", cleaned_versions);
    return TEE_SUCCESS;
}
```

#### 2. 乐观并发控制

```c
// 乐观锁条目
struct optimistic_lock_entry {
    TEE_UUID uuid;                         // 条目 UUID
    uint64_t version_number;               // 版本号
    uint64_t last_modified;                // 最后修改时间
    uint32_t checksum;                     // 数据校验和
    struct mutex validation_mutex;         // 验证互斥锁
};

// 乐观读取
static TEE_Result optimistic_read_begin(const TEE_UUID *uuid,
                                       struct optimistic_lock_entry *lock_entry)
{
    struct tadb_entry *entry = find_entry_by_uuid(uuid);
    
    if (!entry)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    // 记录读取时的版本信息
    lock_entry->uuid = *uuid;
    lock_entry->version_number = entry->version_number;
    lock_entry->last_modified = entry->last_modified;
    lock_entry->checksum = calculate_checksum(entry);
    
    return TEE_SUCCESS;
}

// 乐观读取验证
static TEE_Result optimistic_read_validate(const struct optimistic_lock_entry *lock_entry)
{
    struct tadb_entry *entry = find_entry_by_uuid(&lock_entry->uuid);
    
    if (!entry)
        return TEE_ERROR_ITEM_NOT_FOUND;
    
    // 检查版本是否发生变化
    if (entry->version_number != lock_entry->version_number ||
        entry->last_modified != lock_entry->last_modified ||
        calculate_checksum(entry) != lock_entry->checksum) {
        return TEE_ERROR_CONFLICT;
    }
    
    return TEE_SUCCESS;
}

// 乐观写入
static TEE_Result optimistic_write_commit(const struct optimistic_lock_entry *lock_entry,
                                         const struct tadb_entry *new_entry)
{
    TEE_Result res = TEE_SUCCESS;
    
    mutex_lock(&((struct optimistic_lock_entry*)lock_entry)->validation_mutex);
    
    // 验证读取的版本仍然有效
    res = optimistic_read_validate(lock_entry);
    if (res != TEE_SUCCESS) {
        mutex_unlock(&((struct optimistic_lock_entry*)lock_entry)->validation_mutex);
        return res;
    }
    
    // 执行写入操作
    struct tadb_entry *entry = find_entry_by_uuid(&lock_entry->uuid);
    if (entry) {
        memcpy(entry, new_entry, sizeof(*new_entry));
        entry->version_number++;
        entry->last_modified = get_time_stamp();
    }
    
    mutex_unlock(&((struct optimistic_lock_entry*)lock_entry)->validation_mutex);
    return TEE_SUCCESS;
}
```

### 分布式存储一致性

#### 1. 分片和副本管理

```c
// 存储分片配置
#define TADB_MAX_SHARDS           8
#define TADB_REPLICATION_FACTOR   3

struct tadb_shard_config {
    uint32_t shard_id;                     // 分片 ID
    uint32_t hash_range_start;             // 哈希范围开始
    uint32_t hash_range_end;               // 哈希范围结束
    
    // 副本节点
    struct {
        uint32_t node_id;                  // 节点 ID
        bool is_primary;                   // 是否为主节点
        bool is_healthy;                   // 节点健康状态
        uint64_t last_heartbeat;           // 最后心跳时间
    } replicas[TADB_REPLICATION_FACTOR];
    
    // 分片状态
    enum shard_state {
        SHARD_STATE_ACTIVE,                // 活跃
        SHARD_STATE_MIGRATING,             // 迁移中
        SHARD_STATE_OFFLINE,               // 离线
        SHARD_STATE_RECOVERING,            // 恢复中
    } state;
    
    struct mutex shard_mutex;              // 分片互斥锁
};

// 分片管理器
struct tadb_shard_manager {
    struct tadb_shard_config shards[TADB_MAX_SHARDS];
    uint32_t num_active_shards;            // 活跃分片数
    struct mutex manager_mutex;            // 管理器互斥锁
    
    // 一致性哈希环
    struct {
        uint32_t ring_positions[TADB_MAX_SHARDS * TADB_REPLICATION_FACTOR];
        uint32_t ring_size;                // 环大小
    } consistent_hash;
    
    // 分片策略
    struct {
        enum sharding_strategy {
            SHARD_BY_UUID_HASH,            // 按 UUID 哈希分片
            SHARD_BY_TA_HASH,              // 按 TA 哈希分片
            SHARD_BY_RANGE,                // 按范围分片
        } strategy;
        
        bool auto_rebalance;               // 自动重平衡
        uint32_t rebalance_threshold;      // 重平衡阈值
    } sharding_policy;
};

// 分片选择算法
static uint32_t select_shard_for_uuid(struct tadb_shard_manager *sm,
                                     const TEE_UUID *uuid)
{
    uint32_t hash = crc32_hash((const uint8_t *)uuid, sizeof(*uuid));
    
    switch (sm->sharding_policy.strategy) {
    case SHARD_BY_UUID_HASH:
        return hash % sm->num_active_shards;
        
    case SHARD_BY_TA_HASH:
        // 使用 UUID 的前 8 字节作为 TA 标识
        hash = crc32_hash((const uint8_t *)uuid, 8);
        return hash % sm->num_active_shards;
        
    case SHARD_BY_RANGE:
        // 基于一致性哈希环
        return find_shard_in_consistent_hash(&sm->consistent_hash, hash);
        
    default:
        return 0;
    }
}

// 分布式读取操作
static TEE_Result distributed_read_entry(struct tadb_shard_manager *sm,
                                        const TEE_UUID *uuid,
                                        struct tadb_entry **entry)
{
    uint32_t shard_id = select_shard_for_uuid(sm, uuid);
    struct tadb_shard_config *shard = &sm->shards[shard_id];
    TEE_Result res = TEE_ERROR_GENERIC;
    
    mutex_lock(&shard->shard_mutex);
    
    // 尝试从主副本读取
    for (int i = 0; i < TADB_REPLICATION_FACTOR; i++) {
        if (shard->replicas[i].is_primary && shard->replicas[i].is_healthy) {
            res = read_from_replica(shard->replicas[i].node_id, uuid, entry);
            if (res == TEE_SUCCESS) {
                goto out;
            }
        }
    }
    
    // 主副本不可用，尝试从其他副本读取
    for (int i = 0; i < TADB_REPLICATION_FACTOR; i++) {
        if (!shard->replicas[i].is_primary && shard->replicas[i].is_healthy) {
            res = read_from_replica(shard->replicas[i].node_id, uuid, entry);
            if (res == TEE_SUCCESS) {
                DMSG("Fallback read from replica %u", shard->replicas[i].node_id);
                goto out;
            }
        }
    }
    
out:
    mutex_unlock(&shard->shard_mutex);
    return res;
}

// 分布式写入操作 (多数派写入)
static TEE_Result distributed_write_entry(struct tadb_shard_manager *sm,
                                         const TEE_UUID *uuid,
                                         const struct tadb_entry *entry)
{
    uint32_t shard_id = select_shard_for_uuid(sm, uuid);
    struct tadb_shard_config *shard = &sm->shards[shard_id];
    uint32_t successful_writes = 0;
    uint32_t required_writes = (TADB_REPLICATION_FACTOR / 2) + 1; // 多数派
    TEE_Result res = TEE_ERROR_GENERIC;
    
    mutex_lock(&shard->shard_mutex);
    
    // 向所有健康副本写入
    for (int i = 0; i < TADB_REPLICATION_FACTOR; i++) {
        if (shard->replicas[i].is_healthy) {
            TEE_Result write_res = write_to_replica(shard->replicas[i].node_id, uuid, entry);
            if (write_res == TEE_SUCCESS) {
                successful_writes++;
            }
        }
    }
    
    // 检查是否满足多数派要求
    if (successful_writes >= required_writes) {
        res = TEE_SUCCESS;
        DMSG("Distributed write successful: %u/%u replicas",
             successful_writes, TADB_REPLICATION_FACTOR);
    } else {
        res = TEE_ERROR_COMMUNICATION;
        EMSG("Distributed write failed: only %u/%u replicas succeeded",
             successful_writes, required_writes);
    }
    
    mutex_unlock(&shard->shard_mutex);
    return res;
}
```

#### 2. 分布式共识算法 (简化版 Raft)

```c
// Raft 节点状态
enum raft_node_state {
    RAFT_STATE_FOLLOWER,                   // 跟随者
    RAFT_STATE_CANDIDATE,                  // 候选者
    RAFT_STATE_LEADER,                     // 领导者
};

// Raft 日志条目
struct raft_log_entry {
    uint64_t term;                         // 任期号
    uint64_t index;                        // 日志索引
    enum log_entry_type {
        LOG_ENTRY_TADB_INSERT,             // TADB 插入
        LOG_ENTRY_TADB_UPDATE,             // TADB 更新
        LOG_ENTRY_TADB_DELETE,             // TADB 删除
        LOG_ENTRY_NOOP,                    // 空操作
    } type;
    
    size_t data_len;                       // 数据长度
    uint8_t data[];                        // 数据内容
};

// Raft 状态机
struct raft_state_machine {
    enum raft_node_state state;           // 节点状态
    uint32_t node_id;                     // 节点 ID
    uint64_t current_term;                // 当前任期
    uint32_t voted_for;                   // 投票对象
    uint64_t commit_index;                // 提交索引
    uint64_t last_applied;                // 最后应用索引
    
    // 日志
    struct {
        struct raft_log_entry **entries;  // 日志条目数组
        size_t log_size;                   // 日志大小
        uint64_t last_log_index;           // 最后日志索引
        uint64_t last_log_term;            // 最后日志任期
    } log;
    
    // 领导者状态
    struct {
        uint64_t *next_index;              // 下一个发送索引
        uint64_t *match_index;             // 匹配索引
        uint64_t last_heartbeat;           // 最后心跳时间
    } leader;
    
    // 集群配置
    struct {
        uint32_t *cluster_nodes;           // 集群节点列表
        size_t cluster_size;               // 集群大小
        uint32_t majority_size;            // 多数派大小
    } cluster;
    
    struct mutex raft_mutex;              // Raft 互斥锁
};

// Raft 日志复制
static TEE_Result raft_replicate_log(struct raft_state_machine *raft,
                                    const struct raft_log_entry *entry)
{
    TEE_Result res = TEE_SUCCESS;
    uint32_t success_count = 1; // 包括领导者自己
    
    if (raft->state != RAFT_STATE_LEADER) {
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    mutex_lock(&raft->raft_mutex);
    
    // 添加到本地日志
    res = append_log_entry(raft, entry);
    if (res != TEE_SUCCESS) {
        goto out;
    }
    
    // 向所有跟随者发送日志
    for (size_t i = 0; i < raft->cluster.cluster_size; i++) {
        if (raft->cluster.cluster_nodes[i] == raft->node_id) {
            continue; // 跳过自己
        }
        
        TEE_Result send_res = send_append_entries(raft, raft->cluster.cluster_nodes[i], entry);
        if (send_res == TEE_SUCCESS) {
            success_count++;
        }
    }
    
    // 检查是否获得多数派同意
    if (success_count >= raft->cluster.majority_size) {
        raft->commit_index = raft->log.last_log_index;
        res = TEE_SUCCESS;
        DMSG("Log entry committed: index=%"PRIu64, raft->log.last_log_index);
    } else {
        res = TEE_ERROR_COMMUNICATION;
        EMSG("Failed to replicate log: only %u/%zu nodes responded",
             success_count, raft->cluster.cluster_size);
    }
    
out:
    mutex_unlock(&raft->raft_mutex);
    return res;
}

// Raft 状态机应用
static TEE_Result raft_apply_committed_entries(struct raft_state_machine *raft)
{
    TEE_Result res = TEE_SUCCESS;
    
    mutex_lock(&raft->raft_mutex);
    
    // 应用已提交但未应用的日志条目
    while (raft->last_applied < raft->commit_index) {
        raft->last_applied++;
        
        struct raft_log_entry *entry = raft->log.entries[raft->last_applied - 1];
        if (!entry) {
            continue;
        }
        
        // 根据条目类型应用到 TADB
        switch (entry->type) {
        case LOG_ENTRY_TADB_INSERT:
            res = apply_tadb_insert(entry->data, entry->data_len);
            break;
            
        case LOG_ENTRY_TADB_UPDATE:
            res = apply_tadb_update(entry->data, entry->data_len);
            break;
            
        case LOG_ENTRY_TADB_DELETE:
            res = apply_tadb_delete(entry->data, entry->data_len);
            break;
            
        case LOG_ENTRY_NOOP:
            // 空操作，用于心跳
            res = TEE_SUCCESS;
            break;
            
        default:
            EMSG("Unknown log entry type: %u", entry->type);
            res = TEE_ERROR_BAD_PARAMETERS;
            break;
        }
        
        if (res != TEE_SUCCESS) {
            EMSG("Failed to apply log entry %"PRIu64": result=0x%x",
                 raft->last_applied, res);
            break;
        }
    }
    
    mutex_unlock(&raft->raft_mutex);
    return res;
}
```

### 高级存储性能优化

#### 1. 自适应缓存策略

```c
// 智能缓存管理
struct adaptive_cache_manager {
    // 缓存层次
    struct {
        struct cache_layer *l1_cache;     // L1 缓存 (热数据)
        struct cache_layer *l2_cache;     // L2 缓存 (温数据)
        struct cache_layer *l3_cache;     // L3 缓存 (冷数据)
    } layers;
    
    // 访问模式分析
    struct {
        uint64_t total_accesses;           // 总访问次数
        uint64_t sequential_accesses;      // 顺序访问次数
        uint64_t random_accesses;          // 随机访问次数
        double locality_factor;            // 局部性因子
        enum access_pattern {
            PATTERN_RANDOM,                // 随机模式
            PATTERN_SEQUENTIAL,            // 顺序模式
            PATTERN_MIXED,                 // 混合模式
        } detected_pattern;
    } access_analysis;
    
    // 自适应策略
    struct {
        bool adaptive_prefetch;            // 自适应预取
        uint32_t prefetch_window_size;     // 预取窗口大小
        bool dynamic_cache_size;           // 动态缓存大小
        uint32_t cache_hit_threshold;      // 缓存命中阈值
    } adaptive_policy;
    
    struct mutex cache_mutex;             // 缓存互斥锁
};

// 访问模式检测
static void detect_access_pattern(struct adaptive_cache_manager *acm,
                                 uint32_t file_number)
{
    static uint32_t last_file_number = UINT32_MAX;
    static uint64_t sequential_count = 0;
    
    mutex_lock(&acm->cache_mutex);
    
    acm->access_analysis.total_accesses++;
    
    // 检测顺序访问
    if (last_file_number != UINT32_MAX) {
        if (file_number == last_file_number + 1 ||
            file_number == last_file_number - 1) {
            sequential_count++;
            acm->access_analysis.sequential_accesses++;
        } else {
            sequential_count = 0;
            acm->access_analysis.random_accesses++;
        }
    }
    
    last_file_number = file_number;
    
    // 更新局部性因子
    if (acm->access_analysis.total_accesses > 0) {
        acm->access_analysis.locality_factor = 
            (double)acm->access_analysis.sequential_accesses / 
            acm->access_analysis.total_accesses;
    }
    
    // 确定访问模式
    if (acm->access_analysis.locality_factor > 0.7) {
        acm->access_analysis.detected_pattern = PATTERN_SEQUENTIAL;
    } else if (acm->access_analysis.locality_factor < 0.3) {
        acm->access_analysis.detected_pattern = PATTERN_RANDOM;
    } else {
        acm->access_analysis.detected_pattern = PATTERN_MIXED;
    }
    
    // 调整预取策略
    if (sequential_count > 3 && acm->adaptive_policy.adaptive_prefetch) {
        acm->adaptive_policy.prefetch_window_size = MIN(16, sequential_count * 2);
    } else {
        acm->adaptive_policy.prefetch_window_size = 1;
    }
    
    mutex_unlock(&acm->cache_mutex);
}

// 智能预取算法
static TEE_Result adaptive_prefetch(struct adaptive_cache_manager *acm,
                                   uint32_t current_file_number)
{
    TEE_Result res = TEE_SUCCESS;
    
    if (!acm->adaptive_policy.adaptive_prefetch) {
        return TEE_SUCCESS;
    }
    
    switch (acm->access_analysis.detected_pattern) {
    case PATTERN_SEQUENTIAL:
        // 顺序预取
        for (uint32_t i = 1; i <= acm->adaptive_policy.prefetch_window_size; i++) {
            uint32_t prefetch_file = current_file_number + i;
            res = prefetch_file_async(prefetch_file);
            if (res != TEE_SUCCESS) {
                break;
            }
        }
        break;
        
    case PATTERN_RANDOM:
        // 随机模式下减少预取
        if (get_cache_hit_rate() < 0.5) {
            // 缓存命中率低，尝试预取相关文件
            res = prefetch_related_files(current_file_number, 2);
        }
        break;
        
    case PATTERN_MIXED:
        // 混合模式下适度预取
        res = prefetch_file_async(current_file_number + 1);
        break;
    }
    
    return res;
}

// 动态缓存大小调整
static TEE_Result adjust_cache_sizes(struct adaptive_cache_manager *acm)
{
    if (!acm->adaptive_policy.dynamic_cache_size) {
        return TEE_SUCCESS;
    }
    
    double hit_rate = get_cache_hit_rate();
    size_t total_memory = get_available_cache_memory();
    
    if (hit_rate > 0.9) {
        // 命中率高，可以减小缓存
        resize_cache_layer(acm->layers.l1_cache, total_memory * 0.4);
        resize_cache_layer(acm->layers.l2_cache, total_memory * 0.35);
        resize_cache_layer(acm->layers.l3_cache, total_memory * 0.25);
    } else if (hit_rate < 0.6) {
        // 命中率低，增大缓存
        resize_cache_layer(acm->layers.l1_cache, total_memory * 0.5);
        resize_cache_layer(acm->layers.l2_cache, total_memory * 0.4);
        resize_cache_layer(acm->layers.l3_cache, total_memory * 0.1);
    }
    
    return TEE_SUCCESS;
}
```

#### 2. 存储负载均衡

```c
// 负载均衡器
struct storage_load_balancer {
    // 存储节点信息
    struct storage_node {
        uint32_t node_id;                  // 节点 ID
        char endpoint[64];                 // 节点地址
        
        // 负载指标
        struct {
            uint32_t active_connections;   // 活跃连接数
            uint64_t bytes_per_second;     // 每秒字节数
            uint64_t operations_per_second; // 每秒操作数
            double cpu_usage;              // CPU 使用率
            double memory_usage;           // 内存使用率
            uint64_t average_latency_us;   // 平均延迟
        } load_metrics;
        
        // 健康状态
        struct {
            bool is_healthy;               // 健康状态
            uint64_t last_heartbeat;       // 最后心跳
            uint32_t consecutive_failures; // 连续失败次数
            double availability;           // 可用性百分比
        } health;
        
        struct mutex node_mutex;           // 节点互斥锁
    } nodes[MAX_STORAGE_NODES];
    
    size_t num_nodes;                      // 节点数量
    
    // 负载均衡策略
    enum lb_strategy {
        LB_ROUND_ROBIN,                    // 轮询
        LB_LEAST_CONNECTIONS,              // 最少连接
        LB_WEIGHTED_ROUND_ROBIN,           // 加权轮询
        LB_LEAST_RESPONSE_TIME,            // 最短响应时间
        LB_HASH_BASED,                     // 基于哈希
    } strategy;
    
    // 权重配置
    struct {
        double latency_weight;             // 延迟权重
        double cpu_weight;                 // CPU 权重
        double memory_weight;              // 内存权重
        double throughput_weight;          // 吞吐量权重
    } weights;
    
    struct mutex lb_mutex;                 // 负载均衡器互斥锁
};

// 节点选择算法
static uint32_t select_storage_node(struct storage_load_balancer *lb,
                                   const TEE_UUID *uuid)
{
    uint32_t selected_node = 0;
    double best_score = -1.0;
    
    mutex_lock(&lb->lb_mutex);
    
    switch (lb->strategy) {
    case LB_ROUND_ROBIN:
        {
            static uint32_t round_robin_index = 0;
            do {
                selected_node = round_robin_index % lb->num_nodes;
                round_robin_index++;
            } while (!lb->nodes[selected_node].health.is_healthy);
        }
        break;
        
    case LB_LEAST_CONNECTIONS:
        {
            uint32_t min_connections = UINT32_MAX;
            for (size_t i = 0; i < lb->num_nodes; i++) {
                if (lb->nodes[i].health.is_healthy &&
                    lb->nodes[i].load_metrics.active_connections < min_connections) {
                    min_connections = lb->nodes[i].load_metrics.active_connections;
                    selected_node = i;
                }
            }
        }
        break;
        
    case LB_LEAST_RESPONSE_TIME:
        {
            uint64_t min_latency = UINT64_MAX;
            for (size_t i = 0; i < lb->num_nodes; i++) {
                if (lb->nodes[i].health.is_healthy &&
                    lb->nodes[i].load_metrics.average_latency_us < min_latency) {
                    min_latency = lb->nodes[i].load_metrics.average_latency_us;
                    selected_node = i;
                }
            }
        }
        break;
        
    case LB_WEIGHTED_ROUND_ROBIN:
        {
            // 基于综合指标计算权重
            for (size_t i = 0; i < lb->num_nodes; i++) {
                if (!lb->nodes[i].health.is_healthy) {
                    continue;
                }
                
                double score = calculate_node_score(lb, i);
                if (score > best_score) {
                    best_score = score;
                    selected_node = i;
                }
            }
        }
        break;
        
    case LB_HASH_BASED:
        {
            // 基于 UUID 的一致性哈希
            uint32_t hash = crc32_hash((const uint8_t *)uuid, sizeof(*uuid));
            selected_node = hash % lb->num_nodes;
            
            // 如果选中的节点不健康，寻找下一个健康节点
            while (!lb->nodes[selected_node].health.is_healthy) {
                selected_node = (selected_node + 1) % lb->num_nodes;
            }
        }
        break;
    }
    
    // 更新节点负载
    lb->nodes[selected_node].load_metrics.active_connections++;
    
    mutex_unlock(&lb->lb_mutex);
    return selected_node;
}

// 节点评分算法
static double calculate_node_score(struct storage_load_balancer *lb, size_t node_index)
{
    struct storage_node *node = &lb->nodes[node_index];
    double score = 0.0;
    
    // 延迟评分 (延迟越低评分越高)
    double latency_score = 1.0 / (1.0 + node->load_metrics.average_latency_us / 1000.0);
    score += latency_score * lb->weights.latency_weight;
    
    // CPU 使用率评分 (使用率越低评分越高)
    double cpu_score = 1.0 - node->load_metrics.cpu_usage;
    score += cpu_score * lb->weights.cpu_weight;
    
    // 内存使用率评分
    double memory_score = 1.0 - node->load_metrics.memory_usage;
    score += memory_score * lb->weights.memory_weight;
    
    // 吞吐量评分
    double throughput_score = node->load_metrics.bytes_per_second / 1024.0 / 1024.0; // MB/s
    score += throughput_score * lb->weights.throughput_weight;
    
    // 可用性调整
    score *= node->health.availability;
    
    return score;
}

// 健康检查
static TEE_Result perform_health_check(struct storage_load_balancer *lb)
{
    for (size_t i = 0; i < lb->num_nodes; i++) {
        struct storage_node *node = &lb->nodes[i];
        
        mutex_lock(&node->node_mutex);
        
        // 发送心跳请求
        TEE_Result res = send_heartbeat(node->node_id);
        uint64_t current_time = get_time_stamp();
        
        if (res == TEE_SUCCESS) {
            node->health.last_heartbeat = current_time;
            node->health.consecutive_failures = 0;
            node->health.is_healthy = true;
            
            // 更新可用性统计
            update_availability_stats(node);
        } else {
            node->health.consecutive_failures++;
            
            // 连续失败超过阈值则标记为不健康
            if (node->health.consecutive_failures >= HEALTH_CHECK_FAILURE_THRESHOLD) {
                node->health.is_healthy = false;
                EMSG("Node %u marked as unhealthy after %u consecutive failures",
                     node->node_id, node->health.consecutive_failures);
            }
        }
        
        mutex_unlock(&node->node_mutex);
    }
    
    return TEE_SUCCESS;
}
```

这些高级并发控制和存储一致性机制的补充为 OP-TEE TADB 存储架构提供了：

1. **多版本并发控制(MVCC)**: 支持高并发读写访问，避免读写冲突
2. **乐观并发控制**: 减少锁争用，提高并发性能
3. **分布式存储一致性**: 支持多节点部署和数据一致性保证
4. **分布式共识算法**: 基于 Raft 的强一致性保证
5. **自适应缓存策略**: 智能的缓存管理和预取机制
6. **存储负载均衡**: 动态负载分配和健康监控

这些特性使得 OP-TEE TADB 成为一个真正企业级的分布式安全存储系统，能够应对大规模部署和高并发访问的挑战。