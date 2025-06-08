# OP-TEE 存储安全威胁分析与防护机制

## 概述

本文档深入分析OP-TEE存储系统面临的各种安全威胁，并详细介绍相应的防护机制和对策。内容包括威胁模型、攻击向量、检测机制和防御策略。

## 威胁模型与攻击面分析

### 1. 威胁角色分类

```c
// 威胁角色定义
enum threat_actor_type {
    THREAT_ACTOR_MALICIOUS_APP,      // 恶意普通应用
    THREAT_ACTOR_MALICIOUS_TA,       // 恶意可信应用
    THREAT_ACTOR_PRIVILEGED_USER,    // 恶意特权用户
    THREAT_ACTOR_SYSTEM_ADMIN,       // 恶意系统管理员
    THREAT_ACTOR_PHYSICAL_ATTACKER,  // 物理攻击者
    THREAT_ACTOR_NATION_STATE,       // 国家级攻击者
};

struct threat_capability {
    bool can_modify_ree_storage;     // 可修改REE存储
    bool can_access_rpmb;            // 可访问RPMB
    bool has_physical_access;        // 有物理访问权限
    bool can_extract_keys;           // 可提取密钥
    bool can_modify_firmware;        // 可修改固件
    bool has_advanced_tools;         // 拥有高级工具
};

// 威胁能力评估矩阵
static const struct threat_capability threat_capabilities[] = {
    [THREAT_ACTOR_MALICIOUS_APP] = {
        .can_modify_ree_storage = false,
        .can_access_rpmb = false,
        .has_physical_access = false,
        .can_extract_keys = false,
        .can_modify_firmware = false,
        .has_advanced_tools = false,
    },
    [THREAT_ACTOR_MALICIOUS_TA] = {
        .can_modify_ree_storage = false,
        .can_access_rpmb = false,
        .has_physical_access = false,
        .can_extract_keys = false,
        .can_modify_firmware = false,
        .has_advanced_tools = false,
    },
    [THREAT_ACTOR_PRIVILEGED_USER] = {
        .can_modify_ree_storage = true,
        .can_access_rpmb = false,
        .has_physical_access = false,
        .can_extract_keys = false,
        .can_modify_firmware = false,
        .has_advanced_tools = true,
    },
    [THREAT_ACTOR_PHYSICAL_ATTACKER] = {
        .can_modify_ree_storage = true,
        .can_access_rpmb = true,
        .has_physical_access = true,
        .can_extract_keys = true,
        .can_modify_firmware = true,
        .has_advanced_tools = true,
    },
};
```

### 2. 攻击面映射

```
┌─────────────────────────────────────────────────────────────┐
│                    Attack Surface Map                      │
├─────────────────────────────────────────────────────────────┤
│  1. Application Layer (GP API)                             │
│     • TA身份伪造  • 对象ID猜测  • 权限提升                  │
├─────────────────────────────────────────────────────────────┤
│  2. TEE OS Layer (System Calls)                           │
│     • 系统调用注入  • 参数篡改  • 竞态条件                  │
├─────────────────────────────────────────────────────────────┤
│  3. Storage Backend Layer                                  │
│     • 后端绕过  • 缓存投毒  • 元数据篡改                    │
├─────────────────────────────────────────────────────────────┤
│  4. Cryptographic Layer                                    │
│     • 密钥泄露  • 算法降级  • 侧信道攻击                    │
├─────────────────────────────────────────────────────────────┤
│  5. RPC Communication Layer                               │
│     • 消息篡改  • 重放攻击  • 中间人攻击                    │
├─────────────────────────────────────────────────────────────┤
│  6. Hardware Layer                                        │
│     • RPMB密钥提取  • 总线监听  • 硬件篡改                 │
└─────────────────────────────────────────────────────────────┘
```

## 具体威胁分析与防护

### 1. 身份伪造与权限提升攻击

#### 威胁描述
攻击者试图伪造TA身份或提升访问权限以访问其他TA的存储数据。

