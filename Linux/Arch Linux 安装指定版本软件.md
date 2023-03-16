能够用上最新的软件包是archlinux的福利，然而可能软件的最新版本有问题，或者必须使用某个软件的固定版本，这时应该如何安装这个老版本软件？
例如我安装 `gdb-multiarch` 依赖 `gdb-common=12.1` ，则可以到 [Arch Linux Archive](https://archive.archlinux.org/packages) 找到包的指定版本的 url 然后用以下指令安装：
```bash
yay -U https://archive.archlinux.org/packages/g/gdb-common/gdb-common-12.1-2-x86_64.pkg.tar.zst
```
或者将其下载下来，再安装：
```bash
yay -U gdb-common-12.1-2-x86_64.pkg.tar.zst
```
