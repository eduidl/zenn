---
title: "[Rust] トレイトオブジェクトから元の型に戻す"
emoji: 🔙
type: tech
topics: [rust]
published: true
---

`Box<dyn Trait>` から元の型を得る方法を調べてみました。

:::message alert

1. ジェネリクスでよいならその方が楽です （`Vec<Box<dyn T>>` とかしたいなら別）
2. enum でよいならその方が楽です
3. 参照カウント（`Rc` や `Arc`）を使えるなら、`Rc<T>` と `Rc<dyn Trait>` を同時に用意できるため、その方が楽かもしれません

:::

## お題

以下のような `Draw` トレイトと、それを実装した `Triangle` を用意し、 `Box<dyn Draw>` を用意します。
ここから、`&T`, `&mut T`, `Box<T>` を取り出す方法を考えてみます。

```rust
pub trait Draw {
    fn draw(&self);
}

#[derive(Debug)]
pub struct Triangle {
    pub size: f32,
}

impl Triangle {
    const fn new(size: f32) -> Self {
        Self { size }
    }
}

impl Draw for Triangle {
    fn draw(&self) {
        println!("Triangle (size: {})", self.size);
    }
}

fn main() {
    let triangle = Box::new(Triangle::new(1.5)) as Box<dyn Draw>;
    triangle.draw(); //=> Triangle (size: 1.5)
}
```

## `&Box<dyn Trait>` -> `&T` / `&mut Box<dyn Trait>` -> `&mut T`

`downcast_ref<T>` / `downcast_mut<T>` が使えます。

https://doc.rust-lang.org/std/any/trait.Any.html#method.downcast_ref

これは、`impl dyn Any` とトレイトオブジェクトに対して `impl` されています。
通常の具体型であれば、トレイトオブジェクトに変えるのは `as` するだけでできますが、トレイトオブジェクトを別のトレイトオブジェクトに変えるのは意外？と面倒です[^1]
ここでは、新たに `AsAny` というトレイトを用意し、`dyn Any` を得る方法をトレイトオブジェクトに与えておくことで実現します。

```rust
pub trait AsAny {
    fn as_any(&self) -> &dyn Any;

    fn as_any_mut(&mut self) -> &mut dyn Any;
}

pub trait Draw: AsAny {
    fn draw(&self);
}

impl AsAny for Triangle {
    fn as_any(&self) -> &dyn Any {
        self
    }

    fn as_any_mut(&mut self) -> &mut dyn Any {
        self
    }
}
```

例えば以下のようにすることで `AsAny` の実装は楽になります。
ただ、これをすると `Box<dyn Trait>` にも `AsAny` が実装され、`as_ref` 等を明示的に呼ばないと、`Box<dyn Trait>` から直接 `&dyn Any` に変換され、`&T` へのダウンキャストに失敗する[^2]ことには注意です。

```rust
impl<T> AaAny for T
where
    T: 'static,
{
    fn as_any(&self) -> &dyn Any {
        self
    }

    fn as_any_mut(&mut self) -> &mut dyn Any {
        self
    }
}
```

使用例は以下の通り（型がわかりやすいように少し冗長に書いています）

```rust
fn main() {
    let mut triangle = Box::new(Triangle::new(1.5)) as Box<dyn Draw>;
    triangle.draw(); //=> Triangle (size: 1.5)

    let triangle_mut: &mut Triangle = triangle.as_mut().as_any_mut().downcast_mut().unwrap();
    triangle_mut.size = 2.5;
    triangle.draw(); //=> Triangle (size: 2.5)
}
```

## `Box<dyn Trait>` -> `Box<T>`

以下の `downcast` が使えそう、と思いきや `Box<dyn Trait>` から `Box<dyn Any>` への変換がさらに大変そうです。自分では `unsafe` なしで書く方法は思いつきませんでした。

https://doc.rust-lang.org/std/any/trait.Any.html#method.downcast

どうせ `unsafe` 使うならと、 `downcast` の実装をほぼそのまま参考にして以下のように書けました。

```rust
impl dyn Draw {
    fn downcast<T: Draw + 'static>(self: Box<Self>) -> Box<T> {
        assert!(self.as_ref().as_any().is::<T>());
        unsafe {
            let raw = Box::into_raw(self);
            Box::from_raw(raw as *mut T)
        }
    }
}
```

## もう少し実用的な例

この記事を書くきっかけとなったより実用的？な例を以下に載せておきます。

https://github.com/eduidl/rust-playgounnd/blob/main/crates/trait-object/src/main.rs

## あとがき

トレイトオブジェクトは最終手段みたいなところはあるので、使わないにこしたことはないかと思います。
かく言う私も、最近書いていたコードでトレイトオブジェクトを使うつもりでしたが、面倒になって結局 enum にしました。

## 参考

使ったことはありませんが、そういうことをしたいときのための crate もあるみたいです。

https://docs.rs/downcast-rs/latest/downcast_rs/

[^1]: trait upcasting という unstable な機能としてはあるみたいです (https://doc.rust-lang.org/beta/unstable-book/language-features/trait-upcasting.html)
[^2]: `Any` は `TypdId` を使って、自身の型と外から渡された型パラメータの照合をしています。`Box<dyn Trait>` と `T` は型としては全くの別物であるため、`TypeId` も異なります。照合を成功させるためには、 `as_ref` 等により `&dyn Trait` にしてから `&dyn Any` にする必要があります。