#### 攻击实现
```c
// 模拟的身份伪造攻击代码
static TEE_Result attempt_uuid_spoofing_attack(void)
{
    TEE_UUID target_uuid = {
        0x12345678, 0x1234, 0x1234,
        { 0x12, 0x34, 0x56, 0x78, 0x9a, 0xbc, 0xde, 0xf0 }
    };
    
    // 攻击尝试1: 直接UUID替换
    TEE_UUID *current_uuid = &tee_ta_get_current_session()->ctx->uuid;
    TEE_UUID original_uuid = *current_uuid;
    
    *current_uuid = target_uuid;  // 伪造UUID
    
    // 尝试访问目标TA的对象
    TEE_ObjectHandle object;
    TEE_Result res = TEE_OpenPersistentObject(TEE_STORAGE_PRIVATE,
                                            "sensitive_data", 13,
                                            TEE_DATA_FLAG_ACCESS_READ,
                                            &object);
    
    *current_uuid = original_uuid;  // 恢复原UUID
    
    return res;  // 如果成功，说明存在漏洞
}
```

#### 防护机制
```c
// UUID完整性验证
struct uuid_integrity_context {
    TEE_UUID verified_uuid;           // 经过验证的UUID
    uint8_t uuid_signature[64];       // UUID数字签名
    uint64_t session_creation_time;   // 会话创建时间
    uint32_t verification_counter;    // 验证计数器
    bool is_verified;                 // 验证状态
};

static struct uuid_integrity_context uuid_contexts[MAX_SESSIONS];

// 强化的UUID验证
static TEE_Result verify_ta_uuid_integrity(struct tee_ta_session *sess)
{
    struct uuid_integrity_context *ctx = &uuid_contexts[sess->id];
    uint8_t computed_signature[64];
    
    // 1. 检查UUID是否已经验证过
    if (ctx->is_verified) {
        // 检查UUID是否被篡改
        if (memcmp(&ctx->verified_uuid, &sess->ctx->uuid, sizeof(TEE_UUID)) != 0) {
            EMSG("SECURITY: UUID tampering detected for session %u", sess->id);
            return TEE_ERROR_SECURITY;
        }
        return TEE_SUCCESS;
    }
    
    // 2. 对新UUID进行加密验证
    TEE_Result res = crypto_sign_uuid(&sess->ctx->uuid, computed_signature);
    if (res != TEE_SUCCESS) {
        EMSG("SECURITY: UUID signature generation failed");
        return res;
    }
    
    // 3. 验证UUID签名（通过安全启动链）
    res = verify_uuid_signature(&sess->ctx->uuid, computed_signature);
    if (res != TEE_SUCCESS) {
        EMSG("SECURITY: UUID signature verification failed");
        return TEE_ERROR_SECURITY;
    }
    
    // 4. 记录验证结果
    ctx->verified_uuid = sess->ctx->uuid;
    memcpy(ctx->uuid_signature, computed_signature, sizeof(computed_signature));
    ctx->session_creation_time = get_current_time();
    ctx->verification_counter = 0;
    ctx->is_verified = true;
    
    DMSG("UUID verification successful for TA: %pUb", &sess->ctx->uuid);
    return TEE_SUCCESS;
}

// 运行时权限检查增强
static TEE_Result enhanced_access_control_check(const TEE_UUID *requesting_uuid,
                                               const TEE_UUID *object_uuid,
                                               uint32_t access_flags)
{
    struct tee_ta_session *sess = tee_ta_get_current_session();
    
    // 1. 基础UUID匹配检查
    if (memcmp(requesting_uuid, object_uuid, sizeof(TEE_UUID)) != 0) {
        EMSG("SECURITY: Cross-TA access attempt denied");
        log_security_event(SECURITY_EVENT_UNAUTHORIZED_ACCESS, 
                          requesting_uuid, object_uuid);
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    // 2. UUID完整性验证
    TEE_Result res = verify_ta_uuid_integrity(sess);
    if (res != TEE_SUCCESS) {
        return res;
    }
    
    // 3. 检查TA权限范围
    if (!is_ta_authorized_for_storage(requesting_uuid)) {
        EMSG("SECURITY: TA not authorized for storage operations");
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    // 4. 时间窗口检查（防止长时间会话劫持）
    uint64_t session_age = get_current_time() - 
                          uuid_contexts[sess->id].session_creation_time;
    if (session_age > MAX_SESSION_LIFETIME) {
        EMSG("SECURITY: Session expired, forcing re-authentication");
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    // 5. 访问模式异常检测
    if (detect_anomalous_access_pattern(requesting_uuid, access_flags)) {
        EMSG("SECURITY: Anomalous access pattern detected");
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    return TEE_SUCCESS;
}
```

