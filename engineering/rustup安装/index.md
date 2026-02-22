# Rustup安装


依赖环境 Rustup 安装:

```shell
mkdir -p $HOME/.cargo
# vim $HOME/.cargo/config

[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```
---

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# vim ~/.bashrc
export PATH=$PATH:$HOME/.cargo/bin
```


---

> 作者: [Lucas](https://www.lucas6.xyz)  
> URL: https://www.lucas6.xyz/engineering/rustup%E5%AE%89%E8%A3%85/  

