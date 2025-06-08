# OP-TEE高级密钥管理深度分析

## 密钥管理架构概述

OP-TEE实现了一个复杂的分层密钥管理系统，从硬件信任根(HUK)开始，通过密码学方法派生出各种用途的子密钥。这个系统不仅支持存储加密，还支持TA加密、RPMB认证等多种安全功能。

## 硬件唯一密钥 (HUK) 系统

### 1. HUK的核心概念

```c
// HUK长度定义
#define HW_UNIQUE_KEY_LENGTH    32      // 256位硬件唯一密钥

struct tee_hw_unique_key {
    uint8_t data[HW_UNIQUE_KEY_LENGTH]; // 32字节HUK数据
};
```

#### HUK的特性
1. **硬件唯一性**: 每个芯片/设备拥有唯一的HUK
2. **不可提取**: HUK存储在硬件安全区域，软件无法直接读取
3. **固化密钥**: 在芯片制造或初始化时写入，不可更改
4. **信任根基**: 所有其他密钥的派生基础

### 2. HUK获取接口

```c
// 获取硬件唯一密钥
TEE_Result tee_otp_get_hw_unique_key(struct tee_hw_unique_key *hwkey);

// 典型平台实现示例
TEE_Result platform_get_hw_unique_key(struct tee_hw_unique_key *hwkey)
{
    // 从硬件安全存储读取HUK
    // 实现方式因平台而异:
    // - ARM TrustZone OTP
    // - HSM (Hardware Security Module)
    // - eFuse
    // - 安全元件
    return read_hw_unique_key_from_secure_storage(hwkey->data);
}
```

## HUK子密钥派生系统

### 1. 子密钥用途分类

```c
enum huk_subkey_usage {
    HUK_SUBKEY_RPMB = 0,        // RPMB认证密钥
    HUK_SUBKEY_SSK = 1,         // 安全存储密钥
    HUK_SUBKEY_DIE_ID = 2,      // 芯片标识符
    HUK_SUBKEY_UNIQUE_TA = 3,   // TA唯一标识密钥
    HUK_SUBKEY_TA_ENC = 4,      // TA加密密钥
    HUK_SUBKEY_SE050 = 5,       // SE050安全元件密钥
    // 新增用途ID需要保持常量，以确保密钥派生一致性
};

#define HUK_SUBKEY_MAX_LEN    TEE_SHA256_HASH_SIZE  // 32字节最大长度
```

### 2. 子密钥派生算法

#### 标准派生方法
```c
TEE_Result __huk_subkey_derive(enum huk_subkey_usage usage,
                              const void *const_data, size_t const_data_len,
                              uint8_t *subkey, size_t subkey_len)
{
    void *ctx = NULL;
    struct tee_hw_unique_key huk = { };
    TEE_Result res;
    
    // 1. 参数验证
    if (subkey_len > HUK_SUBKEY_MAX_LEN)
        return TEE_ERROR_BAD_PARAMETERS;
    if (!const_data && const_data_len)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 2. 分配HMAC上下文
    res = crypto_mac_alloc_ctx(&ctx, TEE_ALG_HMAC_SHA256);
    if (res)
        return res;
    
    // 3. 获取HUK
    res = tee_otp_get_hw_unique_key(&huk);
    if (res)
        goto out;
    
    // 4. 初始化HMAC (HUK作为密钥)
    res = crypto_mac_init(ctx, huk.data, sizeof(huk.data));
    if (res)
        goto out;
    
    // 5. 更新用途标识符
    res = mac_usage(ctx, usage);
    if (res)
        goto out;
    
    // 6. 更新常量数据 (如果提供)
    if (const_data) {
        res = crypto_mac_update(ctx, const_data, const_data_len);
        if (res)
            goto out;
    }
    
    // 7. 计算最终子密钥
    res = crypto_mac_final(ctx, subkey, subkey_len);
    
out:
    crypto_mac_free_ctx(ctx);
    memzero_explicit(&huk, sizeof(huk));  // 清除HUK
    return res;
}

// 用途标识符MAC更新
static TEE_Result mac_usage(void *ctx, uint32_t usage)
{
    return crypto_mac_update(ctx, (const void *)&usage, sizeof(usage));
}
```

