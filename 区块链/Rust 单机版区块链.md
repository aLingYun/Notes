## 概念
### Block
我们构建一个 Block 如下：
```rust
struct Block {
    //data: String,
    data: Vec<Transaction>,
    pre_hash: String,
    time_stamp: i64,
    nonce: String,
    hash: String,
}
```
区块链是由一个一个区块构成的有序链表，每一个区块都记录了一系列交易: `data`。并且每个区块都指向前一个区块，从而形成一个链条。区块通过记录上一个区块的哈希来指向上一个区块: `pre_hash`。每个区块都有一个唯一的哈希标识，被称为区块哈希: `hash`。每个 Block 都需要一个时间戳: `time_stamp`。以及一个随机量: `nonce`，用于挖矿时，不改变交易数据的情况下改变区块的 HASH 值。
### Transaction
构建一个 Transaction 如下：
```rust
struct Transaction {
    from: String,
    to: String,
    amount: u32,
    signature: String,
}
```
Transaction 用于记录交易信息，那必然有交易的三要素：

* 付钱方：`from`
* 收钱方：`to`
* 金额：`amount`

`signature` 用于存储私钥对 Transaction 加密产生的签名。这个签名可以用公钥解密出原始数据。

### Chain
构造一个 Chain 如下：
```rust
struct Chain {
    chain: Vec<Block>,
    transaction_pool: Vec<Transaction>,
    miner_reward: u8,
    diffculty: u8,
}
```
其中包含：
* 区块链主体：`chain`
* 交易池：`transaction_pool`
* 矿工奖励：`miner_reward`
* 挖矿难度：`difficulty`

## 实战
### Code 如下
```
# Cargo.toml
[package]
name = "study"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
sha256 = "1.1.1"
rand = "0.8.5"
rust-crypto = "0.2.36"
```

