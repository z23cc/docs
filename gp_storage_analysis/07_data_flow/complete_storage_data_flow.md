---
title: '完整存储数据流分析'
---

# OP-TEE GP存储完整数据流分析

## 总体架构数据流

### 系统层次结构
```
┌─────────────────────────────────────────────────────────────┐
│                    用户应用层                                │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │     TA代码      │    │      主机应用程序               │ │
│  │  (Trusted App)  │    │   (Host Application)            │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         ↓ GP API调用                    ↓ TEEC API调用
┌─────────────────────────────────────────────────────────────┐
│                   OP-TEE系统层                               │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │  TEE存储服务    │    │    TEE客户端库                 │ │
│  │ (Storage SVC)   │    │  (Client Library)               │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         ↓ 系统调用                      ↓ RPC调用
┌─────────────────────────────────────────────────────────────┐
│                   存储后端层                                │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   RPMB文件系统  │    │    REE文件系统                 │ │
│  │   (RPMB FS)     │    │    (REE FS)                     │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         ↓ 硬件访问                      ↓ RPC到tee-supplicant
┌─────────────────────────────────────────────────────────────┐
│                   物理存储层                                │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   eMMC RPMB分区 │    │    Linux文件系统                │ │
│  │  (Hardware)     │    │  (REE Filesystem)               │ │
│  └─────────────────┘    └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 详细数据流程分析

### 1. 对象创建流程 (TEE_CreatePersistentObject)

#### 第一阶段: TA API调用
```
TA代码调用:
TEE_CreatePersistentObject(storage_id, object_id, object_id_len, 
                          flags, attr_handle, data, data_len, &object)
         ↓
lib/libutee/tee_api_objects.c:
- 参数验证和转换
- 调用utee_storage_obj_create系统调用
         ↓
系统调用: utee_storage_obj_create()
```

#### 第二阶段: 系统调用处理
```
core/tee/tee_svc_storage.c: syscall_storage_obj_create()
         ↓
1. 参数验证和类型检查
2. 获取当前TA的UUID
3. 选择存储后端 (tee_svc_storage_file_ops)
4. 创建持久化对象结构 (tee_pobj_get)
         ↓
核心对象管理:
- tee_pobj_get(): 创建/获取对象引用
- 设置对象属性和访问权限
- 分配对象句柄
```

#### 第三阶段: 后端文件创建
```
存储后端选择:
if (storage_id == TEE_STORAGE_PRIVATE_RPMB)
    → rpmb_fs_ops.create()
else
    → ree_fs_ops.create()
         ↓
```

##### REE FS路径:
```
core/tee/tee_ree_fs.c: ree_fs_create_dfh()
         ↓
1. 构建文件路径: /TA_UUID/ObjectID/
2. 初始化哈希树 (tee_fs_htree_open)
3. 生成FEK密钥 (tee_fs_generate_fek)
4. 写入GP文件头 (tee_svc_storage_head)
5. 加密并写入数据
         ↓
RPC调用到REE:
core/tee/tee_fs_rpc.c: tee_fs_rpc_create()
- 设置RPC参数
- 调用thread_rpc_cmd()
         ↓
```

##### RPMB FS路径:
```
core/tee/tee_rpmb_fs.c: rpmb_fs_create()
         ↓
1. 分配RPMB文件条目
2. 生成文件加密密钥
3. 更新FAT表
4. 写入数据到RPMB分区
5. 更新write counter
```

#### 第四阶段: REE文件系统操作
```
REE端: tee-supplicant接收RPC
tee-supplicant/src/tee_supp_fs.c: tee_supp_fs_process()
         ↓
根据命令分发:
OPTEE_MRF_CREATE → ree_fs_new_create()
         ↓
1. 生成绝对路径: tee_fs_root + filename
2. 创建目录层次结构 (mkpath)
3. 调用Linux系统调用: open(O_CREAT | O_RDWR | O_TRUNC)
4. 返回文件描述符
         ↓
Linux文件系统:
- VFS层处理
- 具体文件系统实现 (ext4, f2fs等)
- 写入到物理存储设备
```

### 2. 对象读取流程 (TEE_ReadObjectData)

#### 第一阶段: TA API调用
```
TA代码调用:
TEE_ReadObjectData(object, buffer, size, &count)
         ↓
lib/libutee/tee_api_objects.c:
- 对象句柄验证
- 调用utee_storage_obj_read系统调用
```

#### 第二阶段: 系统调用处理
```
core/tee/tee_svc_storage.c: syscall_storage_obj_read()
         ↓