#### 子密钥派生公式
```
SubKey = HMAC-SHA256(HUK, Usage_ID || Const_Data)

其中:
- HUK: 硬件唯一密钥 (32字节)
- Usage_ID: 用途标识符 (4字节)
- Const_Data: 可选的常量数据 (可变长度)
- ||: 连接操作
```

### 3. 兼容性处理

#### 传统兼容模式
```c
#ifdef CFG_CORE_HUK_SUBKEY_COMPAT
static TEE_Result huk_compat(void *ctx, enum huk_subkey_usage usage)
{
    TEE_Result res = TEE_SUCCESS;
    uint8_t chip_id[TEE_FS_KM_CHIP_ID_LENGTH] = { 0 };
    static uint8_t ssk_str[] = "ONLY_FOR_tee_fs_ssk";
    
    switch (usage) {
        case HUK_SUBKEY_RPMB:
            // RPMB密钥保持新格式
            return TEE_SUCCESS;
            
        case HUK_SUBKEY_SSK:
            // SSK使用传统派生方法保持向后兼容
            res = get_otp_die_id(chip_id, sizeof(chip_id));
            if (res)
                return res;
            res = crypto_mac_update(ctx, chip_id, sizeof(chip_id));
            if (res)
                return res;
            return crypto_mac_update(ctx, ssk_str, sizeof(ssk_str));
            
        default:
            // 其他密钥使用标准方法
            return mac_usage(ctx, usage);
    }
}

// 传统SSK派生公式:
// SSK = HMAC-SHA256(HUK, Die_ID || "ONLY_FOR_tee_fs_ssk")
#endif
```

## 存储密钥管理

### 1. 安全存储密钥 (SSK) 管理

#### SSK初始化
```c
// 位置: /core/tee/tee_fs_key_manager.c
struct tee_fs_ssk {
    bool is_init;                           // 初始化标志
    uint8_t key[TEE_FS_KM_SSK_SIZE];       // SSK数据 (32字节)
};

static struct tee_fs_ssk tee_fs_ssk;       // 全局SSK实例

static TEE_Result tee_fs_init_key_manager(void)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 编译时断言: SSK大小不超过子密钥最大长度
    COMPILE_TIME_ASSERT(TEE_FS_KM_SSK_SIZE <= HUK_SUBKEY_MAX_LEN);
    
    // 从HUK派生SSK
    res = huk_subkey_derive(HUK_SUBKEY_SSK, NULL, 0,
                           tee_fs_ssk.key, sizeof(tee_fs_ssk.key));
    if (res == TEE_SUCCESS)
        tee_fs_ssk.is_init = 1;
    else
        memzero_explicit(&tee_fs_ssk, sizeof(tee_fs_ssk));
    
    return res;
}

// 服务后期初始化
service_init_late(tee_fs_init_key_manager);
```

### 2. 信任应用存储密钥 (TSK) 派生

#### 基于TA UUID的TSK派生
```c
static TEE_Result derive_tsk_for_ta(const TEE_UUID *uuid, uint8_t *tsk, size_t tsk_size)
{
    TEE_Result res;
    
    if (uuid) {
        // TSK = HMAC-SHA256(SSK, TA_UUID)
        res = do_hmac(tsk, tsk_size,
                     tee_fs_ssk.key, TEE_FS_KM_SSK_SIZE,
                     uuid, sizeof(*uuid));
    } else {
        // 无UUID的情况 (如系统级操作)
        uint8_t dummy[1] = { 0 };
        res = do_hmac(tsk, tsk_size,
                     tee_fs_ssk.key, TEE_FS_KM_SSK_SIZE,
                     dummy, sizeof(dummy));
    }
    
    return res;
}
```

### 3. 文件加密密钥 (FEK) 管理

