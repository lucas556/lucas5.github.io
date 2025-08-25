# 用 Bitcoin2john 和 Hashcat 爆破 Wallet.dat 密码


在较早期的 Bitcoin Core 钱包中,私钥是保存在一个名为 `wallet.dat` 的加密文件里.很多人在创建钱包时设置了密码用于保护私钥,但多年之后很可能已经忘记了当初的密码.如果没有这个密码,`wallet.dat` 是无法解锁的.

目前并不存在可以绕过该加密机制的“万能工具”.即便有类似声称支持恢复的软件或服务,多数也只能破解弱密码,甚至可能是恶意诱导用户上传钱包文件的骗局.

不过,由于 Bitcoin Core 使用的是标准的 `PBKDF2-HMAC-SHA512` 作为加密算法,我们可以借助 `bitcoin2john` 提取哈希,再用 `hashcat` 进行暴力破解(太复杂密码不可能破解).

这篇文章将完整演示这个过程.

---

## 提取 wallet.dat 哈希

从 Bitcoin Core 的文件夹中提取出`wallet.dat` 文件.

然后需要使用 `bitcoin2john.py` 脚本将加密的 `wallet.dat` 文件转为 hashcat 支持的格式.

注意：John 官方的原始版本在一些新环境（如 Ubuntu 22.04 + Python 3.10）上已经不可用.推荐使用这个修复后的版本：

[https://github.com/lucas556/btc_wallet-recover/blob/main/bitcoin2john.py](https://github.com/lucas556/btc_wallet-recover/blob/main/bitcoin2john.py)

此外,该脚本依赖 `berkeleydb`,你需要先安装：

```bash
pip install berkeleydb
```

执行提取命令如下：

```bash
python3 bitcoin2john.py wallet.dat
```

输出结果类似：

```
$bitcoin$64$6dabee7730bb1d6f20f7f8019ef2fc8922753f35cb258a52add31114899e19fd$16$70813ad5382f7a5a$166925$2$00$2$00
```

保存这行哈希到一个文件,比如 `wallet.hash`.

---

## 使用 hashcat 进行破解

接下来我们使用 hashcat 对刚才提取的哈希进行爆破.下面以尝试 6 到 9 位纯数字密码为例：

```bash
hashcat -m 11300 wallet.hash -a 3 ?d?d?d?d?d?d --increment --increment-min=6 --increment-max=9
```

如果你使用的是独立显卡,可以使用以下方式启用 GPU：

需要安装nvcc(nvidia),参考: [https://lucas5.xyz/posts/0909320100/](https://lucas5.xyz/posts/0909320100/)

```bash
./hashcat.bin -m 11300 ../wallet.hash -a 3 ?d?d?d?d?d?d --increment --backend-ignore-opencl --force
```

**参数说明：**

- `-m 11300`：指定 wallet.dat 所用的哈希算法（Bitcoin Core）
- `-a 3`：使用掩码模式
- `?d`：每一位是数字
- `--increment`：开启位数逐步尝试,从 6 位递增至 9 位

运行中的状态输出示例：

```
Status...........: Exhausted
Hash.Mode........: 11300 (Bitcoin/Litecoin wallet.dat)
Recovered........: 0/1 (0.00%) Digests
Progress.........: 100.00%
...
The wordlist or mask that you are using is too small.
```

如果破解成功,终端会显示类似：

```
$bitcoin$... : myoldpass2020
```

这时候你就可以拿着这个密码重新打开 Bitcoin Core 钱包.

---

## 额外说明

- 如果你设置的密码较为复杂（如字母、符号、大小写组合）,纯掩码暴力破解几乎不现实.
- 可以使用字典攻击,例如：

  ```bash
  hashcat -m 11300 wallet.hash -a 0 rockyou.txt
  ```

- 千万不要将 wallet.dat 上传到任何不可信网站,即使它声称“免费恢复”或“AI破解”——这很可能是诈骗.
- 本方法适用于你**已拥有本地 wallet.dat 文件且忘记密码**的场景.

---

这是一个完全离线、透明、受控的恢复流程,适用于你希望亲自操作而不依赖第三方服务的情况.如果你使用的是冷钱包、遗忘了早期钱包密码,这种方式是目前最稳妥的尝试路径.


---

> 作者: [Lucas](https://lucas5.xyz)  
> URL: https://www.lucas6.xyz/posts/20252607660128/  

