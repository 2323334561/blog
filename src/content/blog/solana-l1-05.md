---
title: 社交项目
pubDatetime: 2024-12-21T11:49:00+08:00
author: shushunie
slug: solana-l1-05
description: ""
---

## 合约代码

### 项目初始化

与 [SPLToken 合约创建](../solana-l1-04) 中的初始化步骤类似

### 状态管理

合约程序的状态管理相关代码在 state.rs 中编写

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::pubkey::Pubkey;

#[derive(BorshDeserialize, BorshSerialize, Debug, Clone)]
pub struct UserProfile {
    pub data_len: u16,
    pub follows: Vec<Pubkey>,
}

#[derive(BorshDeserialize, BorshSerialize, Debug, Clone)]
pub struct UserPost {
    pub post_count: u64,
}

#[derive(BorshDeserialize, BorshSerialize, Debug, Clone)]
pub struct Post {
    pub content: String,
    pub timestamp: u64,
}

impl UserProfile {
    pub fn new() -> Self {
        Self {
            data_len: 0,
            follows: Vec::new(),
        }
    }

    pub fn follow(&mut self, user_to_follow: Pubkey) {
        self.follows.push(user_to_follow);
        self.data_len = self.follows.len() as u16 ;
    }

    pub fn unfollow(&mut self, user_to_unfollow: Pubkey) {
        self.follows.retain(|&pubkey| pubkey != user_to_unfollow);
        self.data_len = self.follows.len() as u16 ;
    }
}

impl UserPost {
    pub fn new() -> Self {
        Self { post_count: 0 }
    }

    pub fn add_post(&mut self) {
        self.post_count += 1;
    }

    pub fn get_count(&self) -> u64 {
        self.post_count
    }
}

impl Post {
    pub fn new(content: String, timestamp: u64) -> Self {
        Self {
            content,
            timestamp
        }
    }
}
```

borsh 的作用是对数据结构进行序列化，这种格式最终会将所有变量类型格式化为字节数组（[u8]）

### 初始化账户

合约程序主要的逻辑代码在 processor.rs 中，后文中的代码在未标注文件的情况下默认都在该文件下

#### 解析输入账户列表

```rust
fn init_user(
    seed_type: String,
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let account_iter = &mut accounts.iter();

    let user_account = next_account_info(account_iter)?;
    let pda_account = next_account_info(account_iter)?;
    let system_program = next_account_info(account_iter)?;
}
```

#### 处理种子

```rust
...
let seed: &str = match seed_type.as_str() {
    "profile" => "profile",
    "post" => "post",
    _ => return Err(ProgramError::InvalidArgument),
};
msg!("种子：{seed}");
```

这一步的目的是防止客户端输入预期之外的种子

#### PDA 生成与验证

```rust
...
let (pda, bump_seed) =
    ::find_program_address(&[user_account.key.as_ref(), seed.as_bytes()], program_id);
msg!("pda：{pda}");
msg!("pda_account：{pda_account}");

if pda != pda_account.key.clone() {
    return Err(ProgramError::InvalidArgument);
}
```

`find_program_address` 用于生成 程序派生地址（PDA），接受两个参数

    * seeds: &[&[u8]]
    * program_id: &Pubkey

种子的类型是多个字节数组切片(&[u8])，`user_account.key` 的类型是 `PubKey`，`as_ref()` 可以将 `Pubkey` 转换为 `&[u8]`。`as_bytes()` 同理，可以将字符串转为 `&[u8]`

程序派生地址（PDA） 地址作为数据存储地址使用，可以看作键值对中的 key

![alt text](../../assets/images/solana-l1-05/pda.svg)

具体用哪些字段作为种子，应当根据具体的业务逻辑来确定。社交项目中，PDA 账户用于存储

    1. 用户数据（关注的用户）
    2. 用户帖子数据（用户发布的帖子数量）
    3. 单个帖子数据（帖子的具体内容）

特定的一组种子会形成唯一的 PDA，而 PDA 作为 key 需要和存储的数据一一对应，所以用哪些字段作为种子来生成 pda 只要满足唯一性即可。例如用于存储用户数据的 PDA 需要满足不同用户生成的 PDA 各不相同，因此可以直接使用 `user_account.key` 作为种子。用户帖子数据也可以基于 `user_account.key` 生成 PDA，于是为了区分用户数据和用户帖子数据，我们添加一个 seed 字段。即用户数据的种子采用 `user_account.key + profile`，用户帖子数据的种子采用 `user_account.key + post`

#### 计算占用空间

```rust
const MAX_FOLLOWER_SIZE: u16 = 200;
const PUBKEY_SIZE: usize = 32;
const USER_PROFILE_SIZE: usize = 6;
const U16_SIZE: usize = 2;

