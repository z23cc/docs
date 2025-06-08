---
title: '存储示例和测试分析'
---

# OP-TEE存储示例和测试分析

## 存储示例应用

### 1. 安全存储示例 (Secure Storage Example)

#### 示例位置
- **TA实现**: `/optee_examples/secure_storage/ta/secure_storage_ta.c`
- **主机应用**: `/optee_examples/secure_storage/host/main.c`
- **头文件**: `/optee_examples/secure_storage/ta/include/secure_storage_ta.h`

#### 主要功能演示

##### TA侧实现关键函数

###### 删除对象操作
```c
static TEE_Result delete_object(uint32_t param_types, TEE_Param params[4])
{
    const uint32_t exp_param_types =
        TEE_PARAM_TYPES(TEE_PARAM_TYPE_MEMREF_INPUT,  // 对象ID
                       TEE_PARAM_TYPE_NONE,
                       TEE_PARAM_TYPE_NONE,
                       TEE_PARAM_TYPE_NONE);
    TEE_ObjectHandle object;
    TEE_Result res;
    char *obj_id;
    size_t obj_id_sz;
    
    // 1. 参数验证
    if (param_types != exp_param_types)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 2. 复制对象ID到TA内存
    obj_id_sz = params[0].memref.size;
    obj_id = TEE_Malloc(obj_id_sz, 0);
    if (!obj_id)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    TEE_MemMove(obj_id, params[0].memref.buffer, obj_id_sz);
    
    // 3. 打开对象(需要写权限才能删除)
    res = TEE_OpenPersistentObject(TEE_STORAGE_PRIVATE,
                                  obj_id, obj_id_sz,
                                  TEE_DATA_FLAG_ACCESS_READ |
                                  TEE_DATA_FLAG_ACCESS_WRITE_META,
                                  &object);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to open persistent object, res=0x%08x", res);
        TEE_Free(obj_id);
        return res;
    }
    
    // 4. 关闭并删除对象
    TEE_CloseAndDeletePersistentObject1(object);
    TEE_Free(obj_id);
    
    return res;
}
```

###### 创建原始对象操作
```c
static TEE_Result create_raw_object(uint32_t param_types, TEE_Param params[4])
{
    const uint32_t exp_param_types =
        TEE_PARAM_TYPES(TEE_PARAM_TYPE_MEMREF_INPUT,  // 对象ID
                       TEE_PARAM_TYPE_MEMREF_INPUT,   // 数据
                       TEE_PARAM_TYPE_NONE,
                       TEE_PARAM_TYPE_NONE);
    TEE_ObjectHandle object;
    TEE_Result res;
    char *obj_id;
    size_t obj_id_sz;
    char *data;
    size_t data_sz;
    uint32_t obj_data_flag;
    
    // 1. 参数验证
    if (param_types != exp_param_types)
        return TEE_ERROR_BAD_PARAMETERS;
    
    // 2. 获取对象ID
    obj_id_sz = params[0].memref.size;
    obj_id = TEE_Malloc(obj_id_sz, 0);
    if (!obj_id)
        return TEE_ERROR_OUT_OF_MEMORY;
    TEE_MemMove(obj_id, params[0].memref.buffer, obj_id_sz);
    
    // 3. 获取数据
    data_sz = params[1].memref.size;
    data = TEE_Malloc(data_sz, 0);
    if (!data) {
        res = TEE_ERROR_OUT_OF_MEMORY;
        goto exit;
    }
    TEE_MemMove(data, params[1].memref.buffer, data_sz);
    
    // 4. 设置对象标志
    obj_data_flag = TEE_DATA_FLAG_ACCESS_READ |
                   TEE_DATA_FLAG_ACCESS_WRITE |
                   TEE_DATA_FLAG_ACCESS_WRITE_META |
                   TEE_DATA_FLAG_OVERWRITE;
    
    // 5. 创建持久化对象
    res = TEE_CreatePersistentObject(TEE_STORAGE_PRIVATE,
                                    obj_id, obj_id_sz,
                                    obj_data_flag,
                                    TEE_HANDLE_NULL,  // 无属性对象
                                    data, data_sz,    // 初始数据
                                    &object);
    if (res != TEE_SUCCESS) {
        EMSG("TEE_CreatePersistentObject failed 0x%08x", res);
        goto exit;
    }
    
    TEE_CloseObject(object);
    
exit:
    TEE_Free(data);
    TEE_Free(obj_id);
    return res;
}
```

