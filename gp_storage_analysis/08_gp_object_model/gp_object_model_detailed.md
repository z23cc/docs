# GP对象模型和属性系统深度分析

## GP对象模型概述

GlobalPlatform (GP) 对象模型是TEE存储系统的核心，它定义了一个完整的对象类型系统、属性框架和生命周期管理机制。OP-TEE完全实现了GP TEE Internal Core API v1.3.1规范中定义的对象模型。

## 对象类型分类体系

### 1. 对象类型编码结构

GP对象类型使用32位标识符，具有特定的编码结构：

```c
// 对象类型编码格式: 0xATTTTTTT
// A: 算法类别 (4位)
// T: 类型标识 (28位)

// 算法类别定义:
// 0xA0: 对称密钥和公钥
// 0xA1: 密钥对
// 0x50: 摘要算法 (无对象类型)
// 0x30: MAC算法 (无对象类型)
```

### 2. 对称密钥对象类型

#### 分组密码密钥
```c
#define TEE_TYPE_AES                        0xA0000010   // AES密钥
#define TEE_TYPE_DES                        0xA0000011   // DES密钥
#define TEE_TYPE_DES3                       0xA0000013   // 3DES密钥
#define TEE_TYPE_SM4                        0xA0000014   // SM4密钥(中国标准)
```

#### MAC密钥类型
```c
#define TEE_TYPE_HMAC_MD5                   0xA0000001   // HMAC-MD5密钥
#define TEE_TYPE_HMAC_SHA1                  0xA0000002   // HMAC-SHA1密钥
#define TEE_TYPE_HMAC_SHA224                0xA0000003   // HMAC-SHA224密钥
#define TEE_TYPE_HMAC_SHA256                0xA0000004   // HMAC-SHA256密钥
#define TEE_TYPE_HMAC_SHA384                0xA0000005   // HMAC-SHA384密钥
#define TEE_TYPE_HMAC_SHA512                0xA0000006   // HMAC-SHA512密钥
#define TEE_TYPE_HMAC_SM3                   0xA0000007   // HMAC-SM3密钥
#define TEE_TYPE_HMAC_SHA3_224              0xA0000008   // HMAC-SHA3-224密钥
#define TEE_TYPE_HMAC_SHA3_256              0xA0000009   // HMAC-SHA3-256密钥
#define TEE_TYPE_HMAC_SHA3_384              0xA000000A   // HMAC-SHA3-384密钥
#define TEE_TYPE_HMAC_SHA3_512              0xA000000B   // HMAC-SHA3-512密钥
```

### 3. 非对称密钥对象类型

#### RSA密钥类型
```c
#define TEE_TYPE_RSA_PUBLIC_KEY             0xA0000030   // RSA公钥
#define TEE_TYPE_RSA_KEYPAIR                0xA1000030   // RSA密钥对
```

#### DSA密钥类型
```c
#define TEE_TYPE_DSA_PUBLIC_KEY             0xA0000031   // DSA公钥
#define TEE_TYPE_DSA_KEYPAIR                0xA1000031   // DSA密钥对
```

#### DH密钥类型
```c
#define TEE_TYPE_DH_KEYPAIR                 0xA1000032   // DH密钥对
```

#### ECC密钥类型
```c
#define TEE_TYPE_ECDSA_PUBLIC_KEY           0xA0000041   // ECDSA公钥
#define TEE_TYPE_ECDSA_KEYPAIR              0xA1000041   // ECDSA密钥对
#define TEE_TYPE_ECDH_PUBLIC_KEY            0xA0000042   // ECDH公钥
#define TEE_TYPE_ECDH_KEYPAIR               0xA1000042   // ECDH密钥对
```

#### Edwards曲线密钥类型
```c
#define TEE_TYPE_ED25519_PUBLIC_KEY         0xA0000043   // Ed25519公钥
#define TEE_TYPE_ED25519_KEYPAIR            0xA1000043   // Ed25519密钥对
#define TEE_TYPE_ED448_PUBLIC_KEY           0xA0000048   // Ed448公钥
#define TEE_TYPE_ED448_KEYPAIR              0xA1000048   // Ed448密钥对
#define TEE_TYPE_X25519_PUBLIC_KEY          0xA0000044   // X25519公钥
#define TEE_TYPE_X25519_KEYPAIR             0xA1000044   // X25519密钥对
#define TEE_TYPE_X448_PUBLIC_KEY            0xA0000049   // X448公钥
#define TEE_TYPE_X448_KEYPAIR               0xA1000049   // X448密钥对
```

