最近 Hermes Agent 非常的火热，所以想安装感受了一下。但是手上又没有云服务器，又不想天天把电脑开着，所以想着是否可以用 Termux 安装，没想到一次就跑通了。这样就只要带着手机即可，方便又便宜。
### 一、安装
简单 4 步走：
1. 从谷歌商店或第三方安装 Termux APP；
2. 按顺序执行以下命令：
``` bash
pkg update                // 更新源
pkg upgrade               // 升级系统
pkg install nodejs        // 安装 hermes 时需要
pkg install uv            // 安装 hermes 时需要

export UV_INDEX_URL="https://pypi.tuna.tsinghua.edu.cn/simple"  // 替换 uv 源，安装时加速
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash           // 中途会从 github 下载 hermes 代码，所以可能需要开个代理。正常者这一步安装完成就会自动启动 hermes, 我们根据提示去配置大模型和机器人就可以了

// hermes 源码下载完成后，如果安装过程卡住很久，就可以先退出来。再到源码路径下继续安装即可
cd .hermes/hermes-agent/scripts/
./install.sh

// 如果安装到最后有东西没配置好，想重新配置，就再执行
hermes setup 

// 配置好后直接启动，记得给 Termux 关闭电池优化，避免系统把后台清了
hermes gateway
```
3. 我选的机器人是微信，配置的时候会出来一个链接，复制到浏览器会有一个二维码，截图再用微信扫描即可。
4. 之后用微信随便给机器人发点东西，比如”你好“。他会回复你几句话，包括一个指令，在 Termux 中执行，在重启网关即可：
```bash
hermes pairing approve weixin XXXXXXXX    // XXXXXXXX 是验证码
hermes gateway stop
hermes gateway
```
### 二、使用
1. 让他安装了 Rust 工具链，并写一个简单的程序
![[img_v3_0211g_71f93433-258c-4240-8da3-4581ffa62e0g.jpg]]
2. 让他每天 8 点给我发分析报告
![[img_v3_0211g_9d5f92f3-9b89-4c19-868b-fb64a4b1812g.jpg]]
3. 次日 8 点准时给我发了分析报告
![[img_v3_0211g_80d04199-cf98-43f9-b687-91a13c477a9g.jpg]]
### 三、感受
我用的是 deepseek-v4-flash，整体使用感受还是很不错的。如果尝鲜的话可以白嫖 NVIDIA 提供的大模型，也包括 deepseek-v4 等。
