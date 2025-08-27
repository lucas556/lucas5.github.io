# Golang安装


依赖环境 Golang 安装(1.16.4):

```shell
wget -c https://studygolang.com/dl/golang/go1.16.4.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local

# vim ~/.bashrc
export PATH=$PATH:/usr/local/go/bin
export GOPROXY=https://goproxy.cn
source ~/.bashrc
```


---

> 作者: [Lucas](https://www.lucas6.xyz)  
> URL: https://www.lucas6.xyz/posts/golang-install/  

