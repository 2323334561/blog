---
title: 社交应用：用户发帖 & 点赞
pubDatetime: 2025-1-6T17:09:00+08:00
author: shushunie
slug: anchor-04
description: "完成发帖和点赞功能"
---

## 发帖

### 指令代码

```rust
pub fn create_post(ctx: Context<CreatePost>, content: String) -> Result<()> {
    let profile = &mut ctx.accounts.profile;
    profile.post_count += 1;

    let post = Post::new(content, ctx.accounts.authority.key());
    ctx.accounts.post.set_inner(post);

    Ok(())
}
```

该指令接收帖子内容 `content`，并使用 `set_inner()` 将新创建的 `Post` 结构体保存到账户中

### 账户信息

```rust
#[derive(Accounts)]
pub struct CreatePost<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Post::INIT_SPACE,
        seeds = [
            Post::SEED_PREFIX.as_bytes(),
            authority.key.as_ref(),
            (profile.post_count + 1).to_string().as_ref(),
        ],
        bump
    )]
    pub post: Account<'info, Post>,

    #[account(
        mut,
        seeds = [
            UserProfile::SEED_PREFIX.as_bytes(),
            authority.key().as_ref()
        ],
        bump
    )]
    pub profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

`post` 账户是一个 pda，用于存储帖子内容

- `init`：标记该账户需要初始化
- `payer = authority`：用户支付创建该账户的费用
- `space = 8 + Post::INIT_SPACE`：指定创建该账户所需的空间大小
- `seeds` 中 `profile.post_count + 1` 是帖子计数器，用于确保每个帖子都有一个唯一的标识符

`profile` 是之前创建的用户账户

### `Post` 结构体

```rust
use anchor_lang::prelude::*;

#[account]
#[derive(InitSpace)]
pub struct Post {
    #[max_len(50)]
    pub content: String,

    pub like_count: u8,

    pub author: Pubkey,
}

impl Post {
    pub const SEED_PREFIX: &'static str = "post";

    pub fn new(content: String, author: Pubkey) -> Self {
        Post {
            content,
            like_count: 0,
            author,
        }
    }
}
```

一个简单的 `Post` 结构体，包含帖子内容、点赞数和作者公钥

---

## 点赞

### 指令代码

```rust
pub fn create_like(ctx: Context<CreateLike>) -> Result<()> {
    let profile = &ctx.accounts.profile;

    let post = &mut ctx.accounts.post;
    post.like_count += 1;

    let like = Like::new(profile.key(), post.key());
    ctx.accounts.like.set_inner(like);

    Ok(())
}
```

逻辑非常简单，先获取 `post` 账户中存储的关注数量，然后将计数 + 1。创建一个新的 `Like` 账户来记录这次点赞。用到的方法和 `create_post` 类似

### 账户信息

```rust
#[derive(Accounts)]
pub struct CreateLike<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Like::INIT_SPACE,
        seeds = [
            Like::SEED_PREFIX.as_bytes(),
            post.key().as_ref(),
            profile.key().as_ref()
        ],
        bump
    )]
    pub like: Account<'info, Like>,

    #[account(mut)]
    pub post: Account<'info, Post>,

    pub profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

相较于 `create_post`，`create_like` 指令只是多了一个 `post` 账户，其他方面区别不大

### `Like` 结构体

```rust
use anchor_lang::prelude::*;

#[account]
#[derive(InitSpace)]
pub struct Like {
    pub profile_pda: Pubkey,

    pub post_pda: Pubkey,
}

impl Like {
    pub const SEED_PREFIX: &'static str = "like";

    pub fn new(profile_pda: Pubkey, post_pda: Pubkey) -> Self {
        Like {
            profile_pda,
            post_pda,
        }
    }
}
```

---

## 前端接入

### 创建帖子

```typescript
export async function createPost(wallet: anchor.Wallet, content: string) {
  const [profile_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("profile"), wallet.publicKey.toBuffer()],
    program.programId
  );

  const profile = await program.account.userProfile.fetch(profile_pda);

  const [post_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [
      Buffer.from("post"),
      wallet.publicKey.toBuffer(),
      Buffer.from(`${profile.postCount + 1}`),
    ],
    program.programId
  );

  const result = program.methods
    .createPost(content)
    .accounts({
      post: post_pda,
    })
    .rpc();

  return result;
}
```

和创建 profile 账户不同，post 账户的种子用到了动态数据（`profile.post_count + 1`）。anchor 是不会查询账户数据的，所以 post 账户的 pda 没办法自动生成，需要前端手动获取到 pda 地址，再传给合约

### 关注帖子

```typescript
export async function createLike(
    wallet: anchor.Wallet,
    post_pda: anchor.web3.PublicKey
) {
    const [profile_pda] = anchor.web3.PublicKey.findProgramAddressSync(
        [Buffer.from("profile"), wallet.publicKey.toBuffer()],
        program.programId
    );

    const { author } = await program.account.post.fetch(post_pda);

    const result = program.methods
        .createLike()
        .accounts({
            post: post_pda,
            profile: profile_pda,
        })
        .rpc();

    return result;
}
```