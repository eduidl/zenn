---
title: RustでROS2
emoji: ⚙️
type: tech
topics: [ros, ros2, rust]
published: true
---

:::message
この記事は[ROS/ROS2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/ros)の 20 日目の記事です．
:::

遅刻しすぎて、最早 2021 年のアドベントカレンダーの方が近くなってしまいましたが、個人的に書いていた ROS2 の Rust 製クライアントが一段落したので投稿します．

## 要約

- Rust で ROS2 クライアントを書いている．
- pub/sub、サービス、タイマー処理、パラメータ等、最低限の機能がある程度動くようになってきた．

https://github.com/rclrust/rclrust

## Rust について

自分が考える Rust のよいところを挙げてみます．

- GC がない
- 公式の周辺ツールの充実
  - まともなパッケージマネージャ
  - テスト
    - メインの実装と同じファイルに書けるのが本当に楽
    - 上記の特徴により、private な関数でも簡単にテストできる
  - ドキュメント生成
    - `cargo doc`で一発
  - Lint
    - rustc + Clippy
  - フォーマット
    - `cargo fmt`
- デフォルトで不変
- 変数が基本的にムーブされる
- コンストラクタの扱い
- モダンな文法
  - if を初めとする制御構文やブロックが式
  - match 式
  - Option 型
  - Result 型によるエラー処理
  - トレイト
  - 扱いやすいタプル型
- 優れた型推論
- コンパイルが通ればメモリ安全が保証される（所有権・借用・ライフタイム etc...）
  - 例外：`unsafe`、コンパイラやライブラリのバグ
- 衛生的マクロ

まあ要するに書いていて楽しいです．

### Rust を始めたい人向けリンク

公式チュートリアル

https://doc.rust-jp.rs/book-ja

Rust ツアー

- 一部未翻訳あり

https://tourofrust.com

日本コミュニティの Slack

https://rust-jp.herokuapp.com

### メモリ安全？

Rust では、以下のような危険になりうるコードに対してコンパイルを通さないことによってメモリ安全性を保証しています．

- 借用されている変数の変更

```rust
fn main() {
    let mut s = 1;
    let r = &s;
    s = 2;  // コンパイルエラー
    println!("{}", r);
}
```

- 1 つの可変参照とは別に 1 つ以上の（不変・可変）参照が存在する

```rust
fn main() {
    let mut s = 1;
    let r1 = &s;
    let r2 = &mut s;  // コンパイルエラー
    println!("{}", r1);
}
```

- 参照は別スレッドに送る

```rust
fn main() {
    let s = 1;
    let r = &s;  // コンパイルエラー
    std::thread::spawn(move || println!("{}", r));
}
```

このため、基本的にコンパイルが通っていれば、メモリ安全であることが基本的に保証されています．
しかし例外があります、一つはコンパイラのバグです．まあほとんど出会うことはないでしょう．

もう一つが`unsafe`です．Rust ではポインタの参照外し、`static mut`へのアクセス等、メモリ安全を壊しうる危険な操作をするために、以下のように`unsafe`キーワードを使う必要があります．そして、当然`unsafe`を使った場合、それに由来するメモリ安全のケアは実装者の責任となります．

```rust
unsafe fn very_very_dangerous_function() {
  // 何か危険な処理
}

fn main() {
  unsafe { very_very_dangerous_function() }
}
```

ROS2 では rcl, rcmw, rcutils といった共通の C API が提供されており、rclcpp や rclpy はそれらをラップする形で実装されています．
話を Rust に戻すと、Rust から FFI による C API の呼び出しを行う場合、全て`unsafe`になります．ということはメモリ安全に関しては、私が保証するしかないということになりますが、Rust が言語として提供するレベルのメモリ安全からは現状程遠いと思っています．

参考

https://doc.rust-jp.rs/book-ja/ch19-01-unsafe-rust.html

## ビルドシステム

ROS2 では通常 colcon や ament を使います．ビルドシステムの役割としてはいくつかあって、ざっと以下のようなフェーズと項目に分けられると思います．

- prebuild
  - ビルド対象の走査
  - `package.xml`の depend を見てビルドの順番を決める
- build
  - メッセージ型等の生成
  - 依存関係の解決
  - ビルド
