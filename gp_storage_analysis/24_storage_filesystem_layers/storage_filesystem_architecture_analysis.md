# OP-TEE Storage File System Layers - Architectural Analysis

## Overview

This document analyzes the layered architecture of OP-TEE's storage file system, covering the hash tree implementation (fs_htree), directory file management (fs_dirfile), and object lifecycle management across the storage stack.

## Hash Tree File System (fs_htree) Implementation

### Core Architecture
- **Location**: `/home/dzb/optee/optee_os/core/tee/fs_htree.c`
- **Purpose**: Provides integrity and confidentiality for block-level storage
- **Structure**: Binary tree with cryptographic hashes for integrity verification

### Hash Tree Node Structure

```c
struct htree_node {
    size_t id;               // Node identifier in tree
    bool dirty;              // Needs synchronization to storage
    bool block_updated;      // Block data has been modified
    struct tee_fs_htree_node_image node;  // Cryptographic metadata
    struct htree_node *parent;            // Parent node pointer
    struct htree_node *child[2];          // Left/right child pointers
};
```

### File Layout Organization
```
Hash Tree File Structure:
+----------------------------+
| htree_image.0              | <- Header version 0
| htree_image.1              | <- Header version 1  
+----------------------------+
| htree_node_image.1.0       | <- Node 1, version 0
| htree_node_image.1.1       | <- Node 1, version 1
+----------------------------+
| htree_node_image.2.0       | <- Node 2, version 0
| htree_node_image.2.1       | <- Node 2, version 1
+----------------------------+
| Data blocks...             |
+----------------------------+
```

### Binary Tree Navigation

```c
// Node ID to level calculation
static size_t node_id_to_level(size_t node_id)
{
    // Root node (1) has level 1, binary tree structure
    return sizeof(unsigned int) * 8 - __builtin_clz(node_id);
}

// Tree traversal for closest node
static struct htree_node *find_closest_node(struct tee_fs_htree *ht, size_t node_id)
{
    struct htree_node *node = &ht->root;
    size_t level = node_id_to_level(node_id);
    
    for (n = 1; n < level; n++) {
        size_t bit_idx = level - n - 1;
        child = node->child[((node_id >> bit_idx) & 1)];
        if (!child) return node;
        node = child;
    }
    return node;
}
```

### Integrity Verification Mechanism

```c
// Node hash calculation including children
static TEE_Result calc_node_hash(struct htree_node *node,
                                struct tee_fs_htree_meta *meta, void *ctx,
                                uint8_t *digest)
{
    // Hash node data (excluding existing hash)
    uint8_t *ndata = (uint8_t *)&node->node + sizeof(node->node.hash);
    size_t nsize = sizeof(node->node) - sizeof(node->node.hash);
    
    crypto_hash_update(ctx, ndata, nsize);
    
    // Include metadata for root node
    if (meta)
        crypto_hash_update(ctx, (void *)meta, sizeof(*meta));
    
    // Include children hashes
    if (node->child[0])
        crypto_hash_update(ctx, node->child[0]->node.hash, sizeof(...));
    if (node->child[1])
        crypto_hash_update(ctx, node->child[1]->node.hash, sizeof(...));
    
    return crypto_hash_final(ctx, digest, TEE_FS_HTREE_HASH_SIZE);
}
```

## Directory File Management (fs_dirfile)

### Directory Structure
- **Location**: `/home/dzb/optee/optee_os/core/tee/fs_dirfile.c`
- **Purpose**: Maps object IDs to file numbers and manages directory metadata
- **Storage**: Single file containing array of directory entries

### Directory Entry Format

```c
struct dirfile_entry {
    TEE_UUID uuid;                              // TA identifier
    uint8_t oid[TEE_OBJECT_ID_MAX_LEN];        // Object identifier
    uint32_t oidlen;                           // Object ID length
    uint8_t hash[TEE_FS_HTREE_HASH_SIZE];     // Root hash of file
    uint32_t file_number;                       // Physical file number
};
```

### File Number Management

```c
struct tee_fs_dirfile_dirh {
    const struct tee_fs_dirfile_operations *fops;
    struct tee_file_handle *fh;    // Underlying file handle
    int nbits;                     // Size of file bitmap
    bitstr_t *files;              // Bitmap of allocated file numbers
    size_t ndents;                // Number of directory entries
};
```

### Object Lifecycle Operations

#### File Creation Flow
```c
// Get temporary file number
TEE_Result tee_fs_dirfile_get_tmp(struct tee_fs_dirfile_dirh *dirh,
                                  struct tee_fs_dirfile_fileh *dfh)
{
    // Find first free bit in file number bitmap
    if (dirh->nbits) {
        bit_ffc(dirh->files, dirh->nbits, &i);
        if (i == -1) i = dirh->nbits;  // Expand if needed
    }
    
    res = set_file(dirh, i);  // Mark file number as used
    if (!res) dfh->file_number = i;
    return res;
}
```

#### File Lookup Process
```c
// Find file by TA UUID and object ID
TEE_Result tee_fs_dirfile_find(struct tee_fs_dirfile_dirh *dirh,
                               const TEE_UUID *uuid, const void *oid,
                               size_t oidlen, struct tee_fs_dirfile_fileh *dfh)
{
    for (n = 0;; n++) {
        res = read_dent(dirh, n, &dent);  // Read directory entry
        if (res) return res;
        
        if (is_free(&dent)) continue;     // Skip free entries
        if (dent.oidlen != oidlen) continue;
        
        // Check UUID and OID match
        if (!memcmp(&dent.uuid, uuid, sizeof(dent.uuid)) &&
            !memcmp(&dent.oid, oid, oidlen))
            break;  // Found matching entry
    }
    
    // Return file handle information
    dfh->idx = n;
    dfh->file_number = dent.file_number;
    memcpy(dfh->hash, dent.hash, sizeof(dent.hash));
    return TEE_SUCCESS;
}
```