const USER_POST_SIZE: usize = 8;

fn init_user() {
    ...
    let space = match seed_type.as_str() {
        "profile" => computer_profile_number(MAX_FOLLOWER_SIZE),
        "post" => USER_POST_SIZE,
        _ => return Err(ProgramError::InvalidArgument),
    };
    let rent = Rent::get()?;
    let lamports = rent.minimum_balance(space);
}

fn computer_profile_number(pubkey_count: u16) -> usize {
    USER_PROFILE_SIZE + pubkey_count as usize * PUBKEY_SIZE
}
```

实例 user_profile 序列化后占用的大小，可以直接通过方法获取。例如：`borsh::to_vec(&user_profile).unwrap().len();`

lamports 参数表示在创建账户时转入的余额，一般通过 Solana 的租金模块 `Rent::get()` 计算租金所需的最低余额。`Rent::get()` 用来获取网络中租金相关的全局配置，返回一个 Rent 对象。`minimum_balance` 是 Rent 对象的一个方法，用于计算指定大小的账户满足 租金豁免（Rent Exemption）所需要的最低租金。租金豁免是一种机制，当账户满足一定条件之后，不需要定期支付租金。

#### 执行指令

```rust
...
let create_account_ix = system_instruction::create_account(
    user_account.key,
    &pda,
    lamports,
    space as u64,
    program_id,
);
  
let account_infos = &[
    user_account.clone(),
    pda_account.clone(),
    system_program.clone(),
];

let signer_seeds: &[&[&[u8]]] =
    &[&[user_account.key.as_ref(), seed.as_bytes(), &[bump_seed]]];
  
invoke_signed(&create_account_ix, account_infos, signer_seeds);
```

#### 存储初始数据

```rust
...
match seed_type.as_str() {
    "profile" => {
        let user_profile = UserProfile::new();
        user_profile.serialize(&mut *pda_account.try_borrow_mut_data()?)?;
    }
    "post" => {
        let user_post = UserPost::new();
        user_post.serialize(&mut *pda_account.try_borrow_mut_data()?)?;
    }
    _ => return Err(ProgramError::InvalidArgument),
};
```

serialize 通过实例调用，接收一个 字节数组 `&[u8]`，将实例序列化后的结果写入参数中。需要注意的是参数类型比较麻烦，有以下两种写法

```rust
user_post.serialize(&mut *pda_account.try_borrow_mut_data()?)?;

user_profile.serialize(&mut &mut pda_account.data.borrow_mut()[..]);
```

### 关注用户

#### 解析输入账户列表

```rust
fn follow_user(accounts: &[AccountInfo], user_to_follow: Pubkey) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let user_account_info = next_account_info(account_info_iter)?;
}
```

#### 计算占用空间

```rust
fn follow_user() {
    ...
    let size;
    {
        let data = pda_account.data.borrow();
        let follow_count_data = &data[..U16_SIZE];
        let follow_count = bytes_to_u16(follow_count_data).unwrap();
        size = computer_profile_number(follow_count);
    }
}

