---
title: "[Rust] SixelでターミナルにWebカメラの映像を表示してみる"
emoji: 📸
type: tech
topics: [rust, sixel]
published: true
---

TUI 周りをよく調べていた時期があり、そのときに知ったのが Sixel （かなり古い技術）であった。
画像をターミナルで表示できるということに夢を感じて試してみた。
かなり前に試したが、今更ながら供養として雑に記事にする。

## 実装

https://github.com/eduidl/sixel-webcam-viewer

## 環境

- OS: Ubuntu 22.04
- ターミナル： [wezterm](https://wezfurlong.org/wezterm/)

ターミナルは Sixel が対応するものを選ぶ必要がある。
筆者は特にこだわりがなく、普段 Ubuntu デフォルトの Terminal を使っているが、これは Sixel をサポートしていないため動かない。
特に理由はないが、Zenn で紹介記事を見かけたこともある wezterm を選んだ。

## 実装

メインの sixel の部分に関しては、画像サイズ・フォーマットを指定し、画像データをコピーするだけで動かすことができた。

```rust
use sixel::encoder::{Encoder, QuickFrameBuilder};
use sixel_sys::PixelFormat;

:

    let encoder = Encoder::new().unwrap();
    loop {
        if rx.recv_timeout(Duration::from_millis(1)).is_ok() {
            break;
        }

        execute!(stdout(), MoveTo(0, 0))?;

        let frame = camera.capture()?;
        let qframe = QuickFrameBuilder::new()
            .width(usize::try_from(opt.width)?)
            .height(usize::try_from(opt.height)?)
            .format(PixelFormat::RGB888)
            .pixels((frame[..]).to_vec());

        encoder.encode_bytes(qframe).unwrap();
    }
```

## 実行

フォーマットは RGB888 で決め打ち、デバイス、画像サイズ、FPS はパラメータとして渡せるようにした。

```
$ cargo run --release -- --help
webcam to sixel 0.1.0

USAGE:
    sixel-webcam-viewer [OPTIONS]

FLAGS:
        --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -d, --device <device>     [default: /dev/video0]
    -f, --fps <fps>           [default: 30]
    -h, --height <height>     [default: 480]
    -w, --width <width>       [default: 640]
$ cargo run --release
```

![](https://storage.googleapis.com/zenn-user-upload/32ecf1118578-20230211.png)

何故か右下が欠けているが、一応動いた。

## 処理の重さ

`top` をみるとかなり CPU 使用率が高い。このプログラムを動かしていないときは、wezterm-gui の CPU 使用率はもっと低い（〜数%）ので、ほぼ 2 つの合算と見ることができる。

![](https://storage.googleapis.com/zenn-user-upload/47a75d19eaba-20230211.png)

比較のため Python で OpenCV を使って動かしてみたところ 10% 程度だった。

```python
import cv2

cap = cv2.VideoCapture('/dev/video0')
while True:
    ret, frame = cap.read()

    cv2.imshow("camera",frame)
    cv2.waitKey(0)
```

## 感想

夢を感じて試してみたが、ターミナルの制約や処理の重さなど難しい点もあると感じた。
