# OP-TEE Storage Monitoring and Diagnostics - Analysis

## Overview

This document analyzes OP-TEE's storage monitoring and diagnostic capabilities, covering debugging interfaces, performance metrics, error reporting, logging mechanisms, and health monitoring features built into the storage subsystem.

## Debugging and Trace Infrastructure

### Debug Message Framework
OP-TEE storage components use extensive debug messaging for monitoring operations:

```c
// Storage operation tracing in fs_htree.c
DMSG("%scrypt block #%u", (mode == TEE_MODE_ENCRYPT) ? "En" : "De", blk_idx);

// Directory operations in fs_dirfile.c
DMSG("clearing duplicate file number %" PRIu32, dent.file_number);

// REE file system status in tee_ree_fs.c
DMSG("dirf.db file not found");
DMSG("dirf.db not found, initializing with a non-zero monotonic counter");
```

### Trace Levels and Categories
Different trace levels provide granular debugging control:

```c
// Key manager operations with detailed tracing
#define TEE_FS_KM_HMAC_ALG        TEE_ALG_HMAC_SHA256
#define TEE_FS_KM_ENC_FEK_ALG     TEE_ALG_AES_ECB_NOPAD

// Trace block encryption/decryption operations
DMSG("%scrypt block #%u", (mode == TEE_MODE_ENCRYPT) ? "En" : "De", blk_idx);
```

### Configuration-Based Debugging
Debug features controlled by build-time configuration:

```c
// Insecure mode warnings for development
if (res == TEE_ERROR_NOT_IMPLEMENTED && IS_ENABLED(CFG_INSECURE)) {
    static bool once;
    if (!once) {
        IMSG("WARNING (insecure configuration): Failed to get monotonic counter for REE FS, using 0");
        once = true;
    }
    min_counter = 0;
}
```

## Performance Monitoring

### Block-Level Operation Tracking
Hash tree operations track block access patterns:

```c
// Block number calculation for performance analysis
static int pos_to_block_num(int position)
{
    return position >> BLOCK_SHIFT;  // BLOCK_SHIFT = 12 (4KB blocks)
}

// Out-of-place write performance monitoring
static TEE_Result out_of_place_write(struct tee_fs_fd *fdp, size_t pos,
                                     const void *buf_core, const void *buf_user,
                                     size_t len)
{
    size_t start_block_num = pos_to_block_num(pos);
    size_t end_block_num = pos_to_block_num(pos + len - 1);
    
    // Track block access range
    while (start_block_num <= end_block_num) {
        // ... process each block with timing potential
        start_block_num++;
    }
}
```

### Memory Pool Monitoring
Storage operations use memory pools with implicit monitoring:

```c
// Temporary block allocation tracking
static void *get_tmp_block(void)
{
    return mempool_alloc(mempool_default, BLOCK_SIZE);
}

static void put_tmp_block(void *tmp_block)
{
    mempool_free(mempool_default, tmp_block);
}
```

### Reference Counting Diagnostics
Directory handle reference counting provides operation tracking:

```c
static void put_dirh_primitive(bool close)
{
    assert(ree_fs_dirh_refcount);  // Diagnostic assertion
    
    ree_fs_dirh_refcount--;
    if (ree_fs_dirh && (!ree_fs_dirh_refcount || close))
        close_dirh(&ree_fs_dirh);
}
```

## Error Reporting and Logging

### Structured Error Codes
Storage operations return detailed error information:

```c
// RPMB-specific error codes for diagnostics
#define RPMB_RESULT_OK                      0x00
#define RPMB_RESULT_GENERAL_FAILURE         0x01
#define RPMB_RESULT_AUTH_FAILURE            0x02
#define RPMB_RESULT_COUNTER_FAILURE         0x03
#define RPMB_RESULT_ADDRESS_FAILURE         0x04
#define RPMB_RESULT_WRITE_FAILURE           0x05
#define RPMB_RESULT_READ_FAILURE            0x06
#define RPMB_RESULT_AUTH_KEY_NOT_PROGRAMMED 0x07
```

