# OP-TEE Storage Migration and Versioning - Analysis

## Overview

This document analyzes OP-TEE's storage migration and versioning mechanisms, covering format versioning, backward compatibility, schema evolution, and migration strategies between different storage backends and versions.

## Storage Format Versioning

### RPMB File System Versioning
- **Location**: `/home/dzb/optee/optee_os/core/tee/tee_rpmb_fs.c`
- **Current Version**: FS_VERSION = 2
- **Magic Number**: 0x52504D42 ("RPMB" in ASCII)

```c
#define RPMB_FS_MAGIC     0x52504D42
#define FS_VERSION        2

struct rpmb_fs_partition {
    uint32_t rpmb_fs_magic;        // Magic: 0x52504D42
    uint32_t fs_version;           // Current: 2
    uint32_t write_counter;        // Anti-rollback counter
    uint32_t fat_start_address;    // File allocation table
    uint8_t reserved[112];         // Future expansion
};
```

### Hash Tree Compatibility
The hash tree implementation supports format evolution through configuration:

```c
// Backward compatibility for hash sizes
if (IS_ENABLED(CFG_REE_FS_HTREE_HASH_SIZE_COMPAT)) {
    // Use old 16-byte hash size for compatibility
    aad_len += TEE_FS_HTREE_FEK_SIZE;
} else {
    // Use full 32-byte SHA256 hash
    aad_len += TEE_FS_HTREE_HASH_SIZE;
}
```

### HUK Subkey Compatibility Layer
Hardware Unique Key derivation includes compatibility mode for legacy systems:

```c
#ifdef CFG_CORE_HUK_SUBKEY_COMPAT
/*
 * Special treatment for RPMB and SSK key derivations to give
 * the same result as when huk_subkey_derive() wasn't used.
 */
static TEE_Result huk_compat(void *ctx, enum huk_subkey_usage usage)
{
    uint8_t chip_id[TEE_FS_KM_CHIP_ID_LENGTH] = { 0 };
    static uint8_t ssk_str[] = "ONLY_FOR_tee_fs_ssk";
    
    switch (usage) {
    case HUK_SUBKEY_RPMB:
        return TEE_SUCCESS;  // No additional data for RPMB
    case HUK_SUBKEY_SSK:
        // Legacy SSK derivation includes chip ID and salt
        res = get_otp_die_id(chip_id, sizeof(chip_id));
        if (res) return res;
        res = crypto_mac_update(ctx, chip_id, sizeof(chip_id));
        if (res) return res;
        return crypto_mac_update(ctx, ssk_str, sizeof(ssk_str));
    default:
        return mac_usage(ctx, usage);  // Standard derivation
    }
}
#endif
```

## Migration Between Storage Backends

### REE File System Migration
The REE file system supports migration from non-integrity-protected to integrity-protected mode:

```c
static TEE_Result open_dirh(struct tee_fs_dirfile_dirh **dirh)
{
    uint32_t min_counter = 0;
    
    // Attempt to get current counter value
    res = nv_counter_get_ree_fs(&min_counter);
    if (res) {
        if (res != TEE_ERROR_NOT_IMPLEMENTED || !IS_ENABLED(CFG_INSECURE))
            return res;
        
        IMSG("WARNING (insecure configuration): Using counter 0");
        min_counter = 0;
    }
    
    // Try to open existing directory file
    res = tee_fs_dirfile_open(false, NULL, min_counter, &ree_dirf_ops, dirh);
    if (res == TEE_ERROR_ITEM_NOT_FOUND) {
        if (min_counter) {
            if (!IS_ENABLED(CFG_REE_FS_ALLOW_RESET)) {
                DMSG("dirf.db file not found");
                return TEE_ERROR_SECURITY;  // Prevent rollback
            }
            DMSG("dirf.db not found, initializing with non-zero counter");
        }
        // Create new directory file with current counter
        return tee_fs_dirfile_open(true, NULL, min_counter, &ree_dirf_ops, dirh);
    }
    
    return res;
}
```

### RPMB to REE Migration Strategy
While not directly implemented, the architecture supports migration through:

1. **Export Phase**: Read all objects from source storage
2. **Key Preservation**: Maintain FEK encryption for seamless transition
3. **Import Phase**: Write objects to destination storage format
4. **Validation**: Verify integrity of migrated data

## Backward Compatibility Mechanisms

### Configuration-Based Compatibility
Multiple compile-time flags control compatibility behavior:

```c
// Configuration options for compatibility
CFG_REE_FS_HTREE_HASH_SIZE_COMPAT    // Hash size compatibility
CFG_CORE_HUK_SUBKEY_COMPAT           // HUK derivation compatibility
CFG_REE_FS_ALLOW_RESET               // Allow counter reset
CFG_INSECURE                         // Debug/insecure mode
```

### Directory File Format Evolution
Directory entries use reserved space for future expansion:

```c
struct dirfile_entry {
    TEE_UUID uuid;                    // 16 bytes
    uint8_t oid[TEE_OBJECT_ID_MAX_LEN];  // 64 bytes
    uint32_t oidlen;                  // 4 bytes
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];  // 32 bytes
    uint32_t file_number;             // 4 bytes
    // Total: 120 bytes, room for expansion
};
```

### Object ID Compatibility
Support for zero-length object IDs with special encoding:

```c
#define OID_EMPTY_NAME 1

/*
 * An object can have an ID of size zero. This object is represented by
 * oidlen == 0 and oid[0] == OID_EMPTY_NAME. When both are zero, the entry is
 * not a valid object.
 */
static bool is_free(struct dirfile_entry *dent)
{
    assert(dent->oidlen || !dent->oid[0] || dent->oid[0] == OID_EMPTY_NAME);
    return !dent->oidlen && !dent->oid[0];
}
```