```rust
// main.rs
use rand::prelude::*;
use sha256;
use std::time::{SystemTime, UNIX_EPOCH};
use crypto::ed25519::{keypair, signature, verify};

// 获取 difficult 个 0 的 String
fn get_answer(difficult: u8) -> String {
    let mut answer = "".to_string();
    for _i in 0..difficult {
        answer.push('0');
    }
    answer
}

// 获取时间戳
fn get_time_stamp() -> i64 {
    let start = SystemTime::now();
    let since_the_epoch = start
        .duration_since(UNIX_EPOCH)
        .expect("Time went backwards");
    let ns = since_the_epoch.as_secs() as i64 * 1_000_000_000_i64
        + (since_the_epoch.subsec_nanos() as f64) as i64;
    ns
}

// 将 &[u8] 转换为 String
fn u8_to_string(array: &[u8]) -> String {
    let mut s = "".to_string();
    for i in array {
        let tmp = format!("{:02X}", i);
        s.push_str(&tmp[..]);
    }
    s
}

// 将 String 转换为 [u8]
fn string_to_u8(ss: &String) -> [u8; 64] {
    let mut array_u8 = [0; 64];
    let mut i = 0_usize;
    for iter in ss.bytes() {
        if iter < 65 {
            if i % 2 == 0 {
                array_u8[i/2] = (iter - 48) * 16;
            } else {
                array_u8[i/2] += iter - 48;
            }
        } else {
            if i % 2 == 0 {
                array_u8[i/2] = (iter - 55) * 16;
            } else {
                array_u8[i/2] += iter - 55;
            }
        }
        i += 1;
    }
    array_u8
}

// transaction: 用于记录每一笔转账
#[derive(Debug, Clone)]
struct Transaction {
    from: String,
    to: String,
    amount: u32,
    signature: String,
}

impl Transaction {
    fn new(from_arg: String, to_arg: String, amount_arg: u32) -> Transaction {
        Transaction {
            from: from_arg,
            to: to_arg,
            amount: amount_arg,
            signature: "".to_string(),
        }
    }

    // 用私钥对 transaction 的 sha256 进行签名
    fn sign(&mut self, private_key: &[u8]) {
        self.signature = u8_to_string(&signature(sha256::digest(self.to_string()).as_bytes(), private_key));
    }

    // 用公钥对签名进行解密，并与 transaction 的 sha256 比较，验证 transaction 是否有效
    fn is_valid_transaction(&self) -> bool {
        if self.from == "".to_string() && self.to != "".to_string() {
            return true;
        }
        if self.signature == "".to_string() {
            return false;
        }
        //println!("is valid: {:?}", &string_to_u8(&self.from));
        return verify(sha256::digest(self.to_string()).as_bytes(), 
                      &string_to_u8(&self.from)[0..32], 
                      &string_to_u8(&self.signature));
    }
}

// 实现 to_string() 以便传入 sha256::digest() 计算整个 transaction 的 SHA256
impl ToString for Transaction {
    fn to_string(&self) -> String {
        format!("{}{}{}", self.from, self.to, self.amount)
    }
}

// Block 定义
#[derive(Debug)]
struct Block {
    //data: String,
    data: Vec<Transaction>,
    pre_hash: String,
    time_stamp: i64,
    nonce: String,
    hash: String,
}

impl Block {
    fn new(data_arg: Vec<Transaction>) -> Block {
        Block {
            data: data_arg,
            pre_hash: "".to_string(),
            time_stamp: get_time_stamp(),
            nonce: "".to_string(),
            hash: "".to_string(),
        }
    }
    // 以便传入 sha256::digest() 计算整个 block 的 SHA256
    fn to_string_for_hash(&self) -> String {
        let mut data_string = "".to_string();
        for iter in &self.data {
            data_string.push_str(&iter.to_string()[..]);
        }
        format!("{}{}{}{}", data_string, self.pre_hash, self.time_stamp, self.nonce)
    }

    // 挖矿 (计时功能不需要)
    fn mine(&mut self, difficult: u8) {
        let mut rng = thread_rng();
        let sys_time = SystemTime::now();
        loop {
            if self.hash[0..(difficult as usize)] != get_answer(difficult) {
                self.nonce = rng
                    .gen_range(0 as usize..18446744073709551615 as usize)
                    .to_string();
                self.hash = sha256::digest(self.to_string_for_hash());
            } else {
                break;
            }
        }
        println!(
            "挖矿结束，用时 {:#?} 微秒",
            sys_time.elapsed().unwrap().as_micros()
        );
    }

    // 遍历整个 transaction pool，验证有效性
    fn all_transaction_is_valid(&self) -> bool {
        for iter in &self.data {
            if !iter.is_valid_transaction() {
                println!("This is invalid transaction");
                return false;
            }
        }
        true
    }
}

// Chain 定义
#[derive(Debug)]
struct Chain {
    chain: Vec<Block>,
    transaction_pool: Vec<Transaction>,
    miner_reward: u8,
    diffculty: u8,
}

impl Chain {
    fn new() -> Chain {
        let tran = Transaction::new(
            "".to_string(), 
            "".to_string(), 
            0_u32);
        let mut blk = Block::new(vec![tran]);
        blk.hash = sha256::digest(blk.to_string_for_hash());
        Chain {
            chain: vec![blk],
            transaction_pool: vec![],
            miner_reward: 50_u8,// 矿工的奖励
            diffculty: 4_u8,    // 挖矿的难度设置，即前面有几个连续的 0
        }
    }

    // 将某个 transaction 添加到 transaction pool 中
    fn add_transaction(&mut self, tran: Transaction) {
        // 如果支付和接受方有一方没有信息，都是无效的
        if tran.from == "".to_string() || tran.to == "".to_string() {
            println!("Invalid from or to!");
            return;
        }
        // transaction 有效才会加入 pool 中
        if tran.is_valid_transaction() {
            self.transaction_pool.push(tran);
        } else {
            println!("Invalid Transaction!");
        }
    }

    // 挖整个 transaction pool
    fn mine_transaction_pool(&mut self, miner: String) {
        // 矿工奖励 transaction
        let tran = Transaction::new(
            "".to_string(),
            miner,
            self.miner_reward as u32,
        );
        // 有效 transaction，加入 pool
        if tran.is_valid_transaction() {
            self.transaction_pool.push(tran);
        }

        // 从 transaction 创建 Block，并添加到 Chain
        let blk = Block::new(self.transaction_pool.clone());
        self.add_block(blk);
        // 将 Chain 上已有的 transaction 都保存到 Block 中，
        // 并加入到 Chain 后，将 pool 清空
        self.transaction_pool = vec![];      
    }

    // 添加 block 到 Chain 上
    fn add_block(&mut self, mut blk: Block) {
        blk.pre_hash = self.chain[self.chain.len() - 1].hash.clone();
        blk.hash = sha256::digest(blk.to_string_for_hash());
        // 挖矿
        blk.mine(self.diffculty);
        // 挖完将 Block 加入到 Chain
        self.chain.push(blk);
    }

    // 验证整个 Chain 是否有效
    fn is_valid_chain(&self) -> bool {
        // 只有一个创世区块
        if self.chain.len() == 1 {
            if self.chain[0].hash != sha256::digest(self.chain[0].to_string_for_hash())
            {
                return false;
            }
            return true;
        }
        // 遍历 Chain 上所有的 Block 
        for iter in 1..self.chain.len() {
            let blk_tmp = &self.chain[iter];
            // Block 中所有 transaction 的有效性
            if !blk_tmp.all_transaction_is_valid() {
                println!("This is a invalid block!");
                return false;
            }
            // Block 整体数据的有效性
            if blk_tmp.hash != sha256::digest(blk_tmp.to_string_for_hash())
            {
                println!("数据被篡改");
                return false;
            }
            // Chain 上所有 Block 的链接关系验证
            if blk_tmp.pre_hash != self.chain[iter - 1].hash {
                println!("区块断裂");
                return false;
            }
        }
        return true;
    }
}

fn main() {
    // 生成发送发和接收方的密钥对
    let seed_string = b"qwertyuiopasdfghjklzxcvbnm012345";  
    let (private_key_s, public_key_s) = keypair(seed_string);
    // println!("public key = {:?}", u8_to_string(&public_key_s));
    // println!("private key = {:?}", u8_to_string(&private_key_s));
    let seed_string = b"012345qwertyuiopasdfghjklzxcvbnm";  
    let (_private_key_r, public_key_r) = keypair(seed_string);
    // println!("public key = {:?}", u8_to_string(&public_key_r));
    // println!("private key = {:?}", u8_to_string(&private_key_r));

    // 新建一个 Chain
    let mut chain = Chain::new();

    // 新建三笔转账
    let mut tran1 = Transaction::new(
        u8_to_string(&public_key_s),
        u8_to_string(&public_key_r), 
        10_u32
    );
    let mut tran2 = Transaction::new(
        u8_to_string(&public_key_s),
        u8_to_string(&public_key_r), 
        20_u32
    );
    let mut tran3 = Transaction::new(
        u8_to_string(&public_key_s),
        u8_to_string(&public_key_r),  
        30_u32
    );
    // 对这三笔转账进行签名
    tran1.sign(&private_key_s);
    tran2.sign(&private_key_s);
    tran3.sign(&private_key_s);
    // println!("{}", tran1.is_valid_transaction());
    // println!("{}", tran2.is_valid_transaction());
    // println!("{}", tran3.is_valid_transaction());

    // 添加到 Chain 的 transaction pool 中，等待矿工挖矿
    chain.add_transaction(tran1);
    chain.add_transaction(tran2);
    chain.add_transaction(tran3);
    // 挖整个 transaction pool
    chain.mine_transaction_pool("miner1".to_string());

    println!("{:#?}", chain);
    println!("A whole chain is valid: {}", chain.is_valid_chain());
}

```