fn bytes_to_u16(bytes: &[u8]) -> Option<u16> {
    if bytes.len() != 2 {
        return None; // 确保输入是16字节
    }
    let mut array = [0u8; 2];
    array.copy_from_slice(bytes); // 将切片复制到固定大小的数组
    Some(u16::from_le_bytes(array)) // 或者使用 from_be_bytes 进行大端序转换
}
```

同一个账户可能会被多个程序处理，data 无法在编译时进行借用规则的检查，只能用 RefCell 推迟到运行时检查。所以这里需要使用 `borrow()` 来获取借用。

Rust 借用规则要求只能同时存在一个可变借用或者多个不可变借用。违反这个规则会导致代码运行出现错误，所以 Rust 直接在编译阶段阻止错误的发生。`user_account_info.data` 使用了 RefCell，这导致编译截阶段的借用规则检查失效，但并不代表可以无视规则。我们需要人工检查代码是否符合借用规则，否则代码运行可能会出现错误。在当前函数中，后续代码需要修改用户数据，所以使用 `borrow_mut` 拿到了可变借用。同时计算空间部分的代码用到了不可变借用，这违反了借用规则。对于这种情况，我们使用大括号将空间计算部分的代码作用域隔离。当 data 离开作用域时，已经被释放了，可以避免可变借用和不可变借用同时存在的问题。

`copy_from_slice` 是标准库的方法，它的作用是将一个切片中的元素复制到另一个切片。`from_le_bytes` 也是一个标准库方法，用于将字节数组 &[u8]（小端序）解释为一个 u16 整数值。`from_le_bytes()` 接收的参数类型是 `[u8; {const}]`，而切片是 `&[u8]` 类型。所以这里先创建一个 `[u8; 2]` 类型的空数组，再将 bytes 的内容复制进去以确保类型正确。

#### 更新数据

```rust
...
let mut user_profile : UserProfile = UserProfile::try_from_slice(&user_account_info.data.borrow()[..size])?;

user_profile.follow(user_to_follow);

user_profile.serialize(&mut *user_account_info.try_borrow_mut_data()?)?;
Ok(())
```

### 取关用户

#### 解析输入账户列表

```rust
fn unfollow_user(user_to_unfollow: Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let user_account = next_account_info(account_iter)?;
}
```
#### 计算占用空间

```rust
...
let size;
{
    let follow_count_data = &user_account.data.borrow()[..U16_SIZE];
    let follow_count = bytes_to_u16(follow_count_data).unwrap();
    size = computer_profile_number(follow_count);
}
```
#### 更新数据

```rust
...
let user_profile_data = &user_account.data.borrow()[..size];
let mut user_profile = UserProfile::try_from_slice(user_profile_data)?;
user_profile.unfollow(user_to_unfollow);

Ok(())
```

### 查询关注

查询关注的就是查询账户数据，代码实现比较简单。实际上这个功能并不需要在合约内实现，因为可以通过其他工具（例如 Solana CLI）直接查询账户信息

```rust
fn query_follows(accounts: &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let pda_account = next_account_info(account_iter)?;

    let user_profile = try_from_slice_unchecked::<UserProfile>(&pda_account.data.borrow());

    msg!("用户资料：{:?}", user_profile);

    Ok(())
}
```

### 发帖

#### 解析输入账户列表

```rust
fn post_content(
    content: String,
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let user_account = next_account_info(account_iter)?;
    let user_post_pda_account = next_account_info(account_iter)?;
    let post_pda_account = next_account_info(account_iter)?;
    let system_program = next_account_info(account_iter)?;
}
```

#### 更新状态

```rust
...
let mut user_post = try_from_slice_unchecked::<UserPost>(&user_post_pda_account.data.borrow())?;
user_post.add_post();
user_post.serialize(&mut *user_post_pda_account.try_borrow_mut_data()?);
```

#### 创建帖子

```rust
...
let clock = Clock::get()?;
let timestamp = clock.unix_timestamp as u64;
let post = Post::new(content, timestamp);
```

#### 生成 post pda

```rust
...
let post_index = user_post.get_count();
let (post_pda, bump) = Pubkey::find_program_address(
    &[
        user_account.key.as_ref(),
        "post".as_bytes(),
        &[post_index as u8],
    ],
    program_id,
);
```

#### 计算占用空间和初始资金

```rust
let post_space = to_vec(&post).unwrap().len();
let rent = Rent::get()?;
let lamports = rent.minimum_balance(post_space);
```

#### 初始化 post 账户

```rust
let create_account_instruction = system_instruction::create_account(
    user_account.key,
    &post_pda,
    lamports,
    post_space as u64,
    program_id,
);