## Schema Evolution Strategies

### Counter-Based Evolution
Monotonic counters provide migration checkpoints:

```c
// Counter validation with minimum requirements
if (ht->head.counter < min_counter)
    return TEE_ERROR_SECURITY;  // Prevent rollback to old format
```

### Versioned Storage Operations
Each storage backend can implement version-specific behavior:

```c
struct tee_fs_htree_storage {
    size_t block_size;
    // Function pointers for version-specific operations
    TEE_Result (*rpc_read_init)(void *aux, struct tee_fs_rpc_operation *op,
                               enum tee_fs_htree_type type, size_t idx,
                               uint8_t vers, void **data);
    TEE_Result (*rpc_write_init)(void *aux, struct tee_fs_rpc_operation *op,
                                enum tee_fs_htree_type type, size_t idx,
                                uint8_t vers, void **data);
    // ... additional operations
};
```

### Test Infrastructure for Migration
Dedicated test framework for storage format validation:

```c
// Location: /home/dzb/optee/optee_os/core/pta/tests/fs_htree.c
#define TEST_BLOCK_SIZE    144  // Minimum viable block size

// Test-specific storage layout
static TEE_Result test_get_offs_size(enum tee_fs_htree_type type, size_t idx,
                                    uint8_t vers, size_t *offs, size_t *size)
{
    // Implement test-specific layout for validation
    switch (type) {
    case TEE_FS_HTREE_TYPE_HEAD:
        *offs = sizeof(struct tee_fs_htree_image) * vers;
        *size = sizeof(struct tee_fs_htree_image);
        return TEE_SUCCESS;
    // ... handle other types
    }
}
```

## Version Detection and Negotiation

### Magic Number Validation
Storage formats use magic numbers for identification:

```c
// RPMB file system magic validation
if (partition->rpmb_fs_magic != RPMB_FS_MAGIC) {
    EMSG("Invalid RPMB FS magic: 0x%x", partition->rpmb_fs_magic);
    return TEE_ERROR_CORRUPT_OBJECT;
}

// Version compatibility check
if (partition->fs_version > FS_VERSION) {
    EMSG("Unsupported FS version: %d > %d", partition->fs_version, FS_VERSION);
    return TEE_ERROR_NOT_SUPPORTED;
}
```

### Feature Detection
Runtime detection of storage capabilities:

```c
// Detect RPMB device capabilities
struct rpmb_dev_info {
    uint8_t cid[16];           // Card identification
    uint8_t rpmb_size_mult;    // Size multiplier
    uint8_t rel_wr_sec_c;      // Reliable write sector count
    uint8_t ret_code;          // Response code
};
```

## Migration Safety Mechanisms

### Atomic Migration Operations
Migration operations use the same atomic commit mechanisms:

1. **Preparation**: Verify source and destination integrity
2. **Copy**: Transfer data while maintaining encryption
3. **Verification**: Validate copied data
4. **Commit**: Atomically switch to new format
5. **Cleanup**: Remove old format data

### Rollback Protection
Counter mechanisms prevent rollback to older formats:

```c
// Enforce minimum counter requirements
res = nv_counter_get_ree_fs(&min_counter);
if (res == TEE_SUCCESS) {
    if (current_counter < min_counter) {
        EMSG("Counter rollback detected: %d < %d", current_counter, min_counter);
        return TEE_ERROR_SECURITY;
    }
}
```

## Platform-Specific Migration

### Platform Detection
Storage behavior adapts to platform capabilities:

```c
// RPMB availability detection
static TEE_Result rpmb_probe_reset(void)
{
    res = thread_rpc_cmd(OPTEE_RPC_CMD_RPMB_PROBE_RESET, 1, params);
    if (res) return res;
    
    rpmb_ctx->legacy_operation = false;
    rpmb_ctx->dev_id = 0;
    
    switch (params[0].u.value.a) {
    case OPTEE_RPC_SHM_TYPE_APPL:
        rpmb_ctx->shm_type = THREAD_SHM_TYPE_APPLICATION;
        return TEE_SUCCESS;
    case OPTEE_RPC_SHM_TYPE_KERNEL:
        rpmb_ctx->shm_type = THREAD_SHM_TYPE_KERNEL_PRIVATE;
        return TEE_SUCCESS;
    default:
        return TEE_ERROR_GENERIC;
    }
}
```

### Legacy Operation Support
Support for older RPC protocols:

```c
// Legacy vs. new RPC command selection
uint32_t cmd = OPTEE_RPC_CMD_RPMB_FRAMES;
if (rpmb_ctx->legacy_operation)
    cmd = OPTEE_RPC_CMD_RPMB;  // Use legacy command

return thread_rpc_cmd(cmd, 2, params);
```

## Future Migration Considerations

### Reserved Space Utilization
Storage structures include reserved space for future features:

```c
struct rpmb_fs_partition {
    // ... existing fields
    uint8_t reserved[112];  // 112 bytes for future expansion
};
```

### Extensible Header Format
Hash tree headers can accommodate new metadata:

```c
struct tee_fs_htree_imeta {
    struct tee_fs_htree_meta meta;  // Current metadata
    uint32_t max_node_id;           // Implementation detail
    // Additional fields can be added here
};
```

### Version-Specific Operations
Framework supports version-specific storage operations through function pointers, enabling smooth transitions between storage format versions while maintaining data integrity and security properties.

This migration and versioning architecture ensures OP-TEE storage can evolve while maintaining backward compatibility and data integrity across different storage backends and format versions.