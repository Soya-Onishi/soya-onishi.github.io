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

## OS内に含めるパッケージの集約

### 集約されるパッケージについて

Redox OSのビルドとお話していますがカーネルのビルドはその一部であり、
その他にも`ion`（Redox OSにおけるターミナル）や
基本的なコマンドが集約された`coreutils`などもビルドされます。

[The Redox Operating System](https://doc.redox-os.org/book/ch09-01-including-programs.html)で例を使って説明されているのですが、
どのパッケージをビルドするかは`/config/<ARCH>/<CONFIG>.toml`内の
`package`セクションに記載します。
ここで、`<ARCH>`は実行のアーキテクチャを示しており基本的には`x86_64`になると思います。
また、`<CONFIG>`は`/mk/config.mk`内の`CONFIG_NAME`で指定した名前が入ります。

例えば、x86_64環境で`CONFIG_NAME`が`desktop`になっている場合に、
greetingsというパッケージを作ってこれをRedox OS内で使いたいとき、
`config/x86_64/desktop.toml`内の
`package`セクションに`greetings = {}`という行を追加します。

### パッケージの集約処理について

パッケージの集約は`/mk/repo.mk`の`$(BUILD)/fetch.tag`ルール内の

```sh
PACKAGES="$$($(INSTALLER) --list-packages -c $(FILESYSTEM_CONFIG))"
```

で実施されます。

`INSTALLER`と`FILESYSTEM_CONFIG`は`/mk/config.mk`で定義されています。

```sh
INSTALLER=installer/target/release/redox_installer
```

```sh
FILESYSTEM_CONFIG?=config/$(ARCH)/$(CONFIG_NAME).toml
```

`INSTALLER`はバイナリなのでソースコードは`/installer/src/bin/installer.rs`内になります。
処理としてはまずコマンドライン引数のパースを行ったあと、`FILESYSTEM_CONFIG`で指定したtomlファイルを読みこんだ後、`packages`に記載されている各パッケージ名を標準出力に流していきます。そうすることで、シェル変数の`PACKAGES`にパッケージ名が追加されていきます。

```rust
if parser.found("list-packages") {
    for (packagename, _package) in &config.packages {
        println!("{}", packagename);
    }
} 
```

## 各パッケージのダウンロード

ダウンロードは`/cookbook/fetch.sh`がエントリポイントになります。
`fetch.sh`では`PACKAGES`の内容が入っている`$recipes`を回して、
ダウンロード処理を`cook`および`cook.sh`で進めていきます。

```sh
for recipe in $recipes
do
    if [ -e "recipes/$recipe/recipe.toml" ]
    then
        target/release/cook --fetch-only "$recipe"
        continue
    fi

    ./cook.sh "$recipe" fetch
done
```

`recipes/$recipe/recipe.toml`の部分について説明していきます。
各パッケージのディレクトリは`/cookbook/recipes/`に入っています。
また各パッケージは`recipe.toml`を持っており、
その内容からどこからソースコードを引っ張ってくるか、
どうやってコンパイルするかがわかります。
例えば`coreutils`パッケージにも`recipe.toml`がありますが、
このパスは`/cookbook/recipes/coreutils/recipe.toml`になります。
`recipe.toml`の内容については後ほど説明します。

`target/release/cook --fetch-only "$recipe"`や`./cook.sh "$recipe" fetch`ではそのオプションの通り、`$recipe`で指定したパッケージのダウンロードを行います。

## 各パッケージのビルド

前節までで一旦`mk/repo.mk`内の`$(BUILD)/fetch.tag`は終わりです。
`$(BUILD)/repo.tag`ルールを実行していきますが、
`repo.sh`内がビルドを実行するスクリプトになるので、この中身を見ていきます。

とはいえ、`repo.sh`でやっていることは引数で渡されたパッケージ名を
一つずつ処理していくだけで、処理内容も`cook`や`cook.sh`の呼び出しとその呼び出しのために
シェル変数を定義するのが主な内容です。
実際のビルドは`cook`の中で行われます（`cook.sh`でも行いますが最終的に`cook`に渡しているのでやっていることは同じです）

