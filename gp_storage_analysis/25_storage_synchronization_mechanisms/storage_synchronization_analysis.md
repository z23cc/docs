# OP-TEE Storage Synchronization Mechanisms - Analysis

## Overview

This document analyzes OP-TEE's storage synchronization mechanisms, including commit protocols, atomic write operations, consistency checking, and recovery mechanisms that ensure data integrity across storage operations.

## Atomic Commit Protocols

### Hash Tree Synchronization
- **Location**: `/home/dzb/optee/optee_os/core/tee/fs_htree.c`
- **Function**: `tee_fs_htree_sync_to_storage()`
- **Strategy**: Two-phase commit with versioning for atomic updates

### Dual-Version Storage Strategy

```c
// Hash tree image header with versioning
struct tee_fs_htree_image {
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];           // Initialization vector
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];         // Authentication tag
    uint8_t enc_fek[TEE_FS_HTREE_FEK_SIZE];     // Encrypted file key
    uint8_t imeta[sizeof(struct tee_fs_htree_imeta)];  // Metadata
    uint32_t counter;                           // Monotonic counter
};

// Node versioning with committed flags
#define HTREE_NODE_COMMITTED_BLOCK    BIT32(0)
#define HTREE_NODE_COMMITTED_CHILD(n) BIT32(1 + (n))
```

### Counter-Based Version Selection

```c
// Determine which version is committed based on counters
static int get_idx_from_counter(uint32_t counter0, uint32_t counter1)
{
    if (!(counter0 & 1)) {
        if (!(counter1 & 1))
            return 0;                    // Both even, use 0
        if (counter0 > counter1)
            return 0;                    // 0 newer
        else
            return 1;                    // 1 newer
    }
    
    if (counter1 & 1)
        return 1;                        // Both odd, use 1
    else
        return -1;                       // Invalid state
}
```

## Write-Ahead Logging (WAL) Implementation

### Node-Level Versioning
Each hash tree node maintains two versions, switching between them during updates:

```c
static TEE_Result htree_sync_node_to_storage(struct traverse_arg *targ,
                                             struct htree_node *node)
{
    if (!node->dirty) return TEE_SUCCESS;   // No changes to commit
    
    if (node->parent) {
        // Toggle child version bit in parent
        uint32_t f = HTREE_NODE_COMMITTED_CHILD(node->id & 1);
        node->parent->dirty = true;
        node->parent->node.flags ^= f;      // Flip version bit
        vers = !!(node->parent->node.flags & f);
    } else {
        // Root node uses counter LSB for versioning
        vers = !(targ->ht->head.counter & 1);
    }
    
    // Calculate new hash and write to storage
    res = calc_node_hash(node, meta, targ->arg, node->node.hash);
    node->dirty = false;
    node->block_updated = false;
    
    return rpc_write_node(targ->ht, node->id, vers, &node->node);
}
```

### Post-Order Traversal for Consistency
Changes committed from leaves to root, ensuring parent hashes reflect child changes:

```c
static TEE_Result traverse_post_order(struct traverse_arg *targ,
                                      struct htree_node *node)
{
    if (!node) return TEE_SUCCESS;
    
    // Process children first (post-order)
    res = traverse_post_order(targ, node->child[0]);
    if (res != TEE_SUCCESS) return res;
    
    res = traverse_post_order(targ, node->child[1]);
    if (res != TEE_SUCCESS) return res;
    
    // Process current node after children
    return targ->cb(targ, node);
}
```

## Directory File Synchronization

### Commit Protocol
- **Location**: `/home/dzb/optee/optee_os/core/tee/fs_dirfile.c`
- **Function**: `tee_fs_dirfile_commit_writes()`
- **Integration**: Calls underlying hash tree commit mechanism

```c
TEE_Result tee_fs_dirfile_commit_writes(struct tee_fs_dirfile_dirh *dirh,
                                       uint8_t *hash, uint32_t *counter)
{
    return dirh->fops->commit_writes(dirh->fh, hash, counter);
}
```