1. 验证对象句柄和权限
2. 获取对象当前位置
3. 调用后端读取操作
         ↓
```

##### REE FS读取路径:
```
core/tee/tee_ree_fs.c: ree_fs_read_dfh()
         ↓
1. 计算读取的块范围
2. 逐块读取和解密:
   for each block:
     - tee_fs_htree_read_block() 读取加密块
     - 验证哈希树完整性
     - 解密数据块 (AES-CBC + ESSIV)
3. 复制数据到用户缓冲区
         ↓
RPC读取:
core/tee/tee_fs_rpc.c: tee_fs_rpc_read()
- 设置读取偏移量和长度
- 调用thread_rpc_cmd(OPTEE_MSG_RPC_CMD_FS)
```

##### RPMB FS读取路径:
```
core/tee/tee_rpmb_fs.c: rpmb_fs_read()
         ↓
1. 定位RPMB文件位置
2. 计算读取块范围
3. 验证RPMB认证:
   - 检查write counter
   - 验证MAC值
4. 读取并解密数据
```

#### 第三阶段: REE文件操作
```
tee-supplicant处理OPTEE_MRF_READ:
ree_fs_new_read()
         ↓
1. 获取文件描述符和偏移量
2. Linux系统调用:
   - lseek(fd, offset, SEEK_SET)
   - read(fd, buffer, length)
3. 返回实际读取字节数
         ↓
数据返回路径:
REE → TEE → 解密 → TA缓冲区
```

### 3. 数据写入流程 (TEE_WriteObjectData)

#### 写入流程概述
```
TA调用 → 系统调用 → 后端选择 → 加密处理 → 完整性保护 → 物理写入
```

#### 详细写入路径

##### REE FS写入:
```
ree_fs_write_dfh()
         ↓
1. 写时复制机制 (out_of_place_write):
   - 分配临时块缓冲区
   - 读取原始块 (如果是部分写入)
   - 合并新数据
   - 加密整个块
2. 哈希树更新:
   - 计算新的块哈希
   - 更新父节点哈希
   - 传播到根节点
3. 原子提交:
   - 写入新版本
   - 更新计数器
   - 切换活跃版本
```

##### RPMB FS写入:
```
rpmb_fs_write()
         ↓
1. 数据准备:
   - 加密数据块
   - 计算新的write counter
2. RPMB操作:
   - 认证写入请求
   - 更新write counter
   - 写入数据到RPMB分区
3. 原子性保证:
   - RPMB硬件保证原子操作
   - write counter防重放
```

### 4. 密钥管理数据流

#### 密钥派生链
```
Hardware Unique Key (HUK)
         ↓ huk_subkey_derive(HUK_SUBKEY_SSK)
Secure Storage Key (SSK) [32字节]
         ↓ HMAC-SHA256(SSK, TA_UUID)
Trusted App Storage Key (TSK) [32字节]
         ↓ AES-ECB(TSK, random_fek)
Encrypted File Encryption Key (E-FEK) [16字节]
         ↓ AES-ECB(TSK, E-FEK) → FEK
File Encryption Key (FEK) [16字节]
         ↓ AES-CBC(FEK, data) + ESSIV
数据块加密
```

#### 密钥使用流程
```
文件创建时:
1. 生成随机FEK: crypto_rng_read()
2. 用TSK加密FEK: tee_fs_fek_crypt(encrypt)
3. 存储E-FEK到文件头

文件访问时:
1. 读取E-FEK从文件头
2. 用TSK解密FEK: tee_fs_fek_crypt(decrypt)
3. 用FEK加密/解密数据块
4. 立即清除内存中的FEK
```

### 5. 完整性保护数据流

#### 哈希树构建
```
数据块层:
Block 0: data0 → hash0
Block 1: data1 → hash1
Block 2: data2 → hash2
Block 3: data3 → hash3
         ↓
节点层:
Node 2: hash(hash0||hash1) + IV + Tag
Node 3: hash(hash2||hash3) + IV + Tag
         ↓
根节点:
Node 1: hash(Node2||Node3) + IV + Tag
         ↓
根哈希存储在文件头
```

#### 完整性验证流程
```
读取时验证:
1. 读取根节点 → 验证根哈希
2. 沿树向下验证:
   - 验证父节点哈希
   - 解密子节点
   - 验证GCM标签
3. 到达数据块:
   - 验证块哈希
   - 解密数据
   - 返回明文