invoke_signed(
    &create_account_instruction,
    &[
        user_account.clone(),
        post_pda_account.clone(),
        system_program.clone(),
    ],
    &[&[
        user_account.key.as_ref(),
        "post".to_string().as_bytes(),
        &[post_index as u8],
        &[bump],
    ]],
)?;
```

#### 写入数据

```rust
post.serialize(&mut *post_pda_account.try_borrow_mut_data()?)?;
msg!("创建帖子成功：{:?}", { post });

Ok(())
```

### 查询帖子

```rust
fn query_post(accounts: &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let user_post_pda_account = next_account_info(account_iter)?;
    let post_pda_account = next_account_info(account_iter)?;

    let user_post = try_from_slice_unchecked::<UserPost>(&user_post_pda_account.data.borrow());
    let post = try_from_slice_unchecked::<Post>(&post_pda_account.data.borrow());

    msg!("用户帖子数：{:?}", user_post);
    msg!("帖子内容：{:?}", post);
    

    Ok(())
}
```

查询帖子和查询关注的思路是一样的，只是简单的将账户数据读取并打印

## 客户端代码

### 初始化部分

```rust
pub struct SocialClient {
    rpc_client: RpcClient,
    program_id: Pubkey,
}

impl SocialClient {
    pub fn new(rpc_url: &str, program_id: Pubkey) -> Self {
        let rpc_client = RpcClient::new(rpc_url);

        Self {
            rpc_client,
            program_id,
        }
    }
}

fn test() -> Result<(), Box<dyn std::error::Error>> {
    let program_id = Pubkey::from_str("AocjgEc7Uk6n8s17p38ueEqmEWcFvB4RxskZrYNRQRJt")?;
    let user_1_keypair = read_keypair_file("/Users/xxx/.config/solana/id1.json")?;
    let user_2_keypair = read_keypair_file("/Users/xxx/.config/solana/id2.json")?;
    let social_clinet = SocialClient::new("http://127.0.0.1:8899", program_id);
}
```

### 公用函数

#### 生成 pda

```rust
fn get_pda(seeds: &[&[u8]], program_id: &Pubkey) -> Pubkey {
    let (pda, _bump) = Pubkey::find_program_address(seeds, program_id);
    println!("pda 地址: {pda}");

    pda
}
```

生成 pda 的方法和合约内一致，都是使用 `Pubkey::find_program_address`。

#### 发送指令

```rust
pub fn send_instruction(
    &self,
    instructions: &[Instruction],
    payer: &Keypair,
) -> Result<(), Box<dyn std::error::Error>> {
    let recent_blockhash = self.rpc_client.get_latest_blockhash()?;

    let tx = Transaction::new_signed_with_payer(
        instructions,
        Some(&payer.pubkey()),
        &[payer],
        recent_blockhash,
    );

    println!("发送交易");
    match self.rpc_client.send_and_confirm_transaction(&tx) {
        Ok(_) => {
            println!("交易成功");
        }
        Err(e) => {
            println!("交易失败: {:?}", e);
        }
    };

    Ok(())
}
```

发送指令的代码比较长而且逻辑都是一样的，这里封装一个函数避免重复

### 初始化账号

```rust
impl SocialClient {
    ...

