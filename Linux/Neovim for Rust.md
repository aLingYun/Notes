### 1. 首先配置好终端代理
```bash
yay -S proxychains  //安装代理工具

vim /etc/proxychains.conf  //在文件最后的[ProxyList]一节中增加代理设置，例如
                           //socks5 127.0.0.1 1080
```

接下来，所有希望走代理的命令，前面增加`proxychains`即可，例如：
```bash
proxychains git clone git@github.com:aLingYun/Notes.git
```

### 2. 配置 neovim
1) 安装 Rust
```bash
proxychains curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
2) Clone 配置文件
```bash
mv ~/.config/nvim ~/.config/nvimbackup
proxychains git clone https://github.com/AstroNvim/AstroNvim ~/.config/nvim
```
3) 启动 nvim 的包管理器去安装插件
```bash
nvim +PackerSync
```
4) 安装 LSP 和 tree-sitter
```bash
nvim .                   // open nvim

:LspInstall rust         // 安装 LSP

:TSInstall rust          // 安装 tree-sitter
```

如此便得到一个比较好用的 neovim for rust。

![](attachments/Pasted%20image%2020230126180313.png)

但是你会发现界面中有很多图标是问号，这需要安装并在终端中使用 Nerd 字体：
