# 基于 YubiKey 的冷钱包探索:使用 GPG 公钥生成以太坊地址

在区块链的世界中,如何安全保管私钥始终是一个核心问题.YubiKey 作为一款硬件安全密钥,长期被用于 SSH 和 GPG 签名.我们尝试探索一个方向:

**是否可以用 YubiKey 来构建一个离线;不可提取私钥的冷钱包?**

在实际调研中发现,YubiKey 的 OpenPGP 模块支持自定义算法,其中包括 secp256k1.虽然它并非以太坊原生格式的密钥,但通过提取 GPG 公钥中的 EC 点数据(即未压缩公钥),我们仍然可以导出与以太坊地址对应的 Keccak-256 哈希,从而进行地址验证.

本文将介绍如何从一个导出的 GPG 公钥(ASCII 格式)中恢复未压缩的 secp256k1 公钥,并转换为对应的以太坊地址.

---

## GPG 生成 secp256k1 密钥

确保你的 GPG 支持 `secp256k1`(需要 gnupg >= 2.2.27):

```bash
gpg --full-generate-key
```

在交互菜单中选择:

- 密钥类型:`ECC` 或 `ECC and ECC`
- 曲线选择:`secp256k1`
- 有效期/用户名/邮箱等按需输入
- 设置密码(可以不设置)

然后使用以下命令导出公钥:

```bash
gpg --export-options export-reset-subkey-passwd --export YOUR_KEY_ID > pubkey.asc
```

---

## 实现逻辑概览

1. 使用 `PGPy` 解析 ASCII 格式的 GPG 公钥(`.asc` 文件)
2. 提取公钥中嵌入的 ECPoint(未压缩椭圆曲线公钥)
3. 使用以太坊标准(Keccak-256)计算地址
4. 得到标准 Ethereum 地址,可用于匹配与验证

---

## 安装依赖

```bash
pip install pgpy eth-utils
```

---

## 脚本示例

```python
from pgpy import PGPKey
from eth_utils import keccak, to_checksum_address
import struct

def extract_pubkey_from_gpg_ascii(path: str) -> bytes:
    with open(path, "r") as f:
        blob = f.read()
    key, _ = PGPKey.from_blob(blob)

    # 获取 ECPoint 对象 → 转为 MPI bytes(2字节bitlen + 65字节数据)
    mpi_bytes = key._key.keymaterial.p.to_mpibytes()

    # 解析 MPI → 跳过前2字节bit长度
    bitlen = struct.unpack(">H", mpi_bytes[:2])[0]
    keydata = mpi_bytes[2:]

    if len(keydata) != 65 or keydata[0] != 0x04:
        raise ValueError("不是未压缩 secp256k1 公钥")
    return keydata

def pubkey_to_eth_address(pubkey_bytes: bytes) -> str:
    keccak_digest = keccak(pubkey_bytes[1:])[-20:]
    return to_checksum_address("0x" + keccak_digest.hex())

# 主程序
pubkey_bytes = extract_pubkey_from_gpg_ascii("pubkey.asc")
print("Ethereum Pubkey:", pubkey_bytes.hex())
eth_address = pubkey_to_eth_address(pubkey_bytes)
print("Ethereum address:", eth_address)
```

---

## 注意事项

- 该方法要求密钥为 **未压缩的 secp256k1 公钥**
- 当前支持的是 GPG 主密钥或子密钥使用 secp256k1 算法的情况
- 若使用的是 RSA 或 ed25519,无法用于 Ethereum 地址恢复
- 要确保 `pgpy` 能正确读取你的密钥文件格式

---

## 应用场景

这种方式的核心意义在于:

- 可以利用 YubiKey 的 GPG 模块中的密钥做链上身份验证
- **无需导出私钥**,确保密钥仅存在于硬件中
- 适用于构建**离线冷钱包**或硬件签名设备
- 可搭配 GPG 签名对离线交易或消息进行认证,验证签名对应的 Ethereum 地址是否一致

---

利用这一方法,可以验证 YubiKey 中保存的密钥是否对应某个以太坊地址,并以 GPG 格式参与链上签名或身份认证.这为构建一个安全/去信任的冷钱包方案,提供了全新的实现路径.


---

> 作者: Lucas  
> URL: https://www.lucas6.xyz/keys/%E5%9F%BA%E4%BA%8E-yubikey-%E7%9A%84%E5%86%B7%E9%92%B1%E5%8C%85%E6%8E%A2%E7%B4%A2%E4%BD%BF%E7%94%A8-gpg-%E5%85%AC%E9%92%A5%E7%94%9F%E6%88%90%E4%BB%A5%E5%A4%AA%E5%9D%8A%E5%9C%B0%E5%9D%80/  

