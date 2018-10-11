## 修改某一文件夹中的文件/目录权限

```shell
### 文件夹默认权限为755, 文件的默认权限为644
find ${dirname} -type d -print -exec chmod 755 {} \;
find ${dirname} -type f -print -exec chmod 644 {} \;
```
