# Ethereum & TRON Address Generation in Python (Secp256k1,Keccak-256,Secure Random Keys)


## Abstract

This article provides a complete, runnable Python example for generating Ethereum and TRON addresses from a securely generated private key. It explains the role of secp256k1 elliptic curve cryptography, Keccak-256 hashing, Base58Check encoding, and cryptographically secure random number generation.

------------------------------------------------------------------------

# Full Working Example (Runnable Code)

## Install Dependencies

``` bash
pip install coincurve eth_utils base58 pysha3
```

-   **coincurve** -- Python wrapper for libsecp256k1\
-   **pysha3** -- Keccak-256 implementation (Ethereum-compatible)\
-   **eth_utils** -- EIP-55 checksum formatting\
-   **base58 + hashlib** -- Used for TRON Base58Check encoding

------------------------------------------------------------------------

## Complete Code

``` python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os, hashlib, base58
import sha3
from coincurve import PublicKey
from eth_utils import to_checksum_address

SECP256K1_N = int("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141", 16)

def keccak256(b: bytes) -> bytes:
    k = sha3.keccak_256()
    k.update(b)
    return k.digest()

def gen_privkey_32() -> bytes:
    while True:
        priv = os.urandom(32)
        d = int.from_bytes(priv, 'big')
        if 1 <= d < SECP256K1_N:
            return priv

def pubkey64_from_priv(priv32: bytes) -> bytes:
    pub65 = PublicKey.from_secret(priv32).format(compressed=False)
    return pub65[1:]

def pubkey33_from_priv(priv32: bytes) -> bytes:
    return PublicKey.from_secret(priv32).format(compressed=True)

def eth_address_from_pub64(pub64: bytes) -> str:
    addr20 = keccak256(pub64)[-20:]
    return to_checksum_address("0x" + addr20.hex())

def tron_address_from_pub64(pub64: bytes) -> str:
    addr20 = keccak256(pub64)[-20:]
    payload = b'\x41' + addr20
    checksum = hashlib.sha256(hashlib.sha256(payload).digest()).digest()[:4]
    return base58.b58encode(payload + checksum).decode()

if __name__ == "__main__":
    priv   = gen_privkey_32()
    pub64  = pubkey64_from_priv(priv)
    pub33  = pubkey33_from_priv(priv)

    core20 = keccak256(pub64)[-20:]

    eth_addr  = eth_address_from_pub64(pub64)
    tron_addr = tron_address_from_pub64(pub64)

    print("Private Key (hex):", priv.hex())
    print("Public Key 64B (x||y):", pub64.hex())
    print("Public Key 33B (compressed):", pub33.hex())
    print("Shared 20B:", core20.hex())
    print("Ethereum Address:", eth_addr)
    print("TRON Address:", tron_addr)
```

------------------------------------------------------------------------

# Security Notes

-   Always use OS CSPRNG (`os.urandom`)\
-   Use rejection sampling for uniform scalar distribution\
-   Ethereum uses Keccak-256 (not FIPS SHA3-256)\
-   TRON uses Base58Check encoding with double SHA-256 checksum\
-   Same private key → Same 20-byte core → Different address encoding

------------------------------------------------------------------------

## Related Articles

-   Cross-Chain Public Key Tracing
-   TRON Offline Transaction Signing
-   Prevent ERC20 Approval Scams

- [Cross-Chain Public Key Tracing](/crypto/cross-chain-public-key-tracing-ethereum-bitcoin/)
- [TRON Offline Transaction Signing](/blockchain/tron-offline-transaction-signing-python/)
- [Prevent ERC20 Approval Scams](/security/prevent-erc20-approval-scams-proactive-defense-node/)


---

> 作者: <no value>  
> URL: https://www.lucas6.xyz/crypto/ethereum-tron-address-generation-python/  

