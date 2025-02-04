---
title: 质押合约：质押逻辑
pubDatetime: 2025-1-13T10:19:00+08:00
author: shushunie
slug: anchor-08
description: ""
---

## 创建 NFT

创建 NFT 的过程和创建 Token 类似，不过这次我们换一种写法：使用 `CpiBuilder` 来完成 cpi 调用

```rust
CreateV1CpiBuilder::new(ctx.accounts.token_metadata_program.as_ref())
    .mint(ctx.accounts.nft_mint_account.as_ref(), true)
    .metadata(ctx.accounts.nft_metadata_account.as_ref())
    .authority(ctx.accounts.nft_mint_account.as_ref())
    .update_authority(ctx.accounts.nft_mint_account.as_ref(), true)
    .payer(ctx.accounts.authority.as_ref())
    .master_edition(Some(ctx.accounts.master_edition_account.as_ref()))
    .seller_fee_basis_points(0)
    .sysvar_instructions(ctx.accounts.rent.as_ref())
    .system_program(ctx.accounts.system_program.as_ref())
    .spl_token_program(Some(ctx.accounts.token_program.as_ref()))
    .name("lcr_nft_hhh".to_string())
    .symbol("LNTFS".to_string())
    .uri("https://note-public-img.oss-cn-beijing.aliyuncs.com/nya/nya.json".to_string())
    .print_supply(PrintSupply::Zero)
    .invoke_signed(signers_seeds)?;
```

可以看到 `CpiBuilder` 的方式是通过链式调用来传递参数的，和使用 `CpiContext::new()` 代码相比看起来更整齐一些，最终效果没有什么区别

- **.master_edition()**

  用于指定 Master Edition account。该账户作为 Printable NFTs（可打印 NFT） 的主版本账户，存储一些数据，例如：

  - supply（当前的供应量）
  - max_supply（最大供应量）

- **.seller_fee_basis_points()**

  NFT 的版税费率，单位是百分之 1。当 NFT 被转售（比如通过二级市场交易）时，这个版税会自动计算并支付给指定的收益账户。版税的具体处理由 NFT 元数据和支持 mpl-token-metadata 标准的市场程序实现

- **.sysvar_instructions()**

  用于传递系统变量

- **.symbol()**

  设置 NFT 的符号。符号类似于股票市场中的股票代码（如 “AAPL” 表示 Apple，“TSLA” 表示 Tesla）。即使两个代币名称相同，符号也可以用来区分它们

- **.uri()**

  一个链接，这个链接通常指向一个 JSON 文件，描述了 NFT 的具体信息

- **.print_supply()**

  设置 Master Edition account 中的 max_supply 字段，参数类型为 `PrintSupply` 枚举

  ```rust
  pub enum PrintSupply {
      Zero,
      Limited(u64),
      Unlimited,
  }
  ```

## Mint NFT

```rust
  pub fn mint_nft(ctx: Context<MintNFT>, id: String) -> Result<()> {
      let signers_seeds: &[&[&[u8]]] = &[&[b"lcr_NFT", id.as_bytes(), &[ctx.bumps.nft_mint_account]]];

      MintV1CpiBuilder::new(ctx.accounts.token_metadata_program.as_ref())
          .token(ctx.accounts.associated_token_account.as_ref())
          .metadata(ctx.accounts.nft_metadata_account.as_ref())
          .master_edition(Some(ctx.accounts.master_edition_account.as_ref()))
          .mint(ctx.accounts.nft_mint_account.as_ref())
          .authority(ctx.accounts.nft_mint_account.as_ref())
          .payer(ctx.accounts.authority.as_ref())
          .system_program(ctx.accounts.system_program.as_ref())
          .sysvar_instructions(ctx.accounts.rent.as_ref())
          .spl_token_program(ctx.accounts.token_program.as_ref())
          .spl_ata_program(ctx.accounts.associated_token_program.as_ref())
          .amount(1)
          .invoke_signed(signers_seeds)?;

      Ok(())
  }
```

同样使用 `CpiBuilder` 来完成，和创建 NFT 的代码类似，这里就不再赘述了

## 质押 NFT

### 创建接收 NFT 的账户

首先需要为每一个 NFT 创建一个 PDA 账户 stake_info 来存储质押关系信息，例如：

