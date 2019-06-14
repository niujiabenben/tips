## 修改某一文件夹中的文件/目录权限

```shell
### 文件夹默认权限为755, 文件的默认权限为644
find ${dirname} -type d -print -exec chmod 755 {} \;
find ${dirname} -type f -print -exec chmod 644 {} \;
```

## Making zsh default shell without root access

Create .bash_profile in your home directory and add these lines:

```shell
if [ -x /bin/zsh ]; then
    export SHELL=/bin/zsh
    exec /bin/zsh -l
fi
```

source: https://unix.stackexchange.com/questions/136423/making-zsh-default-shell-without-root-access

## linux添加账户

```shell
sudo useradd -d /home/chenli -m chenli -s /bin/bash
sudo passwd chenli
sudo nano /etc/sudoers
chenli  ALL=(ALL:ALL) ALL

### 删除账户
sudo userdel -r chenli
```

## 安装第三方库

```shell
sudo apt install build-essential
sudo apt install automake autoconf
sudo apt install clang libclang-dev
sudo apt install libgoogle-glog-dev libgflags-dev libboost-all-dev
sudo apt install libopencv-dev libjsoncpp-dev libcurl4-openssl-dev libfreetype6-dev
```

## 安装emacs

```shell
wget https://mirrors.ustc.edu.cn/gnu/emacs/emacs-26.1.tar.gz
wget https://mirrors.sjtug.sjtu.edu.cn/gnu/emacs/emacs-26.1.tar.gz

### without x-window:
sudo apt-get install libncurses-dev
./configure --without-x --with-gnutls=no --prefix=${HOME}/Documents/tools/emacs
make -j4 && make install

### all packages:
sudo apt install texinfo libx11-dev libxpm-dev libjpeg-dev libpng-dev
sudo apt install libgif-dev libtiff-dev libgtk2.0-dev libncurses-dev libxpm-dev
./configure --prefix=${HOME}/Documents/tools/emacs
make -j4 && make install
```

## Ubuntu Error: System program problem detected

sudo nano /etc/default/apport
Change the line that says enabled=1 to enabled=0 to disable Apport.
