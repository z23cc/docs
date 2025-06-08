# OP-TEE Storage Security Features - Comprehensive Analysis

## Overview

This document provides a comprehensive analysis of OP-TEE's storage security features, including anti-rollback mechanisms, tamper detection, secure deletion, forensics prevention, and defense against various attack vectors targeting secure storage.

## Anti-Rollback Mechanisms

### Monotonic Counter Integration
OP-TEE implements robust anti-rollback protection through monotonic counters at multiple layers:

```c
// REE File System Counter Protection
static TEE_Result commit_dirh_writes(struct tee_fs_dirfile_dirh *dirh)
{
    uint32_t counter = 0;
    
    // Commit changes and get new counter value
    res = tee_fs_dirfile_commit_writes(dirh, NULL, &counter);
    if (res) return res;
    
    // Increment monotonic counter in normal world
    res = nv_counter_incr_ree_fs_to(counter);
    if (res == TEE_ERROR_NOT_IMPLEMENTED && IS_ENABLED(CFG_INSECURE)) {
        IMSG("WARNING (insecure configuration): Failed to commit dirh counter %"PRIu32, counter);
        return TEE_SUCCESS;  // Only allowed in debug builds
    }
    return res;
}
```

### Hash Tree Counter Validation
Every hash tree header includes a monotonic counter for anti-rollback:

```c
struct tee_fs_htree_image {
    uint8_t iv[TEE_FS_HTREE_IV_SIZE];
    uint8_t tag[TEE_FS_HTREE_TAG_SIZE];
    uint8_t enc_fek[TEE_FS_HTREE_FEK_SIZE];
    uint8_t imeta[sizeof(struct tee_fs_htree_imeta)];
    uint32_t counter;                    // Anti-rollback counter
};

// Counter validation during file open
static TEE_Result init_head_from_data(struct tee_fs_htree *ht,
                                     const uint8_t *hash, uint32_t min_counter)
{
    // ... read headers
    
    // Enforce minimum counter requirement
    if (ht->head.counter < min_counter) {
        EMSG("SECURITY VIOLATION: Counter rollback detected %u < %u", 
             ht->head.counter, min_counter);
        return TEE_ERROR_SECURITY;
    }
    
    return TEE_SUCCESS;
}
```

### RPMB Hardware Counter
RPMB storage leverages hardware write counters for ultimate anti-rollback protection:

```c
struct rpmb_data_frame {
    uint8_t stuff_bytes[RPMB_STUFF_DATA_SIZE];
    uint8_t key_mac[RPMB_KEY_MAC_SIZE];      // HMAC authentication
    uint8_t data[RPMB_DATA_SIZE];
    uint8_t nonce[RPMB_NONCE_SIZE];
    uint32_t write_counter;                   // Hardware monotonic counter
    uint16_t address;
    uint16_t block_count;
    uint16_t result;
    uint16_t req_resp;
};
```

## Tamper Detection Mechanisms

### Cryptographic Hash Tree Integrity
Hash tree provides comprehensive tamper detection through cryptographic verification:

```c
// Node integrity verification
static TEE_Result verify_node(struct traverse_arg *targ, struct htree_node *node)
{
    void *ctx = targ->arg;
    uint8_t digest[TEE_FS_HTREE_HASH_SIZE];
    
    // Calculate expected hash
    if (node->parent)
        res = calc_node_hash(node, NULL, ctx, digest);
    else
        res = calc_node_hash(node, &targ->ht->imeta.meta, ctx, digest);
    
    // Constant-time comparison to prevent timing attacks
    if (res == TEE_SUCCESS && 
        consttime_memcmp(digest, node->node.hash, sizeof(digest))) {
        EMSG("SECURITY EVENT: Hash tree tampering detected for node %zu", node->id);
        return TEE_ERROR_CORRUPT_OBJECT;
    }
    
    return res;
}
```

### Authenticated Encryption for Metadata
Critical metadata protected with AES-GCM authenticated encryption:

