---
title: anchor 入门：pda 的用法
pubDatetime: 2025-1-1T10:56:00+08:00
author: shushunie
slug: anchor-02
description: ""
---

## 使用 pda 存储计数器数据

### 合约改动

counter 从之前的普通账户变成了 PDA 账户，在合约侧需要添加额外的声明

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        space = 8 + 8,
        seeds = [payer.key().as_ref()],
        bump
    )]
    pub counter: Account<'info, Counter>,

    pub system_program: Program<'info, System>,
}
```

[seeds](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/f86304a8-4e38-4733-b98d-dfdfa359c915)

[bump](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/1812cf60-c929-4e72-9b1a-cea8a4d075ee)

### 在测试代码中省略 counter

构建完成后会发现测试代码出现报错。原因是 counter 是 pda 账户，生成该账户需要的 payer 以及 programId 都是已知的。所以在调用指令时也没必要传入，anchor 会自动处理

```typescript
it("Is initialized!", async () => {
  // Add your test here.
  const tx = await program.methods
    .initialize()
    .accounts({
      payer: wallet.publicKey,
      // counter: pda,
    })
    .signers([wallet.payer])
    .rpc();
  console.log("Your transaction signature", tx);
});
```

### 添加其他种子

实际应用中可能会用到字符串+账户作为种子来生成 pda，这样字符串可以用来描述 pda 的具体作用。例如本例中我们可以使用 counter+账户地址 作为种子

```rust
#[account(
    init,
    payer = payer,
    space = 8 + 8,
    seeds = [b"counter", payer.key().as_ref()],
    bump
)]
```

```typescript
const [pda, _] = anchor.web3.PublicKey.findProgramAddressSync(
  [Buffer.from("counter"), wallet.publicKey.toBytes()],
  program.programId
);
```
