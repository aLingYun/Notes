```sh
yay -S rdesktop
```
首先需要允许此windows远程访问。
基本操作：计算机---属性---远程设置---远程，
勾选：允许远程连接到此计算机。去掉默认勾选：仅允许运行使用网络级别验证...，（如果不取消这个，在运行时会出现“ERROR: recv: 连接被对端重置”）。
```sh
rdesktop -f ip_address -d domain -u user -p password
```
注： `-f` 参数默认全屏打开，使用 `Ctrl + Alt + Enter` 可以退出全屏模式；
    `-d` 后面跟 域名；
	`-u` 后面跟 用户名；
	`-p` 后面跟 账户密码。