#### FEK生成和保护
```c
// FEK生成
TEE_Result tee_fs_generate_fek(const TEE_UUID *uuid, void *buf, size_t buf_size)
{
    TEE_Result res;
    
    if (buf_size != TEE_FS_KM_FEK_SIZE)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 1. 生成随机FEK
    res = generate_fek(buf, TEE_FS_KM_FEK_SIZE);
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 使用TSK加密FEK
    return tee_fs_fek_crypt(uuid, TEE_MODE_ENCRYPT, buf,
                           TEE_FS_KM_FEK_SIZE, buf);
}

// FEK加密/解密
TEE_Result tee_fs_fek_crypt(const TEE_UUID *uuid, TEE_OperationMode mode,
                           const uint8_t *in_key, size_t size,
                           uint8_t *out_key)
{
    TEE_Result res;
    void *ctx = NULL;
    uint8_t tsk[TEE_FS_KM_TSK_SIZE];
    
    // 1. 派生TSK
    res = derive_tsk_for_ta(uuid, tsk, sizeof(tsk));
    if (res != TEE_SUCCESS)
        return res;
    
    // 2. 使用AES-ECB加密/解密FEK
    res = crypto_cipher_alloc_ctx(&ctx, TEE_FS_KM_ENC_FEK_ALG);
    if (res != TEE_SUCCESS)
        goto exit;
    
    res = crypto_cipher_init(ctx, mode, tsk, sizeof(tsk), NULL, 0, NULL, 0);
    if (res != TEE_SUCCESS)
        goto exit;
    
    res = crypto_cipher_update(ctx, mode, true, in_key, size, out_key);
    if (res != TEE_SUCCESS)
        goto exit;
    
    crypto_cipher_final(ctx);
    
exit:
    crypto_cipher_free_ctx(ctx);
    memzero_explicit(tsk, sizeof(tsk));      // 立即清除TSK
    return res;
}
```

## RPMB密钥管理

### 1. RPMB密钥派生策略

#### 生产环境密钥派生
```c
static TEE_Result tee_rpmb_key_gen(uint8_t *key, uint32_t len)
{
    uint8_t message[RPMB_EMMC_CID_SIZE];
    
    if (!key || RPMB_KEY_MAC_SIZE != len)
        return TEE_ERROR_BAD_PARAMETERS;
    
    /*
     * 基于eMMC CID派生RPMB密钥
     * CID字段处理:
     * - 屏蔽PRV (Product Revision) [55:48]
     * - 屏蔽CRC (CRC7 checksum) [07:01]
     * - 保留其他字段用于密钥派生
     */
    memcpy(message, rpmb_ctx->cid, RPMB_EMMC_CID_SIZE);
    memset(message + RPMB_CID_PRV_OFFSET, 0, 1);
    memset(message + RPMB_CID_CRC_OFFSET, 0, 1);
    
    // RPMB_Key = HUK_SubKey(RPMB, Masked_CID)
    return huk_subkey_derive(HUK_SUBKEY_RPMB, message, sizeof(message),
                           key, len);
}
```

#### 测试环境固定密钥
```c
#ifdef CFG_RPMB_TESTKEY
static const uint8_t rpmb_test_key[RPMB_KEY_MAC_SIZE] = {
    0xD3, 0xEB, 0x3E, 0xC3, 0x6E, 0x33, 0x4C, 0x9F,
    0x98, 0x8C, 0xE2, 0xC0, 0xB8, 0x59, 0x54, 0x61,
    0x0D, 0x2B, 0xCF, 0x86, 0x64, 0x84, 0x4D, 0xF2,
    0xAB, 0x56, 0xE6, 0xC6, 0x1B, 0xB7, 0x01, 0xE4
};

static TEE_Result tee_rpmb_key_gen(uint8_t *key, uint32_t len)
{
    if (!key || RPMB_KEY_MAC_SIZE != len)
        return TEE_ERROR_BAD_PARAMETERS;
    
    DMSG("RPMB: Using test key");
    memcpy(key, rpmb_test_key, RPMB_KEY_MAC_SIZE);
    return TEE_SUCCESS;
}
#endif
```

