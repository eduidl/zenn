---
title: [Rust] ユーザ定義型をシリアライズしてZenohでやり取りする
emoji: 📮
type: tech
topics: [rust, zenoh]
published: true
---

以下の記事を読んで Zenoh に興味が出てきました。

https://qiita.com/Shintaro_Hosoai/items/0bde489cde43a00d6f96

ROS を多少触っている身として、ROS の Pub/Sub 的な使い方ができないかを試してみました。

## 実装

https://github.com/eduidl/zenoh-typed-pubsub-example

## シリアライズ方法

いわゆる publish をおこなう `put()` に渡せる値には、 `Into<Value>` トレイトが実装されている必要があります。

https://docs.rs/zenoh/0.7.0-rc/zenoh/publication/struct.Publisher.html#method.put

探してみると、 `Vec<u8>` に `Into<Value>` が実装されているので、バイト列へのシリアライズ方法を考えてみます。

https://docs.rs/zenoh/0.7.0-rc/zenoh/value/struct.Value.html#impl-From%3C%26[u8]%3E-for-Value

ROS 的な使い方であることや、シリアライズ効率などを考えて、スキーマありで、かつ serde と組み合わせて簡単に使える [bincode](https://github.com/bincode-org/bincode) にしてみました。
もちろん、MessagePack, Protocol Buffers, FlatBuffers 等を使っても問題ないはずです。

## やりとりするメッセージの型

データ構造に大して意味はないんですが、Vec や String そしてネスト構造がある型を適当に用意してみました。
Rust では [serde](https://serde.rs) という超有名ライブラリを使えば、derive するだけでシリアライズ・デシリアライズできるようになるので非常に便利です。

https://github.com/eduidl/zenoh-typed-pubsub-example/blob/main/src/lib.rs

`Serialize` を derive しているおかげで、[bincode::serialize](https://docs.rs/bincode/1.3.3/bincode/fn.serialize.html) を呼ぶだけでシリアライズできます。

https://github.com/eduidl/zenoh-typed-pubsub-example/blob/main/src/bin/serialize.rs

```
serialized: [1, 0, 0, 0, 11, 0, 0, 0, 0, 0, 0, 0, 104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100, 3, 0, 0, 0, 0, 0, 0, 0, 20, 34, 2, 255, 25, 0, 0, 0, 0, 0, 0, 0, 4, 0, 0, 0, 0, 0, 0, 0, 104, 111, 103, 101, 244]
```

データを整理すると以下のようになります。フィールド名を無視してデータを素直に上から詰めていった感じですね。

```
1, 0, 0, 0 => u32(1)

11, 0, 0, 0, 0, 0, 0, 0 => usize(11)
104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100 => "hello world"

3, 0, 0, 0, 0, 0, 0, 0 => usize(3)
20, 34, 2 => Vec<u8> [20, 34, 2]

255 => i8(-1)

25, 0, 0, 0, 0, 0, 0, 0 => i64(25)

4, 0, 0, 0, 0, 0, 0, 0 => usize(4)
104, 111, 103, 101 => "hoge"

244 => u8(244)
```

## Publish

https://github.com/eclipse-zenoh/zenoh/blob/b06d91f5986fb87b607af92be874ecda28d021d9/examples/examples/z_pub.rs

を参考に以下のように実装してみました。エラー処理とかは適当です。

https://github.com/eduidl/zenoh-typed-pubsub-example/blob/main/src/bin/pub.rs

## Subscribe

https://github.com/eclipse-zenoh/zenoh/blob/b06d91f5986fb87b607af92be874ecda28d021d9/examples/examples/z_sub.rs

を参考に以下のように実装しました。

https://github.com/eduidl/zenoh-typed-pubsub-example/blob/main/src/bin/sub.rs

`&[u8]` を引数に取れる [bincode::deserialize](https://docs.rs/bincode/1.3.3/bincode/fn.deserialize.html) がありますが、これを Zenoh と組み合わせて使うにはコピーが必要なので非効率です。
代わりに [std::io::Read](https://doc.rust-lang.org/std/io/trait.Read.html) を実装した型を取れる [bincode::deserialize_from](https://docs.rs/bincode/1.3.3/bincode/fn.deserialize_from.html) を使えば、余分なコピーをせずに処理できます。

https://docs.rs/zenoh/0.7.0-rc/zenoh/value/struct.Value.html

https://docs.rs/zenoh-buffers/0.7.0-rc/zenoh_buffers/struct.ZBuf.html

https://docs.rs/zenoh-buffers/0.7.0-rc/zenoh_buffers/traits/reader/trait.HasReader.html#associatedtype.Reader

## 動作

以上の pub/sub を同時に動かしたところ以下のようになりました。

![](https://storage.googleapis.com/zenn-user-upload/02d3d55eafb3-20230205.gif)

## 課題

実用には、pub と sub 側でシリアライズ・デシリアライズが一貫しているかをどこかで保証する必要があります。
適当なヘッダを用意したりすればいいのでしょうが、まだあまり考えられていません。