#### 中国商用密码算法
```c
#define TEE_TYPE_SM2_DSA_PUBLIC_KEY         0xA0000045   // SM2数字签名公钥
#define TEE_TYPE_SM2_DSA_KEYPAIR            0xA1000045   // SM2数字签名密钥对
#define TEE_TYPE_SM2_KEP_PUBLIC_KEY         0xA0000046   // SM2密钥交换公钥
#define TEE_TYPE_SM2_KEP_KEYPAIR            0xA1000046   // SM2密钥交换密钥对
#define TEE_TYPE_SM2_PKE_PUBLIC_KEY         0xA0000047   // SM2公钥加密公钥
#define TEE_TYPE_SM2_PKE_KEYPAIR            0xA1000047   // SM2公钥加密密钥对
```

### 4. 特殊对象类型

#### 通用对象类型
```c
#define TEE_TYPE_GENERIC_SECRET             0xA0000000   // 通用密钥材料
#define TEE_TYPE_DATA                       0xA00000BF   // 原始数据对象
#define TEE_TYPE_CORRUPTED_OBJECT           0xA00000BE   // 损坏对象标识
#define TEE_TYPE_HKDF                       0xA000004A   // HKDF密钥派生对象
#define TEE_TYPE_ILLEGAL_VALUE              0xEFFFFFFF   // 非法类型值
```

## 对象属性系统

### 1. 属性标识符编码

GP属性使用32位标识符，编码结构如下：

```c
// 属性编码格式: 0xFTTTTTTT
// F: 属性类型标志 (4位)
// T: 属性标识 (28位)

// 属性类型标志:
#define TEE_ATTR_FLAG_PUBLIC            (1 << 28)  // 0x10000000: 公开属性
#define TEE_ATTR_FLAG_VALUE             (1 << 29)  // 0x20000000: 值属性
// 位30: 保护标志 (0=公开, 1=保护)
// 位31: 类型标志 (0=缓冲区, 1=值)

// 属性类型组合:
// 0xC0: 保护值属性 (私钥材料)
// 0xD0: 公开缓冲区属性 (公钥材料)
// 0xF0: 公开值属性 (数值参数)
```

### 2. 通用密钥属性

#### 密钥值属性
```c
#define TEE_ATTR_SECRET_VALUE               0xC0000000   // 对称密钥值
```

#### 密钥用途属性
```c
#define TEE_USAGE_EXTRACTABLE              0x00000001   // 可提取
#define TEE_USAGE_ENCRYPT                  0x00000002   // 可加密
#define TEE_USAGE_DECRYPT                  0x00000004   // 可解密
#define TEE_USAGE_MAC                      0x00000008   // 可MAC
#define TEE_USAGE_SIGN                     0x00000010   // 可签名
#define TEE_USAGE_VERIFY                   0x00000020   // 可验签
#define TEE_USAGE_DERIVE                   0x00000040   // 可派生
#define TEE_USAGE_DEFAULT                  0xffffffff   // 默认用途
```

### 3. RSA密钥属性

```c
// RSA公钥属性
#define TEE_ATTR_RSA_MODULUS                0xD0000130   // RSA模数 (n)
#define TEE_ATTR_RSA_PUBLIC_EXPONENT        0xD0000230   // RSA公钥指数 (e)

// RSA私钥属性
#define TEE_ATTR_RSA_PRIVATE_EXPONENT       0xC0000330   // RSA私钥指数 (d)
#define TEE_ATTR_RSA_PRIME1                 0xC0000430   // RSA质数p
#define TEE_ATTR_RSA_PRIME2                 0xC0000530   // RSA质数q
#define TEE_ATTR_RSA_EXPONENT1              0xC0000630   // RSA指数1 (dp)
#define TEE_ATTR_RSA_EXPONENT2              0xC0000730   // RSA指数2 (dq)
#define TEE_ATTR_RSA_COEFFICIENT            0xC0000830   // RSA系数 (qinv)

// RSA算法参数
#define TEE_ATTR_RSA_OAEP_LABEL             0xD0000930   // OAEP标签
#define TEE_ATTR_RSA_OAEP_MGF_HASH          0xD0000931   // OAEP MGF哈希算法
#define TEE_ATTR_RSA_PSS_SALT_LENGTH        0xF0000A30   // PSS盐长度
```

