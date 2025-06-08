# NVMEM Framework Integration Analysis

## Overview

OP-TEE 的 NVMEM (Non-Volatile Memory) 框架提供了统一的抽象层，用于访问包含关键安全数据的非易失性存储器单元。该框架支持多种平台特定的 OTP (One-Time Programmable) 和熔丝 (fuse) 实现，为硬件唯一密钥 (HUK)、设备标识符和制造数据提供安全访问。

## NVMEM Framework Architecture

### 系统架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                TEE Application Layer                       │
│     • tee_otp_get_hw_unique_key()  • tee_otp_get_die_id() │
├─────────────────────────────────────────────────────────────┤
│              Crypto Framework Integration                  │
│    • huk_subkey_derive()  • Storage key derivation        │
├─────────────────────────────────────────────────────────────┤
│                NVMEM Framework API                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │   nvmem_get_cell_by_name() • nvmem_cell_read()        ││
│  │   nvmem_get_cell_by_index() • nvmem_cell_malloc()     ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│              Device Tree Integration Layer                │
│    • FDT parsing  • Cell binding  • Compatible matching   │
├─────────────────────────────────────────────────────────────┤
│           Platform-Specific NVMEM Drivers                │
│  ┌───────────────┐ ┌───────────────┐ ┌──────────────────┐  │
│  │  Atmel SFC    │ │  STM32 BSEC   │ │   iMX OCOTP     │  │
│  │ • Secure Fuse │ │ • Boot & OTP  │ │ • On-Chip OTP   │  │
│  │ • SAMA5D2     │ │ • STM32MP1x   │ │ • iMX 6/7/8     │  │
│  └───────────────┘ └───────────────┘ └──────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│              Hardware Abstraction Layer                   │
│    • Register access • Security config • Lock management │
├─────────────────────────────────────────────────────────────┤
│                Hardware NVMEM Devices                     │
│     • OTP cells • eFuses • Shadow registers • Locks      │
└─────────────────────────────────────────────────────────────┘
```

### 核心数据结构

#### NVMEM Cell 结构

```c
struct nvmem_cell {
    paddr_t offset;                    // 存储器单元字节偏移
    size_t len;                        // 单元字节大小
    const struct nvmem_ops *ops;       // 设备驱动操作表
    void *drv_data;                    // 驱动私有数据
};

struct nvmem_ops {
    TEE_Result (*read_cell)(struct nvmem_cell *cell, uint8_t *data);
    void (*put_cell)(struct nvmem_cell *cell);
};
```

#### NVMEM Provider 结构

```c
struct nvmem_provider {
    const char *compatible;            // 设备树兼容字符串
    TEE_Result (*probe)(const void *fdt, int node, const void *compat_data);
    const void *compat_data;           // 兼容性数据
};

#define NVMEM_PROVIDER(compat, probe_func, data) \
    static const struct nvmem_provider __nvmem_provider_##probe_func \
    __used __section(".nvmem_providers") = { \
        .compatible = compat, \
        .probe = probe_func, \
        .compat_data = data, \
    }
```

## Framework API Functions

### 1. 设备树集成 API

#### 按名称获取单元

```c
TEE_Result nvmem_get_cell_by_name(const void *fdt, int nodeoffset,
                                  const char *name, struct nvmem_cell **cell)
{
    int index = -1;
    
    // 在 nvmem-cell-names 属性中查找指定名称
    index = fdt_stringlist_search(fdt, nodeoffset, "nvmem-cell-names", name);
    if (index < 0) {
        DMSG("NVMEM cell name '%s' not found", name);
        return TEE_ERROR_ITEM_NOT_FOUND;
    }
    
    // 根据索引获取对应的单元引用
    return nvmem_get_cell_by_index(fdt, nodeoffset, index, cell);
}
```

#### 按索引获取单元

```c
TEE_Result nvmem_get_cell_by_index(const void *fdt, int nodeoffset,
                                   unsigned int index, struct nvmem_cell **out_cell)
{
    struct dt_driver_phandle_args args = { 0 };
    struct nvmem_cell *cell = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 解析设备树 phandle 引用
    res = dt_driver_device_from_node_idx_prop("nvmem-cells", fdt, nodeoffset,
                                            index, DT_DRIVER_NVMEM, &args);
    if (res != TEE_SUCCESS)
        return res;
    
    // 分配并初始化 NVMEM 单元
    cell = calloc(1, sizeof(*cell));
    if (!cell)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    // 从设备树参数提取偏移和长度
    if (args.args_count >= 2) {
        cell->offset = args.args[0];
        cell->len = args.args[1];
    } else {
        free(cell);
        return TEE_ERROR_BAD_FORMAT;
    }
    
    // 获取 NVMEM 设备操作表
    cell->ops = (const struct nvmem_ops *)args.driver_data;
    cell->drv_data = args.node_data;
    
    *out_cell = cell;
    return TEE_SUCCESS;
}
```

### 2. 单元访问 API

#### 读取单元数据

```c
TEE_Result nvmem_cell_read(struct nvmem_cell *cell, uint8_t *data)
{
    if (!cell || !cell->ops || !cell->ops->read_cell)
        return TEE_ERROR_NOT_IMPLEMENTED;
        
    return cell->ops->read_cell(cell, data);
}
```

#### 分配内存并读取

```c
TEE_Result nvmem_cell_malloc_and_read(struct nvmem_cell *cell, uint8_t **data)
{
    uint8_t *buf = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    if (!cell || !data)
        return TEE_ERROR_BAD_PARAMETERS;
    
    buf = malloc(cell->len);
    if (!buf)
        return TEE_ERROR_OUT_OF_MEMORY;
    
    res = nvmem_cell_read(cell, buf);
    if (res != TEE_SUCCESS) {
        free(buf);
        return res;
    }
    
    *data = buf;
    return TEE_SUCCESS;
}
```

#### 释放单元资源

```c
void nvmem_put_cell(struct nvmem_cell *cell)
{
    if (!cell)
        return;
        
    if (cell->ops && cell->ops->put_cell)
        cell->ops->put_cell(cell);
        
    free(cell);
}
```

## Platform-Specific NVMEM Implementations

### 1. Atmel SFC (Secure Fuse Controller)

#### 驱动实现

**文件**: `core/drivers/nvmem/atmel_sfc.c`

```c
#define ATMEL_SFC_CELLS_32    17
#define ATMEL_SFC_CELLS_8     (ATMEL_SFC_CELLS_32 * sizeof(uint32_t))