### 2. 重放攻击与时间操控

#### 威胁描述
攻击者记录并重放存储操作，或操控时间戳绕过安全检查。

#### 攻击向量
```c
// 重放攻击示例
struct replay_attack_context {
    uint8_t captured_rpc_data[4096];   // 捕获的RPC数据
    size_t data_size;                  // 数据大小
    uint64_t original_timestamp;       // 原始时间戳
    uint32_t sequence_number;          // 序列号
};

// 模拟重放攻击
static TEE_Result attempt_replay_attack(struct replay_attack_context *attack)
{
    // 1. 重放捕获的RPC数据
    struct thread_param params[4];
    
    memcpy(params, attack->captured_rpc_data, sizeof(params));
    
    // 2. 修改时间戳试图绕过检查
    params[3].attr = THREAD_PARAM_ATTR_VALUE_IN;
    params[3].u.value.a = attack->original_timestamp - 1000; // 回退时间
    
    // 3. 发送重放的RPC请求
    return thread_rpc_cmd(OPTEE_RPC_CMD_FS_WRITE, 4, params);
}
```

#### 防护机制
```c
// 高级防重放保护系统
#define NONCE_SIZE 16
#define MAX_PENDING_OPERATIONS 1024

struct anti_replay_context {
    uint8_t session_nonce[NONCE_SIZE];    // 会话随机数
    uint64_t monotonic_counter;           // 单调计数器
    uint64_t last_operation_time;         // 上次操作时间
    uint32_t operation_sequence;          // 操作序列号
    
    // 操作历史记录（Bloom Filter实现）
    uint8_t operation_history[1024];     // 布隆过滤器位数组
    uint32_t hash_functions[8];           // 哈希函数参数
    
    struct mutex ctx_lock;
};

static struct anti_replay_context replay_contexts[MAX_SESSIONS];

// 生成操作唯一标识符
static uint64_t generate_operation_id(const TEE_UUID *uuid, 
                                     const void *object_id,
                                     size_t obj_id_len,
                                     uint64_t timestamp,
                                     uint32_t sequence)
{
    uint8_t combined_data[256];
    size_t offset = 0;
    
    // 组合所有参数
    memcpy(combined_data + offset, uuid, sizeof(TEE_UUID));
    offset += sizeof(TEE_UUID);
    
    memcpy(combined_data + offset, object_id, obj_id_len);
    offset += obj_id_len;
    
    memcpy(combined_data + offset, &timestamp, sizeof(timestamp));
    offset += sizeof(timestamp);
    
    memcpy(combined_data + offset, &sequence, sizeof(sequence));
    offset += sizeof(sequence);
    
    // 计算哈希值作为操作ID
    uint64_t operation_id;
    hash_sha256((uint8_t *)&operation_id, sizeof(operation_id),
                combined_data, offset);
    
    return operation_id;
}

// 检查操作是否为重放
static bool is_replay_operation(struct anti_replay_context *ctx,
                               uint64_t operation_id)
{
    // 使用布隆过滤器检查操作历史
    for (int i = 0; i < 8; i++) {
        uint32_t hash = (operation_id * ctx->hash_functions[i]) % (1024 * 8);
        uint32_t byte_index = hash / 8;
        uint32_t bit_index = hash % 8;
        
        if (!(ctx->operation_history[byte_index] & (1 << bit_index))) {
            return false;  // 操作不在历史中，不是重放
        }
    }
    
    return true;  // 可能是重放（布隆过滤器可能误报）
}

// 记录操作到历史
static void record_operation(struct anti_replay_context *ctx,
                            uint64_t operation_id)
{
    // 在布隆过滤器中设置位
    for (int i = 0; i < 8; i++) {
        uint32_t hash = (operation_id * ctx->hash_functions[i]) % (1024 * 8);
        uint32_t byte_index = hash / 8;
        uint32_t bit_index = hash % 8;
        
        ctx->operation_history[byte_index] |= (1 << bit_index);
    }
}

// 强化的防重放检查
static TEE_Result check_anti_replay(struct tee_ta_session *sess,
                                   const void *object_id,
                                   size_t obj_id_len,
                                   uint32_t operation_type)
{
    struct anti_replay_context *ctx = &replay_contexts[sess->id];
    uint64_t current_time = get_secure_monotonic_time();
    TEE_Result res;
    
    mutex_lock(&ctx->ctx_lock);
    
    // 1. 时间单调性检查
    if (current_time <= ctx->last_operation_time) {
        EMSG("SECURITY: Time rollback detected, possible replay attack");
        res = TEE_ERROR_SECURITY;
        goto exit;
    }
    
    // 2. 序列号检查
    uint32_t expected_sequence = ctx->operation_sequence + 1;
    
    // 3. 生成当前操作ID
    uint64_t operation_id = generate_operation_id(&sess->ctx->uuid,
                                                 object_id, obj_id_len,
                                                 current_time, expected_sequence);
    
    // 4. 检查是否为重放操作
    if (is_replay_operation(ctx, operation_id)) {
        EMSG("SECURITY: Replay attack detected for operation ID: %lx", operation_id);
        log_security_event(SECURITY_EVENT_REPLAY_ATTACK, 
                          &sess->ctx->uuid, object_id);
        res = TEE_ERROR_SECURITY;
        goto exit;
    }
    
    // 5. 记录当前操作
    record_operation(ctx, operation_id);
    ctx->last_operation_time = current_time;
    ctx->operation_sequence = expected_sequence;
    
    // 6. RPMB计数器验证（如果使用RPMB）
    if (operation_type == STORAGE_OP_WRITE) {
        res = verify_rpmb_counter_monotonicity();
        if (res != TEE_SUCCESS) {
            EMSG("SECURITY: RPMB counter check failed");
            goto exit;
        }
    }
    
    res = TEE_SUCCESS;
    
exit:
    mutex_unlock(&ctx->ctx_lock);
    return res;
}
```