### 4. DSA密钥属性

```c
// DSA域参数
#define TEE_ATTR_DSA_PRIME                  0xD0001031   // DSA质数p
#define TEE_ATTR_DSA_SUBPRIME               0xD0001131   // DSA子质数q
#define TEE_ATTR_DSA_BASE                   0xD0001231   // DSA生成元g

// DSA密钥材料
#define TEE_ATTR_DSA_PUBLIC_VALUE           0xD0000131   // DSA公钥值
#define TEE_ATTR_DSA_PRIVATE_VALUE          0xC0000231   // DSA私钥值
```

### 5. DH密钥属性

```c
// DH域参数
#define TEE_ATTR_DH_PRIME                   0xD0001032   // DH质数p
#define TEE_ATTR_DH_SUBPRIME                0xD0001132   // DH子质数q (可选)
#define TEE_ATTR_DH_BASE                    0xD0001232   // DH生成元g
#define TEE_ATTR_DH_X_BITS                  0xF0001332   // DH私钥位长度

// DH密钥材料
#define TEE_ATTR_DH_PUBLIC_VALUE            0xD0000132   // DH公钥值
#define TEE_ATTR_DH_PRIVATE_VALUE           0xC0000232   // DH私钥值
```

### 6. ECC密钥属性

#### 通用ECC属性
```c
#define TEE_ATTR_ECC_PUBLIC_VALUE_X         0xD0000141   // ECC公钥X坐标
#define TEE_ATTR_ECC_PUBLIC_VALUE_Y         0xD0000241   // ECC公钥Y坐标
#define TEE_ATTR_ECC_PRIVATE_VALUE          0xC0000341   // ECC私钥值
#define TEE_ATTR_ECC_CURVE                  0xF0000441   // ECC曲线标识符
```

#### 支持的ECC曲线
```c
#define TEE_ECC_CURVE_NIST_P192             0x00000001   // NIST P-192
#define TEE_ECC_CURVE_NIST_P224             0x00000002   // NIST P-224
#define TEE_ECC_CURVE_NIST_P256             0x00000003   // NIST P-256
#define TEE_ECC_CURVE_NIST_P384             0x00000004   // NIST P-384
#define TEE_ECC_CURVE_NIST_P521             0x00000005   // NIST P-521
#define TEE_ECC_CURVE_25519                 0x00000300   // Curve25519
#define TEE_ECC_CURVE_SM2                   0x00000400   // SM2曲线
```

#### 短期密钥属性
```c
#define TEE_ATTR_ECC_EPHEMERAL_PUBLIC_VALUE_X 0xD0000146   // 短期公钥X坐标
#define TEE_ATTR_ECC_EPHEMERAL_PUBLIC_VALUE_Y 0xD0000246   // 短期公钥Y坐标
```

### 7. Edwards曲线属性

#### Ed25519属性
```c
#define TEE_ATTR_ED25519_PUBLIC_VALUE       0xD0000743   // Ed25519公钥值
#define TEE_ATTR_ED25519_PRIVATE_VALUE      0xC0000843   // Ed25519私钥值
#define TEE_ATTR_EDDSA_CTX                  0xD0000643   // EdDSA上下文
#define TEE_ATTR_EDDSA_PREHASH              0xF0000004   // EdDSA预哈希标志
```

#### X25519属性
```c
#define TEE_ATTR_X25519_PUBLIC_VALUE        0xD0000944   // X25519公钥值
#define TEE_ATTR_X25519_PRIVATE_VALUE       0xC0000A44   // X25519私钥值
```

#### Ed448/X448属性
```c
#define TEE_ATTR_X448_PUBLIC_VALUE          0xD0000A45   // X448公钥值
#define TEE_ATTR_X448_PRIVATE_VALUE         0xC0000A46   // X448私钥值
```

### 8. SM2密钥属性

