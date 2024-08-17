---
title: "ブラウザバック広告のデモ"
emoji: 🙅
type: idea
topics: [javascript, historyapi]
published: true
---

先日、以下の『ブラウザの履歴を操作して「戻る」ボタンで広告を出すやつについて』という記事が話題になっていました。

https://blog.maripo.org/2024/08/history-api-abuse/

技術的な仕組みをより理解するため、実際に動くデモを作ってみました。ただし、Safari対策等は省略しています。

::: message
お使いの環境によっては動かないかもしれません。
筆者は Linux の Vivaldi, Chrome, Edge, Firefox で動作を確認しました。
:::

https://eduidl.github.io/history-api-demo/

上のリンクを開くと以下のページが開き、

![](https://storage.googleapis.com/zenn-user-upload/ea3bcf1955e3-20240817.png)

そこから戻ろうとすると以下の広告ページ？が出るはずです。

![](https://storage.googleapis.com/zenn-user-upload/6c689c48c8ab-20240817.png)

JavaScript のコード自体は20行もありません。

https://github.com/eduidl/history-api-demo/blob/main/index.html#L37-L53

軽く説明すると、初めてページを開くとこの箇所で履歴が追加されます。
History API といえども、直接過去の履歴を変えることはできないので、 `history.replaceState` で「現在の履歴」を「広告用の履歴」に変えて、本来の「現在の履歴」を `history.pushState` で入れ直しているのがみそでしょうか。
`performance.getEntriesByType("navigation")` は複数回 `pushState` されない対策として雑にやっています。本来、自ドメインリンクがあるはずなので、そのあたりもう少し真面目に対策する必要はあるはずです。リファラのドメインチェックしているのでしょうか？

```js
if (performance.getEntriesByType("navigation")[0].type !== "reload") {
  console.log("履歴追加");
  const currentState = window.history.state;
  window.history.replaceState({ ad: 1 }, "");
  window.history.pushState(currentState, "");
}
```

そして、ユーザが戻るボタンを押すと、`history.replaceState` で追加した「広告用の履歴」によって popstate イベントが発行され、登録したイベントハンドラが呼ばれます。
`e.state` には `history.replaceState` の第1引数のコピーが格納されているため、`e.state.ad === 1` で照合できます。

```js 
window.addEventListener("popstate", (e) => {
  const modal = new bootstrap.Modal(
    document.getElementById("modal")
  );
  if (e.state && e.state.ad === 1) {
    modal.show();
  } else {
    modal.hide();
  }
});
```

---

仕組みは大体わかったので、対策案を個人的に考えました。
副作用に責任をとれないのと、万一対策されても嫌なので公開はしませんが、大枠は関数を差し替えるユーザスクリプトを追加しています。
単純にやるならホワイトリスト方式で一部ドメイン以外は空の関数で置き換えるでも十分かもしれません。

```js
const origPushState = window.history.pushState.bind(window.history);
const origReplaceState = window.history.replaceState.bind(window.history);

window.history.pushState = (...args) => {
    // 何らかの処理
}
window.history.replaceState = (...args) => {
    // 何らかの処理
}
```
