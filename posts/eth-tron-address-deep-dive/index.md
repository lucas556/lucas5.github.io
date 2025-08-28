# 区块链地址生成实战


> 这篇文章把可运行的 **完整示例代码** 放在最前面,随后给出**逐步解析**与 **Q&A**.拷贝本文代码即可直接生成 ETH 与 Tron 地址,并理解背后的安全细节.

---

## 一、完整示例代码(可直接运行)

### 依赖安装

```bash
pip install coincurve eth_utils base58 pysha3
```

- **coincurve**：libsecp256k1 的 Python 封装（负责 secp256k1 椭圆曲线点乘/签名等）  
- **pysha3**：以太坊使用的 Keccak-256（注意与 FIPS 的 SHA3-256 有细微差别）  
- **eth_utils**：EIP-55 校验地址  
- **base58 + hashlib**：Tron 的 Base58Check 与“双 SHA-256”校验

### 代码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os, hashlib, base58
import sha3                      # Keccak-256 (pysha3)
from coincurve import PublicKey  # secp256k1
from eth_utils import to_checksum_address

SECP256K1_N = int("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141", 16)

def keccak256(b: bytes) -> bytes:
    k = sha3.keccak_256()
    k.update(b)
    return k.digest()

# Step 1: 生成私钥（CSPRNG + 拒绝采样；不再做额外哈希混合）
def gen_privkey_32() -> bytes:
    while True:
        priv = os.urandom(32)
        d = int.from_bytes(priv, 'big')
        if 1 <= d < SECP256K1_N:
            return priv

# Step 2: 公钥（未压缩 64B 与压缩 33B）
def pubkey64_from_priv(priv32: bytes) -> bytes:
    pub65 = PublicKey.from_secret(priv32).format(compressed=False)  # 65B: 0x04|x|y
    return pub65[1:]  # 去掉 0x04 → 64B (x||y)

def pubkey33_from_priv(priv32: bytes) -> bytes:
    return PublicKey.from_secret(priv32).format(compressed=True)    # 33B: 0x02/0x03|x

# Step 3: 以太坊地址（Keccak → 后 20B → EIP-55 checksum）
def eth_address_from_pub64(pub64: bytes) -> str:
    addr20 = keccak256(pub64)[-20:]
    return to_checksum_address("0x" + addr20.hex())

# Step 4: Tron 地址（同源 20B → 0x41 → 双 SHA-256 → Base58Check）
def tron_address_from_pub64(pub64: bytes) -> str:
    addr20 = keccak256(pub64)[-20:]
    payload = b'\x41' + addr20
    checksum = hashlib.sha256(hashlib.sha256(payload).digest()).digest()[:4]
    return base58.b58encode(payload + checksum).decode()

if __name__ == "__main__":
    # 1) 私钥
    priv   = gen_privkey_32()

    # 2) 公钥（两种形态）
    pub64  = pubkey64_from_priv(priv)       # 64B,用于地址派生
    pub33  = pubkey33_from_priv(priv)       # 33B,备用

    # 3) 共同 20B（ETH 与 Tron 的共同核心）
    core20 = keccak256(pub64)[-20:]

    # 4) 生成地址
    eth_addr  = eth_address_from_pub64(pub64)
    tron_addr = tron_address_from_pub64(pub64)

    # 5) 输出
    print("私钥 (hex):", priv.hex())
    print("公钥(64B x||y) hex:", pub64.hex())
    print("公钥(33B 压缩) hex:", pub33.hex())
    print("共同20B (Keccak(pub64)后20B) hex:", core20.hex())
    print("ETH  地址:", eth_addr)
    print("TRON 地址:", tron_addr)
```

---

## 二、逐步解析(为什么这样写)

### 1. 随机数：`os.urandom(32)` 和 `os.urandom(8) × 4`

- **等价**：两者都来自操作系统的 CSPRNG,总熵都是 256 bit.  
- 多次 `os.urandom(8)` 并不会“更安全”,只是多了几次系统调用.  
- 只有在**合并不同熵源**(例如硬件 RNG + OS RNG)时,才考虑拼接后做一次哈希白化.

**对比代码：**
```python
# 一次取 32B（推荐简洁写法）
priv = os.urandom(32)