// SFC 寄存器定义
#define SFC_DR(n)            (0x20 + ((n) * 4))    // 数据寄存器

struct atmel_sfc {
    vaddr_t base;                      // 寄存器基地址
    uint8_t fuses[ATMEL_SFC_CELLS_8];  // 熔丝数据缓存 (68字节)
};

static const struct nvmem_ops atmel_sfc_nvmem_ops = {
    .read_cell = atmel_sfc_read_cell,
    .put_cell = atmel_sfc_put_cell,
};
```

#### 单元读取实现

```c
static TEE_Result atmel_sfc_read_cell(struct nvmem_cell *cell, uint8_t *data)
{
    struct atmel_sfc *sfc = cell->drv_data;
    
    // 验证偏移和长度
    if (cell->offset + cell->len > ATMEL_SFC_CELLS_8) {
        EMSG("Cell access out of range: offset=0x%lx, len=%zu", 
             cell->offset, cell->len);
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    // 从缓存的熔丝数据中复制
    memcpy(data, &sfc->fuses[cell->offset], cell->len);
    
    return TEE_SUCCESS;
}
```

#### 设备初始化

```c
static TEE_Result atmel_sfc_probe(const void *fdt, int node,
                                  const void *compat_data __unused)
{
    struct atmel_sfc *sfc = NULL;
    paddr_t pbase = 0;
    size_t size = 0;
    uint32_t val = 0;
    
    // 获取设备树寄存器信息
    if (dt_map_dev(fdt, node, &sfc->base, &size, DT_MAP_AUTO) < 0) {
        EMSG("Failed to map SFC registers");
        return TEE_ERROR_GENERIC;
    }
    
    // 配置硬件安全性
    matrix_configure_periph_secure(AT91C_ID_SFC);
    
    // 读取所有熔丝数据到缓存
    for (unsigned int i = 0; i < ATMEL_SFC_CELLS_32; i++) {
        val = io_read32(sfc->base + SFC_DR(i));
        memcpy(&sfc->fuses[i * sizeof(uint32_t)], &val, sizeof(val));
    }
    
    return nvmem_register_provider(fdt, node, &atmel_sfc_nvmem_ops, sfc);
}

NVMEM_PROVIDER("atmel,sama5d2-sfc", atmel_sfc_probe, NULL);
```

### 2. STM32 BSEC (Boot and Security and OTP Controller)

#### 驱动特性

**文件**: `core/drivers/stm32_bsec.c`

```c
// BSEC 寄存器偏移
#define BSEC_OTP_CONF_OFF       U(0x000)
#define BSEC_OTP_CTRL_OFF       U(0x004)
#define BSEC_OTP_WRDATA_OFF     U(0x008)
#define BSEC_OTP_STATUS_OFF     U(0x00C)
#define BSEC_OTP_LOCK_OFF       U(0x010)
#define BSEC_OTP_DATA_OFF       U(0x200)

// 状态位定义
#define BSEC_MODE_SECURED       BIT(0)     // 安全模式
#define BSEC_MODE_BUSY          BIT(3)     // 忙状态
#define BSEC_MODE_PROGFAIL      BIT(4)     // 编程失败
#define BSEC_MODE_PWR_FAIL      BIT(5)     // 电源失败

// 平台兼容性
#ifdef CFG_STM32MP13
#define DT_BSEC_COMPAT "st,stm32mp13-bsec"
#define STM32MP1_OTP_MAX_ID     U(95)
#endif

#ifdef CFG_STM32MP15  
#define DT_BSEC_COMPAT "st,stm32mp15-bsec"
#define STM32MP1_OTP_MAX_ID     U(95)
#endif
```

#### 安全读取机制

```c
TEE_Result stm32_bsec_shadow_read_otp(uint32_t *val, uint32_t otp_id)
{
    TEE_Result result = TEE_SUCCESS;
    vaddr_t otp_data_base = stm32_bsec.base + BSEC_OTP_DATA_OFF;
    
    // 验证 OTP ID 范围
    if (otp_id > STM32MP1_OTP_MAX_ID) {
        EMSG("OTP ID %u out of range", otp_id);
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    // 检查设备状态
    if (stm32_bsec_busy()) {
        EMSG("BSEC device is busy");
        return TEE_ERROR_BUSY;
    }
    
    // 从影子寄存器读取
    *val = io_read32(otp_data_base + otp_id * sizeof(uint32_t));
    
    return result;
}
```

#### 锁定机制管理

```c
TEE_Result stm32_bsec_read_permanent_lock(uint32_t otp_id, bool *locked)
{
    uint32_t bank = otp_id / STM32MP1_OTP_BANK_SIZE;
    uint32_t otp_mask = BIT(otp_id & STM32MP1_OTP_BANK_MASK);
    uint32_t bank_value = 0;
    vaddr_t bank_address = 0;
    
    // 验证参数
    if (otp_id > STM32MP1_OTP_MAX_ID || !locked)
        return TEE_ERROR_BAD_PARAMETERS;
        
    bank_address = stm32_bsec.base + BSEC_OTP_LOCK_OFF + bank * sizeof(uint32_t);
    
    // 读取锁定状态寄存器
    bank_value = io_read32(bank_address);
    *locked = (bank_value & otp_mask) != 0;
    
    return TEE_SUCCESS;
}
```

### 3. iMX OCOTP (On-Chip OTP Controller)

#### 驱动架构

**文件**: `core/drivers/imx_ocotp.c`

```c
// OCOTP 寄存器定义
#define OCOTP_CTRL              0x0000
#define OCOTP_TIMING            0x0010
#define OCOTP_DATA              0x0020
#define OCOTP_READ_CTRL         0x0030
#define OCOTP_READ_FUSE_DATA    0x0040
#define OCOTP_SW_STICKY         0x0050
#define OCOTP_SCS               0x0060
#define OCOTP_VERSION           0x0090

// 控制位定义
#define OCOTP_CTRL_BUSY         BIT(8)
#define OCOTP_CTRL_ERROR        BIT(9)
#define OCOTP_CTRL_RELOAD_SHADOWS BIT(10)

// 影子寄存器偏移计算
#define OCOTP_SHADOW_OFFSET(bank, word) \
    (0x400 + (bank) * 0x80 + (word) * 0x10)

struct ocotp_instance {
    unsigned char nb_banks;            // 存储体数量
    unsigned char nb_words;            // 每个存储体字数
    TEE_Result (*get_die_id)(uint64_t *ret_uid);  // 获取芯片ID回调
};
```

#### 安全读取实现

```c
TEE_Result imx_ocotp_read(unsigned int bank, unsigned int word, uint32_t *val)
{
    TEE_Result ret = TEE_ERROR_GENERIC;
    
    // 参数验证
    if (!val || bank > g_ocotp->nb_banks || word > g_ocotp->nb_words) {
        EMSG("Invalid parameters: bank=%u, word=%u", bank, word);
        return TEE_ERROR_BAD_PARAMETERS;
    }
    
    mutex_lock(&fuse_read);
    
    // 启用 OCOTP 时钟
    ocotp_clock_enable();
    
    // 清除错误位并等待完成
    io_clrbits32(g_base_addr + OCOTP_CTRL, OCOTP_CTRL_ERROR);
    ret = ocotp_ctrl_wait_for(OCOTP_CTRL_ERROR | OCOTP_CTRL_BUSY);
    if (ret != TEE_SUCCESS) {
        EMSG("OCOTP controller timeout or error");
        goto out;
    }
    
    // 从影子寄存器读取
    *val = io_read32(g_base_addr + OCOTP_SHADOW_OFFSET(bank, word));
    ret = TEE_SUCCESS;
    
out:
    ocotp_clock_disable();
    mutex_unlock(&fuse_read);
    return ret;
}
```

#### 设备 ID 提取

```c
static TEE_Result imx6_get_die_id(uint64_t *ret_uid)
{
    uint32_t uid_low = 0, uid_high = 0;
    TEE_Result ret = TEE_SUCCESS;
    
    // 读取 UID 的低32位 (Bank 0, Word 1)
    ret = imx_ocotp_read(0, 1, &uid_low);
    if (ret != TEE_SUCCESS)
        return ret;
        
    // 读取 UID 的高32位 (Bank 0, Word 2)  
    ret = imx_ocotp_read(0, 2, &uid_high);
    if (ret != TEE_SUCCESS)
        return ret;
    
    // 组合64位设备ID
    *ret_uid = ((uint64_t)uid_high << 32) | uid_low;
    
    return TEE_SUCCESS;
}
```

## Device Tree Integration Patterns

### 设备树绑定规范

#### NVMEM Provider 节点

```dts
sfc: sfc@f804c000 {
    compatible = "atmel,sama5d2-sfc";
    reg = <0xf804c000 0x64>;
    read-only;
    #address-cells = <1>;
    #size-cells = <1>;

    // 定义 NVMEM 单元
    sfc_dr0: sfc_dr0@20 {
        reg = <0x20 0x20>;      // 偏移 0x20, 大小 0x20 字节
    };

    sfc_dr1: sfc_dr1@24 {
        reg = <0x24 0x20>;      // 偏移 0x24, 大小 0x20 字节
    };

    // 硬件唯一密钥单元
    hw_unique_key: hw_unique_key@40 {
        reg = <0x40 0x20>;      // 32字节 HUK
    };
};
```

#### NVMEM Consumer 节点

```dts
// 设备 ID 消费者
die_id: die_id {
    compatible = "optee,nvmem-die-id";
    nvmem-cells = <&sfc_dr0>;
    nvmem-cell-names = "die_id";
};

// HUK 消费者
hw_unique_key_node: hw_unique_key {
    compatible = "optee,nvmem-huk";
    nvmem-cells = <&hw_unique_key>;
    nvmem-cell-names = "hw_unique_key";
};

// 多单元消费者示例
secure_keys: secure_keys {
    compatible = "optee,nvmem-secure-keys";
    nvmem-cells = <&hw_unique_key>, <&sfc_dr1>;
    nvmem-cell-names = "huk", "device_cert";
};
```

### 设备树解析实现

```c
static TEE_Result nvmem_cell_parse_dt(const void *fdt, int node, 
                                     const char *cell_name,
                                     struct nvmem_cell **out_cell)
{
    struct nvmem_cell *cell = NULL;
    const fdt32_t *prop = NULL;
    int prop_len = 0;
    int phandle = 0;
    int provider_node = 0;
    
    // 获取 nvmem-cells 属性
    prop = fdt_getprop(fdt, node, "nvmem-cells", &prop_len);
    if (!prop) {
        DMSG("No nvmem-cells property found");
        return TEE_ERROR_ITEM_NOT_FOUND;
    }
    
    // 解析 phandle 引用
    for (int i = 0; i < prop_len / sizeof(fdt32_t); i++) {
        phandle = fdt32_to_cpu(prop[i]);
        provider_node = fdt_node_offset_by_phandle(fdt, phandle);
        
        if (provider_node < 0) {
            EMSG("Invalid phandle: 0x%x", phandle);
            continue;
        }
        
        // 查找对应的 NVMEM provider
        return nvmem_get_provider_cell(fdt, provider_node, cell_name, out_cell);
    }
    
    return TEE_ERROR_ITEM_NOT_FOUND;
}
```

## Hardware Unique Key (HUK) Management

### 通用 NVMEM HUK 实现

**文件**: `core/drivers/nvmem/nvmem_huk.c`

```c
static uint8_t *huk = NULL;  // 全局 HUK 存储

TEE_Result tee_otp_get_hw_unique_key(struct tee_hw_unique_key *hwkey)
{
    if (!huk) {
        EMSG("Hardware Unique Key not available");
        return TEE_ERROR_GENERIC;
    }
    
    // 复制 HUK 数据
    memcpy(hwkey->data, huk, HW_UNIQUE_KEY_LENGTH);
    return TEE_SUCCESS;
}

static TEE_Result nvmem_huk_probe(const void *fdt, int node,
                                  const void *compat_data __unused)
{
    struct nvmem_cell *cell = NULL;
    uint8_t *data = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 从设备树获取 HUK 单元
    res = nvmem_get_cell_by_name(fdt, node, "hw_unique_key", &cell);
    if (res != TEE_SUCCESS) {
        EMSG("Failed to get HUK cell from device tree");
        return res;
    }
    
    // 验证单元大小
    if (cell->len < HW_UNIQUE_KEY_LENGTH) {
        EMSG("HUK cell too small: %zu < %u", cell->len, HW_UNIQUE_KEY_LENGTH);
        nvmem_put_cell(cell);
        return TEE_ERROR_GENERIC;
    }
    
    // 分配内存并读取 HUK
    res = nvmem_cell_malloc_and_read(cell, &data);
    if (res == TEE_SUCCESS) {
        huk = data;  // 设置全局 HUK
        DMSG("Hardware Unique Key loaded successfully");
    } else {
        EMSG("Failed to read HUK from NVMEM: 0x%x", res);
    }
    
    nvmem_put_cell(cell);
    return res;
}

NVMEM_PROVIDER("optee,nvmem-huk", nvmem_huk_probe, NULL);
```

### HUK 子密钥派生框架

**文件**: `core/kernel/huk_subkey.c`

```c
// HUK 子密钥用途枚举
enum huk_subkey_usage {
    HUK_SUBKEY_RPMB = 0,        // RPMB 密钥
    HUK_SUBKEY_SSK = 1,         // 安全存储密钥
    HUK_SUBKEY_DIE_ID = 2,      // 设备 ID 表示
    HUK_SUBKEY_UNIQUE_TA = 3,   // TA 唯一密钥
    HUK_SUBKEY_TA_ENC = 4,      // TA 加密密钥
    HUK_SUBKEY_SE050 = 5,       // SCP03 加密密钥
    HUK_SUBKEY_MAX
};

TEE_Result __huk_subkey_derive(enum huk_subkey_usage usage,
                               const void *const_data, size_t const_data_len,
                               uint8_t *subkey, size_t subkey_len)
{
    void *ctx = NULL;
    struct tee_hw_unique_key huk = { };
    TEE_Result res = TEE_SUCCESS;
    
    // 分配 HMAC 上下文
    res = crypto_mac_alloc_ctx(&ctx, TEE_ALG_HMAC_SHA256);
    if (res != TEE_SUCCESS)
        return res;
    
    // 获取硬件唯一密钥
    res = tee_otp_get_hw_unique_key(&huk);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 初始化 HMAC 使用 HUK 作为密钥
    res = crypto_mac_init(ctx, huk.data, sizeof(huk.data));
    if (res != TEE_SUCCESS)
        goto out;
    
    // 添加用途标识符
    res = mac_usage(ctx, usage);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 添加常量数据（如果提供）
    if (const_data && const_data_len > 0) {
        res = crypto_mac_update(ctx, const_data, const_data_len);
        if (res != TEE_SUCCESS)
            goto out;
    }
    
    // 生成子密钥
    res = crypto_mac_final(ctx, subkey, subkey_len);
    
out:
    crypto_mac_free_ctx(ctx);
    memzero_explicit(&huk, sizeof(huk));
    return res;
}

// 便利宏定义
#define huk_subkey_derive(usage, const_data, const_data_len, subkey, subkey_len) \
    __huk_subkey_derive((usage), (const_data), (const_data_len), \
                       (subkey), (subkey_len))
```

#### 用途特定的子密钥派生

```c
// RPMB 密钥派生
static TEE_Result derive_rpmb_key(uint8_t *key, size_t key_len)
{
    return huk_subkey_derive(HUK_SUBKEY_RPMB, NULL, 0, key, key_len);
}

// 安全存储密钥派生
static TEE_Result derive_ssk(uint8_t *ssk, size_t ssk_len) 
{
    const uint8_t label[] = "SECURE_STORAGE_KEY";
    return huk_subkey_derive(HUK_SUBKEY_SSK, label, sizeof(label) - 1, 
                           ssk, ssk_len);
}

// TA 特定密钥派生
TEE_Result derive_ta_unique_key(const TEE_UUID *ta_uuid, 
                               uint8_t *key, size_t key_len)
{
    return huk_subkey_derive(HUK_SUBKEY_UNIQUE_TA, ta_uuid, sizeof(*ta_uuid),
                           key, key_len);
}
```

### 平台特定 HUK 实现

#### STM32MP15 HUK 实现

**文件**: `core/drivers/stm32mp15_huk.c`

```c
#define HUK_NB_OTP (HW_UNIQUE_KEY_LENGTH / sizeof(uint32_t))

TEE_Result tee_otp_get_hw_unique_key(struct tee_hw_unique_key *hwkey)
{
    uint32_t otp_key[HUK_NB_OTP] = { };
    uint32_t otp_id[HUK_NB_OTP] = { };
    TEE_Result ret = TEE_ERROR_GENERIC;
    size_t len = HW_UNIQUE_KEY_LENGTH;
    bool locked = false;
    
    // 根据配置选择 HUK 获取方式
    if (IS_ENABLED(CFG_STM32MP15_HUK_BSEC_KEY)) {
        // 直接从 BSEC 密钥区域读取
        for (size_t i = 0; i < HUK_NB_OTP; i++) {
            ret = stm32_bsec_shadow_read_otp(&otp_key[i], HUK_BSEC_KEY_0 + i);
            if (ret != TEE_SUCCESS) {
                EMSG("Failed to read BSEC key OTP %zu", i);
                return ret;
            }
        }
        
        // 检查密钥锁定状态
        ret = stm32_bsec_read_permanent_lock(HUK_BSEC_KEY_0, &locked);
        if (ret == TEE_SUCCESS && !locked) {
            IMSG("HUK not permanently locked - security warning");
        }
        
        memcpy(hwkey->data, otp_key, HW_UNIQUE_KEY_LENGTH);
        
    } else if (IS_ENABLED(CFG_STM32MP15_HUK_BSEC_DERIVE_UID)) {
        // 从设备 UID 派生 HUK
        for (size_t i = 0; i < HUK_NB_OTP; i++) {
            ret = stm32_bsec_shadow_read_otp(&otp_id[i], UID_WORD_NB_0 + i);
            if (ret != TEE_SUCCESS) {
                EMSG("Failed to read UID OTP %zu", i);
                return ret;
            }
        }
        
        // 使用 AES-GCM 从 UID 派生 HUK
        ret = aes_gcm_encrypt_uid((uint8_t *)otp_key, sizeof(otp_key),
                                hwkey->data, &len);
        if (ret != TEE_SUCCESS) {
            EMSG("Failed to derive HUK from UID");
            return ret;
        }
    } else {
        EMSG("No HUK source configured");
        return TEE_ERROR_NOT_SUPPORTED;
    }
    
    return TEE_SUCCESS;
}

// AES-GCM UID 派生实现
static TEE_Result aes_gcm_encrypt_uid(uint8_t *key, size_t key_len,
                                      uint8_t *out, size_t *out_len)
{
    const uint8_t nonce[12] = { 0x55 };  // 常量随机数
    uint32_t uid[4] = { 0 };
    uint8_t tag[TEE_AES_BLOCK_SIZE] = { 0 };
    size_t tag_len = sizeof(tag);
    void *ctx = NULL;
    TEE_Result ret = TEE_SUCCESS;
    
    // 读取设备唯一标识符
    ret = stm32mp15_read_uid(uid);
    if (ret != TEE_SUCCESS)
        return ret;
    
    // 创建 AES-GCM 上下文
    ret = crypto_authenc_alloc_ctx(&ctx, TEE_ALG_AES_GCM);
    if (ret != TEE_SUCCESS)
        return ret;
    
    // 初始化认证加密
    ret = crypto_authenc_init(ctx, TEE_MODE_ENCRYPT, key, key_len,
                            nonce, sizeof(nonce), TEE_AES_BLOCK_SIZE, 0, sizeof(uid));
    if (ret != TEE_SUCCESS)
        goto out;
    
    // 执行加密并生成认证标签
    ret = crypto_authenc_enc_final(ctx, (uint8_t *)uid, sizeof(uid),
                                 out, out_len, tag, &tag_len);
    
out:
    crypto_authenc_free_ctx(ctx);
    return ret;
}
```

#### Xilinx ZynqMP HUK 实现

**文件**: `core/drivers/zynqmp_huk.c`

```c
#define ZYNQMP_HUK_EFUSE_COUNT  8

// 设备 DNA 寄存器偏移
#define ZYNQMP_EFUSE_DNA_0_OFFSET   0x100C
#define ZYNQMP_EFUSE_DNA_1_OFFSET   0x1010  
#define ZYNQMP_EFUSE_DNA_2_OFFSET   0x1014

// 用户 eFuse 偏移
#define USER_0                      0x1020
#define USER_1                      0x1024
#define USER_2                      0x1028
#define USER_3                      0x102C
#define USER_4                      0x1030
#define USER_5                      0x1034
#define USER_6                      0x1038
#define USER_7                      0x103C

static TEE_Result tee_zynqmp_generate_huk_src(const uint8_t *device_dna,
                                              size_t device_dna_size,
                                              uint8_t *huk_source,
                                              size_t huk_source_size)
{
    uint32_t user_efuse = 0;
    void *ctx = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 创建 SHA-256 哈希上下文
    res = crypto_hash_alloc_ctx(&ctx, TEE_ALG_SHA256);
    if (res != TEE_SUCCESS)
        return res;
    
    // 添加设备 DNA 作为基础熵源
    res = crypto_hash_update(ctx, device_dna, device_dna_size);
    if (res != TEE_SUCCESS)
        goto out;
    
    // 添加用户 eFuse 作为额外熵源
    for (size_t i = 0; i < ZYNQMP_HUK_EFUSE_COUNT; i++) {
        res = zynqmp_efuse_read((uint8_t *)&user_efuse, sizeof(user_efuse),
                              USER_0 + (i * 4), false);
        if (res != TEE_SUCCESS) {
            EMSG("Failed to read user eFuse %zu", i);
            goto out;
        }
        
        res = crypto_hash_update(ctx, (uint8_t *)&user_efuse, sizeof(user_efuse));
        if (res != TEE_SUCCESS)
            goto out;
    }
    
    // 生成最终的 HUK 源
    res = crypto_hash_final(ctx, huk_source, huk_source_size);
    
out:
    crypto_hash_free_ctx(ctx);
    return res;
}

TEE_Result tee_otp_get_hw_unique_key(struct tee_hw_unique_key *hwkey)
{
    uint8_t device_dna[16] = { 0 };     // 128位设备DNA
    uint8_t huk_source[32] = { 0 };     // 256位HUK源
    TEE_Result res = TEE_SUCCESS;
    
    // 读取设备 DNA (Device DNA)
    res = zynqmp_efuse_read(&device_dna[0], 4, ZYNQMP_EFUSE_DNA_0_OFFSET, false);
    if (res != TEE_SUCCESS)
        return res;
        
    res = zynqmp_efuse_read(&device_dna[4], 4, ZYNQMP_EFUSE_DNA_1_OFFSET, false);
    if (res != TEE_SUCCESS)
        return res;
        
    res = zynqmp_efuse_read(&device_dna[8], 4, ZYNQMP_EFUSE_DNA_2_OFFSET, false);
    if (res != TEE_SUCCESS)
        return res;
    
    // 生成 HUK 源
    res = tee_zynqmp_generate_huk_src(device_dna, sizeof(device_dna),
                                     huk_source, sizeof(huk_source));
    if (res != TEE_SUCCESS)
        return res;
    
    // 截断到 HUK 长度
    memcpy(hwkey->data, huk_source, HW_UNIQUE_KEY_LENGTH);
    
    return TEE_SUCCESS;
}
```

## Die ID and Manufacturing Data Access

### 通用 Die ID 实现

**文件**: `core/drivers/nvmem/nvmem_die_id.c`

```c
static uint8_t *die_id = NULL;
static size_t die_id_len = 0;

int tee_otp_get_die_id(uint8_t *buffer, size_t len)
{
    if (!buffer || len == 0)
        return -1;
    
    if (!die_id) {
        // 回退到基于 HUK 的 die ID
        if (huk_subkey_derive(HUK_SUBKEY_DIE_ID, NULL, 0, buffer, len) != TEE_SUCCESS)
            return -1;
        return 0;
    }
    
    // 复制实际的 die ID 数据
    memcpy(buffer, die_id, MIN(die_id_len, len));
    return 0;
}

static TEE_Result nvmem_die_id_probe(const void *fdt, int node,
                                     const void *compat_data __unused)
{
    struct nvmem_cell *cell = NULL;
    uint8_t *data = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 从设备树获取 die_id 单元
    res = nvmem_get_cell_by_name(fdt, node, "die_id", &cell);
    if (res != TEE_SUCCESS) {
        DMSG("No die_id cell found in device tree");
        return res;
    }
    
    // 读取 die ID 数据
    res = nvmem_cell_malloc_and_read(cell, &data);
    if (res == TEE_SUCCESS) {
        die_id = data;
        die_id_len = cell->len;
        DMSG("Die ID loaded: %zu bytes", die_id_len);
    } else {
        EMSG("Failed to read die ID: 0x%x", res);
    }
    
    nvmem_put_cell(cell);
    return res;
}

NVMEM_PROVIDER("optee,nvmem-die-id", nvmem_die_id_probe, NULL);
```

### 制造数据访问模式

#### 芯片版本信息

```c
struct chip_revision_info {
    uint32_t chip_id;                  // 芯片标识符
    uint32_t revision;                 // 修订版本号
    uint32_t package_info;             // 封装信息
    uint8_t manufacturing_date[8];     // 制造日期
};

static TEE_Result read_manufacturing_data(struct chip_revision_info *info)
{
    struct nvmem_cell *cell = NULL;
    uint8_t *data = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 读取芯片标识符
    res = nvmem_get_cell_by_name(fdt, node, "chip_id", &cell);
    if (res == TEE_SUCCESS) {
        res = nvmem_cell_read(cell, (uint8_t *)&info->chip_id);
        nvmem_put_cell(cell);
    }
    
    // 读取修订版本
    res = nvmem_get_cell_by_name(fdt, node, "revision", &cell);
    if (res == TEE_SUCCESS) {
        res = nvmem_cell_read(cell, (uint8_t *)&info->revision);
        nvmem_put_cell(cell);
    }
    
    return res;
}
```

#### 校准数据访问

```c
struct calibration_data {
    uint32_t temperature_offset;       // 温度校准偏移
    uint32_t voltage_reference;        // 电压参考值
    uint32_t adc_gain_correction;      // ADC 增益校正
    uint32_t oscillator_trim;          // 振荡器调整值
};

static TEE_Result load_calibration_data(struct calibration_data *cal_data)
{
    const char *cal_cell_names[] = {
        "temp_offset", "vref", "adc_gain", "osc_trim"
    };
    uint32_t *cal_values[] = {
        &cal_data->temperature_offset,
        &cal_data->voltage_reference,
        &cal_data->adc_gain_correction,
        &cal_data->oscillator_trim
    };
    
    for (size_t i = 0; i < ARRAY_SIZE(cal_cell_names); i++) {
        struct nvmem_cell *cell = NULL;
        TEE_Result res = nvmem_get_cell_by_name(fdt, node, cal_cell_names[i], &cell);
        
        if (res == TEE_SUCCESS) {
            res = nvmem_cell_read(cell, (uint8_t *)cal_values[i]);
            nvmem_put_cell(cell);
            
            if (res != TEE_SUCCESS) {
                EMSG("Failed to read calibration data: %s", cal_cell_names[i]);
                return res;
            }
        }
    }
    
    return TEE_SUCCESS;
}
```

## Security Mechanisms and Access Control

### 硬件安全特性

#### 1. OTP 锁定机制

```c
enum otp_lock_type {
    OTP_LOCK_NONE = 0,              // 无锁定
    OTP_LOCK_WRITE = 1,             // 写锁定
    OTP_LOCK_READ_WRITE = 2,        // 读写锁定
    OTP_LOCK_PERMANENT = 3,         // 永久锁定
};

static TEE_Result check_otp_lock_status(uint32_t otp_id, enum otp_lock_type *lock_type)
{
    bool write_locked = false, read_locked = false, permanent_locked = false;
    TEE_Result res = TEE_SUCCESS;
    
    // 检查写锁定状态
    res = stm32_bsec_read_write_lock(otp_id, &write_locked);
    if (res != TEE_SUCCESS)
        return res;
    
    // 检查读锁定状态
    res = stm32_bsec_read_read_lock(otp_id, &read_locked);
    if (res != TEE_SUCCESS)
        return res;
    
    // 检查永久锁定状态
    res = stm32_bsec_read_permanent_lock(otp_id, &permanent_locked);
    if (res != TEE_SUCCESS)
        return res;
    
    // 确定锁定类型
    if (permanent_locked) {
        *lock_type = OTP_LOCK_PERMANENT;
    } else if (read_locked && write_locked) {
        *lock_type = OTP_LOCK_READ_WRITE;
    } else if (write_locked) {
        *lock_type = OTP_LOCK_WRITE;
    } else {
        *lock_type = OTP_LOCK_NONE;
    }
    
    return TEE_SUCCESS;
}
```

#### 2. 安全世界访问控制

```c
static TEE_Result verify_secure_access(const void *fdt, int node)
{
    int status = fdt_get_status(fdt, node);
    
    // 检查设备树中的安全状态
    if (status != DT_STATUS_OK_SEC) {
        EMSG("NVMEM device not marked as secure");
        return TEE_ERROR_ACCESS_DENIED;
    }
    
    // 验证安全世界访问权限
    if (!dt_have_prop(fdt, node, "secure-status")) {
        DMSG("No secure-status property, assuming secure access");
    }
    
    return TEE_SUCCESS;
}
```

#### 3. 物理安全保护

```c
static TEE_Result configure_physical_security(vaddr_t base_addr)
{
    // 启用防篡改检测
    io_setbits32(base_addr + SECURITY_CONFIG_REG, TAMPER_DETECT_EN);
    
    // 配置访问监控
    io_setbits32(base_addr + ACCESS_MONITOR_REG, UNAUTHORIZED_ACCESS_IRQ_EN);
    
    // 设置时钟故障检测
    io_setbits32(base_addr + CLOCK_MONITOR_REG, CLOCK_FAIL_DETECT_EN);
    
    return TEE_SUCCESS;
}
```

### 软件安全措施

#### 1. 数据完整性验证

```c
static TEE_Result verify_nvmem_integrity(const uint8_t *data, size_t len,
                                        const uint8_t *expected_hash)
{
    uint8_t computed_hash[TEE_SHA256_HASH_SIZE] = { 0 };
    void *ctx = NULL;
    TEE_Result res = TEE_SUCCESS;
    
    // 计算数据哈希值
    res = crypto_hash_alloc_ctx(&ctx, TEE_ALG_SHA256);
    if (res != TEE_SUCCESS)
        return res;
    
    res = crypto_hash_update(ctx, data, len);
    if (res != TEE_SUCCESS)
        goto out;
    
    res = crypto_hash_final(ctx, computed_hash, sizeof(computed_hash));
    if (res != TEE_SUCCESS)
        goto out;
    
    // 常量时间比较
    if (crypto_memneq(computed_hash, expected_hash, sizeof(computed_hash)) != 0) {
        EMSG("NVMEM data integrity verification failed");
        res = TEE_ERROR_CORRUPT_OBJECT;
    }
    
out:
    crypto_hash_free_ctx(ctx);
    return res;
}
```

#### 2. 访问日志和审计

```c
struct nvmem_access_log {
    uint64_t timestamp;                // 访问时间戳
    uint32_t cell_offset;             // 单元偏移
    size_t access_len;                // 访问长度
    enum nvmem_operation {            // 操作类型
        NVMEM_OP_READ = 1,
        NVMEM_OP_WRITE = 2,
        NVMEM_OP_LOCK = 3
    } operation;
    TEE_Result result;                // 操作结果
};

static void log_nvmem_access(struct nvmem_cell *cell, 
                            enum nvmem_operation op,
                            TEE_Result result)
{
    struct nvmem_access_log log_entry = {
        .timestamp = get_time_stamp(),
        .cell_offset = cell->offset,
        .access_len = cell->len,
        .operation = op,
        .result = result
    };
    
    // 写入安全审计日志
    secure_audit_log(&log_entry, sizeof(log_entry));
}
```

## Configuration and Build System

### 核心配置选项

```makefile
# NVMEM 框架配置
CFG_DRIVERS_NVMEM ?= y                    # 启用 NVMEM 框架
CFG_NVMEM_DIE_ID ?= n                     # 启用 NVMEM Die ID 支持
CFG_NVMEM_HUK ?= n                        # 启用 NVMEM HUK 支持

# 平台特定驱动
CFG_ATMEL_SFC ?= n                        # Atmel SFC 驱动
CFG_STM32_BSEC ?= n                       # STM32 BSEC 驱动
CFG_IMX_OCOTP ?= n                        # iMX OCOTP 驱动

# HUK 配置选项
CFG_STM32MP15_HUK ?= n                    # STM32MP15 HUK 支持
CFG_STM32_HUK_FROM_DT ?= n                # 从设备树获取 HUK 配置
CFG_STM32MP15_HUK_BSEC_KEY ?= y           # 直接使用 BSEC 密钥
CFG_STM32MP15_HUK_BSEC_DERIVE_UID ?= n    # 从 UID 派生 HUK

# 安全选项
CFG_OTP_SUPPORT ?= y                      # OTP 支持
CFG_HUK_SUBKEY_DERIVE ?= y                # HUK 子密钥派生
```

### 构建系统集成

**文件**: `core/drivers/nvmem/sub.mk`

```makefile
# NVMEM 框架核心文件
srcs-y += nvmem.c

# 可选的 NVMEM 服务
srcs-$(CFG_NVMEM_DIE_ID) += nvmem_die_id.c
srcs-$(CFG_NVMEM_HUK) += nvmem_huk.c

# 平台特定驱动
srcs-$(CFG_ATMEL_SFC) += atmel_sfc.c

# 条件编译示例
ifeq ($(CFG_STM32_BSEC),y)
$(call force,CFG_DRIVERS_NVMEM,y)
srcs-y += ../stm32_bsec.c
endif

# 依赖检查
ifeq ($(CFG_NVMEM_HUK),y)
$(call force,CFG_HUK_SUBKEY_DERIVE,y)
endif
```

## Performance Optimization and Best Practices

### 1. 缓存优化

```c
#define NVMEM_CACHE_SIZE 16

struct nvmem_cache_entry {
    uint32_t cell_offset;              // 单元偏移
    size_t len;                        // 数据长度
    uint8_t *data;                     // 缓存数据
    uint64_t access_time;              // 访问时间
    bool valid;                        // 有效标志
};

static struct {
    struct nvmem_cache_entry entries[NVMEM_CACHE_SIZE];
    struct mutex cache_mutex;
    uint64_t lru_counter;
} nvmem_cache = {
    .cache_mutex = MUTEX_INITIALIZER,
    .lru_counter = 0
};

static TEE_Result nvmem_cache_read(uint32_t offset, size_t len, uint8_t *data)
{
    struct nvmem_cache_entry *entry = NULL;
    
    mutex_lock(&nvmem_cache.cache_mutex);
    
    // 查找缓存条目
    for (size_t i = 0; i < NVMEM_CACHE_SIZE; i++) {
        entry = &nvmem_cache.entries[i];
        if (entry->valid && entry->cell_offset == offset && entry->len == len) {
            memcpy(data, entry->data, len);
            entry->access_time = ++nvmem_cache.lru_counter;
            mutex_unlock(&nvmem_cache.cache_mutex);
            return TEE_SUCCESS;
        }
    }
    
    mutex_unlock(&nvmem_cache.cache_mutex);
    return TEE_ERROR_ITEM_NOT_FOUND;
}
```

### 2. 批量访问优化

```c
struct nvmem_batch_request {
    uint32_t offset;                   // 访问偏移
    size_t len;                        // 访问长度
    uint8_t *buffer;                   // 数据缓冲区
};

static TEE_Result nvmem_batch_read(struct nvmem_cell *cell,
                                  struct nvmem_batch_request *requests,
                                  size_t num_requests)
{
    TEE_Result res = TEE_SUCCESS;
    
    // 排序请求以优化访问模式
    qsort(requests, num_requests, sizeof(*requests), compare_offset);
    
    // 合并连续的访问请求
    for (size_t i = 0; i < num_requests; ) {
        size_t merge_count = 1;
        size_t total_len = requests[i].len;
        
        // 查找连续的请求
        while (i + merge_count < num_requests &&
               requests[i + merge_count - 1].offset + requests[i + merge_count - 1].len ==
               requests[i + merge_count].offset) {
            total_len += requests[i + merge_count].len;
            merge_count++;
        }
        
        // 执行合并的读取
        if (merge_count > 1) {
            res = nvmem_batch_read_merged(&requests[i], merge_count, total_len);
        } else {
            res = nvmem_cell_read(cell, requests[i].buffer);
        }
        
        if (res != TEE_SUCCESS)
            return res;
            
        i += merge_count;
    }
    
    return TEE_SUCCESS;
}
```

### 3. 错误处理和重试机制

```c
#define NVMEM_MAX_RETRIES 3
#define NVMEM_RETRY_DELAY_MS 10

static TEE_Result nvmem_read_with_retry(struct nvmem_cell *cell, uint8_t *data)
{
    TEE_Result res = TEE_ERROR_GENERIC;
    
    for (int retry = 0; retry < NVMEM_MAX_RETRIES; retry++) {
        res = nvmem_cell_read(cell, data);
        
        if (res == TEE_SUCCESS)
            break;
            
        if (res == TEE_ERROR_BUSY) {
            // 等待后重试
            mdelay(NVMEM_RETRY_DELAY_MS * (retry + 1));
            continue;
        }
        
        // 非临时错误，直接返回
        if (res != TEE_ERROR_COMMUNICATION) {
            EMSG("NVMEM read failed with non-recoverable error: 0x%x", res);
            break;
        }
        
        DMSG("NVMEM read retry %d/%d", retry + 1, NVMEM_MAX_RETRIES);
    }
    
    return res;
}
```

## Summary

OP-TEE 的 NVMEM 框架提供了一个强大、安全且灵活的解决方案，用于访问关键的硬件安全数据。该框架的主要特点包括：

### 核心优势

1. **统一抽象**: 为多种硬件 OTP/eFuse 实现提供一致的 API
2. **设备树集成**: 标准的 Linux 设备树绑定，便于硬件描述
3. **安全设计**: 多层次的安全机制和访问控制
4. **平台兼容**: 支持主流 ARM SoC 平台的 NVMEM 实现
5. **密钥管理**: 完整的 HUK 管理和子密钥派生框架

### 安全特性

1. **硬件保护**: OTP 锁定机制和物理安全功能
2. **访问控制**: 安全世界访问验证和权限管理
3. **完整性验证**: 数据完整性检查和审计日志
4. **密钥隔离**: HUK 子密钥派生确保密钥隔离

### 适用场景

- 硬件唯一密钥管理
- 设备身份认证和证明
- 安全启动和固件验证
- 制造数据和校准参数访问
- 平台特定的安全配置

NVMEM 框架是 OP-TEE 安全架构的重要组成部分，为基于硬件的信任根提供了可靠的基础设施支持。