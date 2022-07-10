---
title: "[ROS] ノードやトピックの情報を相互に辿れる自作TUIツールの紹介"
emoji: 🤖
type: tech
topics: [ros, ros1, ros2, tui]
published: true
---

タイトルの通り ROS のノードやトピックの情報を相互に辿れる自作 TUI ツールを作ったので紹介します。

https://github.com/eduidl/rtui

## デモ動画

適当に turtlebot4 のシミュレータを動かしながら、rtui を走らせている様子です。
TUI なの（とそういったセンスがないの）で非常に簡素な見た目ですが、トピックの一覧から個別トピックの情報に飛んだり、そのトピックを echo したり、トピックを購読しているノードの情報に飛んだりできます。

![](https://storage.googleapis.com/zenn-user-upload/e053c115d176-20220710.gif)

※この動画も含め画像は全て ROS2 galactic で動かしたものです。

## 環境

以下の環境をサポートしています

- ROS1
  - melodic
    - ただし、Poetry が Python 3.6 をサポートしていなそうなので 3.8 以上としています
    - Ubuntu 20.04 だとデフォルトで Python 3.8 が入っているのでそれに合わせる形です。
  - noetic
- ROS2
  - foxy 以降

## インストール

[pipx](https://github.com/pypa/pipx)自体の宣伝も兼ねて、pipx でのインストール方法を記載します。
既に知っている方は大勢いらっしゃるでしょうし、Node.js を使ったことがある人なら、`npx`的なやつといえば伝わるでしょうか（仕組みは大分異なる気がしますが）。
venv を使いながらも使い勝手そのままに、CLI ツールを環境をあまり汚さずにインストールできるツールです。

https://github.com/pypa/pipx

```sh-session
$ pipx install git+https://github.com/eduidl/rtui.git
```

ただし、melodic だけはホストの Python を使えないので、以下のように使う Python のバージョンを変えてください。

```sh-session
$ pipx install --python python3.8 git+https://github.com/eduidl/rtui.git
```

もちろん、pip でもいれられます。

## コマンド

現状はサブコマンドが 4 つあります。が、基本的に直接/間接的に遷移可能なので実質 1 つです。

### `rtui nodes`

引数はありません。最初は左のカラムしか表示されていませんが、いずれかのノード名をクリックすると右画面にそのノードの情報が表示されます。
ROS1 はアクションの情報が取る API が無さそうなので表示されていませんが、似たような表示です。
トピック名、サービス名、アクション名（ROS2 のみ）をクリックするとその情報に飛べます。トピックの情報に飛ぶとともに左のカラムもトピック一覧に更新されます（他も同様）

![](https://storage.googleapis.com/zenn-user-upload/41ccc9781028-20220710.png)

### `rtui topics`

型名と Publish/Subscribe しているノードが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/87b4bfbbf3e4-20220710.png)

またこのトピックの画面だけは、e キーを押すとトピックを購読し、右画面の下半分に表示することができます。再度 e を押すと停止します。

![](https://storage.googleapis.com/zenn-user-upload/38d45dcc53b9-20220710.png)

ROS2 では ROS1 と異なり^[ROS1 は 後勝ちですかね？開発時とかで 1 回吐いたら二度と変えられないというのも面倒そうなので、まあ妥当…なのかな]、同一のトピックを別の型で同時に吐けるようです ^[私にはそうする理由がわかりません]。
この仕様には正直頭を抱えましたが、あまり考えずに両方の型を表示するようにしました。e キー押したときに購読するのは決め打ちです…。

![](https://storage.googleapis.com/zenn-user-upload/64e417a02c4e-20220710.png)

### `rtui services`

ROS1 ではサーバノード名しか取れなくて残念と思っていたら、ROS2 ではどっちも取ってこれないことに気づいてさらに残念な気持ちになりました。
ROS2 だとノード名からサービスには飛べるものの、ここからどこにも行けず、立ち上げ直すしかなくなってしまうので、常時履歴を保存しておき、b キーで戻れるようにしました。
ROS2 のパラメータサーバは多すぎるのでリストからは非表示にしています。

![](https://storage.googleapis.com/zenn-user-upload/f6ab51f78999-20220710.png)

### `rtui actions`

アクションはサーバもクライアントも両方取ってこれます。サービスよりアクションを使いましょうという思し召しなのでしょうか？

![](https://storage.googleapis.com/zenn-user-upload/d51ef1a91688-20220710.png)

## 終わりに

機能は今のところ以上です。便利そうだなと思ったらぜひ使ってみてください。
フィードバックもお待ちしています。

## やり残し

- メッセージ型等の情報表示（`rosmsg`や`ros2 interface`相当）
- `rqt_console`あたりも TUI に向いていそうに思う
- textual よりいい TUI ライブラリあったら乗り換えたい
