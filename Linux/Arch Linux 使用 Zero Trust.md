  
### 一、安装
在 archlinux 下只需要执行以下指令即可：
```bash
yay -S cloudflare-warp-bin
```
其他平台可以去官网（在浏览器输入 1.1.1.1 即可）安装。

如果通过 yay 遇到问题，可以去缓存文件夹里改脚本 `PKGBUILD`。
```bash
cd ~/.cache/yay/cloudflare-warp-bin/

nvim PKGBUILD

makepkg
```
我遇到了 sha256sum error 的问题，检查发现脚本中计算 sha256sum 的指令加了参数，导致无法获得结果。然后把参数删了，保存再执行 `makepkg` 指令即可。

### 二、注册登录
之后再去 Zero Trust 网站去注册账号，最好不要用国内的邮箱。注册之后选择免费的套餐即可，至于如何避开使用信用卡，可以网上搜索一下，很简单。
```bash
sudo systemctl start warp-svc.service  //启用服务

systemctl --user enable --now warp-taskbar   //启用状态图标
```
在状态栏图标上右键选择 登录Zero Trust。填写你注册过程中填的企业名称，邮箱，密码，验证码。

有可能你会遇到这样的问题：**Unable to connect Failed DNS lookup**

解决办法是在配置文件 `/etc/systemd/resolved.conf` 中开启如下配置，重启即可连接成功：
```
ResolveUnicastSingleLabel=yes
```