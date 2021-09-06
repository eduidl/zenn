---
title: '[Rust] OptionやResultのハンドリング'
emoji: 🏎
type: tech
topics: [rust]
published: false
---

Rustでは`Option<T>`や`Result<T, E>`を扱うことが多いですが、色々やり方があるので「こういう場合どうするんだ…？」と迷うことありませんか？私は迷います．
ということで、処理方法大全みたい（網羅できているかは怪しい）なのをまとめてみました．
Rustの入門記事ではないので、その点はあしからず．

## 汎用

### `match`式

`match`は式なので値を返せますし、`return`や`?`を使って早期リターンしたりできるので、大体これさえあればなんとかなります．
ただし、往々にしても別のより良い方法があったりします．

```rust
let v = Some(10);
match v {
    Some(v) => println!("v is some {}", v),
    None => println!("v is none"),
}

let v = "10".parse::<u8>();
match v {
    Ok(v) => println!("v is ok: {}", v),
    Err(e) => println!("v is err: {}", e),
}
```

### `if let`

`match`式と異なり、全てのパターンを網羅する必要がないので、「`Some`のときだけある処理をしたい」みたいな場合に有用です．

```rust
let v = Some(10);
if let Some(v) = v {
    println!("v is some {}", v);
}

let v = "10".parse::<u8>();
if let Err(e) = v {
    println!("v is err: {}", e);
}
```

網羅する場合も、`else`等使えば対応できます．
使い分けに関しては私は、`if let`から値を返されるときの個人的に感じる違和感と、`match`の方が場合によってネストが1段深くなることのデメリットを天秤にかけて決めています．
値返さないときは、概ね`if let`使っている気がします．

```rust
let v = Some(10);

// 当然普通は`unwrap_or()`を使うので、あくまで例です
let _ = match v {
    Some(v) => v,
    None => 42,
}

// ---------------------------
let _ = if let Some(v) = v {
    v
} else {
    42,
}

// ---------------------------
match v {
    Ok(v) => {
        // hoge
        // hoge
        // hoge
    }
    _ => {
        // fuga
    },
}

// ---------------------------
if let Some(v) = v {
    // hoge
    // hoge
    // hoge
} else {
    // fuga
}
```

## 値を取り出す

ここでは、`match`や`if let`のような、条件ブロック内で扱える`T`ではなく、同じネストで扱える`T`を作る話です．
そのためには、例えば`Option<T>`の場合、`None`のときに何かしらの処理（何か適当な値で補完するのか etc...）をする必要があります．
ということで、その処理がどういうものかというのを軸にして分類していきます．

### ある値で補完する

別の手段として、`None`や`Err`の場合、にある値で補完するという方法もあります．
明示的に渡された値を使う`unwrap_or()`、`T::default()`を使う`unwrap_or_default()`（もちろん`T: Default`）、渡したクロージャを呼び出したときの返り値を使う`unwrap_or_else()`があります．

```rust
assert_eq!(None.unwrap_or(42), 42);
let a: Option<i32> = None;
assert_eq!(a.unwrap_or_default(), 0);
assert_eq!(None.unwrap_or_else(|| 10 * 10), 100);
```

### `panic!`する

`None`や`Err`のときに`panic!`してしまえば、以降の処理は`T`型の可能性のみを考えればよくなります．

そのためには、`unwrap()`や`expect()`が使えます．`expect()`の方がメッセージを渡せるので、親切です．自信があれば、`unwrap_unchecked()`を使ってもよいでしょう．
ちなみに、`Result<T, E>`から`E`を取り出す、`unwrap_err()`および`unwrap_err_unchecked()`もあります．

```rust
let v: i32 = Some(10).unwrap();
let v: i32 = Some(10).expect("ここにNoneは来ません");
```

### 関数の呼び出し元に返す

以下のように、`None`や`Err`のときそのまま`return`したい場合は`?`演算子を使うことできれいに書くことができます．

```
fn func1() -> Option<i32> {
    // 何らかの処理
}

fn func2() -> Option<i32> {
    let v = match func1() {
        Some(v) => v,
        None => return None,
    );
    Some(v + 10)
}

fn func3() -> Option<i32> {
    Some(func1()? + 10)
}
```

### 入れ子

#### `Option<Result<T, E>>` <-> `Result<Option<T>, E>`

`transpose`が使えます．

```rust
#[derive(Debug, Eq, PartialEq)]
struct SomeErr;

let x: Result<Option<i32>, SomeErr> = Ok(Some(5));
let y: Option<Result<i32, SomeErr>> = Some(Ok(5));
assert_eq!(x, y.transpose());

let x: Result<Option<i32>, SomeErr> = Ok(Some(5));
let y: Option<Result<i32, SomeErr>> = Some(Ok(5));
assert_eq!(y, x.transpose());
```

#### `Option<Option<T>>` -> `Option<T>`

`flatten`が使えます．

#### `Result<Result<T, E>, E>` -> `Result<T, E>`

こちらも`flatten`が使えますが、こちらはnightlyです．
