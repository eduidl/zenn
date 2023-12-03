---
title: "[Rust] cargo workspace で使える tips"
emoji: 🛰
type: tech
topics: [rust]
published: true
---

:::message
この記事は [Rust Advent Calendar 2023](https://qiita.com/advent-calendar/2023/rust) の 3 日目の記事です
:::

## はじめに

cargo workspace とは、複数のパッケージを一つのプロジェクトとして扱う機能です。
パッケージを分割しワークスペースで管理することで、保守性や再利用性が上がるなどのメリットがある一方で、 `Cargo.toml` の管理が多少が面倒になることもあります。
この記事では、これを少しだけでも解消する方法を紹介します。

## `[workspace.package]` でパッケージの情報をまとめる

パッケージは必ず `Cargo.toml` を持ち、そこにビルドの依存関係や、crates.io に公開する際に使われる情報などを記述する必要があります。これは、ワークスペースを使う場合も例外ではありません。
その一方で、ライセンスやリポジトリの情報など、ワークスペース内で全てに共通するものもあります。
こういったときに使えるのが、 Rust 1.64 で導入されたワークスペース内の継承です。

https://blog.rust-lang.org/2022/09/22/Rust-1.64.0.html#cargo-improvements-workspace-inheritance-and-multi-target-builds

使い方は簡単で、ワークスペース の `Cargo.toml` に `[workspace.package]` セクションを記述し、子パッケージの `Cargo.toml` で `<key>.workspace = true ` と書くだけです。
利用例は以下の通り。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar"]

[workspace.package]
version = "1.2.3"
authors = ["Nice Folks"]
description = "A short description of my package"
documentation = "https://example.com/bar"
```

```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version.workspace = true
authors.workspace = true
description.workspace = true
documentation.workspace = true
```

（https://doc.rust-lang.org/cargo/reference/workspaces.html#the-package-table より引用）

他にも継承可能な項目はいくつかあり、以下のリンク先で記述されています。

https://doc.rust-lang.org/cargo/reference/workspaces.html#the-package-table

私は以下の 6 つをよく使っています。

- authors
- edition
- license
- repository
- rust-version
- version

特に上 4 つはパッケージごとに変えることは非常に少ないのではないかと思います。
`version` や `rust-version` はパッケージによって変えられなくもないですが、パッケージの数が増えてくると非現実的になってくると思います。

逆に、readme や description 等は、私はパッケージごとに多少変えることが多く、あまり使いどころがないと感じます。

## `[workspace.dependencies]` で依存バージョンをまとめる

これも Rust 1.64 からできるようになりました。
こちらは、同じを複数のパッケージで使いたい場合に使えます。特に、複数のパッケージにわたり、あるクレートのバージョンを同じにしておきたい場合には有用です。

ワークスペースの `Cargo.toml` に `[workspace.dependencies]` セクションを記述することで、子パッケージから利用できます。feature の設定も可能な他、 `dev-dependencies` や `build-dependencies` でも使えます。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar"]

[workspace.dependencies]
cc = "1.0.73"
rand = "0.8.5"
regex = { version = "1.6.0", default-features = false, features = ["std"] }
```

```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version = "0.2.0"

[dependencies]
regex = { workspace = true, features = ["unicode"] }

[build-dependencies]
cc.workspace = true

[dev-dependencies]
rand.workspace = true
```

(https://doc.rust-lang.org/cargo/reference/workspaces.html#the-dependencies-table より引用)

例えば、 [protobuf](https://crates.io/crates/protobuf) と [protobuf-codegen](https://crates.io/crates/protobuf-codegen) のバージョンを複数のパッケージにわたり一元的に固定したいことがあり、これが、`workspace.dependencies` を使うことで非常に対処しやすくなりました。

さらに、ワークスペースを利用する場合、内部の依存関係も書く必要があるはずですが、これも `workspace.dependencies` を使うことで管理しやすくなります^[このあたりは[cargo-release](https://crates.io/crates/cargo-release) でも十分かもしれませんが、ワークスペースに設定が集まっているのはそれだけで価値があると思っています]。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace]
members = ["bar", "bar-baz", "bar-foo"]

[workspace.package]
version = "0.1.0"

[workspace.dependencies]
bar-baz = { path = "bar-baz", version = "=0.1.0" }
bar-foo = { path = "bar-foo", version = "=0.1.0" }
```

```toml
# [PROJECT_DIR]/bar/Cargo.toml
[package]
name = "bar"
version.workspace = true

[dependencies]
# bar-baz = { path = "../bar-baz", version = "=0.1.0" } と同じ
bar-baz.workspace = true
bar-foo.workspace = true
```

### 参考

https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#inheriting-a-dependency-from-a-workspace

## cargo release を使ってリリースする

ワークスペース内にクレートが沢山ある場合には、`cargo publish` するだけでもそこそこ大変です^[といいつつ `cargo publish` しか使ってませんでした]。数もそうですが、依存関係があるので publish する順番に気を使う必要があるからです。
そんなとき、 [cargo-release](https://crates.io/crates/cargo-release) を使うと、クレートを一括で publish できるようになります。
一度インストールしたあとは、ワークスペースの直下で以下のコマンドを叩くだけです。

```
$ cargo release --execute
```

ただ、デフォルト設定から少し変えた方がいいと思う部分があるので、それを紹介します。

### タグの設定

cargo-release は crates.io 等に publish する前に、デフォルトでバージョン（vX.Y.Z）を git タグを切ってくれるのですが、ワークスペースの場合、属する全てのパッケージそれぞれについてタグが切られてしまいます。
例えば、`foo` と `bar` というパッケージがあった場合、 `foo-v0.1.0` と `bar-v0.1.0` というタグが切られるということです。
パッケージごとにバージョンを管理するならそれもありでしょうが、バージョンを共通化する場合には冗長に感じます。

そこで、ワークスペースの `Cargo.toml` に以下のように設定をしています。このようにすることで `v0.1.0` のようなタグだけが切られるようになります。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace.metadata.release]
tag-prefix = ""
```

しかしこれにも少し注意が必要で異なるバージョンが混ざっている場合、 `foo` が `v0.1.0`、 `bar` が `v0.2.0` だと、 `v0.1.0` と `v0.2.0` の両方が切られてしまいます^[そのための `shared-version` という設定なのかなと思いましたが、true にしても特に何も起こらないようでした]。

こちらが気になる場合には、少し記述量が増えますが、ワークスペースの `Cargo.toml` に次のように設定して全体のタグ付けを無効にしつつ、メインのパッケージだけタグ付けを明示的に設定するのがよいでしょう。

```toml
# [PROJECT_DIR]/Cargo.toml
[workspace.metadata.release]
tag = false
```

```toml
# メインのパッケージのCargo.toml
[package.metadata.release]
tag = true
tag-prefix =""
```

### 一部のパッケージの除外

cargo-release はワークスペースに属する全てのパッケージをリリースしようとしますが、リリース対象でないパッケージがあるかもしれません（たとえ、`publish = false` されていてもバージョンが更新される）
その場合は、リリースしたくないパッケージの `Cargo.toml` に以下のように設定すると、リリース対象から外すことができます。
その場限りの場合は、 `cargo release --exclude` で指定するだけでもよいでしょう。

```toml
# Cargo.toml
[package.metadata.release]
release = false
```

### 参考

- https://github.com/crate-ci/cargo-release/blob/master/docs/reference.md

## おわりに

以上の tips は、個人的にですが、実際に以下のリポジトリで使ってみていて、特にワークスペースで設定を集約できるのは非常に便利だと感じています。

https://github.com/eduidl/gz-rs

ご存じなかった方は、是非使ってみてください。
