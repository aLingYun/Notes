由于我使用的 ftp server 使用了 TLSv1.2 的加密算法，所以必须使用 ftp crate 中的 `secure` features。故而我们需要为编译做一些准备工作，否则会编译失败。
## 一、准备工作
在 ftp crate 的 build.rs 中可以发现其依赖 openssl 的版本是 `1.0.1 / 1.0.2 / 1.1.0` 这三个版本中的任意一个。
```rust
    match env::var("DEP_OPENSSL_VERSION") {
        Ok(ref v) if v == "101" => {
            println!("cargo:rustc-cfg=ossl101");
            println!("cargo:rustc-cfg=ossl10x");
        }
        Ok(ref v) if v == "102" => {
            println!("cargo:rustc-cfg=ossl102");
            println!("cargo:rustc-cfg=ossl10x");
        }
        Ok(ref v) if v == "110" => {
            println!("cargo:rustc-cfg=ossl110");
        }
        _ => panic!("Unable to detect OpenSSL version"),
    }
```
所以我们需要先编译好这三个版本中的一个，我编译的是 `1.1.0l` :
- 从 openssl 网站的 `source/old` 选择下载 `openssl-1.1.0l` 代码；并将压缩包解压；
- 下载安装 `perl` ，我用的是 5.32 版本；安装完执行 `perl -v` 查看版本，确定安装成功；
- 下载 `nasm` ，我用的是 `nasm-2.16.01-win64` ；将其解压，并把两个 exe 文件拷贝到代码解压目录；
- 然后运行 `perl configure VC-WIN64A --prefix=D:\OpenSSL\x64` ，这样配置最终会安装到 `D:\tools\OpenSSL` 目录下；
- 再执行 `nmake` ，等待执行编译结束；
- 最后用管理员权限运行 cmd，并在代码目录下执行 `nmake install` ；这一步就会将编译产物拷贝到  `D:\tools\OpenSSL` 目录下。
- 再添加几个环境变量：
```toml
OPENSSL_DIR=D:\tools\OpenSSL
OPENSSL_INCLUDE_DIR=D:\tools\OpenSSL\include
OPENSSL_LIB_DIR=D:\tools\OpenSSL\lib
```
## 二、代码编译
首先添加 ftp crate 依赖，需要使用 `secure` features:
```toml
[dependencies]
ftp = { version = "3.0.1", features = ["secure"] }
```
示例代码：
```rust
use ftp::FtpStream;
use ftp::openssl::ssl::{ SslContext, SslMethod };

fn main() {
    let ftp_stream = FtpStream::connect("127.0.0.1:21").unwrap();
    let ctx = SslContext::builder(SslMethod::tls()).unwrap().build();
    let mut ftp_stream = ftp_stream.into_secure(ctx).unwrap();
    ftp_stream.login("username", "password").unwrap();
    println!("{}", ftp_stream.pwd().unwrap());  //列出当前目录
    let _ = ftp_stream.quit();
}
```
编译执行：
```cmd
D:\Codes\rust\study>cargo run
   Compiling study v0.1.0 (D:\Codes\rust\study)
    Finished dev [unoptimized + debuginfo] target(s) in 0.56s
warning: the following packages contain code that will be rejected by a future version of Rust: winapi v0.2.8
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 12`
     Running `target\debug\study.exe`
/                                         #刚登陆在根目录

D:\Codes\rust\study>
```

**追加 Note:** ftp crate 不支持 openssl 1.1.0 以上的版本，且依赖 winapi crate 版本较低，将被丢弃。所以建议使用 suppaftp crate 而不是 ftp crate。