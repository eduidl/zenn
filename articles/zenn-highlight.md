---
title: シンタックスハイライトの言語名はサービスによって多少違うことがある
emoji: ✨
type: idea
topics: [zenn]
published: false
---

それはそうって内容ですが、間違えていたので記事にしました．
ZennやQiita等の（エンジニア向け）ブログサービスでは、以下のように言語名を指定して書くとシンタックスハイライトをしてくれる機能があります．

```
`​``rust
fn main() {
    println!("Hello World!");
}
`​``
```

```rust
fn main() {
    println!("Hello World!");
}
```

ところが、シンタックスハイライトに使っているライブラリが、Zennでは[Prism.js](https://prismjs.com)、Qiitaでは[Rouge](http://rouge.jneen.net)（こっちはRubyで書かれています）が使われており、当然ながらサポートしている言語だけではなく、その名称が異なる場合があります．


> シンタックスハイライトには Prism.js を使用しています。
> （https://zenn.dev/zenn/articles/markdown-guide#%E3%82%B3%E3%83%BC%E3%83%89%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF より引用）

> シンタックスハイライトには Rouge を利用しており、(後略)
> （https://qiita.com/Qiita/items/e84f5aad7757afce82ba より引用）

例えば、以下のようなハイライトをしたい場合、Rougeの場合、`console`, `terminal`, `shell-session`, `shell-session`という名前が使えますが、Prism.jsでは`shell-session`, `sh-session`, `shellsession`です．（一方で、RougeというかQiitaの方はハイライト当たってない気がします．公式サンプルと明らかに違うし……．というかそれに慣れていたので、当たっていないことに気づかなかった．）

```shell-session
$ echo ok
```

その他にも、Rougeでは、C++に対して`cpp`,`c++`両方が使えるのに対し、Prism.jsでは`cpp`のみである等、基本的にRougeの方がエイリアスが多く存在している印象です．

## 対策

Zenn CLIを使っている場合、`zenn preview`をしたコンソールに、`Language does not exist: hogehoge`とファイル保存のタイミングで表示されるので気づけるでしょう．私はそれで気が付きました．
Web版はGitHub連携切るのが面倒なので確認はしていませんが、私の記憶が正しく、そして変更がされていなければ、特に警告はされなかったかと思います．

https://prismjs.com/#supported-languages で確認できるので、色とか何も変わってないなと思ったら確認しておくのがよいでしょう．

## プレーンテキストについて

プレーンテキストに対して、Rougeでは`plaintext`,`text`があり、Prism.jsには存在しないようなので、ファイル名だけ出したい場合いい方法がないなと思っています．

例えば、以下のように種類を書かないとファイル名は無視されてしまいます．

```
`​``:出力
Hello World!
`​``
```

```:出力
Hello World!
```

また、以下のように嘘の種類を書くと、ファイル名は表示されるものの、Zenn CLIを使っていると、`Language does not exist: text`と保存する度出力されるため、少し気になります．

```
`​``text:出力
Hello World!
`​``
```

```text:出力
Hello World!
```

今は、行頭に+や-が来なければ、`diff`つければいいかなと思っています．何かもっといい言語名知っていたら教えてください．

```
`​``diff :出力
Hello World!
`​``
```

```diff :出力
Hello World!
```