### REE File System Integration
- **Location**: `/home/dzb/optee/optee_os/core/tee/tee_ree_fs.c`
- **Counter Management**: Monotonic counter synchronization with normal world

```c
static TEE_Result commit_dirh_writes(struct tee_fs_dirfile_dirh *dirh)
{
    uint32_t counter = 0;
    
    // Commit changes to storage
    res = tee_fs_dirfile_commit_writes(dirh, NULL, &counter);
    if (res) return res;
    
    // Update monotonic counter in normal world
    res = nv_counter_incr_ree_fs_to(counter);
    if (res == TEE_ERROR_NOT_IMPLEMENTED && IS_ENABLED(CFG_INSECURE)) {
        IMSG("WARNING (insecure configuration): Failed to commit dirh counter %"PRIu32, counter);
        return TEE_SUCCESS;  // Allow operation without counter in debug builds
    }
    return res;
}
```

## Consistency Checking Mechanisms

### Hash Tree Verification
Complete tree verification during open operations:

```c
static TEE_Result verify_tree(struct tee_fs_htree *ht)
{
    void *ctx;
    
    res = crypto_hash_alloc_ctx(&ctx, TEE_FS_HTREE_HASH_ALG);
    if (res != TEE_SUCCESS) return res;
    
    // Traverse tree and verify all node hashes
    res = htree_traverse_post_order(ht, verify_node, ctx);
    crypto_hash_free_ctx(ctx);
    
    return res;
}

static TEE_Result verify_node(struct traverse_arg *targ, struct htree_node *node)
{
    void *ctx = targ->arg;
    uint8_t digest[TEE_FS_HTREE_HASH_SIZE];
    
    // Calculate expected hash
    if (node->parent)
        res = calc_node_hash(node, NULL, ctx, digest);
    else
        res = calc_node_hash(node, &targ->ht->imeta.meta, ctx, digest);
    
    // Verify hash matches stored value
    if (res == TEE_SUCCESS && 
        consttime_memcmp(digest, node->node.hash, sizeof(digest)))
        return TEE_ERROR_CORRUPT_OBJECT;
    
    return res;
}
```

### Authenticated Encryption Verification
Root metadata protected with AES-GCM:

```c
static TEE_Result verify_root(struct tee_fs_htree *ht)
{
    void *ctx;
    
    // Decrypt FEK using TA-specific key
    res = tee_fs_fek_crypt(ht->uuid, TEE_MODE_DECRYPT, ht->head.enc_fek,
                          sizeof(ht->fek), ht->fek);
    if (res != TEE_SUCCESS) return res;
    
    // Initialize authenticated decryption
    res = authenc_init(&ctx, TEE_MODE_DECRYPT, ht, NULL, sizeof(ht->imeta));
    if (res != TEE_SUCCESS) return res;
    
    // Verify and decrypt metadata
    return authenc_decrypt_final(ctx, ht->head.tag, ht->head.imeta,
                                sizeof(ht->imeta), &ht->imeta);
}
```

## Recovery After Power Failure

### Rollback-Safe Design
The dual-version storage ensures atomic commits:

1. **Phase 1**: Write new versions of all changed nodes
2. **Phase 2**: Update header counter to commit transaction
3. **Recovery**: On restart, select valid version based on counter

### Counter Validation
```c
static TEE_Result init_head_from_data(struct tee_fs_htree *ht,
                                     const uint8_t *hash, uint32_t min_counter)
{
    struct tee_fs_htree_image head[2];
    
    // Read both header versions
    for (idx = 0; idx < 2; idx++) {
        res = rpc_read_head(ht, idx, head + idx);
        if (res != TEE_SUCCESS) return res;
    }
    
    // Select valid version based on counter algorithm
    idx = get_idx_from_counter(head[0].counter, head[1].counter);
    if (idx < 0) return TEE_ERROR_SECURITY;  // Invalid state
    
    // Verify counter meets minimum requirement (anti-rollback)
    if (head[idx].counter < min_counter)
        return TEE_ERROR_SECURITY;
    
    ht->head = head[idx];
    return TEE_SUCCESS;
}
```

