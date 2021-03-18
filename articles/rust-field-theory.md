---
title: '[Rust] 群・環・体を作る'
emoji: 📐
type: tech
topics: [rust]
published: false
---

群・環・体（たい）は数学上の概念で代数的構造と呼ばれるものですが，簡単に言うと演算の抽象化であり，特に体は四則演算の抽象化です．そして，それらをにプログラミング言語で実装するという試みがいくつかなされています．

- [Swiftで代数学入門 〜 2. 群・環・体の定義](https://qiita.com/taketo1024/items/733e0ecf12da359db729) 
- [Pythonに型付けして代数拡大を実装する (1) 〜モノイド・群・環・整数環〜](https://qiita.com/vbkaisetsu/items/c1cdcfc2bd40b804c0c6)
  - Swiftの記事を参考にしているよう 

自分もやってみたいと思っていたので，自分が好きなRustを用いて実装してみることにしました．
Swiftの先例を参考にしつつ，アレンジを加えています．
数学の説明はSwiftの記事に既に丁寧に書かれているので，そちらを適宜参照してください．

コードは以下に上げています．

https://github.com/eduidl/rust-group-theory

## 注意

オブジェクト指向の話で，正方形クラスが長方形クラスを継承してはいけないみたいな話がありますが，数学的正しさとプログラミング的正しさはしばしば相容れません．
そういった意味で以下の実装は，自分なりに数学的に"正しく"実装しようとした結果，所謂「早すぎる抽象化」に陥っていたり，structやtraitの乱用等，プログラミング的な所作としてはあまり正しくないと思います．
そういったことはわかった上で戯れとしてやっているということで，初心者の方はまかり間違っても参考にしないようにお願いします．

## 群の実装

**群**とは，ある条件を満たす集合$G$とその上の二項演算$\circ:G\times G\rightarrow G$の組から与えられるものです．例えば，「整数全体の集合$\mathbb Z$は加法 $+$ について群をなす」といいます．
後々の実装の都合上群より広い概念である**モノイド**から考えていきます．

### モノイド

モノイドの定義は以下の通りです．

集合$G$とその上の二項演算$\circ:G\times G\rightarrow G$が与えられ、以下の条件を満たすとき，組$(G,\circ)$を**モノイド**という．
1. $G$ の任意の元 $a,b,c$ について $(a\circ b)\circ c=a\circ(b\circ c)$ が成り立つ （**結合法則**）
2. $G$ の任意の元 $a$ について $e\circ a=a\circ e=a$ を満たす $G$ の元 $e$ が存在する（**単位元**の存在）

早速実装を考えていきます．モノイド自体は抽象的な概念なので，トレイトで扱うことにしましょう（以下もトレイトベースで実装していきます）．
Swiftの先例をRustに直すと，こんな感じでしょうか．

```rust
use std::ops::Mul;

trait Monoid: Mul<Output=Self> + Sized {
    // 単位元
    fn identity() -> Self;
}
```

プログラミング的にはこれで十分というか，むしろこれから紹介するのは抽象化しすぎなんですが，これだと一つの集合に対して一つしか実装できないので，数学的には少し正確でないように見えます．モノイドというのは集合と演算のペアによって既定されるので，集合だけからは決まらないからです．
ということで，自分は演算に対してtraitを実装し，集合は関連型で表すようにしました．

```rust
trait Monoid {
    type Element: Clone + Debug + Eq;

    fn call(a: Self::Element, b: Self::Element) -> Self::Element;

    fn identity() -> Self::Element;
}

struct U8Add;

impl Monoid for U8Add {
    type Element = u8;
    
    fn call(a: u8, b: u8) -> u8 {
        a.wrapping_add(b)
    }
    
    fn identity() -> u8 {
    }
}
```

演算子を決め打ちしたくないので，`call`という抽象化した名前にしています．これはこれで，構造体なのにインスタンス作る気がなくて，構造体の乱用感ありますが，後々

```rust
struct Monoid<T, F>
where F: Fn(T, T) -> T {
  op: F,
  identity: T,
}

fn main() {
    let monoid = Monoid {
        op: |a: u32, b: u32| a.wrapping_add(b),
        identity: 0u32
    };
    
    assert_eq!((monoid.op)(2, 3), 5);
}
```

## 環

numという比較的有名なcrateがありますが，その中に，[`One`](https://docs.rs/num/0.3.1/num/traits/trait.One.html)や[`Zero`]というtraitがありますが，これらはこの単位元を意識しています．
