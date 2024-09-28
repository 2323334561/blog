---
title: Rust入门 第一节、环境搭建
pubDatetime: 2024-09-27T15:00:00+08:00
author: shushunie
slug: rust-l1
description: ''
---

# 基本环境搭建

### 安装 rust

通过 rustup 下载 Rust，这是一个管理 Rust 版本和相关工具的命令行工具。下载时需要联网，选择默认安装即可

`curl --proto '=https'--tlsv1.2 -sSf https://sh.rustup.rs | sh`

使用 `rustc -V` 查看版本号如果正确显示则证明安装成功。以下是一些基础命令

- `rustc -V`

- `rustup update`

- `rustup self uninstall`

- `rustup doc`

### vscode 插件推荐

- [rust-analyzer](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/00905c9e-d077-4602-9aa0-7496502d74f8) rust 语法提示

- [Even Better TOML](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/41a191df-3fe1-44cc-a87e-31741cd16574) toml 文件支持

- [crates](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/7a2793dc-7605-4178-8a36-dc063252bb29)（已弃用）[Dependi](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/d365b6e2-38dc-4bdf-9358-1d44a3dca509) rust 依赖包的可用版本提示

### hello world

编写 rust 版 hello world，文件以 .rs 结尾

```rust
fn main(){
    println!("hello world");
}
```

rust 是一种 [预编译](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/ee90c001-7e6e-4480-89d0-6c34920c1679) [静态类型](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/4c140d07-ec14-4fe0-bf53-752916e07901) 语言，编译后生成的执行文件可以在不安装 rust 的情况下直接运行

编译命令是 `rustc {FileName}`

# Cargo 使用

### 初始化项目

rust 的包管理工具是 [cargo](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/af94b5a0-1783-4c27-b630-b1247555c0ee) 和 [npm](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/16e11abe-c09e-4428-bda8-d51c9f260762) 类似，它是不需要单独下载的，rust 安装之后就可以直接使用 cargo命令，以下是一些基本命令

- `cargo new {FileName}`

- `cargo build`

- `cargo build --release`

- `cargo run`

- `cargo add`

- `cargo install`

- `cargo update`

使用 new 命令后会生成一个简单项目，其中 Cargo.toml 类似 package.json


📦 hello_cargo  
 ┣ 📂 src  
 ┃ ┗ 🦀 main.rs  
 ┣ ⚙️ .gitignore  
 ┗ 📜 Cargo.toml  

### 更新依赖

rust 的依赖称为 crate，中文翻译是箱

添加依赖可以使用 `cargo add` 命令，或者直接改 Cargo.toml 里的 dependencies 板块。使用 add 命令添加后会在 dependencies 板块显示并且生成 [Cargo.lock](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/d6a88e45-79f9-4cee-970a-1bc1198e2683) 文件。这个文件类似 [package-lock.json](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/15d9b347-9931-4d48-b0ee-45f68f8930c5)，会保存详细的依赖信息，以此来保证其他开发者能获取到完全相同的依赖从而构建出完全相同的结果

更新依赖可以使用`cargo update` 命令。它会忽略 Cargo.lock 文件，并计算出所有符合 Cargo.toml声明的最新版本。Cargo接下来会把这些版本写入Cargo.lock 文件。不过，Cargo 默认只会寻找大于 0.8.5 而小于 0.9.0的版本

例如 rand crate 发布了两个新版本，0.8.6 和 0.9.0，任运行 cargo update 时会出现如下内容提示你从0.8.5 更新到了 0.8.6。如果需要更新至 0.9 版本，则需要手动修改 Cargo.toml 文件

```shell
$ cargo update
  Updating crates.io index
  Updating rand v0.8.5 →> v0.8.6
```

### 更换国内镜像

cargo 和 npm 一样也是需要更换国内镜像的，这里搜索了一下发现也有现成的工具可以使用

[GitHub - wtklbm/crm: Cargo registry manager (Cargo 注册表管理器)，用于方便的管理和更换 Rust 国内镜像源](https://github.com/wtklbm/crm)

推荐使用 `cargo install crm` 命令进行安装，这里说明一下 add 用于在项目中引入新的依赖，install 用于安装工具或命令行应用程序

crm 有一个比较友好的功能是支持评估网络延迟并自动切换到最优的镜像，命令为 `crm best` 。此外还有`crm best sparse` 可以选择支持 [sparse](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/6b6a25e6-3f3c-420a-8048-fd5b9c09dd37) 的源

另外，也可以通过修改 Cargo.toml 来单独指定某个依赖项所使用的镜像源

```rust
[dependencies]
rand = { registry = "ustc", version="0.8.5" }
```

### Cargo 稀疏注册协议 （sparse protocol）

[sparse](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/6b6a25e6-3f3c-420a-8048-fd5b9c09dd37) 是 rust 的新版索引协议从 1.68 版本开始提供，这种新的“稀疏”协议通常应该在访问 [crates.io](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/f80b03f6-a285-4b32-bfbc-166846ee7be2) 时提供显着的性能改进

[crates.io](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/f80b03f6-a285-4b32-bfbc-166846ee7be2) 是 crate 的官方存储库，类似 npm 官网

[crates.io: Rust Package Registry](https://crates.io/)

为了让 Cargo 确定 [crates.io](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/f80b03f6-a285-4b32-bfbc-166846ee7be2) 上存在哪些 crate，它会下载并读取一个“索引”，其中列出了所有 crate 的所有版本，称为 `crates.io index`。该索引位于 GitHub 上托管的 [git 存储库](https://github.com/rust-lang/crates.io-index/)。Cargo 获取索引并将其存储在 Cargo 的主目录中。该系统让 GitHub 处理服务器端，并提供一种便捷的方式来增量获取新更新

然而，随着索引随着时间的推移大幅增长，该系统已开始达到扩展限制，并且初始提取和更新继续减慢。您可能已经注意到，当 Cargo 显示`Updating crates.io index`时或在经历“解决增量”阶段时出现了暂停，其原因就是 `crates.io index` 过于庞大导致更新卡顿

通过 [RFC 2789](https://rust-lang.github.io/rfcs/2789-sparse-index.html) ，rust 引入了一种新协议来改进 Cargo 访问索引的方式。它不使用 git，而是直接通过 HTTPS 从索引中获取文件。Cargo 将仅下载有关项目中特定 crate 依赖项的信息，详情可查看下方链接

[Help test Cargo’s new index protocol | Inside Rust Blog](https://blog.rust-lang.org/inside-rust/2023/01/30/cargo-sparse-protocol.html)
