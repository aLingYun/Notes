## 一、启动一条链
配置环境：
```bash
# 1.安装预编译包
sudo apt update && sudo apt install -y git clang curl libssl-dev llvm libudev-dev gcc g++ make

# 2.安装Rust编译环境
curl https://sh.rustup.rs -sSf | sh
source ~/.cargo/env
rustup default stable
rustup update
rustup update nightly
rustup target add wasm32-unknown-unknown --toolchain nightly
```
克隆模板源码：
```bash
git clone https://github.com/substrate-developer-hub/substrate-node-template
cd substrate-node-template
git checkout latest
```
编译：
```bash
cargo build --release
```
有可能会遇到如下问题：
```bash
error: failed to run custom build command for `libp2p-core v0.37.0`

Caused by:
  process didn't exit successfully: `/root/substrate-node-template/target/debug/build/libp2p-core-52d7442f1559e8d5/build-script-build` (exit status: 101)
  --- stderr
  thread 'main' panicked at 'Could not find `protoc` installation and this build crate cannot proceed without
      this knowledge. If `protoc` is installed and this crate had trouble finding
      it, you can set the `PROTOC` environment variable with the specific path to your
      installed `protoc` binary.If you're on debian, try `apt-get install protobuf-compiler` or download it from https://github.com/protocolbuffers/protobuf/releases

  For more information: https://docs.rs/prost-build/#sourcing-protoc
  ', /root/.cargo/registry/src/github.com-1ecc6299db9ec823/prost-build-0.11.4/src/lib.rs:1296:10
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
warning: build failed, waiting for other jobs to finish...
```
看提示可知需要安装 protoc：
```bash
sudo apt update
sudo apt install libprotobuf-dev protobuf-compiler
```
编译成功后使用如下命令运行：
```bash
./target/release/node-template --dev
```

![](attachments/Pasted%20image%2020230128155904.png)

启动后按如下步骤连接连链：
```
1、在浏览器中输入https://polkadot.js.org/apps；
2、点击左上角会展开；
3、在展开的菜单中点击DEVELOPMENT；
4、点击Local Node；
5、点击switch。
```

![](attachments/Pasted%20image%2020230128160039.png)