##### 主机应用关键函数

###### TEE会话管理
```c
struct test_ctx {
    TEEC_Context ctx;     // TEE上下文
    TEEC_Session sess;    // TA会话
};

void prepare_tee_session(struct test_ctx *ctx)
{
    TEEC_UUID uuid = TA_SECURE_STORAGE_UUID;
    uint32_t origin;
    TEEC_Result res;
    
    // 1. 初始化TEE上下文
    res = TEEC_InitializeContext(NULL, &ctx->ctx);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_InitializeContext failed with code 0x%x", res);
    
    // 2. 打开TA会话
    res = TEEC_OpenSession(&ctx->ctx, &ctx->sess, &uuid,
                          TEEC_LOGIN_PUBLIC, NULL, NULL, &origin);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_Opensession failed with code 0x%x origin 0x%x",
            res, origin);
}

void terminate_tee_session(struct test_ctx *ctx)
{
    TEEC_CloseSession(&ctx->sess);
    TEEC_FinalizeContext(&ctx->ctx);
}
```

###### 读取安全对象
```c
TEEC_Result read_secure_object(struct test_ctx *ctx, char *id,
                              char *data, size_t data_len)
{
    TEEC_Operation op;
    uint32_t origin;
    TEEC_Result res;
    size_t id_len = strlen(id);
    
    // 1. 设置操作参数
    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(TEEC_MEMREF_TEMP_INPUT,   // 对象ID
                                    TEEC_MEMREF_TEMP_OUTPUT,    // 数据缓冲区
                                    TEEC_NONE, TEEC_NONE);
    
    op.params[0].tmpref.buffer = id;
    op.params[0].tmpref.size = id_len;
    
    op.params[1].tmpref.buffer = data;
    op.params[1].tmpref.size = data_len;
    
    // 2. 调用TA命令
    res = TEEC_InvokeCommand(&ctx->sess,
                            TA_SECURE_STORAGE_CMD_READ_RAW,
                            &op, &origin);
    switch (res) {
        case TEEC_SUCCESS:
        case TEEC_ERROR_SHORT_BUFFER:    // 缓冲区太小
        case TEEC_ERROR_ITEM_NOT_FOUND:  // 对象不存在
            break;
        default:
            printf("Command READ_RAW failed: 0x%x / %u\n", res, origin);
    }
    
    return res;
}
```

## 存储测试套件

### 1. 存储功能测试 (Storage TA)

#### 测试位置
- **TA实现**: `/optee_test/ta/storage/storage.c`
- **测试用例**: `/optee_test/host/xtest/regression_6000.c`
- **头文件**: `/optee_test/ta/include/ta_storage.h`

#### 核心测试功能