```c
// Root metadata authentication
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
    
    // Verify authentication tag and decrypt
    res = authenc_decrypt_final(ctx, ht->head.tag, ht->head.imeta,
                               sizeof(ht->imeta), &ht->imeta);
    if (res == TEE_ERROR_MAC_INVALID) {
        EMSG("SECURITY EVENT: Metadata authentication failed - tampering detected");
        return TEE_ERROR_CORRUPT_OBJECT;
    }
    
    return res;
}
```

### RPMB MAC Protection
RPMB data frames include HMAC protection against tampering:

```c
static TEE_Result tee_rpmb_mac_calc(uint8_t *mac, size_t macsize,
                                   const uint8_t *key, size_t keysize,
                                   struct rpmb_data_frame *datafrm,
                                   size_t datanfrm)
{
    void *ctx = NULL;
    
    res = crypto_mac_alloc_ctx(&ctx, TEE_ALG_HMAC_SHA256);
    if (res != TEE_SUCCESS) return res;
    
    res = crypto_mac_init(ctx, key, keysize);
    if (res != TEE_SUCCESS) goto func_exit;
    
    // MAC covers all frame data
    for (i = 0; i < datanfrm; i++) {
        res = crypto_mac_update(ctx, datafrm[i].data, RPMB_DATA_SIZE);
        if (res != TEE_SUCCESS) goto func_exit;
        
        res = crypto_mac_update(ctx, datafrm[i].nonce, RPMB_NONCE_SIZE);
        if (res != TEE_SUCCESS) goto func_exit;
        
        // Include all metadata in MAC calculation
        res = crypto_mac_update(ctx, (uint8_t *)&datafrm[i].write_counter,
                               sizeof(datafrm[i].write_counter));
        // ... continue for all fields
    }
    
    return crypto_mac_final(ctx, mac, macsize);
}
```

## Secure Deletion and Sanitization

### Explicit Memory Clearing
All sensitive cryptographic material explicitly cleared after use:

```c
// Key manager secure cleanup
TEE_Result tee_fs_fek_crypt(const TEE_UUID *uuid, TEE_OperationMode mode,
                           const uint8_t *in_key, size_t size,
                           uint8_t *out_key)
{
    uint8_t tsk[TEE_FS_KM_TSK_SIZE];
    uint8_t dst_key[size];
    
    // ... perform encryption/decryption
    
exit:
    crypto_cipher_free_ctx(ctx);
    memzero_explicit(tsk, sizeof(tsk));      // Secure TSK deletion
    memzero_explicit(dst_key, sizeof(dst_key)); // Secure temp key deletion
    return res;
}

// Block encryption secure cleanup
TEE_Result tee_fs_crypt_block(const TEE_UUID *uuid, uint8_t *out,
                             const uint8_t *in, size_t size,
                             uint16_t blk_idx, const uint8_t *encrypted_fek,
                             TEE_OperationMode mode)
{
    uint8_t fek[TEE_FS_KM_FEK_SIZE];
    uint8_t iv[TEE_AES_BLOCK_SIZE];
    
    // ... perform block encryption/decryption
    
wipe:
    memzero_explicit(fek, sizeof(fek));      // Secure FEK deletion
    memzero_explicit(iv, sizeof(iv));        // Secure IV deletion
    return res;
}
```

### File Deletion Security
File deletion removes encryption keys, making recovery cryptographically impossible:

```c
// Directory entry deletion
TEE_Result tee_fs_dirfile_remove(struct tee_fs_dirfile_dirh *dirh,
                                 const struct tee_fs_dirfile_fileh *dfh)
{
    struct dirfile_entry dent = { };
    
    // Read current entry
    res = read_dent(dirh, dfh->idx, &dent);
    if (res) return res;
    
    if (is_free(&dent)) return TEE_SUCCESS;
    
    uint32_t file_number = dent.file_number;
    
    // Zero out entire directory entry (including encrypted FEK)
    memset(&dent, 0, sizeof(dent));
    res = write_dent(dirh, dfh->idx, &dent);
    
    // Mark file number as available for reuse
    if (!res) clear_file(dirh, file_number);
    
    return res;
}
```

