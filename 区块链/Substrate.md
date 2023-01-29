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

## 二、添加一个 pallet
第一节克隆的代码中 `./pallet/template` 文件夹是一个模板，我门可以直接复制一份放在同级目录下（这里命名为exam）。并删除暂时不需要的几个文件：
```sh
rm benchmarking.rs mock.rs tests.rs
```

在 `./pallet/exam/src/lib.rs` 文件中写入如下代码：
```rust
#![cfg_attr(not(feature = "std"), no_std)]

// 1. Imports and Dependencies
pub use pallet::*;
#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;

    // 2. Declaration of the Pallet type
    // This is a placeholder to implement traits and methods.
    #[pallet::pallet]
    #[pallet::generate_store(pub(super) trait Store)]
    pub struct Pallet<T>(_);

    // 3. Runtime Configuration Trait
    // All types and constants go here.
    // Use #[pallet::constant] and #[pallet::extra_constants]
    // to pass in values to metadata.
    #[pallet::config]
    pub trait Config: frame_system::Config { 
                type RuntimeEvent: From<Event<Self>> 
                        + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        }

    // 4. Runtime Storage
    // Use to declare storage items.
        #[pallet::storage]
    pub type Proofs<T: Config> =
        StorageMap<_, Blake2_128Concat, u32, u128>;

    // 5. Runtime Events
    // Can stringify event types to metadata.
    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
                ClaimCreated(u32, u128),
        }

    // 7. Extrinsics
    // Functions that are callable from outside the runtime.
        #[pallet::call]
    impl<T:Config> Pallet<T> { 
        #[pallet::weight(0)]
        pub fn create_claim(origin: OriginFor<T>, id: u32, claim: u128) -> DispatchResultWithPostInfo {
            ensure_signed(origin)?;

            Proofs::<T>::insert(
                &id,
                &claim,
            );

            Self::deposit_event(Event::ClaimCreated(id, claim));

            Ok(().into())
        }
    }
}
```

在 `./pallet/exam/Cargo.toml` 文件中修改如下：
```toml
[package]
name = "pallet-exam"
# ...... 省略
# 其他的可以不用修改
```

在 `./runtime/Cargo.toml` 文件中添加如下代码：
```toml
#...... 省略

# Local Dependencies
pallet-template = { version = "4.0.0-dev", default-features = false, path = "../pallets/template" }
pallet-exam = { version = "4.0.0-dev", default-features = false, path = "../pallets/exam" }     # 新加此行

#....... 省略

[features]
default = ["std"]
std = [
	#...... 省略
	"pallet-template/std",
	#...... 省略
	"pallet-exam/std",  #新加此行
]
```

在 `./` 目录下执行 `cargo build --release` 
```sh
~/substrate-node-template# pwd
/root/substrate-node-template
~/substrate-node-template# cargo build --release
```

运行：
```sh
./target/release/node-template --dev
```

在浏览器中登录 `https://polkadot.js.org/apps` ，可见有了我们添加的 `examPallet` 以及 `createClaim` 。

![](attachments/Pasted%20image%2020230129150224.png)

如此，前端页面（这个网页）就和后端交互上了。