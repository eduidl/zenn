---
title: "[Docker] RUN --mount=type=ssh の uid オプション"
emoji: 🐳
type: tech
topics: [docker]
published: true
---

## はじめに

Docker BuildKit はある程度知っている前提で書くので、BuildKit について詳しく知りたい方は公式ドキュメントや他の記事を参照してください。

https://docs.docker.com/build/buildkit/

上だけ読んでも正直よくわからないので、例えば `--mount` のことであれば Dockerfile reference の `RUN` のところを見るなど、機能ごとに探しに行く必要がありそうです。

https://docs.docker.com/engine/reference/builder/

## `--mount` について

例えば、Git のプライベートリポジトリの中身に依存するような Docker イメージをビルドしたくなったとします。
そんなとき、昔であれば（鍵情報を持った）ホストで `git clone` してきて `COPY` コマンドで送るか、ホストに秘密鍵を `COPY` コマンドで送って build 中に `git clone` するという大きく 2 つの方法しかありませんでした。
特に後者であれば、マルチステージビルドを行うなどして、秘密鍵が不用意に残らないように注意する必要がありました [^1]。

そんな中、Docker 18.09 から [BuildKit](https://docs.docker.com/build/buildkit/configure/) というものが利用できるようになり、並列化等いくつかの便利な機能の恩恵に与れるようになりました。
その内の一つが `--mount` であり、Dockerfile 内に書くことで、ファイルシステムをマウントしてビルド中に参照することができます。

https://docs.docker.com/engine/reference/builder/#run---mount

マウントする対象の種類を bind, cache, secret, ssh といったものの中から指定することができ、`git clone`したいときは ssh が使えます。

例えば、次のような内容の Dockerfile を作成して単純にビルドすると、当然 Permission denied となります。

```Dockerfile
FROM ubuntu

RUN apt-get update
RUN apt-get install -y ssh

RUN mkdir -m 700 ~/.ssh
RUN ssh-keyscan github.com > ~/.ssh/known_hosts

RUN ssh -T git@github.com
```

```terminal
$ docker build .
:
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
git@github.com: Permission denied (publickey)
:
```

ところが`--mount=type=ssh`を先頭につけて、更に `DOCKER_BUILDKIT=1` を付けて BuildKit を有効化[^2]し、`--ssh default`を渡すと以下のように SSH の認証が通るようになります。
（ただ、それでも exit code が 1 なのでビルドは失敗しています）

```Dockerfile
FROM ubuntu

RUN apt-get update
RUN apt-get install -y ssh

RUN --mount=type=ssh ssh -o StrictHostKeyChecking=no -T git@github.com
```

```
$ DOCKER_BUILDKIT=1 docker build --ssh default --progress plain build .
:
#9 1.720 Hi eduidl! You've successfully authenticated, but GitHub does not provide shell access.
:
```

## root 以外で動かそうとすると

ところが、dev ユーザを作成し、dev ユーザで同じことをしようとすると失敗します。

```Dockerfile
FROM ubuntu

RUN apt-get update
RUN apt-get install -y ssh

RUN useradd -m -u 1000 dev
USER dev

RUN mkdir -m 700 ~/.ssh
RUN ssh-keyscan github.com > ~/.ssh/known_hosts

RUN --mount=type=ssh ssh -T git@github.com
```

```
$ DOCKER_BUILDKIT=1 docker build --ssh default --progress plain build .
:
#10 1.214 git@github.com: Permission denied (publickey).
:
```

これはデフォルトで uid=0 ユーザにしか見えないようになっているからで、Dockerfile 内で `uid=1000`（`useradd -u 1000`と対応）と明示的に指定すると再び動きます。

```Dockerfile
FROM ubuntu

RUN apt-get update
RUN apt-get install -y ssh

RUN useradd -m -u 1000 dev
USER dev

RUN mkdir -m 700 ~/.ssh
RUN ssh-keyscan github.com > ~/.ssh/known_hosts

RUN --mount=type=ssh,uid=1000 ssh -T git@github.com
```

```
$ DOCKER_BUILDKIT=1 docker build --ssh default --progress plain build .
:
#10 1.942 Hi eduidl! You've successfully authenticated, but GitHub does not provide shell access.
:
```

その他指定可能なオプションは以下にあります。

https://docs.docker.com/engine/reference/builder/#run---mounttypessh

## 参考

全く原因に見当がつかなかったときにこれを見て解決しました

https://stackoverflow.com/questions/74793665/can-i-use-docker-buildkit-to-provide-an-ssh-key-to-a-non-root-user

[^1]: とはいえ、イメージサイズを小さくしようとすると、自然にそうなるような感じもします
[^2]: `/etc/docker/daemon.json` で常に BuildKit を有効化することも可能です（https://docs.docker.com/build/buildkit/#getting-started）