    pub fn init_user(&self, user_keypair: &Keypair, seed_type: String) {
        let pda = get_pda(
            &[user_keypair.pubkey().as_ref(), seed_type.as_bytes()],
            &self.program_id,
        );

        let init_user_instruction = Instruction::new_with_borsh(
            self.program_id,
            &SocialInstruction::InitUser { seed_type },
            vec![
                AccountMeta::new(user_keypair.pubkey(), true),
                AccountMeta::new(pda, false),
                AccountMeta::new(system_program::id(), false),
            ],
        );

        self.send_instruction(&[init_user_instruction], user_keypair);
    }
}
```

使用 `new_with_borsh` 函数创建指令可以省去序列化的步骤，简化代码

客户端调用合约都是一个套路，先创建对应的账户，然后创建并发送命令就可以了。可以看到 `init_user` 以及后面的方法都是类似写法

### 关注用户

```rust
pub fn follow_user(&self, user: &Keypair, user_to_follow: &Keypair) {
    let pda = get_pda(
        &[user.pubkey().as_ref(), USER_PROFILE_SEED.as_bytes()],
        &self.program_id,
    );

    let instruction = Instruction::new_with_borsh(
        self.program_id,
        &SocialInstruction::FollowUser {
            user_to_follow: user_to_follow.pubkey(),
        },
        vec![AccountMeta::new(pda, false)],
    );

    self.send_instruction(&[instruction], user);
}
```

### 取关用户

```rust
pub fn unfollow_user(&self, user: &Keypair, user_to_unfollow: &Keypair) {
    let pda = get_pda(
        &[user.pubkey().as_ref(), USER_PROFILE_SEED.as_bytes()],
        &self.program_id,
    );

    let instruction = Instruction::new_with_borsh(
        self.program_id,
        &SocialInstruction::UnfollowUser {
            user_to_unfollow: user_to_unfollow.pubkey(),
        },
        vec![AccountMeta::new(pda, false)],
    );

    self.send_instruction(&[instruction], user);
}
```

### 查询关注

```rust
pub fn query_follows(&self, user_keypair: &Keypair) {
    let pda = get_pda(
        &[user_keypair.pubkey().as_ref(), USER_PROFILE_SEED.as_bytes()],
        &self.program_id,
    );

    let instruction = Instruction::new_with_borsh(
        self.program_id,
        &SocialInstruction::QueryFollows,
        vec![AccountMeta::new(pda, false)],
    );

    self.send_instruction(&[instruction], user_keypair);
}
```

### 发帖

```rust
pub fn post_content(&self, user_keypair: &Keypair, content: String, id: u64) {
    let user_post_pda = get_pda(
        &[user_keypair.pubkey().as_ref(), USER_POST_SEED.as_bytes()],
        &self.program_id,
    );

    let post_pda = get_pda(
        &[
            user_keypair.pubkey().as_ref(),
            USER_POST_SEED.as_bytes(),
            &[id as u8],
        ],
        &self.program_id,
    );

    let instruction = Instruction::new_with_borsh(
        self.program_id,
        &SocialInstruction::PostContent { content },
        vec![
            AccountMeta::new(user_keypair.pubkey(), true),
            AccountMeta::new(user_post_pda, false),
            AccountMeta::new(post_pda, false),
            AccountMeta::new(system_program::id(), false),
        ],
    );

    self.send_instruction(&[instruction], user_keypair);
}
```

### 查询帖子

```rust
pub fn query_post(&self, user_keypair: &Keypair, id: u64) {
    let user_post_pda = get_pda(
        &[user_keypair.pubkey().as_ref(), USER_POST_SEED.as_bytes()],
        &self.program_id,
    );

    let post_pda = get_pda(
        &[
            user_keypair.pubkey().as_ref(),
            USER_POST_SEED.as_bytes(),
            &[id as u8],
        ],
        &self.program_id,
    );

    let instruction = Instruction::new_with_borsh(
        self.program_id,
        &SocialInstruction::QueryPosts,
        vec![
            AccountMeta::new(user_post_pda, false),
            AccountMeta::new(post_pda, false),
        ],
    );

    self.send_instruction(&[instruction], user_keypair);
}
```