##### 对象打开测试
```c
TEE_Result ta_storage_cmd_open(uint32_t command,
                              uint32_t param_types, TEE_Param params[4])
{
    TEE_Result res = TEE_ERROR_GENERIC;
    TEE_ObjectHandle o = TEE_HANDLE_NULL;
    void *object_id = NULL;
    
    // 参数类型验证
    ASSERT_PARAM_TYPE(TEE_PARAM_TYPES(
        TEE_PARAM_TYPE_MEMREF_INPUT,    // 对象ID
        TEE_PARAM_TYPE_VALUE_INOUT,     // 标志和句柄
        TEE_PARAM_TYPE_VALUE_INPUT,     // 存储ID
        TEE_PARAM_TYPE_NONE));
    
    switch (command) {
        case TA_STORAGE_CMD_OPEN:
            // 复制对象ID到TA内存
            if (params[0].memref.buffer) {
                object_id = TEE_Malloc(params[0].memref.size, 0);
                if (!object_id)
                    return TEE_ERROR_OUT_OF_MEMORY;
                TEE_MemMove(object_id, params[0].memref.buffer,
                           params[0].memref.size);
            }
            break;
            
        case TA_STORAGE_CMD_OPEN_ID_IN_SHM:
            // 直接使用共享内存中的对象ID
            object_id = params[0].memref.buffer;
            break;
            
        default:
            return TEE_ERROR_NOT_SUPPORTED;
    }
    
    // 打开持久化对象
    res = TEE_OpenPersistentObject(params[2].value.a,      // 存储ID
                                  object_id,               // 对象ID
                                  params[0].memref.size,   // ID长度
                                  params[1].value.a,       // 访问标志
                                  &o);                     // 对象句柄
    
    params[1].value.b = (uintptr_t)o;  // 返回句柄给测试
    
    if (command == TA_STORAGE_CMD_OPEN)
        TEE_Free(object_id);
    
    return res;
}
```

##### 对象创建测试
```c
TEE_Result ta_storage_cmd_create(uint32_t command,
                                uint32_t param_types, TEE_Param params[4])
{
    TEE_Result res = TEE_ERROR_GENERIC;
    TEE_ObjectHandle o = TEE_HANDLE_NULL;
    void *object_id = NULL;
    TEE_ObjectHandle ref_handle = TEE_HANDLE_NULL;
    
    // 参数验证
    ASSERT_PARAM_TYPE(TEE_PARAM_TYPES(
        TEE_PARAM_TYPE_MEMREF_INPUT,    // 对象ID
        TEE_PARAM_TYPE_VALUE_INOUT,     // 标志和句柄
        TEE_PARAM_TYPE_VALUE_INPUT,     // 句柄和存储ID
        TEE_PARAM_TYPE_MEMREF_INPUT));  // 初始数据
    
    // 获取对象ID
    switch (command) {
        case TA_STORAGE_CMD_CREATE:
            if (params[0].memref.buffer) {
                object_id = TEE_Malloc(params[0].memref.size, 0);
                if (!object_id)
                    return TEE_ERROR_OUT_OF_MEMORY;
                TEE_MemMove(object_id, params[0].memref.buffer,
                           params[0].memref.size);
            }
            break;
            
        case TA_STORAGE_CMD_CREATE_ID_IN_SHM:
            object_id = params[0].memref.buffer;
            break;
            
        default:
            return TEE_ERROR_NOT_SUPPORTED;
    }
    
    ref_handle = (TEE_ObjectHandle)(uintptr_t)params[2].value.a;
    
    // 创建持久化对象
    res = TEE_CreatePersistentObject(params[2].value.b,     // 存储ID
                                    object_id,              // 对象ID
                                    params[0].memref.size,  // ID长度
                                    params[1].value.a,      // 访问标志
                                    ref_handle,             // 属性参考句柄
                                    params[3].memref.buffer, // 初始数据
                                    params[3].memref.size,  // 数据长度
                                    &o);                    // 对象句柄
    
    params[1].value.b = (uintptr_t)o;
    
    if (command == TA_STORAGE_CMD_CREATE)
        TEE_Free(object_id);
    
    return res;
}
```

#### 回归测试框架

##### 多存储后端测试宏
```c
#define DEFINE_TEST_MULTIPLE_STORAGE_IDS(test_name) \
static void test_name(ADBG_Case_t *c) \
{ \
    size_t i = 0; \
    \
    if (init_storage_info()) { \
        Do_ADBG_Log("init_storage_info() failed"); \
        return; \
    } \
    for (i = 0; i < ARRAY_SIZE(storage_info); i++) { \
        uint32_t id = storage_info[i].id; \
        \
        if (!storage_info[i].available) \
            continue; \
        Do_ADBG_BeginSubCase(c, "Storage id: %08x", id); \
        test_name##_single(c, id); \
        Do_ADBG_EndSubCase(c, "Storage id: %08x", id); \
    } \
}
```