- 质押人
- 质押的 NFT
- 质押时间

**slot**
slot（插槽）是 Solana 网络中的基本时间单位，当前设置为 400 毫秒。每一个 slot 都会被分配一个 leader，他们负责在被指定的 slot 时间内计算出新的区块。如果 leader 并没有在规定的时间内生成区块，网络会直接移动到下一个 slot，为其他的 validator 提供出块的机会。这可以让 Solana 网络保持高吞吐量

**epoch**
Solana 中的时间可以用 epoch（纪元）表示。纪元是 Solana 中是一个较长的时间段，每个纪元约由 432,000 个 slot 组成。在每个 slot 为 400 毫秒的情况下，加起来刚好是 48 小时。然而在现实世界中，有些区块可能因为错误无法产生，再加上网络延迟等问题，一个 epoch 大概是 2～3 天

在一个 epoch 内，验证者和其他网络参与者有机会质押或取消质押其 SOL 代币，以此来调整各节点对 Solana 的共识机制 的参与程度。每个 epoch 结束后，网络都会向符合条件的参与者分配奖励，所以 Solana 网络运行效率越高，越接近理想的 48 小时，就会更加频繁的发放奖励。这也为用户维持网络稳定的行为提供了反馈

```rust
#[account]
#[derive(InitSpace)]
pub struct StakeInfo {
    pub staker: Pubkey,
    pub nft_mint_account: Pubkey,
    pub stake_on: u64,
}

impl StakeInfo {
    pub fn new(staker: Pubkey, nft_mint_account: Pubkey) -> Self {
        let clock = Clock::get().unwrap();
        let stake_on = clock.epoch;

        Self {
            staker,
            nft_mint_account,
            stake_on,
        }
    }
}
```

在合约逻辑开头处设置 `stake_info` 账户的数据，记录质押关系信息

```rust
pub fn stake_nft(ctx: Context<StakeNFT>) -> Result<()> {
    let stake_info = StakeInfo::new(
        ctx.accounts.authority.key(),
        ctx.accounts.nft_mint_account.key(),
    );
    ctx.accounts.stake_info_account.set_inner(stake_info);
}

#[derive(Accounts)]
pub struct StakeNFT<'info> {
    #[account(
        init_if_needed,
        payer = authority,
        space = StakeInfo::INIT_SPACE,
    )]
    pub stake_info_account: Account<'info, StakeInfo>,

    #[account(mut)]
    pub nft_mint_account: AccountInfo<'info>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

其次还需要一个 `associated_token_account` 来接收用户质押的 NFT。这个 ata 账户由 `nft + stake_info` 作为种子，这样完成转账后 `stake_info` 账户就拥有了用户的 NFT

```rust
#[account(
    init_if_needed,
    payer = authority,
    associated_token::authority = authority,
    associated_token::mint = nft_mint_account,
)]
pub associated_token_account: Account<'info, TokenAccount>,
```

### 转移 NFT

创建好接收账号后就可以用户的 NFT 转移到我们的账号下。转移 NFT 的逻辑和转移代币是一样的，使用 transfer 进行转账即可。对于 NFT 来说，转移的数量是 1

```rust
pub fn stake_nft(ctx: Context<StakeNFT>) -> Result<()> {
    ...
    transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.nft_associated_token_account.to_account_info(),
                to: ctx.accounts.program_receipt_nft_ata.to_account_info(), // todo
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        1,
    )?;
}
```

### 为用户 mint 代币

在收到用户质押的 NFT 后，调用 `mint_to` 给用户铸造一些代币。amount 一定要根据创建代币时设置的精度来确定，以避免代币数量不符合预期

```rust
pub fn stake_nft(ctx: Context<StakeNFT>) -> Result<()> {
    ...
    let signer_seeds: &[&[&[u8]]] = &[&[
        IbuidlToken::SEED_PREFIX.as_bytes(),
        &[ctx.bumps.token_mint_account],
    ]];

    mint_to(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.token_mint_account.to_account_info(),
                to: ctx.accounts.associated_token_account.to_account_info(),
                authority: ctx.accounts.token_mint_account.to_account_info(),
            },
        )
        .with_signer(signer_seeds),
        10000,
    )?;

    Ok(());
}
```