### Hardware Unique Key Protection
Root keys derived from hardware, making extraction extremely difficult:

```c
TEE_Result __huk_subkey_derive(enum huk_subkey_usage usage,
                              const void *const_data, size_t const_data_len,
                              uint8_t *subkey, size_t subkey_len)
{
    void *ctx = NULL;
    struct tee_hw_unique_key huk = { };
    
    // Get hardware-specific unique key
    res = tee_otp_get_hw_unique_key(&huk);
    if (res) goto out;
    
    // Derive subkey using HMAC
    res = crypto_mac_init(ctx, huk.data, sizeof(huk.data));
    // ... perform derivation
    
out:
    if (res)
        memzero_explicit(subkey, subkey_len);    // Clear on failure
    memzero_explicit(&huk, sizeof(huk));         // Always clear HUK
    crypto_mac_free_ctx(ctx);
    return res;
}
```

## Forensics Prevention

### Key Isolation Architecture
Each TA gets unique storage keys, preventing cross-TA access:

```c
// Per-TA key derivation
TEE_Result tee_fs_fek_crypt(const TEE_UUID *uuid, TEE_OperationMode mode,
                           const uint8_t *in_key, size_t size,
                           uint8_t *out_key)
{
    uint8_t tsk[TEE_FS_KM_TSK_SIZE];
    
    if (uuid) {
        // Derive TA-specific key: TSK = HMAC(SSK, TA_UUID)
        res = do_hmac(tsk, sizeof(tsk), tee_fs_ssk.key,
                     TEE_FS_KM_SSK_SIZE, uuid, sizeof(*uuid));
    } else {
        // Non-TA storage uses different salt
        uint8_t dummy[1] = { 0 };
        res = do_hmac(tsk, sizeof(tsk), tee_fs_ssk.key,
                     TEE_FS_KM_SSK_SIZE, dummy, sizeof(dummy));
    }
    
    // Use TSK to encrypt/decrypt FEK
    res = crypto_cipher_init(ctx, mode, tsk, sizeof(tsk), NULL, 0, NULL, 0);
    // ... perform operation
}
```

### ESSIV Prevents Pattern Analysis
Enhanced Sector Sequence IV prevents block-level pattern analysis:

```c
// ESSIV computation prevents pattern correlation
static TEE_Result essiv(uint8_t iv[TEE_AES_BLOCK_SIZE],
                       const uint8_t fek[TEE_FS_KM_FEK_SIZE],
                       uint16_t blk_idx)
{
    uint8_t sha[TEE_SHA256_HASH_SIZE];
    uint8_t pad_blkid[TEE_AES_BLOCK_SIZE] = { 0, };
    
    // Hash FEK to create encryption key for IV
    res = sha256(sha, sizeof(sha), fek, TEE_FS_KM_FEK_SIZE);
    if (res != TEE_SUCCESS) return res;
    
    // Pad block index to AES block size
    pad_blkid[0] = (blk_idx & 0xFF);
    pad_blkid[1] = (blk_idx & 0xFF00) >> 8;
    
    // Encrypt block index with hashed FEK to get unique IV
    res = aes_ecb(iv, pad_blkid, sha, 16);
    
    memzero_explicit(sha, sizeof(sha));  // Clear intermediate key
    return res;
}
```

### Constant-Time Operations
Cryptographic operations use constant-time implementations to prevent timing attacks:

```c
// Constant-time memory comparison
if (res == TEE_SUCCESS && 
    consttime_memcmp(digest, node->node.hash, sizeof(digest)))
    return TEE_ERROR_CORRUPT_OBJECT;
```

## Attack Vector Mitigation

### Replay Attack Prevention
IV uniqueness and counters prevent replay attacks:

```c
// Unique IV generation for each write
if (mode == TEE_MODE_ENCRYPT) {
    res = crypto_rng_read(iv, TEE_FS_HTREE_IV_SIZE);  // Fresh random IV
    if (res != TEE_SUCCESS) return res;
}

// Counter increment prevents replay
ht->head.counter++;  // Increment on every sync operation
```

### Side-Channel Attack Mitigation
Key operations use secure implementations:

```c
// Secure random number generation for keys
static TEE_Result generate_fek(uint8_t *key, uint8_t len)
{
    return crypto_rng_read(key, len);  // Hardware RNG when available
}
```

### Brute Force Protection
Strong cryptographic algorithms with sufficient key sizes:

```c
// 256-bit HMAC keys for storage
#define TEE_FS_KM_SSK_SIZE    TEE_SHA256_HASH_SIZE  // 32 bytes
#define TEE_FS_KM_TSK_SIZE    TEE_SHA256_HASH_SIZE  // 32 bytes

// AES-256 equivalent protection through key derivation
// Even with 128-bit FEK, effective security through hierarchical keys
#define TEE_FS_KM_FEK_SIZE    U(16)  // 16 bytes, but protected by 256-bit keys
```

## Security Event Logging

### Anomaly Detection
Security violations trigger detailed logging:

```c
// Counter rollback detection
if (ht->head.counter < min_counter) {
    EMSG("SECURITY VIOLATION: Counter rollback detected %u < %u", 
         ht->head.counter, min_counter);
    return TEE_ERROR_SECURITY;
}

// Invalid state detection
idx = get_idx_from_counter(head[0].counter, head[1].counter);
if (idx < 0) {
    EMSG("SECURITY VIOLATION: Invalid counter state - possible corruption");
    return TEE_ERROR_SECURITY;
}

// Duplicate file number detection
if (test_file(dirh, dent.file_number)) {
    DMSG("SECURITY EVENT: Duplicate file number %" PRIu32 " detected", 
         dent.file_number);
    // ... cleanup duplicate entry
}
```

### Authentication Failure Logging
All authentication failures are logged for analysis:

```c
// MAC verification failures
if (res == TEE_ERROR_MAC_INVALID) {
    EMSG("SECURITY EVENT: MAC verification failed - potential tampering");
    return TEE_ERROR_CORRUPT_OBJECT;
}

// Hash verification failures
if (consttime_memcmp(digest, node->node.hash, sizeof(digest))) {
    EMSG("SECURITY EVENT: Hash verification failed for node %zu", node->id);
    return TEE_ERROR_CORRUPT_OBJECT;
}
```

## Configuration Security

### Secure Defaults
Security features enabled by default with explicit opt-out for development:

```c
// Insecure configurations require explicit enablement
if (res == TEE_ERROR_NOT_IMPLEMENTED && IS_ENABLED(CFG_INSECURE)) {
    static bool once;
    if (!once) {
        IMSG("WARNING (insecure configuration): Failed to get monotonic counter");
        once = true;
    }
    // Only allow in explicitly insecure builds
}

// Reset protection enabled by default
if (!IS_ENABLED(CFG_REE_FS_ALLOW_RESET)) {
    DMSG("Security policy: REE FS reset not allowed");
    return TEE_ERROR_SECURITY;
}
```

### Build-Time Security Validation
Compile-time assertions ensure security properties:

```c
// Ensure adequate key sizes
COMPILE_TIME_ASSERT(TEE_FS_KM_SSK_SIZE <= HUK_SUBKEY_MAX_LEN);

// Validate block size constraints for security
COMPILE_TIME_ASSERT(TEST_BLOCK_SIZE > sizeof(struct tee_fs_htree_node_image) * 2);
```

This comprehensive security architecture provides defense-in-depth protection for OP-TEE storage, combining cryptographic protection, anti-rollback mechanisms, tamper detection, secure deletion, and forensics prevention to protect sensitive data against a wide range of attack vectors.