### Corruption Detection and Reporting
Multiple layers of corruption detection with detailed error reporting:

```c
// Hash tree integrity verification
static TEE_Result verify_node(struct traverse_arg *targ, struct htree_node *node)
{
    uint8_t digest[TEE_FS_HTREE_HASH_SIZE];
    
    res = calc_node_hash(node, NULL, ctx, digest);
    if (res == TEE_SUCCESS && 
        consttime_memcmp(digest, node->node.hash, sizeof(digest))) {
        EMSG("Hash tree node corruption detected for node %zu", node->id);
        return TEE_ERROR_CORRUPT_OBJECT;
    }
    return res;
}

// Authenticated encryption verification failures
if (res == TEE_ERROR_MAC_INVALID) {
    EMSG("MAC verification failed - potential tampering detected");
    return TEE_ERROR_CORRUPT_OBJECT;
}
```

### File System Consistency Warnings
Directory file system includes consistency checking:

```c
// Duplicate file number detection
if (test_file(dirh, dent.file_number)) {
    DMSG("clearing duplicate file number %" PRIu32, dent.file_number);
    memset(&dent, 0, sizeof(dent));
    res = write_dent(dirh, n, &dent);
    if (res) goto out;
    continue;
}
```

## Health Monitoring Features

### Counter Monitoring
Monotonic counter health tracking:

```c
// Counter increment validation
res = nv_counter_incr_ree_fs_to(counter);
if (res == TEE_ERROR_NOT_IMPLEMENTED && IS_ENABLED(CFG_INSECURE)) {
    static bool once;
    if (!once) {
        IMSG("WARNING (insecure configuration): Failed to commit dirh counter %"PRIu32, counter);
        once = true;
    }
    return TEE_SUCCESS;
}
```

### RPMB Device Health
RPMB device status and capability monitoring:

```c
// Device capability reporting
struct rpmb_dev_info {
    uint8_t cid[16];           // Card identification for tracking
    uint8_t rpmb_size_mult;    // Size for capacity monitoring  
    uint8_t rel_wr_sec_c;      // Reliability parameter
    uint8_t ret_code;          // Operation status
};

// Health check through device probing
static TEE_Result rpmb_probe_next(struct rpmb_dev_info *dev_info)
{
    // Probe device capabilities and report status
    if (params[0].u.value.a != OPTEE_RPC_RPMB_EMMC)
        return TEE_ERROR_NOT_SUPPORTED;
    
    *dev_info = (struct rpmb_dev_info){
        .rpmb_size_mult = params[0].u.value.b,
        .rel_wr_sec_c = params[0].u.value.c,
        .ret_code = RPMB_CMD_GET_DEV_INFO_RET_OK,
    };
    
    return TEE_SUCCESS;
}
```

## Test and Validation Infrastructure

### Storage Test Framework
Dedicated test infrastructure for storage validation:

```c
// Location: /home/dzb/optee/optee_os/core/pta/tests/fs_htree.c

// Test block size for validation
#define TEST_BLOCK_SIZE    144

// Test storage auxiliary structure
struct test_aux {
    uint8_t *data;           // Test data buffer
    size_t data_len;         // Current data length
    size_t data_alloced;     // Allocated buffer size
    uint8_t *block;          // Working block buffer
};

// Test storage operations with monitoring
static TEE_Result test_read_init(void *aux, struct tee_fs_rpc_operation *op,
                                enum tee_fs_htree_type type, size_t idx,
                                uint8_t vers, void **data)
{
    struct test_aux *a = aux;
    size_t offs = 0;
    size_t sz = 0;
    
    // Calculate test offsets and validate bounds
    res = test_get_offs_size(type, idx, vers, &offs, &sz);
    if (res != TEE_SUCCESS) return res;
    
    // Monitor test data access patterns
    if (offs + sz > a->data_len) {
        EMSG("Test read beyond data bounds: %zu + %zu > %zu", offs, sz, a->data_len);
        return TEE_ERROR_ITEM_NOT_FOUND;
    }
    
    *data = a->data + offs;
    return TEE_SUCCESS;
}
```