### 运行结果如下
```bash
wlb@WIN11:~/Documents/Codes/rust/block-chain-study$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/study`
挖矿结束，用时 218443 微秒
Chain {
    chain: [
        Block {
            data: [
                Transaction {
                    from: "",
                    to: "",
                    amount: 0,
                    signature: "",
                },
            ],
            pre_hash: "",
            time_stamp: 1673464411042997400,
            nonce: "",
            hash: "e3c5dee4432a97a3b5805675e9d8a8b4753dc606470cad6399d21a6af677d1f0",
        },
        Block {
            data: [
                Transaction {
                    from: "9A45D11C17962EA80B1B7CEF03FC3C7F96C2421B094C346536D08AAA30DE9C11",
                    to: "4E2CA997BD43C83A7CB2615D82379151618F8B94EA00AF45382E113FFF8CB144",
                    amount: 10,
                    signature: "607DCECD2D9EF92D5D9CDA4364F291C7A6ECBE4A45F645F166D063F5A3006B3BA12978D76D8CC12B9838860B36FD12023DF0FD6B2639F5F1482C5007D075FE02",
                },
                Transaction {
                    from: "9A45D11C17962EA80B1B7CEF03FC3C7F96C2421B094C346536D08AAA30DE9C11",
                    to: "4E2CA997BD43C83A7CB2615D82379151618F8B94EA00AF45382E113FFF8CB144",
                    amount: 20,
                    signature: "C473BCADF5A36DC9D3C4288896F06F56BB0719E659CDE47C2B8FB99E65AD85732DD50B69996C84E37878B2A384709BFAC2900453BE1F7AEE52E3CFF163C14F08",
                },
                Transaction {
                    from: "9A45D11C17962EA80B1B7CEF03FC3C7F96C2421B094C346536D08AAA30DE9C11",
                    to: "4E2CA997BD43C83A7CB2615D82379151618F8B94EA00AF45382E113FFF8CB144",
                    amount: 30,
                    signature: "56BA16409B7669F08EDCCB1C786B5D56D1BF397956898425701945053EBFCB726FB3A31C8A39021AD390B43C1D9B1B6FBD0738C208350B2DA4D3456ABB114709",
                },
                Transaction {
                    from: "",
                    to: "miner1",
                    amount: 50,
                    signature: "",
                },
            ],
            pre_hash: "e3c5dee4432a97a3b5805675e9d8a8b4753dc606470cad6399d21a6af677d1f0",
            time_stamp: 1673464411046467000,
            nonce: "5833095891360095079",
            hash: "0000e14282c94154e8257c93f0692322b46f5a28422574362670f2c3623d264d",
        },
    ],
    transaction_pool: [],
    miner_reward: 50,
    diffculty: 4,
}
A whole chain is valid: true
```

### 篡改
在代码中如下位置添加篡改代码：
```rust
    ......
    // 挖整个 transaction pool
    chain.mine_transaction_pool("miner1".to_string());
    
    // 篡改区块中的数据
    chain.chain[1].data[1].amount = 1;

    println!("{:#?}", chain);
    println!("A whole chain is valid: {}", chain.is_valid_chain());
```

执行会发现在结尾出现如下的错误：
```bash
This is invalid transaction
This is a invalid block!
A whole chain is valid: false
```