##### 文件系统操作封装
```c
static TEEC_Result fs_open(TEEC_Session *sess, void *id, uint32_t id_size,
                          uint32_t flags, uint32_t *obj, uint32_t storage_id)
{
    TEEC_Operation op = TEEC_OPERATION_INITIALIZER;
    TEEC_Result res = TEEC_ERROR_GENERIC;
    uint32_t org = 0;
    
    // 设置参数
    op.params[0].tmpref.buffer = id;
    op.params[0].tmpref.size = id_size;
    op.params[1].value.a = flags;
    op.params[1].value.b = 0;
    op.params[2].value.a = storage_id;
    
    op.paramTypes = TEEC_PARAM_TYPES(TEEC_MEMREF_TEMP_INPUT,
                                    TEEC_VALUE_INOUT, 
                                    TEEC_VALUE_INPUT,
                                    TEEC_NONE);
    
    res = TEEC_InvokeCommand(sess, TA_STORAGE_CMD_OPEN, &op, &org);
    
    if (res == TEEC_SUCCESS)
        *obj = op.params[1].value.b;
    
    return res;
}
```

### 2. 存储性能测试 (Storage Benchmark)

#### 测试位置
- **TA实现**: `/optee_test/ta/storage_benchmark/benchmark.c`
- **头文件**: `/optee_test/ta/storage_benchmark/include/storage_benchmark.h`

#### 性能测试关键功能

##### 测试配置
```c
#define DEFAULT_CHUNK_SIZE (1 << 10)    // 1KB块大小
#define DEFAULT_DATA_SIZE (1024)        // 1KB数据大小
#define SCRAMBLE(x) ((x & 0xff) ^ 0xaa) // 数据生成模式

static uint8_t filename[] = "BenchmarkTestFile";
```

##### 数据生成和验证
```c
// 填充测试数据
static void fill_buffer(uint8_t *buf, size_t size)
{
    size_t i = 0;
    
    if (!buf)
        return;
    
    for (i = 0; i < size; i++)
        buf[i] = SCRAMBLE(i);  // 生成可预测的测试数据
}

// 验证数据完整性
static TEE_Result verify_buffer(uint8_t *buf, size_t size)
{
    size_t i = 0;
    
    if (!buf)
        return TEE_ERROR_BAD_PARAMETERS;
    
    for (i = 0; i < size; i++) {
        uint8_t expect_data = SCRAMBLE(i);
        
        if (expect_data != buf[i]) {
            return TEE_ERROR_CORRUPT_OBJECT; // 数据损坏
        }
    }
    
    return TEE_SUCCESS;
}
```

##### 时间测量
```c
static inline uint32_t tee_time_to_ms(TEE_Time t)
{
    return t.seconds * 1000 + t.millis;
}

static inline uint32_t get_delta_time_in_ms(TEE_Time start, TEE_Time stop)
{
    return tee_time_to_ms(stop) - tee_time_to_ms(start);
}
```

##### 测试文件准备
```c
static TEE_Result prepare_test_file(size_t data_size, uint8_t *chunk_buf,
                                   size_t chunk_size)
{
    size_t remain_bytes = data_size;
    TEE_Result res = TEE_SUCCESS;
    TEE_ObjectHandle object = TEE_HANDLE_NULL;
    
    // 创建测试文件
    res = TEE_CreatePersistentObject(TEE_STORAGE_PRIVATE,
                                    filename, sizeof(filename),
                                    TEE_DATA_FLAG_ACCESS_READ |
                                    TEE_DATA_FLAG_ACCESS_WRITE |
                                    TEE_DATA_FLAG_ACCESS_WRITE_META |
                                    TEE_DATA_FLAG_OVERWRITE,
                                    NULL, NULL, 0, &object);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to create persistent object, res=%#"PRIx32, res);
        goto exit;
    }
    
    // 分块写入数据
    while (remain_bytes) {
        size_t write_size = 0;
        
        if (remain_bytes < chunk_size)
            write_size = remain_bytes;
        else
            write_size = chunk_size;
            
        res = TEE_WriteObjectData(object, chunk_buf, write_size);
        if (res != TEE_SUCCESS) {
            EMSG("Failed to write data, res=%#"PRIx32, res);
            goto exit_close_object;
        }
        remain_bytes -= write_size;
    }
    
exit_close_object:
    TEE_CloseObject(object);
exit:
    return res;
}
```

