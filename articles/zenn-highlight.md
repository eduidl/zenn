---
title: シンタックスハイライトの言語名はサービスによって多少異なることがある
emoji: ✨
type: idea
topics: [zenn]
published: true
---

それはそうという内容ですが、間違えていた自戒として記事にしました．

## 事象

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

ところが、シンタックスハイライトに使っているライブラリが、Zennでは[Prism.js](https://prismjs.com)、Qiitaでは[Rouge](http://rouge.jneen.net)（こっちはRubyで書かれています）が使われており、当然ながらサポートしている言語だけではなく、その名前が異なる場合があります．


> シンタックスハイライトには Prism.js を使用しています。
> （https://zenn.dev/zenn/articles/markdown-guide#%E3%82%B3%E3%83%BC%E3%83%89%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF より引用）

> シンタックスハイライトには Rouge を利用しており、(後略)
> （https://qiita.com/Qiita/items/e84f5aad7757afce82ba より引用）

例えば、コンソールの入出力であることを示したいとき、Rougeでは`console`,`terminal`,`shell_session`,`shell-session`という名前が使えますが、Prism.jsでは`shell-session`,`sh-session`,`shellsession`と、`shell-session`を除き異なります．（一方で、RougeというかQiitaの方はシンタックスハイライトが当たっていない気がします．公式サンプルと明らかに違うし……．それに慣れていたせいで、当たっていないことに違和感を持てませんでした）

```shell-session
$ echo ok
```

その他にもC++に対して、Rougeでは`cpp`,`c++`という2つの名前が使えるのに対し、Prism.jsでは`cpp`のみしか使えない等、基本的にRougeの方がエイリアスが多く存在している印象です．

## 対策

Zenn CLIを使っている場合、ファイル保存したタイミングで、`zenn preview`をしたコンソールに`Language does not exist: hogehoge`とログが表示されるので、すぐに気付くことができるでしょう．
Web版はGitHub連携切るのが面倒なので確認はしていませんが、私の記憶が正しく、そして変更がされていなければ、特に警告はされなかったと思います．

正しい名称は https://prismjs.com/#supported-languages で確認できるので、何も変わっていないと思ったら一度確認しておくのがよいでしょう．

## プレーンテキストについて

プレーンテキストに対して、Rougeでは`plaintext`,`text`で指定できますが、Prism.jsではできないようなので（多分）、ファイル名だけ表示したい場合に良い方法がないなと思っています．

例えば、以下のように種類を書かないとファイル名は無視されてしまいます．

```
`​``:出力
Hello World!
`​``
```

```:出力
Hello World!
```

また、以下のようにPrism.jsが認識できない言語名を書くと、ファイル名は表示されるものの、Zenn CLIを使っていると、`Language does not exist: text`と編集中何度も出力されるため、少し気になります．

```
`​``text:出力
Hello World!
`​``
```

```text:出力
Hello World!
```

今は、行頭に+や-が来なければ、`diff`をつければいいかなと思っています．何かもっと良い方法を知っていたら教えてください．

```
`​``diff:出力
Hello World!
`​``
```

```diff:出力
Hello World!
```