### 3. 侧信道攻击防护

#### 威胁描述
攻击者通过分析系统的时间、功耗、电磁辐射等侧信道信息推断密钥或敏感数据。

#### 防护机制
```c
// 侧信道攻击防护框架
struct side_channel_protection {
    bool constant_time_enabled;      // 常数时间算法
    bool power_analysis_protection;  // 功耗分析保护
    bool timing_randomization;       // 时间随机化
    bool cache_line_alignment;       // 缓存行对齐
    uint32_t noise_generation_level; // 噪声生成级别
};

static struct side_channel_protection scp_config = {
    .constant_time_enabled = true,
    .power_analysis_protection = true,
    .timing_randomization = true,
    .cache_line_alignment = true,
    .noise_generation_level = 3,
};

// 常数时间内存比较
static int constant_time_memcmp(const void *s1, const void *s2, size_t n)
{
    const uint8_t *p1 = (const uint8_t *)s1;
    const uint8_t *p2 = (const uint8_t *)s2;
    uint8_t result = 0;
    
    // 确保总是访问所有字节，不因提前匹配而短路
    for (size_t i = 0; i < n; i++) {
        result |= p1[i] ^ p2[i];
    }
    
    return result;
}

// 时间随机化延迟
static void add_timing_noise(uint32_t base_delay_us)
{
    if (!scp_config.timing_randomization) {
        return;
    }
    
    // 生成伪随机延迟
    uint32_t random_delay = secure_random() % (base_delay_us / 4);
    
    // 执行无用计算以消耗时间
    volatile uint32_t dummy = 0;
    for (uint32_t i = 0; i < random_delay * 100; i++) {
        dummy += i * 7919;  // 使用素数增加不可预测性
    }
}

// 缓存行对齐的安全内存访问
static void secure_memory_access(void *ptr, size_t size)
{
    if (!scp_config.cache_line_alignment) {
        return;
    }
    
    // 预加载整个缓存行，避免访问模式泄露信息
    uintptr_t aligned_start = ((uintptr_t)ptr) & ~(CACHE_LINE_SIZE - 1);
    uintptr_t aligned_end = (((uintptr_t)ptr + size - 1) | (CACHE_LINE_SIZE - 1)) + 1;
    
    for (uintptr_t addr = aligned_start; addr < aligned_end; addr += CACHE_LINE_SIZE) {
        volatile uint8_t dummy = *(volatile uint8_t *)addr;
        (void)dummy;  // 防止编译器优化
    }
}

// 抗功耗分析的AES实现
static TEE_Result power_analysis_resistant_aes(const uint8_t *key,
                                              const uint8_t *input,
                                              uint8_t *output)
{
    // 1. 随机掩码生成
    uint8_t mask[16];
    secure_random_buffer(mask, sizeof(mask));
    
    // 2. 掩码输入数据
    uint8_t masked_input[16];
    for (int i = 0; i < 16; i++) {
        masked_input[i] = input[i] ^ mask[i];
    }
    
    // 3. 执行掩码AES运算
    uint8_t masked_output[16];
    TEE_Result res = aes_encrypt_masked(key, masked_input, masked_output, mask);
    if (res != TEE_SUCCESS) {
        return res;
    }
    
    // 4. 移除输出掩码
    for (int i = 0; i < 16; i++) {
        output[i] = masked_output[i] ^ mask[i];
    }
    
    // 5. 添加时间噪声
    add_timing_noise(100);  // 100微秒基础延迟
    
    return TEE_SUCCESS;
}
```