- install
  - バイナリ等を`install`以下に配置する

Rust における依存解決について少し説明します．Rust ではいわゆるライブラリを crate と呼びます．自分が実装している crate 内で別の crate を利用したい場合`Cargo.toml`というファイルに記述します．指定方法は以下の 3 つです．

- レジストリ（通常 crates.io）
- git
- ローカルパス

```toml
[dependencies]
time = "0.1.12"
rand = { git = "https://github.com/rust-lang-nursery/rand" }
hello_utils = { path = "../hello_utils", version = "0.1.0" }
```

[ros2_rust](https://github.com/ros2-rust/ros2_rust)では、esteve さんが土台を作ったのもあり、ROS2 のビルドシステムに則った形になっています．
現状は`ament_cmeke`の枠組みで`CMakeLists.txt`での力技になっており、特に依存関係については、CMake のプロセス中に、dependencies セクションにローカルパスの形で追記するような形を取っています（ build ディレクトリにコピーし、コピー先に追記しているので、作業ディレクトリへの影響はありません）．
ただ、この形だと元の作業ディレクトリでは`Cargo.toml`の記述が不完全なままなので、エディタの補完が受けにかったり、ローカルで`cargo`コマンドを叩いても上手くいかないと何かと不便そうだなと感じました．かといって、作業ディレクトリに直接追記するのもあまりいけてないし、綺麗な方法が思い浮かびませんでした．

そこで [rosrust](https://github.com/adnanademovic/rosrust) でやっているように、クライアントライブラリは、ROS2 のビルドシステムから切り離して作ることにしました．元のビルドシステムの役割に話を戻すと、prebuild/install フェーズは無関係であり、唯一の障害はメッセージ型等の生成のみです．
rosrust では手続きマクロを使ってコード生成を行っています．自分も一度、手続きマクロを作ってみたかったのでそれに倣いました^[はじめは [sailfish](https://github.com/Kogia-sima/sailfish)というテンプレートエンジンで生成していました]．

### 型生成

ROS2 では、`*.msg/*.srv/*.action` -> rosidl_adapter -> `*.idl` -> rosidl_parser -> rosidl_generator\_\* -> ソースコード、といった流れで変換しています^[正直何のために`*.idl`を介しているのかよくわかりません]．
ですが、`*.idl`は表現力が高い分パースが少し大変で、どう考えても`*.msg/*.srv/*.action`の方が格段に簡単だったのでそちらから直接コードを生成するようにしました^[実は`*.idl`で直接定義することもできるようですが、そんな人おらんやろ]．
私は[nom](https://github.com/Geal/nom)というパーサコンビネータを使いましたが、rosidl_adapter のパーサは雑^[`#`より右側をコメント扱いしているので、`#`を含む文字列を定数にすることはできません https://github.com/ros2/rosidl/blob/2.4.0/rosidl_adapter/rosidl_adapter/parser.py#L472-L485]なので、正規表現で十分です．なお、rosidl_parser の方は [lark](https://github.com/lark-parser/lark)というパーサを使っているので正規表現では厳しいです．

楽をするなら rosidl_generator_rust とやらを Python で実装して、`std::process::Command`とかで呼ぶみたいな手もあったと思います．

参考：[ROS2 Client Library を作るということ](https://qiita.com/nonanonno/items/332a585fe8c9930692bd#%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8%E7%94%9F%E6%88%90%E3%81%AE%E3%82%84%E3%82%8A%E6%96%B9)

### その他

クライアント自体は、prebuild/install と無関係と書きましたが、それを利用するノード crate はそうではありません．
build プロセスは cargo に丸投げできるので、`pacakge.xml`と`CMakeLists.txt` をちょろっと書いたら、インストールまでできると思います．

参考

https://github.com/adnanademovic/rosrust/issues/113

## ビルド

現在は Ubunttu 20.04、Foxy のみで動作確認をしているため、他の環境だと動かない可能性があります．

Rust をまだインストールしていない人は https://www.rust-lang.org/tools/install からインストールしておきましょう．

ビルドは以下の手順で行えます．ビルドは ROS2 の一部に依存しているので、`source /opt/ros/foxy/setup.bash` 等は忘れずに

```
$ git clone git@github.com:rclrust/rclrust.git
$ cd rclrust
$ cargo build
```

## サンプル

ここでは Pub/Sub だけ簡単に示しておきます．

### `publisher.rs`

ミニマムなサンプルにしようと思ったので Timer を使っていませんが、Timer も実装済みです．
Rust を触ったことがなくても大体何をやろうとしているか、わかるのではないでしょうか？

```rust
use std::thread::sleep;
use std::time::Duration;

use anyhow::Result;
use rclrust::rclrust_info;
use rclrust::qos::QoSProfile;
use rclrust_msg::std_msgs::msg::String as String_;

fn main() -> Result<()> {
    let ctx = rclrust::init()?;
    let node = ctx.create_node("examples_publisher")?;
    let logger = node.logger();
    let publisher = node.create_publisher::<String_>("message", &QoSProfile::default())?;

    for count in 0..1000 {
        publisher.publish(&String_ {
            data: format!("hello {}", count),
        })?;
        rclrust_info!(logger, "hello {}", count);
        sleep(Duration::from_secs_f32(0.1));
    }

    Ok(())
}
```

### `subscription.rs`

```rust
use anyhow::Result;
use rclrust::qos::QoSProfile;
use rclrust::rclrust_info;
use rclrust_msg::std_msgs::msg::String as String_;

fn main() -> Result<()> {
    let ctx = rclrust::init()?;
    let node = ctx.create_node("examples_subscriber")?;
    let logger = node.logger();

    let _subscription = node.create_subscription(
        "message",
        move |msg: String_| {
            rclrust_info!(logger, "{}", msg.data);
        },
        &QoSProfile::default(),
    )?;

    rclrust::spin(&node)?;

    Ok(())
}
```

---

左のターミナルで

```
$ cargo run --examples publisher
```

右のターミナルで

```sh-session
$ cargo run --examples subscription
```

と叩くと以下のように動くはずです．

![out](https://user-images.githubusercontent.com/25898373/128894819-f925b31f-d814-4046-a328-68bfe854d03b.gif)

`--ros-args`等渡したい場合、`--`で区切ってから渡します．

```sh-session
$ cargo run --example publisher -- --ros-args -r __name:=another_node_name
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/examples/publisher --ros-args -r '__name:=another_node_name'`
[INFO] [1628683463.609400191] [another_node_name]: hello 0
[INFO] [1628683463.709645029] [another_node_name]: hello 1
[INFO] [1628683463.809899030] [another_node_name]: hello 2
[INFO] [1628683463.910187507] [another_node_name]: hello 3
[INFO] [1628683464.010399712] [another_node_name]: hello 4
:
```

興味があれば、他にもいくつかサンプルコードを置いているので動かしてみてください．

https://github.com/rclrust/rclrust/tree/main/rclrust/examples

## おわりに

何かフィードバックあればリポジトリの issue の方にお願いします．

## 参考：その他言語別クライント

最低 Pub/Sub が動く（きそうな）ものを集めました．抜けあったら教えてください．基本アルファベット順

| 言語    | クライアント                                                          |
| ------- | --------------------------------------------------------------------- |
| C++     | [ros2/rclcpp](https://github.com/ros2/rclcpp)                         |
| Python  | [ros2/rclpy](https://github.com/ros2/rclpy)                           |
| Ada     | [ada-ros/rclada](https://github.com/ada-ros/rclada)                   |
| C       | [ros2/rclc](https://github.com/ros2/rclc)                             |
| C#      | [ros2-dotnet](https://github.com/ros2-dotnet/ros2_dotnet)             |
| D       | [nananonno/ros2_d](https://github.com/nonanonno/ros2_d)               |
| Elixir  | [rclex/rclex](https://github.com/rclex/rclex)                         |
| Go      | [juaruipav/rclgo](https://github.com/juaruipav/rclgo)                 |
| Java    | [ros2-java/ros2_java](https://github.com/ros2-java/ros2_java)         |
| Node.js | [RobotWebTools/rclnodejs](https://github.com/RobotWebTools/rclnodejs) |
| Rust    | [ros2-rust/ros2_rust](https://github.com/ros2-rust/ros2_rust)         |
| 〃      | [sequenceplanner/r2r](https://github.com/sequenceplanner/r2r)         |
