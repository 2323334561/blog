---
title: ETF：介绍
pubDatetime: 2025-01-21T08:03:26.685Z
author: shushunie
slug: anchor-10
description: ""
---

## 概念

- **债券**

  小明借我 2w，约定每年给他 5% 的利息作为回报。不管每年公司赚了多少钱，在没有还给小明 2w 之前，小明每年能获得 20000\*0.05=1000 作为利息

- **股票**

  小明用 2w 购买公司 20% 的股权。每年可以获得公司利润的 20% 作为分红，利润高分的就多，利润少分的就少

- **证券**

  各种有价值的凭证都可以称为证券，例如：债券、股票

- **基金**

  - 广义：

    为了达成某种目的而设立的具有一定数量的资金，例如：住房公积金、社保基金

  - 狭义：

    一般指金融投资产品，即证券投资基金。用户投入一定的资金后，管理员会将收集到的资金聚集起来去做股票、债券或者其他理财产品来获得盈利，然后再返还给投资者。往往资金的管理员每年都会按百分比收取一定的管理费，和质押池类似

  - 分类

  可以按照基金资产的投资比例分类，例如：

  - 股票基金：80% 以上投资股票
  - 债券基金：80% 以上投资债券

  按运作方式分类：

  - 开放式基金：随时可以申购或赎回
  - 封闭式基金：到期前不能申购或赎回，一般会在基金名称上明确标明
  - 门槛低

  当然也可以按其他方式分类，这里不进行列举

  由于股票和债券的风险不同，因此在购买基金之前，基金构思都会要求购买者做风险评估测试，并以此来选择不同种类的基金

- **ETF**

  ETF（交易型开放式指数基金），重点在**指数基金**上。指的是基金**跟踪某个特定指数**（如标普500、纳斯达克100等）的表现来确定投资目标，而不是主动挑选个股。有以下几个优点：

  - 门槛低

    假设你看好白酒板块，但一手贵州茅台的价格非常贵，你买不起。这时候就可以购买一手「酒」 ETF，价格比贵州茅台便宜很多

  - 管理费低

    被动追踪指数，不需要基金经理主动管理，因此每年的管理费较低

  - 高透明度

    ETF的投资资讯十分透明，投资者对其持仓资讯能一目了然。相比之下，其他投资工具就未必如此。例如：管理型基金的基金经理可随时改动投资组合而不必向投资者披露基金的持仓资讯

## 创建 ETF

```rust
pub fn create_etf(ctx: Context<CreateEtf>, args: EtfArgs) -> Result<()> {
    let etf_token_mint_account_key = ctx.accounts.etf_token_mint_account.key();
    let signer_seeds: &[&[&[u8]]] = &[&[
        b"etf_v1",
        etf_token_mint_account_key.as_ref(),
        &[ctx.bumps.etf_token_info_account],
    ]];

    create_metadata_accounts_v3(
        CpiContext::new(
            ctx.accounts.token_metadata_program.to_account_info(),
            CreateMetadataAccountsV3 {
                metadata: ctx.accounts.etf_token_metadata_account.to_account_info(),
                mint: ctx.accounts.etf_token_mint_account.to_account_info(),
                mint_authority: ctx.accounts.etf_token_info_account.to_account_info(),
                payer: ctx.accounts.payer.to_account_info(),
                update_authority: ctx.accounts.etf_token_info_account.to_account_info(),
                system_program: ctx.accounts.system_program.to_account_info(),
                rent: ctx.accounts.rent.to_account_info(),
            },
        )
        .with_signer(signer_seeds),
        DataV2 {
            name: args.name.to_string(),
            symbol: args.symbol.to_string(),
            uri: args.url.to_string(),
            seller_fee_basis_points: 0,
            creators: None,
            collection: None,
            uses: None,
        },
        false,
        true,
        None,
    )?;

    let etf_token_info = args.create_etf_token_info(
        ctx.accounts.etf_token_mint_account.key(),
        ctx.accounts.payer.key(),
    );
    ctx.accounts
        .etf_token_info_account
        .set_inner(etf_token_info);

    Ok(())
}
```

```rust
#[derive(Accounts)]
#[instruction(args: EtfArgs)]
pub struct CreateEtf<'info> {
    #[account(
        init_if_needed,
        payer = payer,
        space = 8 + EtfTokenInfo::INIT_SPACE,
        seeds = [
            b"etf_v1",
            etf_token_mint_account.key().as_ref(),
        ],
        bump
    )]
    pub etf_token_info_account: Account<'info, EtfTokenInfo>,

    /// CHECK
    #[account(
        mut,
        seeds = [
            b"metadata",
            token_metadata_program.key().as_ref(),
            etf_token_mint_account.key().as_ref()
        ],
        bump,
        seeds::program = token_metadata_program.key(),
    )]
    pub etf_token_metadata_account: UncheckedAccount<'info>,

    #[account(
        init_if_needed,
        payer = payer,
        seeds = [
            b"ETF_TOKEN",
            args.symbol.as_bytes()
        ],
        bump,
        mint::decimals = 9,
        mint::authority = etf_token_info_account.key(),
    )]
    pub etf_token_mint_account: Account<'info, Mint>,

    #[account(mut)]
    pub payer: Signer<'info>,

    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub token_metadata_program: Program<'info, Metadata>,

    pub rent: Sysvar<'info, Rent>,
}
```

ETF 可以看作一类 token，和创建 token 的步骤是一样的，这里不再重复说明

## 前端部分

### 设定 assets

```rust
const assets = [
    {
        token: new anchor.web3.PublicKey(
            "Bm35wQhXhh9oDpxiYTUekFKiL7hQDxeWPWhRyU8FPZLh"
        ),
        weight: 90,
    },
    {
        token: new anchor.web3.PublicKey(
            "B7tQY7bF6u5mX6p5XTD6KPAcoHuJWi5gpMuBDfkfjttX"
        ),
        weight: 10,
    },
];

const eftArgs = {
    name: "LCR_ETF",
    symbol: "LETF",
    url: "https://note-public-img.oss-cn-beijing.aliyuncs.com/nya/nya.json",
    description: "description_test",
    assets,
};
```

为了设定 assets，我们需要传入两个 token 及其权重。权重可以随意设定（总共 100%），但 token 需要现使用 `spl-token create-token` 来创建，然后将地址填入

### 调用指令

```rust
it("create_etf", async () => {
    await program.methods.createEft(eftArgs).rpc();
});
```

没有特殊账号，所以只需要传入 eftArgs