### 4. 数据完整性攻击与防护

#### 威胁描述
攻击者篡改存储数据或元数据，绕过完整性检查。

#### 防护机制
```c
// 多层次完整性保护系统
struct integrity_protection_context {
    uint8_t primary_hash[32];        // 主哈希值（SHA-256）
    uint8_t backup_hash[32];         // 备份哈希值
    uint8_t merkle_root[32];         // Merkle树根哈希
    uint64_t version_counter;        // 版本计数器
    uint32_t checksum;              // 快速校验和
    bool integrity_verified;         // 完整性验证状态
};

// 增强的完整性验证算法
static TEE_Result enhanced_integrity_verification(struct tee_file_handle *fh,
                                                 const void *data,
                                                 size_t data_size)
{
    struct integrity_protection_context ctx;
    uint8_t computed_hash[32];
    uint32_t computed_checksum;
    TEE_Result res;
    
    // 1. 读取存储的完整性信息
    res = read_integrity_metadata(fh, &ctx);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to read integrity metadata");
        return res;
    }
    
    // 2. 快速校验和检查（第一道防线）
    computed_checksum = calculate_crc32(data, data_size);
    if (computed_checksum != ctx.checksum) {
        EMSG("SECURITY: Checksum mismatch detected");
        log_security_event(SECURITY_EVENT_INTEGRITY_VIOLATION, NULL, NULL);
        return TEE_ERROR_CORRUPT_OBJECT;
    }
    
    // 3. 主哈希验证（第二道防线）
    res = hash_sha256(computed_hash, sizeof(computed_hash), data, data_size);
    if (res != TEE_SUCCESS) {
        return res;
    }
    
    if (constant_time_memcmp(computed_hash, ctx.primary_hash, 32) != 0) {
        // 主哈希失败，尝试备份哈希
        if (constant_time_memcmp(computed_hash, ctx.backup_hash, 32) != 0) {
            EMSG("SECURITY: Both primary and backup hash verification failed");
            return TEE_ERROR_CORRUPT_OBJECT;
        } else {
            IMSG("Primary hash failed, but backup hash verified - updating primary");
            memcpy(ctx.primary_hash, computed_hash, 32);
            update_integrity_metadata(fh, &ctx);
        }
    }
    
    // 4. Merkle树验证（第三道防线，用于大文件）
    if (data_size > MERKLE_TREE_THRESHOLD) {
        res = verify_merkle_tree_integrity(data, data_size, ctx.merkle_root);
        if (res != TEE_SUCCESS) {
            EMSG("SECURITY: Merkle tree integrity verification failed");
            return res;
        }
    }
    
    // 5. 版本单调性检查
    uint64_t current_version = get_file_version_counter(fh);
    if (current_version < ctx.version_counter) {
        EMSG("SECURITY: Version rollback detected");
        return TEE_ERROR_SECURITY;
    }
    
    // 6. 添加侧信道保护
    add_timing_noise(50);
    
    return TEE_SUCCESS;
}

// 写入时的完整性保护
static TEE_Result protected_write_with_integrity(struct tee_file_handle *fh,
                                                const void *data,
                                                size_t data_size)
{
    struct integrity_protection_context ctx;
    TEE_Result res;
    
    // 1. 计算新的完整性信息
    ctx.checksum = calculate_crc32(data, data_size);
    
    res = hash_sha256(ctx.primary_hash, sizeof(ctx.primary_hash), data, data_size);
    if (res != TEE_SUCCESS) {
        return res;
    }
    
    // 备份哈希与主哈希相同（写入时）
    memcpy(ctx.backup_hash, ctx.primary_hash, 32);
    
    // 2. 大文件计算Merkle树
    if (data_size > MERKLE_TREE_THRESHOLD) {
        res = calculate_merkle_tree_root(data, data_size, ctx.merkle_root);
        if (res != TEE_SUCCESS) {
            return res;
        }
    }
    
    // 3. 更新版本计数器
    ctx.version_counter = get_next_version_counter(fh);
    ctx.integrity_verified = true;
    
    // 4. 原子写入数据和完整性信息
    res = atomic_write_with_integrity(fh, data, data_size, &ctx);
    if (res != TEE_SUCCESS) {
        EMSG("Atomic write with integrity failed");
        return res;
    }
    
    return TEE_SUCCESS;
}
```

