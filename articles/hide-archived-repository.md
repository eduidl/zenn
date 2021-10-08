---
title: GitHubのアーカイブ済みのリポジトリを目立たなくするUser Script
emoji: 🐙
type: idea
topics: [JavaScript, GitHub, userscript]
published: true
---

## はじめに

GitHubではリポジトリをアーカイブする機能があります．
ところで、皆さんはリポジトリ一覧で見たときに、アーカイブされているものでも、されていないものと同じように表示されることに不満を覚えたことはないでしょうか？私はあります．
ということで、アーカイブ済みのリポジトリのopacityを下げて、目立たなくさせるスクリプトを作りました．


## 作ったもの

https://gist.github.com/eduidl/3a408c67a5b486ee4002104c4afcea82

## 動作例

以下のように「Public archive」または「Private archive」であるリポジトリの表示が薄くなり、目立たなくなります．

![](https://storage.googleapis.com/zenn-user-upload/7fa373fbb5f99fa685b14da6.png)

完全に非表示というのも考えましたが、再表示等のロジックが面倒なのでやめました．

## 実装

アーカイブされていたら、`opacity`を0.3にしています．お好みに応じて変えてください．

```js
// ==UserScript==
// @name         HideArchivedRepository
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  Hide archived repository
// @author       eduidl
// @match        https://github.com/*
// @icon         https://github.com/favicon.ico
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const getRepos = () => {
        let container = document.getElementById('org-repositories');
        if (container == undefined) {
            container = document.getElementById('user-repositories-list');
        }
        if (container == undefined) {
            return [];
        }

        return container.getElementsByTagName('li');
    };

    const archived = (repo) => {
        const tags = repo.querySelectorAll('span.Label');
        for (const tag of tags) {
            if (tag.innerText.includes('archive')) {
                return true;
            }
        }
        return false;
    };

    const hideArchived = () => {
        const repos = getRepos();
        for (const repo of repos) {
            repo.style.opacity = archived(repo) ? 0.3 : 1.0;
        }
    };


    (function run() {
        hideArchived();
        setTimeout(run, 200);
    })();
})();
```