## RPMB File System Synchronization

### Write Counter Integration
- **Location**: `/home/dzb/optee/optee_os/core/tee/tee_rpmb_fs.c`
- **Version**: FS_VERSION constant for format compatibility
- **Counter**: RPMB hardware write counter for anti-rollback

```c
struct rpmb_fs_partition {
    uint32_t rpmb_fs_magic;      // Magic number: 0x52504D42
    uint32_t fs_version;         // File system version: 2
    uint32_t write_counter;      // Current write counter
    uint32_t fat_start_address;  // File allocation table location
    uint8_t reserved[112];       // Reserved space
};
```

### Atomic RPMB Operations
RPMB operations are inherently atomic due to hardware write counter:

```c
// RPMB request/response with MAC protection
struct rpmb_data_frame {
    uint8_t stuff_bytes[RPMB_STUFF_DATA_SIZE];
    uint8_t key_mac[RPMB_KEY_MAC_SIZE];      // HMAC protection
    uint8_t data[RPMB_DATA_SIZE];
    uint8_t nonce[RPMB_NONCE_SIZE];
    uint32_t write_counter;                   // Monotonic counter
    uint16_t address;
    uint16_t block_count;
    uint16_t result;
    uint16_t req_resp;
};
```

## Thread Safety and Concurrency

### Mutex Protection
Global mutex protects directory operations:

```c
static struct mutex ree_fs_mutex = MUTEX_INITIALIZER;

static TEE_Result ree_fs_open(struct tee_pobj *po, size_t *size,
                              struct tee_file_handle **fh)
{
    mutex_lock(&ree_fs_mutex);
    
    res = get_dirh(&dirh);           // Get directory handle
    if (res != TEE_SUCCESS) goto out;
    
    res = tee_fs_dirfile_find(dirh, &po->uuid, po->obj_id, po->obj_id_len, &dfh);
    // ... perform operations
    
out:
    if (res) put_dirh(dirh, true);   // Release on error
    mutex_unlock(&ree_fs_mutex);
    return res;
}
```

### Reference Counting
Directory handle shared across operations with reference counting:

```c
static void put_dirh_primitive(bool close)
{
    assert(ree_fs_dirh_refcount);
    
    ree_fs_dirh_refcount--;
    if (ree_fs_dirh && (!ree_fs_dirh_refcount || close))
        close_dirh(&ree_fs_dirh);    // Close when no references
}
```

## Error Handling and Cleanup

### Transactional Cleanup
Failed operations trigger automatic cleanup:

```c
TEE_Result tee_fs_htree_sync_to_storage(struct tee_fs_htree **ht_arg,
                                       uint8_t *hash, uint32_t *counter)
{
    struct tee_fs_htree *ht = *ht_arg;
    
    if (!ht->dirty) return TEE_SUCCESS;  // No changes
    
    // Commit all nodes in post-order
    res = htree_traverse_post_order(ht, htree_sync_node_to_storage, ctx);
    if (res != TEE_SUCCESS) goto out;
    
    // Update and commit root header
    res = update_root(ht);
    if (res != TEE_SUCCESS) goto out;
    
    res = rpc_write_head(ht, ht->head.counter & 1, &ht->head);
    
out:
    crypto_hash_free_ctx(ctx);
    if (res != TEE_SUCCESS)
        tee_fs_htree_close(ht_arg);      // Clean up on failure
    return res;
}
```

## Performance Considerations

### Batched Operations
- **Node Updates**: All dirty nodes committed in single traversal
- **I/O Optimization**: Minimal number of storage operations
- **Memory Efficiency**: Temporary blocks reused across operations

### Lazy Synchronization
- **Dirty Tracking**: Only modified nodes synchronized to storage
- **Deferred Writes**: Changes batched until explicit sync
- **Reference Counting**: Shared resources minimize overhead

This synchronization architecture ensures data consistency and durability while providing reasonable performance for OP-TEE's secure storage operations.