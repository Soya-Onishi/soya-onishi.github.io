---
title: "Redox OSのビルドの流れ"
date: 2023-01-06T23:04:46+09:00
draft: true
---

Redox OSについてどのようにビルドが進んでいくのか気になったので調べたのでその備忘録になります。
Redox OSはまだまだ開発中であること、RustやCargoにも今後さらなる機能が追加されることを考えると、
今後はビルド方法が変わるなどする可能性がありますので、この記事は2023年1月現在のものと考えていただければ幸いです。

例えば、Redox OSには現状動的リンクの機能がないため、すべて静的リンクでバイナリを作る必要があり、
そのリンク対象にはlibc（実態は[relibc]()）も含まれています。
これを実施するために静的ライブラリのパスを指定したりなどするスクリプトの実行などが間に挟まったりします。

## 全体の流れ

Redox OSはmakeを使ってディスクイメージの作成まで一気に実施します。
大まかな流れは以下になります。

![abstract_flow]()

今回は各パッケージのビルドの部分までまとめていきたいと思います。

## ビルドに使うツールのビルド

`make all`によって`/Makefile`を実行すると、
`/Makefile` → `/mk/disk.mk` → `/mk/repo.mk`と流れていき、`repo.mk`内の`$(BUILD)/fetch.tag`のルールがまずは実行されます。

```make
$(BUILD)/fetch.tag: cookbook installer prefix $(FILESYSTEM_CONFIG) $(CONTAINER_T
AG)
ifeq ($(PODMAN_BUILD),1)
        $(PODMAN_RUN) $(MAKE) $@
else
        $(HOST_CARGO) build --manifest-path cookbook/Cargo.toml --release
        $(HOST_CARGO) build --manifest-path installer/Cargo.toml --release
```

最初に`/installer/`と`/cookbook/`のクレートがビルドされます。
あとで詳細を記載しますが、前者はOS内で使用する各パッケージを設定ファイルから取得、
後者は各パッケージのダウンロードやビルドが主な処理になります。

`HOST_CARGO`は`/mk/config.mk`内で定義されており、
その`/mk/config.mk`は`Makefile`の初めの方でincludeされています。

```sh
HOST_CARGO=env --unset=RUSTUP_TOOLCHAIN cargo
```

見た限り、`RUSTUP_TOOLCHAIN`が定義されていたときに外してあげてから
`cargo`でビルドしているみたいですね。

