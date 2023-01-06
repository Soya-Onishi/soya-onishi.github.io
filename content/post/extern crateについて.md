---
title: "extern crateについて"
date: 2023-01-05T22:25:29+09:00
draft: false
---

`extern crate`についてedition 2018以降はほぼ使う必要がないとRustBookあたりに書いてあったと記憶しているが、
その感覚でいるとたまにlinterに`extern crate`を書けと言われたりする。
どういうときに`extern crate`が必要になるのかちゃんと理解できてなかったので、いい機会なので調べてみました。

## TL;DR

基本的にはRustBookの言うとおり、不要。

例外として、`core`や`alloc`クレートのような`Cargo.toml`で指定せずtoolchainに紐付いているようなクレートなど、
`Cargo.toml`で指定しないクレートに対しては`extern crate`が必要。

例外の例外として、`core`などと同様にtoolchainに紐づく`std`はプレリュードとして`extern crate`を記載しなくても使えるようになっている。

## もともとはどうだったのか

用法としてはGCCなどでの`-l`オプションに近く、
edition 2018以前は各クレートごとに`extern crate`が必要だったらしい。
しかし、大半は`Cargo.toml`の指定から導き出せるため、`extern crate`を不要にして余計な記述を廃したという流れとのこと。

## edition 2018以降はどうなのか

参考にしているページではボイラープレートなどなどと言っているので`extern crate <クレート名>`みたいなのが大量に記述されていたんじゃないかと思いますが、
edition 2018以降はそのような記述が不要になっています。

ただし例外として、`Cargo.toml`で記載せずに使うような標準ライブラリレベルの`alloc`や`core`クレートについては
`extern crate`での指定が必要のようです。
これらはもともと特定のtriple（例：x86_64_unknown-linux-gnuなど）のtoolchainに紐付いているものになります。
toolchainに紐付いているクレートには`std`も含まれるわけですが、
こちらについては自動的に参照できる状態になっているため、わざわざ`extern crate std`と記述する必要はないです。

## 参考ページ

[Forumでの回答](https://users.rust-lang.org/t/usage-of-extern-crate/73619/8)  
[The Rust Referenceでの記載](https://doc.rust-lang.org/reference/items/extern-crates.html)