```c
// SM2身份标识
#define TEE_ATTR_SM2_ID_INITIATOR           0xD0000446   // SM2发起方ID
#define TEE_ATTR_SM2_ID_RESPONDER           0xD0000546   // SM2响应方ID

// SM2密钥交换属性
#define TEE_ATTR_SM2_KEP_USER               0xF0000646   // SM2密钥交换用户类型
#define TEE_ATTR_SM2_KEP_CONFIRMATION_IN    0xD0000746   // SM2密钥交换输入确认值
#define TEE_ATTR_SM2_KEP_CONFIRMATION_OUT   0xD0000846   // SM2密钥交换输出确认值
```

### 9. HKDF属性

```c
#define TEE_ATTR_HKDF_SALT                  0xD0000946   // HKDF盐值
#define TEE_ATTR_HKDF_INFO                  0xD0000A46   // HKDF信息字段
#define TEE_ATTR_HKDF_HASH_ALGORITHM        0xF0000B46   // HKDF哈希算法
#define TEE_ATTR_KDF_KEY_SIZE               0xF0000C46   // 派生密钥大小
```

## 对象结构体系

### 1. 核心对象结构

#### tee_obj结构 (TEE内核对象)
```c
struct tee_obj {
    TAILQ_ENTRY(tee_obj) link;      // 链表节点
    TEE_ObjectInfo info;            // 对象信息
    bool busy;                      // 操作忙标志
    uint32_t have_attrs;            // 属性位掩码
    void *attr;                     // 属性数据指针
    size_t ds_pos;                  // 数据流位置
    struct tee_pobj *pobj;          // 持久化对象指针
    struct tee_file_handle *fh;     // 文件句柄
};
```

#### TEE_ObjectInfo结构 (GP标准对象信息)
```c
typedef struct {
    uint32_t objectType;            // 对象类型
    uint32_t objectSize;            // 对象大小(位)
    uint32_t maxObjectSize;         // 最大对象大小
    uint32_t objectUsage;           // 对象用途标志
    uint32_t dataSize;              // 数据大小(字节)
    uint32_t dataPosition;          // 数据位置
    uint32_t handleFlags;           // 句柄标志
} TEE_ObjectInfo;
```

### 2. 句柄标志系统

```c
#define TEE_HANDLE_FLAG_PERSISTENT         0x00010000   // 持久化对象
#define TEE_HANDLE_FLAG_INITIALIZED        0x00020000   // 已初始化
#define TEE_HANDLE_FLAG_KEY_SET            0x00040000   // 密钥已设置
#define TEE_HANDLE_FLAG_EXPECT_TWO_KEYS    0x00080000   // 期望两个密钥
#define TEE_HANDLE_FLAG_EXTRACTING         0x00100000   // 正在提取
```

### 3. 数据访问标志

```c
#define TEE_DATA_FLAG_ACCESS_READ          0x00000001   // 读访问
#define TEE_DATA_FLAG_ACCESS_WRITE         0x00000002   // 写访问
#define TEE_DATA_FLAG_ACCESS_WRITE_META    0x00000004   // 元数据写访问
#define TEE_DATA_FLAG_SHARE_READ           0x00000010   // 共享读
#define TEE_DATA_FLAG_SHARE_WRITE          0x00000020   // 共享写
#define TEE_DATA_FLAG_OVERWRITE            0x00000400   // 覆盖标志
```

## 对象生命周期管理

### 1. 对象创建流程

#### 临时对象创建
```c
TEE_Result TEE_AllocateTransientObject(uint32_t objectType,
                                      uint32_t maxObjectSize,
                                      TEE_ObjectHandle *object);
```

#### 持久化对象创建
```c
TEE_Result TEE_CreatePersistentObject(uint32_t storageID,
                                     const void *objectID,
                                     size_t objectIDLen,
                                     uint32_t flags,
                                     TEE_ObjectHandle attributes,
                                     const void *initialData,
                                     size_t initialDataLen,
                                     TEE_ObjectHandle *object);
```

### 2. 对象打开和关闭

#### 对象打开
```c
TEE_Result TEE_OpenPersistentObject(uint32_t storageID,
                                   const void *objectID,
                                   size_t objectIDLen,
                                   uint32_t flags,
                                   TEE_ObjectHandle *object);
```

#### 对象关闭
```c
void TEE_CloseObject(TEE_ObjectHandle object);
void TEE_CloseAndDeletePersistentObject1(TEE_ObjectHandle object);
```

