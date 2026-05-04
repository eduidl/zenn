---
title: "MCAP をターミナルでざっと眺める TUI ツール を作った"
emoji: "🧢"
type: "tech"
topics: ["rust", "mcap", "tui", "cli"]
published: true
---

`mcaptui` という、自分で作った MCAP 閲覧用の TUI ツールを紹介します。
リポジトリとしては [`mcapdecode-rs`](https://github.com/eduidl/mcapdecode-rs/tree/main/tools/mcaptui) の一部として公開していて、 crates.io からもインストールできます。

https://github.com/eduidl/mcapdecode-rs/tree/main/tools/mcaptui

MCAP を見るとき、いきなり本格的な解析を始める前に、まずは「どんな topic があるか」「どんな schema なのか」「実際に 1 件開くとどういう構造になっているか」を素早く確認したいことがあります。
`mcaptui` は、そういう最初の確認をターミナルの中で完結させるために自作したツールです。

## `mcaptui` でできること

`mcaptui` は、MCAP ファイルの中身を TUI で順番に見ていくためのツールです。
主に次のようなことができます。

- topic 名の一覧を見る
- topic ごとのメッセージ件数や schema 名を確認する
- 選択した topic の schema をその場で確認する
- topic に含まれるメッセージを 1 件ずつ開いて、デコード済みの内容を見る
- デコーダがない場合でも raw payload として中身を確認する

個人的には、特に「最初に topic 一覧を見て、気になるものを開き、schema と実データを交互に眺める」という流れがやりやすいのが気に入っています。

起動直後は topic 一覧が表示されます。

![](/images/mcaptui-introduction/mcaptui-topics.png)

`s` キーを押すと、その topic をデコードするときに使われる schema をポップアップで確認できます。

![](/images/mcaptui-introduction/mcaptui-schema.png)

`Enter` で topic を開くと、上にメッセージ一覧、下に選択中メッセージの詳細が出ます。

![](/images/mcaptui-introduction/mcaptui-messages.png)

## `mcaptui` の使い方

インストールは `cargo install` でできます。

```bash
cargo install mcaptui
```

GitHub Releases には、少なくとも Linux 向けの prebuilt archive も置いてあります。

基本的な使い方はかなり単純です。

```bash
mcaptui sample.mcap
```

特定の topic を最初から開きたい場合は `--topic` が使えます。

```bash
mcaptui sample.mcap --topic /imu/data
```

並列で chunk の展開や decode を進めたい場合は `--parallel` もあります。

```bash
mcaptui sample.mcap --parallel
```

操作は次のキーを覚えておけばだいたい足ります。

- `Up/Down` または `j/k` で移動
- `Enter` で topic を開く
- `s` で schema 表示の切り替え
- `Tab` でフォーカスの切り替え
- `Esc` で topic 一覧に戻る
- `q` で終了

「とりあえず開いて topic を眺める」用途なら、これだけでほぼ困らないと思います。

## まとめ

`mcaptui` は、自分が MCAP を調べるときに欲しかった導線を、そのまま TUI にしたようなツールです。
topic 一覧、schema 確認、メッセージ詳細確認までをターミナル内でつなげて扱えるので、まずファイルの中身をざっと把握したいときに向いています。

MCAP を触る人で、最初の確認をもう少し軽くしたい人にはそれなりに役立つはずです。
もし用途が合いそうなら、試してもらえると嬉しいです。

https://crates.io/crates/mcaptui
