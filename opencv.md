## different behavior of cv2.imread and cv2.imdecode

对于某些图片，`cv2.imread()`和`cv2.imdecode()`两个函数给出的输出并不一样.
目前尚不清楚原因. 可能是图片遭到损坏.

训练的时候两者可以通用，测试的时候最好用cv2.imread，已保持输入的一致性.

## 用anaconda安装opencv

通常的conda install opencv操作安装的opencv没有视频io的功能，需要设置conda-forge源,
.condarc配置文件内容如下：
```
channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
  - conda-forge
  - defaults
show_channel_urls: true
```
再执行：
```shell
conda install opencv ffmpeg
```

如果安装失败，可以考虑下载最近版的anaconda，彻底重装一遍。

说明:
1. .condarc中，前两个都是清华镜像地址，后两个是官方的源;

2. .condarc中，conda-forge必须排在前面，因为默认的源中也有opencv，
    如果conda-forge排在后面，则会被默认的源中的资源给覆盖掉.
