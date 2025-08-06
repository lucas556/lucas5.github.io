# 从签名中恢复账户公钥


文中所有数据为虚构,可根据实际情况从区块链获取.

#### 注意:使用助记词(bip39/bip44)生成的地址使用了path,每个地址的公钥也不同(即不同的链的公钥和私钥是不同的),不适合使用该方法(无法使用公钥跨链追踪).

本文代码已托管：
📎 [https://github.com/lucas556/Blockchain-Tools/blob/main/TRON/sig2pub.py](https://github.com/lucas556/Blockchain-Tools/blob/main/TRON/sig2pub.py)


## 为什么要恢复公钥？

在区块链的世界里,私钥签名、公钥验证是最基本也最重要的安全机制之一.

我们通常：

- 使用私钥对一笔交易进行签名
- 网络节点用公钥验证这笔交易是否合法

## 背景知识

### 签名结构

区块链签名基于椭圆曲线 secp256k1,结构通常为 (r, s, v)：

- `r`：签名的第一部分
- `s`：签名的第二部分
- `v`：恢复参数,用于帮助从签名中恢复对应的公钥

### raw_data 是什么？

`raw_data`：是一笔交易在签名前的原始字节数据,系统会先对其 SHA256 响应,将响应结果用于签名.

## 核心原理简图

```
raw_data  --SHA256-->  message_hash
                       |
                       ↓
        signature (r, s, v) + message_hash
                       |
                       ↓
        recover → public key
```

## 使用 Python 实现公钥恢复

### 安装依赖

```bash
pip install eth-keys
```

### 编写恢复函数

```python
import hashlib
from eth_keys.datatypes import Signature

def recover_public_key_from_tron_signature(raw_data_hex: str, signature_hex: str) -> str:
    raw_data_bytes = bytes.fromhex(raw_data_hex)
    message_hash = hashlib.sha256(raw_data_bytes).digest()

    signature_bytes = bytes.fromhex(signature_hex)
    if len(signature_bytes) != 65:
        raise ValueError("签名必须是 65 字节 (r:32 + s:32 + v:1)")

    r = int.from_bytes(signature_bytes[0:32], byteorder='big')
    s = int.from_bytes(signature_bytes[32:64], byteorder='big')
    v = signature_bytes[64]
    if v >= 27:
        v -= 27

    sig = Signature(vrs=(v, r, s))
    public_key = sig.recover_public_key_from_msg_hash(message_hash)
    return public_key.to_bytes().hex()
```

### 示例演示

```python
raw_data_hex = "0a0265782208706c61792d6d6f6465120f0a0b08c0843d1215e1a0e7c2d103"
signature_hex = (
    "7db18bd6ccfffd4f8a5c208a3fd3f34d8f6e2db6f6ea84539b1b3e42b3d47809"
    "79f18e5f741188cb80795d5f8a590b92edbb3e351c80e58208e7b4bbdcbb4ffb1b"
)

recover_public_key_from_tron_signature(raw_data_hex, signature_hex)
```

**输出结果：**

```
Message Hash (SHA256 of raw_data): 9f1430d6ee3d00b8bd5f2a78cbf60b683bdd3d3e1a7b2cf361942c5e110ce1a1
Parsed Signature:
  r = 0x7db18bd6ccfffd4f8a5c208a3fd3f34d8f6e2db6f6ea84539b1b3e42b3d47809
  s = 0x79f18e5f741188cb80795d5f8a590b92edbb3e351c80e58208e7b4bbdcbb4ffb
  v = 1
Recovered Public Key (hex): 04a3f5d0c72ae1955bbf61a69b276bda7e3a7dcff4790d85e3c166e2c99f3b37d3d5f31a7e8876e37e63c6d2dfd4717e7e8c5f8c99b624d6c1441188ac11bca5ed
```

该公钥为未压缩格式,长度为 130 个 hex 字符 (65 字节),前缀 `04` 表示未压缩格式.


---

> 作者: [Lucas](https://lucas5.xyz)  
> URL: https://www.lucas5.pro/posts/2025060560122/  