```

### 6. RPC通信数据流

#### TEE到REE的RPC流程
```
TEE侧发起:
core/tee/tee_fs_rpc.c
         ↓
1. 准备RPC参数:
   - 命令类型 (OPTEE_MSG_RPC_CMD_FS)
   - 文件操作码 (OPTEE_MRF_*)
   - 共享内存缓冲区
2. 调用thread_rpc_cmd()
         ↓
内核驱动:
drivers/tee/optee/
- 切换到REE上下文
- 唤醒tee-supplicant
         ↓
REE侧处理:
tee-supplicant/src/tee_supp_fs.c
1. 解析RPC参数
2. 执行文件系统操作
3. 返回结果
         ↓
结果返回:
REE → 内核 → TEE → TA
```

#### 支持的RPC操作
```
OPTEE_MRF_OPEN      → 打开文件
OPTEE_MRF_CREATE    → 创建文件
OPTEE_MRF_CLOSE     → 关闭文件
OPTEE_MRF_READ      → 读取文件
OPTEE_MRF_WRITE     → 写入文件
OPTEE_MRF_TRUNCATE  → 截断文件
OPTEE_MRF_REMOVE    → 删除文件
OPTEE_MRF_RENAME    → 重命名文件
OPTEE_MRF_OPENDIR   → 打开目录
OPTEE_MRF_READDIR   → 读取目录
OPTEE_MRF_CLOSEDIR  → 关闭目录
```

### 7. 错误处理和恢复流程

#### 错误处理机制
```
TA层错误:
- 参数验证失败 → TEE_ERROR_BAD_PARAMETERS
- 权限检查失败 → TEE_ERROR_ACCESS_DENIED
- 对象不存在 → TEE_ERROR_ITEM_NOT_FOUND

系统层错误:
- 内存分配失败 → TEE_ERROR_OUT_OF_MEMORY
- 加密操作失败 → TEE_ERROR_GENERIC
- 完整性验证失败 → TEE_ERROR_CORRUPT_OBJECT

后端错误:
- 存储空间不足 → TEE_ERROR_STORAGE_NO_SPACE
- 文件系统错误 → TEE_ERROR_STORAGE_NOT_AVAILABLE
- RPMB认证失败 → TEE_ERROR_SECURITY
```

#### 恢复机制
```
对象损坏检测:
1. 哈希验证失败
2. GCM认证失败
3. 文件头损坏
         ↓
自动恢复:
- remove_corrupt_obj(): 自动删除损坏对象
- 回滚到上一个有效版本
- 清理相关资源

原子操作保证:
- 双版本机制: 失败时回滚到上一版本
- 写时复制: 不影响原始数据
- 事务性更新: 要么全部成功，要么全部失败
```

### 8. 性能优化数据流

#### 缓存机制
```
RPMB FS缓存:
- FAT条目缓存: CFG_RPMB_FS_CACHE_ENTRIES
- 读缓冲区: CFG_RPMB_FS_RD_ENTRIES
- 减少RPMB访问次数

REE FS优化:
- 块对齐读写
- 批量操作
- 预读机制
```

#### 内存管理
```
内存池使用:
- get_tmp_block(): 从内存池分配临时块
- put_tmp_block(): 释放到内存池
- 避免频繁malloc/free

敏感数据清除:
- memzero_explicit(): 清除密钥材料
- 函数退出时自动清理
- 防止内存泄露密钥信息
```

## 数据流总结

### 关键特征
1. **多层安全**: 从API到物理存储的全链条保护
2. **原子操作**: 双版本机制确保数据一致性
3. **强加密**: 四层密钥体系和AES-GCM加密
4. **完整性**: SHA256哈希树保护
5. **灵活性**: 支持多种存储后端
6. **性能优化**: 缓存和内存池机制

### 安全保证
1. **机密性**: 分层加密保护数据内容
2. **完整性**: 哈希树检测任何篡改
3. **原子性**: 双版本确保更新原子性
4. **隔离性**: 基于TA UUID的完全隔离
5. **防重放**: RPMB计数器防止重放攻击

### 性能特点
1. **REE FS**: 高性能，大容量，软件安全性
2. **RPMB FS**: 硬件安全性，小容量，较低性能
3. **缓存优化**: 减少底层存储访问
4. **并发支持**: 多TA并发访问
5. **块级操作**: 4KB块大小优化

这个完整的数据流程确保了OP-TEE存储系统既满足GP标准要求，又提供了工业级的安全性和性能。