### 5. 高级持续威胁 (APT) 检测

```c
// APT行为模式检测系统
#define MAX_BEHAVIOR_PATTERNS 1000
#define ANOMALY_THRESHOLD 0.75

struct behavior_pattern {
    TEE_UUID ta_uuid;                    // TA标识
    uint32_t operation_type;             // 操作类型
    uint64_t timestamp;                  // 时间戳
    size_t data_size;                    // 数据大小
    uint32_t frequency;                  // 操作频率
    char object_id_pattern[64];          // 对象ID模式
};

struct apt_detection_context {
    struct behavior_pattern patterns[MAX_BEHAVIOR_PATTERNS];
    size_t pattern_count;
    uint64_t baseline_established_time;  // 基线建立时间
    bool learning_mode;                  // 学习模式
    float anomaly_scores[MAX_SESSIONS];  // 异常评分
    struct mutex detection_lock;
};

static struct apt_detection_context apt_detector = {
    .learning_mode = true,
    .detection_lock = MUTEX_INITIALIZER,
};

// 建立行为基线
static void establish_behavior_baseline(void)
{
    mutex_lock(&apt_detector.detection_lock);
    
    if (apt_detector.learning_mode && 
        get_current_time() - apt_detector.baseline_established_time > 
        BASELINE_LEARNING_PERIOD) {
        
        IMSG("APT: Behavior baseline established with %zu patterns",
             apt_detector.pattern_count);
        
        apt_detector.learning_mode = false;
        
        // 分析模式建立统计模型
        analyze_behavioral_patterns();
    }
    
    mutex_unlock(&apt_detector.detection_lock);
}

// 异常行为检测
static float calculate_anomaly_score(const struct behavior_pattern *current)
{
    float score = 0.0;
    int matches = 0;
    
    // 与历史模式比较
    for (size_t i = 0; i < apt_detector.pattern_count; i++) {
        struct behavior_pattern *historical = &apt_detector.patterns[i];
        
        // UUID匹配
        if (memcmp(&current->ta_uuid, &historical->ta_uuid, sizeof(TEE_UUID)) == 0) {
            matches++;
            
            // 操作类型异常
            if (current->operation_type != historical->operation_type) {
                score += 0.3;
            }
            
            // 数据大小异常
            float size_ratio = (float)current->data_size / historical->data_size;
            if (size_ratio > 10.0 || size_ratio < 0.1) {
                score += 0.4;
            }
            
            // 频率异常
            float freq_ratio = (float)current->frequency / historical->frequency;
            if (freq_ratio > 5.0 || freq_ratio < 0.2) {
                score += 0.3;
            }
            
            // 对象ID模式异常
            if (strstr(current->object_id_pattern, "..") ||
                strstr(current->object_id_pattern, "admin") ||
                strstr(current->object_id_pattern, "system")) {
                score += 0.5;
            }
        }
    }
    
    // 如果没有历史匹配，可能是新的攻击
    if (matches == 0) {
        score = 0.8;
    }
    
    return MIN(score, 1.0);
}

// 实时威胁检测
static TEE_Result detect_apt_activity(struct tee_ta_session *sess,
                                     uint32_t operation_type,
                                     const void *object_id,
                                     size_t data_size)
{
    struct behavior_pattern current;
    float anomaly_score;
    
    // 构建当前行为模式
    current.ta_uuid = sess->ctx->uuid;
    current.operation_type = operation_type;
    current.timestamp = get_current_time();
    current.data_size = data_size;
    current.frequency = calculate_operation_frequency(sess);
    snprintf(current.object_id_pattern, sizeof(current.object_id_pattern),
             "%.*s", (int)MIN(strlen((char *)object_id), 63), (char *)object_id);
    
    mutex_lock(&apt_detector.detection_lock);
    
    // 学习模式：记录行为模式
    if (apt_detector.learning_mode) {
        if (apt_detector.pattern_count < MAX_BEHAVIOR_PATTERNS) {
            apt_detector.patterns[apt_detector.pattern_count++] = current;
        }
        mutex_unlock(&apt_detector.detection_lock);
        return TEE_SUCCESS;
    }
    
    // 检测模式：评估异常
    anomaly_score = calculate_anomaly_score(&current);
    apt_detector.anomaly_scores[sess->id] = anomaly_score;
    
    mutex_unlock(&apt_detector.detection_lock);
    
    // 威胁响应
    if (anomaly_score > ANOMALY_THRESHOLD) {
        EMSG("APT: High anomaly score (%.2f) detected for TA %pUb",
             anomaly_score, &sess->ctx->uuid);
        
        // 记录详细的威胁情报
        log_security_event(SECURITY_EVENT_APT_DETECTED, 
                          &sess->ctx->uuid, object_id);
        
        // 可选的自动响应措施
        if (anomaly_score > CRITICAL_ANOMALY_THRESHOLD) {
            EMSG("APT: Critical threat detected, initiating containment");
            return TEE_ERROR_SECURITY;
        }
    }
    
    return TEE_SUCCESS;
}
```

