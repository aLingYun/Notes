这个没啥好说的，直接上代码：
```toml
[dependencies]
suppaftp = "^4.7.0"
```
如果使用了 SSL/TLS 加密了，则需要启用  `native-tls` 或者 `rustls` ，例如 `native-tls` :
```toml
[dependencies]
suppaftp = { version = "^4.7.0", features = ["native-tls"] }
```
代码示例：
```rust
use suppaftp::FtpStream;
use suppaftp::native_tls::TlsConnector;

fn main() {
    let ftp = FtpStream::connect("127.0.0.1:21").expect("connect error");
    let mut ftp = ftp.into_secure(TlsConnector::new().expect("Tls connetor new error").into(), "127.0.0.1").expect("into secure error");
    ftp.login("username", "password").expect("login error");

    ftp.set_mode(suppaftp::types::Mode::ExtendedPassive);
    ftp.cwd("/wlb").expect("cwd error");
    println!("{}", ftp.pwd().expect("pwd error"));
    let list_dir  = ftp.nlst(Some("/wlb")).expect("nlst error");
    println!("{:#?}", list_dir);

    ftp.quit().unwrap();
}
```
有一个需要注意的点，连接时有三种模式：
```rust
pub enum Mode {
    Active,
    ExtendedPassive,
    Passive,
}
```
登录上去默认是 `Passive` 模式，在这个模式下无法使用 `nlst()` 等方法。需要将其设置为 `ExtendedPassive` 模式。即
```rust
ftp.set_mode(suppaftp::types::Mode::ExtendedPassive);
```
以上代码执行结果如下：
```powershell
PS D:\Codes\rust\suppaftp> cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
     Running `target\debug\suppaftp.exe`
/wlb
[
    "/wlb/aaaa",
    "/wlb/bbbbbbbb",
    "/wlb/cccccccccc",
    "/wlb/2023-02-22-18-16-34_2.7z",
    "/wlb/2023-02-23-10-22-45_2.7z",
]
Hello, world!
PS D:\Codes\rust\suppaftp>
```
其他信息可以在 docs.rs 上查看 [FtpStream in suppaftp - Rust (docs.rs)](https://docs.rs/suppaftp/4.7.0/suppaftp/struct.FtpStream.html) 