## REE File System Integration

### Storage Backend Architecture
- **Location**: `/home/dzb/optee/optee_os/core/tee/tee_ree_fs.c`
- **Purpose**: Integrates hash tree with REE (Rich Execution Environment) file system
- **Block Management**: 4KB blocks with out-of-place write operations

### File Descriptor Structure

```c
struct tee_fs_fd {
    struct tee_fs_htree *ht;              // Hash tree handle
    int fd;                               // REE file descriptor
    struct tee_fs_dirfile_fileh dfh;      // Directory file handle
    const TEE_UUID *uuid;                 // TA identifier
};
```

### Out-of-Place Write Implementation

```c
static TEE_Result out_of_place_write(struct tee_fs_fd *fdp, size_t pos,
                                     const void *buf_core, const void *buf_user,
                                     size_t len)
{
    size_t start_block_num = pos_to_block_num(pos);
    size_t end_block_num = pos_to_block_num(pos + len - 1);
    uint8_t *block = get_tmp_block();
    
    while (start_block_num <= end_block_num) {
        size_t offset = pos % BLOCK_SIZE;
        size_t size_to_write = MIN(remain_bytes, (size_t)BLOCK_SIZE);
        
        // Read existing block if partial write
        if (start_block_num * BLOCK_SIZE < ROUNDUP(meta->length, BLOCK_SIZE)) {
            res = tee_fs_htree_read_block(&fdp->ht, start_block_num, block);
        } else {
            memset(block, 0, BLOCK_SIZE);  // New block
        }
        
        // Modify block content
        if (data_core_ptr) {
            memcpy(block + offset, data_core_ptr, size_to_write);
        } else if (data_user_ptr) {
            copy_from_user(block + offset, data_user_ptr, size_to_write);
        }
        
        // Write modified block back
        res = tee_fs_htree_write_block(&fdp->ht, start_block_num, block);
        
        // Update metadata if file length increased
        if (pos > meta->length) {
            meta->length = pos;
            tee_fs_htree_meta_set_dirty(fdp->ht);
        }
        
        start_block_num++;
        pos += size_to_write;
    }
}
```

## Object Lifecycle Management

### Creation Workflow
1. **Directory Setup**: Open/create directory file using `tee_fs_dirfile_open()`
2. **File Number**: Allocate unique file number via `tee_fs_dirfile_get_tmp()`
3. **Hash Tree**: Create new hash tree with `tee_fs_htree_open(create=true)`
4. **Encryption**: Generate FEK and initialize encryption context
5. **Metadata**: Store object metadata in directory entry

### Access Control Integration
```c
static TEE_Result get_dirh(struct tee_fs_dirfile_dirh **dirh)
{
    if (!ree_fs_dirh) {
        TEE_Result res = open_dirh(&ree_fs_dirh);  // Open directory
        if (res) {
            *dirh = NULL;
            return res;
        }
    }
    ree_fs_dirh_refcount++;  // Reference counting
    *dirh = ree_fs_dirh;
    return TEE_SUCCESS;
}
```

### Deletion and Cleanup
```c
TEE_Result tee_fs_dirfile_remove(struct tee_fs_dirfile_dirh *dirh,
                                 const struct tee_fs_dirfile_fileh *dfh)
{
    struct dirfile_entry dent = { };
    
    // Read current directory entry
    res = read_dent(dirh, dfh->idx, &dent);
    if (res) return res;
    
    if (is_free(&dent)) return TEE_SUCCESS;  // Already deleted
    
    uint32_t file_number = dent.file_number;
    
    // Clear directory entry
    memset(&dent, 0, sizeof(dent));
    res = write_dent(dirh, dfh->idx, &dent);
    
    // Mark file number as available
    if (!res) clear_file(dirh, file_number);
    
    return res;
}
```

## Layer Interaction and Data Flow

### Read Operation Flow
1. **Lookup**: `tee_fs_dirfile_find()` locates file by UUID/OID
2. **Open**: `tee_fs_htree_open()` initializes hash tree with root hash
3. **Verification**: Hash tree verifies integrity during read
4. **Decryption**: Block-level decryption using stored FEK
5. **Return**: Decrypted data returned to caller

### Write Operation Flow
1. **Acquire**: Get directory handle with reference counting
2. **Modify**: Out-of-place write updates blocks in hash tree
3. **Sync**: `tee_fs_htree_sync_to_storage()` commits changes
4. **Update**: Directory entry updated with new root hash
5. **Commit**: Directory changes committed to storage

## Performance Optimizations

### Memory Management
- **Block Pooling**: Temporary blocks allocated from memory pools
- **Reference Counting**: Directory handles shared across operations
- **Lazy Sync**: Changes batched and committed atomically

### I/O Optimization
- **Block Alignment**: All operations aligned to block boundaries
- **Minimal Reads**: Only read blocks that need modification
- **Batch Operations**: Multiple changes committed in single transaction

This layered architecture provides robust storage management while maintaining security guarantees and performance optimization for OP-TEE's secure storage implementation.