### 6. 取证与事件响应

```c
// 安全事件记录系统
#define MAX_SECURITY_EVENTS 10000
#define FORENSIC_BUFFER_SIZE (1024 * 1024)  // 1MB

enum security_event_type {
    SECURITY_EVENT_UNAUTHORIZED_ACCESS,
    SECURITY_EVENT_INTEGRITY_VIOLATION,
    SECURITY_EVENT_REPLAY_ATTACK,
    SECURITY_EVENT_APT_DETECTED,
    SECURITY_EVENT_SIDE_CHANNEL_ATTACK,
    SECURITY_EVENT_PRIVILEGE_ESCALATION,
    SECURITY_EVENT_DATA_EXFILTRATION,
};

struct security_event {
    enum security_event_type type;       // 事件类型
    uint64_t timestamp;                  // 时间戳
    TEE_UUID source_uuid;                // 源TA UUID
    char target_object[64];              // 目标对象
    uint32_t severity;                   // 严重级别 (1-10)
    char description[256];               // 事件描述
    uint8_t forensic_data[512];          // 取证数据
    size_t forensic_data_size;           // 取证数据大小
};

struct forensic_context {
    struct security_event events[MAX_SECURITY_EVENTS];
    size_t event_count;
    uint64_t oldest_event_index;         // 环形缓冲区索引
    uint8_t forensic_buffer[FORENSIC_BUFFER_SIZE];
    size_t buffer_offset;
    bool forensic_mode_enabled;
    struct mutex forensic_lock;
};

static struct forensic_context forensic_ctx = {
    .forensic_mode_enabled = true,
    .forensic_lock = MUTEX_INITIALIZER,
};

// 记录安全事件
static void log_security_event(enum security_event_type type,
                              const TEE_UUID *source_uuid,
                              const void *target_object)
{
    if (!forensic_ctx.forensic_mode_enabled) {
        return;
    }
    
    mutex_lock(&forensic_ctx.forensic_lock);
    
    size_t index = forensic_ctx.event_count % MAX_SECURITY_EVENTS;
    struct security_event *event = &forensic_ctx.events[index];
    
    // 填充事件信息
    event->type = type;
    event->timestamp = get_secure_monotonic_time();
    if (source_uuid) {
        event->source_uuid = *source_uuid;
    } else {
        memset(&event->source_uuid, 0, sizeof(TEE_UUID));
    }
    
    if (target_object) {
        snprintf(event->target_object, sizeof(event->target_object),
                "%.*s", 63, (char *)target_object);
    } else {
        event->target_object[0] = '\0';
    }
    
    // 设置严重级别
    switch (type) {
    case SECURITY_EVENT_APT_DETECTED:
    case SECURITY_EVENT_PRIVILEGE_ESCALATION:
        event->severity = 9;
        break;
    case SECURITY_EVENT_UNAUTHORIZED_ACCESS:
    case SECURITY_EVENT_REPLAY_ATTACK:
        event->severity = 7;
        break;
    case SECURITY_EVENT_INTEGRITY_VIOLATION:
        event->severity = 6;
        break;
    default:
        event->severity = 5;
        break;
    }
    
    // 生成事件描述
    generate_event_description(event);
    
    // 收集取证数据
    collect_forensic_data(event);
    
    forensic_ctx.event_count++;
    
    // 立即响应高严重级别事件
    if (event->severity >= 8) {
        immediate_threat_response(event);
    }
    
    mutex_unlock(&forensic_ctx.forensic_lock);
    
    EMSG("SECURITY EVENT [%u]: %s (Severity: %u)",
         type, event->description, event->severity);
}

// 威胁响应机制
static void immediate_threat_response(struct security_event *event)
{
    switch (event->type) {
    case SECURITY_EVENT_APT_DETECTED:
        // 隔离相关TA
        quarantine_ta(&event->source_uuid);
        // 通知管理员
        notify_security_admin(event);
        break;
        
    case SECURITY_EVENT_PRIVILEGE_ESCALATION:
        // 撤销权限
        revoke_ta_privileges(&event->source_uuid);
        // 记录详细调用栈
        capture_call_stack(event);
        break;
        
    case SECURITY_EVENT_REPLAY_ATTACK:
        // 刷新防重放状态
        refresh_anti_replay_state(&event->source_uuid);
        // 增强监控
        enable_enhanced_monitoring(&event->source_uuid);
        break;
        
    default:
        break;
    }
}

// 取证数据导出
static TEE_Result export_forensic_data(void *buffer, size_t *buffer_size)
{
    if (!buffer || !buffer_size) {
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    mutex_lock(&forensic_ctx.forensic_lock);
    
    size_t required_size = forensic_ctx.event_count * sizeof(struct security_event);
    
    if (*buffer_size < required_size) {
        *buffer_size = required_size;
        mutex_unlock(&forensic_ctx.forensic_lock);
        return TEE_ERROR_SHORT_BUFFER;
    }
    
    // 导出所有安全事件
    memcpy(buffer, forensic_ctx.events, required_size);
    *buffer_size = required_size;
    
    mutex_unlock(&forensic_ctx.forensic_lock);
    
    IMSG("Forensic data exported: %zu events, %zu bytes",
         forensic_ctx.event_count, required_size);
    
    return TEE_SUCCESS;
}
```

## 威胁缓解策略总结

### 1. 纵深防御架构
- **第一层**: 应用层身份验证和访问控制
- **第二层**: 系统调用层参数验证和权限检查
- **第三层**: 存储层加密和完整性保护
- **第四层**: 硬件层安全机制 (RPMB、HUK)

### 2. 主动防御措施
- **实时监控**: 行为分析和异常检测
- **威胁情报**: 已知攻击模式识别
- **自适应响应**: 基于威胁级别的自动响应
- **取证能力**: 完整的事件记录和分析

### 3. 合规性保证
- **CC EAL4+**: 符合通用准则评估保证级别
- **FIPS 140-2**: 密码模块安全要求
- **GP标准**: 完全符合GlobalPlatform规范
- **行业标准**: 满足金融、医疗等行业安全要求

通过这些综合性的安全措施，OP-TEE存储系统能够抵御各种复杂的安全威胁，为敏感数据提供军事级别的保护。