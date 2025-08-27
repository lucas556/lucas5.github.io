# CUDA NVCC安装


CUDA运算需要安装NVCC

安装cuda-toolkit-11-2(ubuntu18.04):

```shell
wget https://developer.download.nvidia.com/compute/cuda/11.2.2/local_installers/cuda_11.2.2_460.32.03_linux.run
sudo sh cuda_11.2.2_460.32.03_linux.run
```

```shell
# set PATH
# vim ~/.bashrc
export PATH=$PATH:/usr/local/cuda-11.2/bin
export LD_LIBRARY_PATH=/usr/local/cuda-11.2/lib64
export CUDA_HOME=/usr/local/cuda

source ~/.bashrc
nvcc --version
```


---

> 作者: [Lucas](https://www.lucas6.xyz)  
> URL: https://www.lucas6.xyz/posts/320100/  