## 测试数据样本

### 预定义测试文件
```c
// 32字节测试文件
static uint8_t file_00[] = {
    0x00, 0x6E, 0x04, 0x57, 0x08, 0xFB, 0x71, 0x96,
    0xF0, 0x2E, 0x55, 0x3D, 0x02, 0xC3, 0xA6, 0x92,
    0xE9, 0xC3, 0xEF, 0x8A, 0xB2, 0x34, 0x53, 0xE6,
    0xF0, 0x74, 0x9C, 0xD6, 0x36, 0xE7, 0xA8, 0x8E
};

// 2字节测试文件
static uint8_t file_01[] = {
    0x01, 0x00
};

// 3字节测试文件
static uint8_t file_02[] = {
    0x02, 0x11, 0x02
};
```

### 测试数据模式
```c
// 第一种数据模式
static uint8_t data_00[] = {
    0x00, 0x6E, 0x04, 0x57, 0x08, 0xFB, 0x71, 0x96,
    0x00, 0x2E, 0x55, 0x3D, 0x02, 0xC3, 0xA6, 0x92,
    0x00, 0xC3, 0xEF, 0x8A, 0xB2, 0x34, 0x53, 0xE6,
    0x00, 0x74, 0x9C, 0xD6, 0x36, 0xE7, 0xA8, 0x00
};

// 第二种数据模式
static uint8_t data_01[] = {
    0x01, 0x6E, 0x04, 0x57, 0x08, 0xFB, 0x71, 0x96,
    0x01, 0x2E, 0x55, 0x3D, 0x02, 0xC3, 0xA6, 0x92,
    0x01, 0xC3, 0xEF, 0x8A, 0xB2, 0x34, 0x53, 0xE6,
    0x01, 0x74, 0x9C, 0xD6, 0x36, 0xE7, 0xA8, 0x01
};
```

## 测试覆盖范围

### 功能测试覆盖
1. **对象生命周期**: 创建、打开、读写、删除
2. **存储后端**: REE FS和RPMB FS
3. **并发访问**: 多TA访问同一存储
4. **错误处理**: 各种异常情况处理
5. **权限控制**: 访问权限验证
6. **数据完整性**: 数据读写正确性验证

### 性能测试覆盖
1. **吞吐量测试**: 不同块大小的读写性能
2. **延迟测试**: 操作响应时间测量
3. **容量测试**: 大文件存储性能
4. **并发性能**: 多线程访问性能
5. **缓存影响**: 缓存对性能的影响

### 安全测试覆盖
1. **隔离验证**: TA间存储隔离
2. **加密验证**: 数据加密正确性
3. **完整性验证**: 哈希树完整性保护
4. **权限验证**: 访问控制有效性
5. **攻击防护**: 各种攻击场景测试

## 测试框架特点

### 1. 自动化测试
- **回归测试**: 自动运行所有存储测试
- **多后端支持**: 自动在所有可用后端上运行测试
- **结果验证**: 自动验证测试结果正确性

### 2. 性能基准
- **标准测试**: 提供标准性能基准测试
- **可配置参数**: 支持不同测试参数配置
- **详细度量**: 提供详细的性能度量数据

### 3. 错误诊断
- **详细日志**: 提供详细的错误诊断信息
- **状态检查**: 检查系统状态和配置
- **异常恢复**: 测试异常情况下的恢复能力

这些示例和测试为开发者提供了：
1. **学习材料**: 如何正确使用GP存储API
2. **测试基准**: 验证存储功能正确性
3. **性能参考**: 了解存储系统性能特征
4. **调试工具**: 诊断存储相关问题