### 3. 对象属性操作

#### 属性获取
```c
TEE_Result TEE_GetObjectBufferAttribute(TEE_ObjectHandle object,
                                       uint32_t attributeID,
                                       void *buffer,
                                       size_t *size);

TEE_Result TEE_GetObjectValueAttribute(TEE_ObjectHandle object,
                                      uint32_t attributeID,
                                      uint32_t *a, uint32_t *b);
```

#### 属性设置
```c
TEE_Result TEE_PopulateTransientObject(TEE_ObjectHandle object,
                                      const TEE_Attribute *attrs,
                                      uint32_t attrCount);
```

### 4. 数据流操作

#### 数据读写
```c
TEE_Result TEE_ReadObjectData(TEE_ObjectHandle object,
                             void *buffer,
                             size_t size,
                             size_t *count);

TEE_Result TEE_WriteObjectData(TEE_ObjectHandle object,
                              const void *buffer,
                              size_t size);
```

#### 数据定位
```c
TEE_Result TEE_SeekObjectData(TEE_ObjectHandle object,
                             int32_t offset,
                             TEE_Whence whence);

TEE_Result TEE_TruncateObjectData(TEE_ObjectHandle object,
                                 size_t size);
```

## 对象枚举系统

### 1. 枚举器操作

#### 枚举器分配和释放
```c
TEE_Result TEE_AllocatePersistentObjectEnumerator(TEE_ObjectEnumHandle *objectEnumerator);
void TEE_FreePersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator);
```

#### 枚举控制
```c
TEE_Result TEE_StartPersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator,
                                              uint32_t storageID);

TEE_Result TEE_GetNextPersistentObject(TEE_ObjectEnumHandle objectEnumerator,
                                      TEE_ObjectInfo *objectInfo,
                                      void *objectID,
                                      size_t *objectIDLen);

void TEE_ResetPersistentObjectEnumerator(TEE_ObjectEnumHandle objectEnumerator);
```

## 对象安全性和验证

### 1. 对象类型验证

#### 类型兼容性检查
- 对象类型与算法兼容性验证
- 密钥长度与算法要求匹配
- 属性完整性检查

### 2. 访问权限控制

#### 对象访问权限
- READ: 读取对象数据和属性
- WRITE: 修改对象数据
- WRITE_META: 修改对象元数据
- SHARE_READ/WRITE: 跨TA共享访问

### 3. 对象完整性保护

#### 属性完整性
- 属性类型和值的一致性检查
- 密钥材料的数学关系验证
- 曲线参数的有效性验证

#### 数据完整性
- 对象数据的哈希验证
- 持久化对象的签名验证
- 属性和数据的关联性检查

## 高级对象特性

### 1. 对象引用和句柄管理

#### 句柄分配策略
- 每个TA独立的句柄空间
- 句柄有效性验证
- 自动资源清理

### 2. 对象缓存和性能优化

#### 属性缓存
- 频繁访问属性的内存缓存
- 惰性属性加载
- 批量属性操作优化

### 3. 对象序列化和持久化

#### 对象存储格式
- GP标准对象头格式
- 属性数据编码
- 版本兼容性管理

## 错误处理和异常情况

### 1. 对象错误类型

```c
#define TEE_ERROR_CORRUPT_OBJECT          0xF0100001   // 对象损坏
#define TEE_ERROR_CORRUPT_OBJECT_2        0xF0100002   // 对象损坏(变体)
#define TEE_ERROR_STORAGE_NOT_AVAILABLE   0xF0100003   // 存储不可用
#define TEE_ERROR_STORAGE_NOT_AVAILABLE_2 0xF0100004   // 存储不可用(变体)
#define TEE_ERROR_STORAGE_NO_SPACE        0xFFFF3041   // 存储空间不足
```

### 2. 对象恢复机制

#### 损坏对象处理
- 自动检测对象损坏
- 损坏对象隔离
- 备份对象恢复

#### 一致性保证
- 原子对象操作
- 事务性更新
- 回滚机制

这个详细的GP对象模型分析涵盖了OP-TEE中实现的完整对象类型系统、属性框架、生命周期管理和安全机制，为理解和使用OP-TEE存储系统提供了全面的技术参考。