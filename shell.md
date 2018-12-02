## 修改某一文件夹中的文件/目录权限

```shell
### 文件夹默认权限为755, 文件的默认权限为644
find ${dirname} -type d -print -exec chmod 755 {} \;
find ${dirname} -type f -print -exec chmod 644 {} \;
```

## Making zsh default shell without root access

Create .bash_profile in your home directory and add these lines:

```shell
export SHELL=/bin/zsh
exec /bin/zsh -l
```

source: https://unix.stackexchange.com/questions/136423/making-zsh-default-shell-without-root-access