# 四次 8B 再拼接（安全性等价,不更强）
priv = b"".join(os.urandom(8) for _ in range(4))
```

### 2. “再次哈希”是否有必要

- 目的：**白化/抽取**随机性；在合并多个熵源或不完全信任某个熵源时有意义.  
- 常规桌面/服务器系统：`os.urandom(32)` 已足够安全；为保持简洁与可审计,本文不再做额外哈希.

**如果确实要合并熵源,可这样：**
```python
x = os.urandom(32) + os.urandom(32)  # 或者 加上硬件 RNG
priv = hashlib.sha256(x).digest()
# 之后仍需做范围校验（拒绝采样）
```

### 3. 取模 vs 拒绝采样（为何脚本里有 while True）

- **取模**：`d = r % n` 实现简单,但因 \(2^{256}\) 不是 \(n\) 的整倍数,会产生**极小分布偏差**（约 2^-128）.  
- **拒绝采样（推荐）**：生成 32B,若 `d >= n` 则丢弃重来；因 \(n pprox 2^{256}\),几乎总是一次成功,且**分布严格均匀**.  
- `while True:` 就是在表达“拒绝采样”,不是要循环很多次.

**示例：**
```python
while True:
    priv = os.urandom(32)
    d = int.from_bytes(priv, "big")
    if 1 <= d < SECP256K1_N:
        return priv
```

### 4. 公钥两种形态: 为什么同时输出 64B 与 33B

- **未压缩公钥**：`0x04 || x(32B) || y(32B)`；派生地址时,规范是用去掉前缀 `0x04` 的 **64B (x||y)** 做 Keccak.  
- **压缩公钥**：`0x02/0x03 || x(32B)`；常用于网络/链上存储、更省空间.  
- 实战里两者都常用,脚本里一并输出更方便.

### 5. "共同 20B"：以太坊与 Tron 地址的共同核心

- 定义：`Keccak-256(pub64)` 的**后 20 字节**.  
- 以太坊：共同 20B → 十六进制 → **EIP-55** 校验（大小写混合）.  
- Tron：共同 20B → 前缀 `0x41` → **双 SHA-256** 取前 4B 校验 → **Base58Check** 编码（`T...`）.  
- **意义**：同一私钥在两条链上的“共同 20B”一致,可用于**跨链同源校验**.

---

## 三、常见问答(Q&A)

**Q1：为何不能使用 coincurve替代 hashlib / pysha3？**  
A：`coincurve` 负责椭圆曲线(secp256k1)运算；地址生成需要 **Keccak-256**(`pysha3`)与 **SHA-256**(`hashlib`,Tron 的双哈希校验),以及 **Base58Check**（`base58`）.

**Q2：为什么不使用 `hashlib.sha3_256`？**  
A：以太坊使用的是“pre‑FIPS”的 **Keccak-256**,与 `hashlib.sha3_256`（FIPS SHA3-256）有细微差别,因此选择 `pysha3` 的 `sha3.keccak_256()`.

**Q3：拒绝采样会不会卡住？**  
A：不会.因 \(n pprox 2^{256}\),超界概率约 3.7×10^-39,几乎总是一次成功.

**Q4：Tron 地址为什么要双 SHA-256 + Base58Check？**  
A：这是一种 **人类友好且带校验** 的编码风格,起源于比特币地址规范,能有效降低抄写/传输错误.

**Q5：是否可以直接用 libsecp256k1（C 库）而不用 coincurve？**  
A：完全可以.`coincurve` 本质是对 libsecp256k1 的 Python 封装；若写 C/C++/Rust/Go,直接用原生库会更硬核与高性能.

---

## 四、要点速记（TL;DR）

- `os.urandom(32)` 足够安全；`os.urandom(8) × 4` 并不会更安全.  
- “再次哈希”仅在**合并独立熵源**时有意义；常规环境不必.  
- **拒绝采样**优于“取模”,能确保私钥分布**严格均匀**.  
- 生成地址要用未压缩公钥的 **64B (x||y)** 作为 Keccak 输入；压缩公钥 **33B** 适合存储/传输.  
- **共同 20B**：ETH 与 Tron 地址的共同核心；ETH 是 Hex+EIP‑55,Tron 是 `0x41` 前缀 + 双 SHA-256 + Base58Check.

---


---

> 作者: [Lucas](https://www.lucas6.xyz)  
> URL: https://www.lucas6.xyz/posts/eth-tron-address-deep-dive/  