### 2. CID字段分析

#### eMMC CID结构
```c
#define RPMB_EMMC_CID_SIZE      16
#define RPMB_CID_PRV_OFFSET     9   // Product Revision位置
#define RPMB_CID_CRC_OFFSET     15  // CRC7位置

/*
 * eMMC CID (Card IDentification) 结构 (128位):
 * [127:120] MID  - Manufacturer ID
 * [119:104] CBX  - Card/BGA
 * [103:96]  OID  - OEM/Application ID
 * [95:56]   PNM  - Product Name
 * [55:48]   PRV  - Product Revision (可变, FFU时改变)
 * [47:16]   PSN  - Product Serial Number
 * [15:12]   MDT  - Manufacturing Date
 * [11:8]    Reserved
 * [7:1]     CRC7 - CRC7 Checksum (可变)
 * [0]       不使用
 */

// CID掩码处理确保密钥稳定性
static void mask_cid_for_key_derivation(uint8_t *cid)
{
    // 清除可变字段以确保密钥一致性
    memset(cid + RPMB_CID_PRV_OFFSET, 0, 1);  // 清除PRV
    memset(cid + RPMB_CID_CRC_OFFSET, 0, 1);  // 清除CRC
}
```

## TA密钥管理

### 1. TA唯一标识密钥

#### TA Unique Key派生
```c
// TA唯一密钥用于TA身份验证和完整性保护
TEE_Result derive_ta_unique_key(const TEE_UUID *ta_uuid, uint8_t *key, size_t key_len)
{
    // TA_Unique_Key = HUK_SubKey(UNIQUE_TA, TA_UUID)
    return huk_subkey_derive(HUK_SUBKEY_UNIQUE_TA, ta_uuid, sizeof(*ta_uuid),
                           key, key_len);
}
```

### 2. TA加密密钥

#### TA Encryption Key派生
```c
// TA加密密钥用于TA镜像加密
TEE_Result derive_ta_encryption_key(uint32_t key_type, uint8_t *key, size_t key_len)
{
    // TA_Enc_Key = HUK_SubKey(TA_ENC, Key_Type)
    return huk_subkey_derive(HUK_SUBKEY_TA_ENC, &key_type, sizeof(key_type),
                           key, key_len);
}

// OTP接口包装
TEE_Result tee_otp_get_ta_enc_key(uint32_t key_type, uint8_t *buffer, size_t len)
{
    return derive_ta_encryption_key(key_type, buffer, len);
}
```

## 安全元件密钥管理

### 1. SE050安全元件支持

#### SE050密钥派生
```c
// SE050 SCP03密钥集派生
TEE_Result derive_se050_keys(uint8_t *scp03_keys, size_t keys_len)
{
    /*
     * SE050需要SCP03协议密钥集:
     * - S-ENC: 会话加密密钥
     * - S-MAC: 会话MAC密钥  
     * - S-DEK: 数据加密密钥
     */
    return huk_subkey_derive(HUK_SUBKEY_SE050, NULL, 0,
                           scp03_keys, keys_len);
}
```

## Die ID管理

### 1. 芯片标识符派生

#### Die ID生成
```c
// 芯片唯一标识符，用于设备指纹和许可管理
TEE_Result derive_die_id(uint8_t *die_id, size_t id_len)
{
    // Die_ID = HUK_SubKey(DIE_ID, NULL)
    return huk_subkey_derive(HUK_SUBKEY_DIE_ID, NULL, 0,
                           die_id, id_len);
}

// 获取Die ID的OTP接口
int tee_otp_get_die_id(uint8_t *buffer, size_t len)
{
    TEE_Result res = derive_die_id(buffer, len);
    return (res == TEE_SUCCESS) ? 0 : -1;
}
```

## 密钥生命周期管理

### 1. 密钥分配和初始化

