在网上找到 secoclient-linux-64-7.0.2.33.run 下载，并安装：
```sh
chomod +x secoclient-linux-64-7.0.2.33.run
sudo ./secoclient-linux-64-7.0.2.33.run
```
可能会报一个错误： `arch: command not found` ,解决方法如下：
```bash
vim secoclient-linux-64-7.0.2.33.run
```
将报错行的 'arch' 换成 x86_64（按照你的架构），再执行安装。
```sh
#############confirm architecture#####
ARCH="x86_64"        //此行
if [ "$ARCH" = "i386" ]; then
    FLAG="x86"
elif [ "$ARCH" = "i486" ]; then
    FLAG="x86"
elif [ "$ARCH" = "i586" ]; then
    FLAG="x86"
elif [ "$ARCH" = "i686" ]; then
    FLAG="x86"
elif [ "$ARCH" = "x86_64" ]; then
    FLAG="x86"
else
    FLAG="UNKNOWN"
fi
```
如果成功应该会出现 ** enjoy ** 的字样。

因为这个安装包是给 ubuntu 使用的，Arch linux 没有 /etc/init.d 这个文件夹，安装程序无法将 `SecoClientPromoteService.sh` 脚本存放进去。所以我们需要先手动启动这个脚本：
```sh
sudo /usr/local/SecoClient/promote/SecoClientPromoteService.sh start
```
再运行 SecoClient 程序。
```
/usr/local/SecoClient/SecoClient &
```
如果报错：
```sh
qt.qpa.plugin: Could not load the Qt platform plugin "xcb" in "" even though it was found.
This application failed to start because no Qt platform plugin could be initialized. Reinstalling the application may fix this problem.

Available platform plugins are: linuxfb, minimal, offscreen, xcb.

Aborted (core dumped)
```
那么到 `/usr/local/SecoClient` 目录下修改 `install.sh` 中的 `arch` 为你的架构，并重新安装：
```sh
sudo ./install.sh
```