### Compile-Time Assertions
Storage structures include compile-time validation:

```c
// Validate test block size constraints
COMPILE_TIME_ASSERT(TEST_BLOCK_SIZE > sizeof(struct tee_fs_htree_node_image) * 2);
COMPILE_TIME_ASSERT(TEST_BLOCK_SIZE > sizeof(struct tee_fs_htree_image) * 2);

// Validate key size constraints
COMPILE_TIME_ASSERT(TEE_FS_KM_SSK_SIZE <= HUK_SUBKEY_MAX_LEN);
```

## Diagnostic Data Collection

### Storage Metadata Monitoring
Hash tree metadata provides diagnostic information:

```c
struct tee_fs_htree_meta {
    uint64_t length;    // File length for size monitoring
};

struct tee_fs_htree_imeta {
    struct tee_fs_htree_meta meta;
    uint32_t max_node_id;    // Tree size for complexity monitoring
};
```

### Operation Status Tracking
Storage operations maintain status information:

```c
struct htree_node {
    size_t id;               // Node identifier
    bool dirty;              // Modification status
    bool block_updated;      // Update tracking
    struct tee_fs_htree_node_image node;
    struct htree_node *parent;
    struct htree_node *child[2];
};
```

### Directory Statistics
Directory file operations provide statistical information:

```c
struct tee_fs_dirfile_dirh {
    const struct tee_fs_dirfile_operations *fops;
    struct tee_file_handle *fh;
    int nbits;               // Bitmap size monitoring
    bitstr_t *files;         // File allocation tracking
    size_t ndents;           // Directory entry count
};
```

## Security Event Monitoring

### Tampering Detection Events
Storage layer monitors for security violations:

```c
// Counter rollback detection
if (ht->head.counter < min_counter) {
    EMSG("SECURITY VIOLATION: Counter rollback detected %u < %u", 
         ht->head.counter, min_counter);
    return TEE_ERROR_SECURITY;
}

// Invalid version state detection
idx = get_idx_from_counter(head[0].counter, head[1].counter);
if (idx < 0) {
    EMSG("SECURITY VIOLATION: Invalid counter state");
    return TEE_ERROR_SECURITY;
}
```

### Authentication Failures
Cryptographic verification failures generate security events:

```c
// MAC verification failure logging
if (res == TEE_ERROR_MAC_INVALID) {
    EMSG("SECURITY EVENT: MAC verification failed - potential tampering");
    return TEE_ERROR_CORRUPT_OBJECT;
}

// Hash verification failure logging
if (consttime_memcmp(digest, node->node.hash, sizeof(digest))) {
    EMSG("SECURITY EVENT: Hash verification failed for node %zu", node->id);
    return TEE_ERROR_CORRUPT_OBJECT;
}
```

## Performance Profiling Support

### Block Access Pattern Analysis
Storage operations can be analyzed for access patterns:

```c
// Sequential vs. random access detection
size_t start_block_num = pos_to_block_num(pos);
size_t end_block_num = pos_to_block_num(pos + len - 1);

// Large sequential operations
if (end_block_num - start_block_num > SEQUENTIAL_THRESHOLD) {
    DMSG("Large sequential operation: blocks %zu-%zu", start_block_num, end_block_num);
}
```

### Cache Effectiveness Monitoring
RPMB file system includes caching metrics:

```c
// Cache configuration for monitoring
#define RPMB_BUF_MAX_ENTRIES (CFG_RPMB_FS_CACHE_ENTRIES + CFG_RPMB_FS_RD_ENTRIES)

struct rpmb_fat_entry_dir {
    struct rpmb_fat_entry *rpmb_fat_entry_buf;
    uint32_t idx_curr;        // Current index
    uint32_t num_buffered;    // Buffered entries
    uint32_t num_total_read;  // Total reads for hit rate calculation
    bool last_reached;        // End-of-data indicator
};
```

This comprehensive monitoring and diagnostic infrastructure enables effective debugging, performance analysis, and security monitoring of OP-TEE's storage subsystem, providing visibility into operations, errors, and security events across all storage layers.