#### 启动时密钥初始化序列
```c
// 系统启动时的密钥管理初始化顺序
static void init_key_management_subsystem(void)
{
    // 1. 验证HUK可用性
    struct tee_hw_unique_key huk;
    if (tee_otp_get_hw_unique_key(&huk) != TEE_SUCCESS) {
        panic("HUK not available");
    }
    
    // 2. 初始化SSK (存储密钥管理器)
    tee_fs_init_key_manager();
    
    // 3. 初始化RPMB密钥上下文
    rpmb_init_key_context();
    
    // 4. 验证密钥派生功能
    verify_key_derivation_functionality();
}
```

### 2. 密钥清理和销毁

#### 安全密钥清理
```c
// 密钥使用后的安全清理
static void secure_key_cleanup(void)
{
    // 使用explicit_bzero或memzero_explicit清除敏感数据
    memzero_explicit(&tee_fs_ssk, sizeof(tee_fs_ssk));
    
    // 清理栈上的密钥数据
    uint8_t stack_keys[256];
    memzero_explicit(stack_keys, sizeof(stack_keys));
}
```

### 3. 密钥缓存管理

#### 密钥缓存策略
```c
// TSK缓存管理 (每个TA会话)
struct ta_key_cache {
    TEE_UUID ta_uuid;
    uint8_t tsk[TEE_FS_KM_TSK_SIZE];
    bool valid;
    uint64_t last_used;
};

// 缓存TSK以避免重复计算
static TEE_Result get_cached_tsk(const TEE_UUID *uuid, uint8_t *tsk)
{
    // 查找缓存
    struct ta_key_cache *cache = find_tsk_cache(uuid);
    if (cache && cache->valid) {
        memcpy(tsk, cache->tsk, TEE_FS_KM_TSK_SIZE);
        cache->last_used = get_time_stamp();
        return TEE_SUCCESS;
    }
    
    // 缓存未命中，重新计算并缓存
    TEE_Result res = derive_tsk_for_ta(uuid, tsk, TEE_FS_KM_TSK_SIZE);
    if (res == TEE_SUCCESS) {
        update_tsk_cache(uuid, tsk);
    }
    return res;
}
```

## 密钥安全特性

### 1. 防侧信道攻击

#### 时间恒定操作
```c
// 密钥比较使用时间恒定算法
static int secure_key_compare(const uint8_t *key1, const uint8_t *key2, size_t len)
{
    int result = 0;
    for (size_t i = 0; i < len; i++) {
        result |= key1[i] ^ key2[i];
    }
    return result;
}
```

### 2. 密钥熵管理

#### 随机数质量保证
```c
// 确保FEK有足够的熵
static TEE_Result generate_high_entropy_fek(uint8_t *fek, size_t len)
{
    TEE_Result res;
    
    // 使用硬件随机数生成器
    res = crypto_rng_read(fek, len);
    if (res != TEE_SUCCESS)
        return res;
    
    // 可选: 混合额外的熵源
    uint64_t timestamp = get_time_stamp();
    for (size_t i = 0; i < len && i < sizeof(timestamp); i++) {
        fek[i] ^= ((uint8_t *)&timestamp)[i];
    }
    
    return TEE_SUCCESS;
}
```

### 3. 密钥版本管理

#### 支持密钥轮换
```c
// 密钥版本支持
struct key_version_info {
    uint32_t version;
    uint32_t algorithm;
    uint32_t key_length;
    uint64_t creation_time;
};

// 版本化密钥派生
TEE_Result derive_versioned_key(enum huk_subkey_usage usage,
                               uint32_t version,
                               const void *const_data, size_t const_data_len,
                               uint8_t *key, size_t key_len)
{
    struct {
        uint32_t usage;
        uint32_t version;
    } versioned_usage = { usage, version };
    
    return huk_subkey_derive(usage, &versioned_usage, sizeof(versioned_usage),
                           key, key_len);
}
```

这个高级密钥管理系统为OP-TEE提供了完整的密钥生命周期管理，从硬件信任根开始，通过标准化的密钥派生过程，支持多种用途的密钥需求，同时保持高度的安全性和向后兼容性。