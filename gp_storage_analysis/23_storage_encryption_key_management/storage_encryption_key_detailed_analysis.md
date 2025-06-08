# OP-TEE Storage Encryption and Key Management - Detailed Analysis

## Overview

This document provides a deep analysis of OP-TEE's storage encryption and key management architecture, focusing on the hierarchical key structure, encryption algorithms, and initialization vector (IV) management for secure storage operations.

## File Encryption Key (FEK) Management

### Key Generation and Protection
- **Location**: `/home/dzb/optee/optee_os/core/tee/tee_fs_key_manager.c`
- **Key Size**: 16 bytes (TEE_FS_KM_FEK_SIZE)
- **Generation**: Uses hardware random number generator via `crypto_rng_read()`
- **Protection**: Encrypted using Trusted App Storage Key (TSK) via AES-ECB

```c
// FEK generation process
TEE_Result tee_fs_generate_fek(const TEE_UUID *uuid, void *buf, size_t buf_size)
{
    // Generate random FEK
    res = generate_fek(buf, TEE_FS_KM_FEK_SIZE);
    // Encrypt FEK with TSK for storage
    return tee_fs_fek_crypt(uuid, TEE_MODE_ENCRYPT, buf, TEE_FS_KM_FEK_SIZE, buf);
}
```

### FEK Cryptographic Operations
- **Algorithm**: AES-ECB-NOPAD for FEK encryption/decryption
- **Per-TA Protection**: Each TA gets unique TSK derived from SSK and TA UUID
- **Runtime Decryption**: FEK decrypted on-demand for data operations

## Secure Storage Key (SSK) Hierarchy

### Hardware Unique Key (HUK) Derivation
- **Location**: `/home/dzb/optee/optee_os/core/kernel/huk_subkey.c`
- **Root Source**: Hardware-specific unique key from OTP or chip ID
- **Size**: 32 bytes (TEE_FS_KM_SSK_SIZE)

```c
// SSK derivation from HUK
static TEE_Result tee_fs_init_key_manager(void)
{
    // Derive SSK from HUK using HMAC-SHA256
    res = huk_subkey_derive(HUK_SUBKEY_SSK, NULL, 0,
                           tee_fs_ssk.key, sizeof(tee_fs_ssk.key));
    if (res == TEE_SUCCESS)
        tee_fs_ssk.is_init = 1;
    return res;
}
```

### Trusted App Storage Key (TSK) Generation
- **Derivation**: HMAC-SHA256(SSK, TA_UUID)
- **Per-TA Isolation**: Each TA gets unique 32-byte TSK
- **Purpose**: Used to encrypt/decrypt FEKs for that specific TA

```c
// TSK derivation for specific TA
res = do_hmac(tsk, sizeof(tsk), tee_fs_ssk.key,
              TEE_FS_KM_SSK_SIZE, uuid, sizeof(*uuid));
```

## Hash Tree (HTREE) Encryption Implementation

### Block-Level Encryption Architecture
- **Location**: `/home/dzb/optee/optee_os/core/tee/fs_htree.c`
- **Algorithm**: AES-CBC with ESSIV (Enhanced Sector-Sequence Initialization Vector)
- **Block Size**: Configurable (typically 4KB for REE FS)

### ESSIV (Encrypted Salt-Sector IV) Implementation

```c
// ESSIV computation for unique per-block IVs
static TEE_Result essiv(uint8_t iv[TEE_AES_BLOCK_SIZE],
                       const uint8_t fek[TEE_FS_KM_FEK_SIZE],
                       uint16_t blk_idx)
{
    uint8_t sha[TEE_SHA256_HASH_SIZE];
    uint8_t pad_blkid[TEE_AES_BLOCK_SIZE] = { 0, };
    
    // Hash FEK to create salt
    res = sha256(sha, sizeof(sha), fek, TEE_FS_KM_FEK_SIZE);
    
    // Pad block index
    pad_blkid[0] = (blk_idx & 0xFF);
    pad_blkid[1] = (blk_idx & 0xFF00) >> 8;
    
    // Encrypt block index with hashed FEK to get IV
    return aes_ecb(iv, pad_blkid, sha, 16);
}
```

### Authenticated Encryption for Metadata
- **Algorithm**: AES-GCM for htree headers and node metadata
- **Tag Size**: 16 bytes (TEE_FS_HTREE_TAG_SIZE)
- **AAD Components**: FEK, IV, hash, counter values

```c
// AES-GCM initialization for metadata protection
res = crypto_authenc_init(ctx, mode, ht->fek, TEE_FS_HTREE_FEK_SIZE, iv,
                         TEE_FS_HTREE_IV_SIZE, TEE_FS_HTREE_TAG_SIZE,
                         aad_len, payload_len);
```

## IV Management and Security

### IV Generation Strategy
- **Random IVs**: Generated using `crypto_rng_read()` for each write operation
- **Uniqueness**: Combination of FEK hash + block index prevents IV reuse
- **Storage**: IVs stored alongside encrypted data in node metadata

### Per-Block IV Architecture
- **Block-Specific**: Each data block gets unique IV derived from block index
- **Collision Prevention**: ESSIV ensures different files with same block patterns have different IVs
- **Replay Protection**: IV changes on every write operation

## Key Lifecycle Management

### Initialization Sequence
1. **Boot Time**: SSK derived from HUK during system initialization
2. **TA Load**: TSK computed when TA requests storage operations
3. **File Open**: FEK decrypted from storage using TSK
4. **Block Access**: Per-block IVs computed using ESSIV

### Key Rotation and Management
- **FEK Rotation**: New FEK generated for new files, existing files keep same FEK
- **TSK Stability**: TSK remains constant for each TA across reboots
- **SSK Persistence**: SSK derived from stable HUK, consistent across system lifecycles

### Secure Key Destruction
```c
// Explicit memory clearing after key operations
memzero_explicit(tsk, sizeof(tsk));
memzero_explicit(dst_key, sizeof(dst_key));
memzero_explicit(fek, sizeof(fek));
memzero_explicit(iv, sizeof(iv));
```

## Cryptographic Algorithms Summary

| Component | Algorithm | Key Size | Purpose |
|-----------|-----------|----------|---------|
| HUK | Hardware-specific | 32 bytes | Root key material |
| SSK | HMAC-SHA256(HUK) | 32 bytes | System storage key |
| TSK | HMAC-SHA256(SSK, UUID) | 32 bytes | Per-TA storage key |
| FEK | Random | 16 bytes | Per-file encryption key |
| Data Blocks | AES-CBC + ESSIV | 16 bytes | File content encryption |
| Metadata | AES-GCM | 16 bytes | Integrity + confidentiality |

## Security Properties

### Confidentiality Guarantees
- **Hardware Root**: Keys ultimately rooted in hardware-unique material
- **TA Isolation**: Each TA cannot access other TA's encrypted storage
- **Forward Secrecy**: File deletion removes FEK, making recovery impossible

### Integrity Protection
- **HMAC Verification**: All key derivations use cryptographic MACs
- **Authenticated Encryption**: Metadata protected with AES-GCM
- **Hash Tree**: Data blocks protected by cryptographic hash tree

### Anti-Rollback Features
- **Counter Integration**: Storage operations tied to monotonic counters
- **Versioning**: Dual-version storage with atomic commit mechanisms
- **Replay Prevention**: IV uniqueness prevents block replay attacks

This encryption and key management architecture provides comprehensive protection for OP-TEE secure storage, ensuring both confidentiality and integrity while maintaining per-TA isolation and hardware-rooted security.