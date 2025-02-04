---
title: 质押合约：解质押逻辑
pubDatetime: 2025-1-15T10:28:17.399Z
author: shushunie
slug: anchor-09
description: ""
---

## 检查质押关系

### 自定义错误

定义一个错误，在检查不通过时抛出

```rust
use anchor_lang::prelude::*;

#[error_code]
pub enum UnstakeError {
    #[msg("can not unstake")]
    NoAuthority,
}
```

### 检查铸造账户

```rust
pub fn unstake_nft(ctx: Context<UnstakeNft>) -> Result<()> {
    require!(
      &ctx.accounts.nft_mint_account.key()
          == &ctx.accounts.stake_info_account.nft_mint_account.key(),
      UnstakeError::NoAuthority
    );
}
```

### 检查质押人

```rust
pub fn unstake_nft(ctx: Context<UnstakeNft>) -> Result<()> {
    ...
    require!(
        &ctx.accounts.nft_mint_account.key()
            == &ctx.accounts.stake_info_account.nft_mint_account.key(),
        UnstakeError::NoAuthority
    );
}
```

## 转移 NFT

调用 `transfer` 进行交易

```rust
pub fn unstake_nft(ctx: Context<UnstakeNft>) -> Result<()> {
    ...
    let nft_mint_account = &ctx.accounts.nft_mint_account.key();
    let signer_seeds: &[&[&[u8]]] = &[&[
        b"stake_v1",
        nft_mint_account.as_ref(),
        &[ctx.bumps.stake_info_account],
    ]];

    transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.program_nft_ata.to_account_info(),
                to: ctx.accounts.user_receipt_nft_ata.to_account_info(),
                authority: ctx.accounts.stake_info_account.to_account_info(),
            },
        )
        .with_signer(signer_seeds),
        1,
    )?;
}
```

## 结算 Token

### 计算收益

根据经过的 epoch 数量来计算用户的收益，可以将方法封装在 StakeInfo 中

直接计算收益大概率不会生效，这是因为默认的 epoch 时间较长。如果想要测试出效果，可以修改本地的 epoch 时间

```rust
impl StakeInfo {
    ...
    pub fn salvage_value(&self, amount: u64) -> u64 {
        let clock = Clock::get().unwrap();
        let now = clock.epoch;

        let p = max(0, (now - self.stake_on) * 2) as f64 / 100.0;
        (amount as f64 * p) as u64
    }
}
```

```rust
let amount = ctx.accounts.stake_info_account.salvage_value(100);
```

### 销毁 Token

调用 `burn` 销毁代币

```rust
pub fn unstake_nft(ctx: Context<UnstakeNft>) -> Result<()> {
    ...
    burn(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Burn {
                mint: ctx.accounts.token_mint_account.to_account_info(),
                from: ctx.accounts.user_token_ata.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        amount,
    )?;

    Ok(())
}
```
