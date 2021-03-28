---
title: "fishの補完について + `catkin --help` から補完を自動生成する試み"
emoji: 🐠
type: tech
topics: [fish]
published: true
---

fish の好きなところを問われたときに、補完を挙げる人は多いのではないかと思います．かく言う私もその一人で、特に画像のようにグレーでコマンドが浮かびあがっているくる点は気に入っています．

![](https://storage.googleapis.com/zenn-user-upload/1m1mmprms26emn3ybtefy8uy3g4j)

さて、そんな便利な補完ですが、当然勝手に生えているわけではなく、それなりに便利に使おうと思うと、基本的に手作業で実装する必要があります．fish は決して多く使われているわけではないため、どうしても bash や zsh に比べると補完が実装されている可能性は低いです．

そんなときに、自分でなんとかできるようにと思って、補完について調べたことをまとめました．最後に補完スクリプトの自動生成の試みについても書きます．

## 補完スクリプトの場所

そもそも補完スクリプトがどこにあるかご存知でしょうか？`$fish_complete_path`で確認することができます．
自分の環境だと以下の通りでした（\*は未使用）．

- `~/.config/fish/completions`
- `/etc/fish/completions`
- `/usr/share/ubuntu/fish/vendor_completions.d` \*
- `/usr/local/share/fish/vendor_completions.d` \*
- `/usr/share/fish/vendor_completions.d`
- `/var/lib/snapd/desktop/fish/vendor_completions.d`\*
- `/usr/share/fish/completions`
- `~/.local/share/fish/generated_completions`

それぞれの中身について下から見ていきます．

### `~/.local/share/fish/generated_completions`

`fish_update_completions`というコマンドを使うと、man をパースして補完スクリプトを生成することができます．そのデフォルトの出力先です．
例えば、`cmake`コマンドは fish,CMake いずれの公式からも補完を提供されていません．しかし、man があるので、`fish_update_completions`を一度叩くことにより、補完の恩恵を受けることができます．
ただし、`fish_update_completions`も万能ではなく、man から読み取れないことをサジェストすることはできないので、限界はあります．

例えば、`docker run [TAB]`としたときに、手元にあるイメージが候補として出てくるのは、以下のコマンドの出力をもらっているからです．こういったことは man 等のパースでは決してできません．

```shell
docker images --format "{{.Repository}}:{{.Tag}}" | command grep -v '<none>'
```

https://github.com/docker/cli/blob/master/contrib/completion/fish/docker.fish#L76-L78

そういうこともあり優先順位が一番低くされているのでしょう．

### `/usr/share/fish/completions`

ここには fish 自体がメンテしているスクリプトが置いてあります．Linux コマンドや、主要なコマンド（`git`, `gcc`, `cargo` など）の補完スクリプトがおいてあります．

https://github.com/fish-shell/fish-shell/tree/master/share/completions

```terminal
$ exa /usr/share/fish/completions/ | wc -w
777
```

結構多い．

### `/usr/share/fish/vendor_completions.d`

サードパーティライブラリから公式に提供される場合の置き場です．
自分の環境だと、以下の 2 つだけがありました．

- `docker.fish`
- `fwupdmgr.fish`

### `/etc/fish/completions`

システム管理者が全ユーザ共通の設定として置きたい場合に使う．

### `~/.config/fish/completions`

ユーザ設定用です．
`gitignore.fish`や`fisher.fish`等、fisher 系の補完もここに置かれていますし、自分で何か追加したくなったらここに置くことになるでしょう．

また、補完スクリプトを生成するコマンドが存在したり、argcomplete を利用して生成できる場合は、出力先をここにします．

```terminal
$ rustup completions fish >~/.config/fish/completions/rustup.fish
$ register-python-argcomplete --shell fish $pipx >~/.config/fish/completions/pipx.fish
```

自分は以下のような関数を用意して、たまに実行しています．

```fish
function command_exist
    if type $argv[1] >/dev/null 2>&1
        return 0
    else
        echo $argv[1]: not found
        return 1
    end
end

function update_completions
    fish_update_completions

    set -l dir $__fish_config_dir/completions

    for cmd in poetry rustup
        if command_exist $cmd
            $cmd completions fish >$dir/$cmd.fish
        end
    end

    if command_exist register-python-argcomplete
        for cmd in pipx
            if command_exist $cmd
                register-python-argcomplete --shell fish $cmd >$dir/$cmd.fish
            end
        end
    end
end
```

## 補完スクリプトの書き方（リンク）

これを最初テーマに書こうと思いましたが、特に 2 つ目見たらいらないなとなったので以下の記事に譲ります．

https://fishshell.com/docs/current/cmds/complete.html

https://qiita.com/ryotako/items/31f9c9153bece58f2d98

## 自動生成 (WIP)

色々調べてさてわかったぞと、`catkin`コマンド（中身は重要ではないので知らなくてよいです）の補完を作ろうと思って書き始めたものの、正直思いました……面倒だと（メンテナの方々には頭があがりません）．でやめました．
しかし思うわけです．man をパース^[普段見ている man ページは自動生成されているものなので、元ファイルはもう少し機械に優しいです．`/usr/share/man/man1/*.1.gz`とか適当に見てみてください．]して生成できるくらいのレベルなら、`--help`の結果をパースすればできるのではないかと．

ということで、`catkin [subcommand] --help`の結果をパースして、オプションについての補完を生成するスクリプトを作りました．

https://github.com/eduidl/completion-generator

以前から少し気になっていたパーサコンビネータを使ってパースしています^[正直に言うとこっちが目的でした]．言語としては Rust を、パーサコンビネータとしては [nom](https://crates.io/crates/nom) を使っています．

できたものがこちら（同じものを https://github.com/eduidl/catkin.fish/blob/main/catkin.fish にも置いています）．これを手打ちすると思うと中々大変そうです．

:::details catkin.fish

```
complete -c catkin -n __fish_use_subcommand -s h -l help -d "show this help message and exit"
complete -c catkin -n __fish_use_subcommand -s a -l list-aliases -d "Lists the current verb aliases and then quits, all other arguments are ignored"
complete -c catkin -n __fish_use_subcommand  -l test-colors -d "Prints a color test pattern to the screen and then quits, all other arguments are ignored"
complete -c catkin -n __fish_use_subcommand  -l version -d "Prints the catkin_tools version."
complete -c catkin -n __fish_use_subcommand  -l force-color -d "Forces catkin to output in color, even when the terminal does not appear to support it."
complete -c catkin -n __fish_use_subcommand  -l no-color -d "Forces catkin to not use color in the output, regardless of the detect terminal type."

# build
complete -c catkin -n __fish_use_subcommand -a build
complete -c catkin -n '__fish_seen_subcommand_from build' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from build' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l profile -d "The name of a config profile to use (default: active profile)"
complete -c catkin -n '__fish_seen_subcommand_from build' -s n -l dry-run -d "List the packages which will be built with the given arguments without building them."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l get-env -d "Print the environment in which PKGNAME is built to stdout."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l this -d "Build the package containing the current working directory."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-deps -d "Only build specified packages, not their dependencies."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l unbuilt -d "Build packages which have yet to be built."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l start-with -d "Build a given package and those which depend on it, skipping any before it."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l start-with-this -d "Similar to --start-with, starting with the package containing the current directory."
complete -c catkin -n '__fish_seen_subcommand_from build' -s c -l continue-on-failure -d "Try to continue building packages whose dependencies built successfully even if some other requested packages fail to build."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l force-cmake -d "Runs cmake explicitly for each catkin package."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l pre-clean -d "Runs `make clean` before building each package."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-install-lock -d "Prevents serialization of the install steps, which is on by default to prevent file install collisions"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l save-config -d "Save any configuration options in this section for the next build invocation."
complete -c catkin -n '__fish_seen_subcommand_from build' -s j -l jobs -d "Maximum number of build jobs to be distributed across active packages. (default is cpu count)"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l jobserver -d "Use the internal GNU Make job server which will limit the number of Make jobs across all active packages."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-jobserver -d "Disable the internal GNU Make job server, and use an external one (like distcc, for example)."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l env-cache -d "Re-use cached environment variables when re-sourcing a resultspace that has been loaded at a different stage in the task."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-env-cache -d "Don't cache environment variables when re-sourcing the same resultspace."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l cmake-args -d "Arbitrary arguments which are passed to CMake. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-cmake-args -d "Pass no additional arguments to CMake."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l make-args -d "Arbitrary arguments which are passed to make. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-make-args -d "Pass no additional arguments to make (does not affect --catkin-make-args)."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l catkin-make-args -d "Arbitrary arguments which are passed to make but only for catkin packages. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-catkin-make-args -d "Pass no additional arguments to make for catkin packages (does not affect --make-args)."
complete -c catkin -n '__fish_seen_subcommand_from build' -s v -l verbose -d "Print output from commands in ordered blocks once the command finishes."
complete -c catkin -n '__fish_seen_subcommand_from build' -s i -l interleave-output -d "Prevents ordering of command output when multiple commands are running at the same time."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-status -d "Suppresses status line, useful in situations where carriage return is not properly supported."
complete -c catkin -n '__fish_seen_subcommand_from build' -s s -l summarize -d "Adds a build summary to the end of a build; defaults to on with --continue-on-failure, off otherwise"
complete -c catkin -n '__fish_seen_subcommand_from build' -s s -l summary -d "Adds a build summary to the end of a build; defaults to on with --continue-on-failure, off otherwise"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-summarize -d "Explicitly disable the end of build summary"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-summary -d "Explicitly disable the end of build summary"
complete -c catkin -n '__fish_seen_subcommand_from build'  -l override-build-tool-check -d "use to override failure due to using differnt build tools on the same workspace."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l limit-status-rate -d "_STATUS_RATE, --status-rate LIMIT_STATUS_RATE Limit the update rate of the status bar to this frequency. Zero means unlimited. Must be positive, default is 10 Hz."
complete -c catkin -n '__fish_seen_subcommand_from build'  -l no-notify -d "Suppresses system pop-up notification."

# clean
complete -c catkin -n __fish_use_subcommand -a clean
complete -c catkin -n '__fish_seen_subcommand_from clean' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from clean' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l profile -d "The name of a config profile to use (default: active profile)"
complete -c catkin -n '__fish_seen_subcommand_from clean' -s n -l dry-run -d "Show the effects of the clean action without modifying the workspace."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s v -l verbose -d "Verbose status output."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s y -l yes -d "Assume "yes" to all interactive checks."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s f -l force -d "Allow cleaning files outside of the workspace root."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l all-profiles -d "Apply the specified clean operation for all profiles in this workspace."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l deinit -d "De-initialize the workspace, delete all build profiles and configuration. This will also clean subdirectories for all profiles in the workspace."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s l -l logs -d "Remove the entire log space."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s b -l build -d "Remove the entire build space."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s d -l devel -d "Remove the entire devel space."
complete -c catkin -n '__fish_seen_subcommand_from clean' -s i -l install -d "Remove the entire install space."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l dependents -d "Clean the packages which depend on the packages to be cleaned."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l deps -d "Clean the packages which depend on the packages to be cleaned."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l orphans -d "Remove products from packages are no longer in the source space. Note that this also removes packages which are blacklisted or which contain `CATKIN_INGORE` marker files."
complete -c catkin -n '__fish_seen_subcommand_from clean'  -l setup-files -d "Clear the catkin-generated setup files from the devel and install spaces."

# config
complete -c catkin -n __fish_use_subcommand -a config
complete -c catkin -n '__fish_seen_subcommand_from config' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from config' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l profile -d "The name of a config profile to use (default: active profile)"
complete -c catkin -n '__fish_seen_subcommand_from config' -s a -l append-args -d "For list-type arguments, append elements."
complete -c catkin -n '__fish_seen_subcommand_from config' -s r -l remove-args -d "For list-type arguments, remove elements."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l init -d "Initialize a workspace if it does not yet exist."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l extend -d "_PATH, -e EXTEND_PATH Explicitly extend the result-space of another catkin workspace, overriding the value of $CMAKE_PREFIX_PATH."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-extend -d "Un-set the explicit extension of another workspace as set by --extend."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l mkdirs -d "Create directories required by the configuration (e.g. source space) if they do not already exist."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l authors -d "Set the default authors of created packages"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l maintainers -d "Set the default maintainers of created packages"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l licenses -d "Set the default licenses of created packages"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l whitelist -d "Set the packages on the whitelist. If the whitelist is non-empty, only the packages on the whitelist are built with a bare call to `catkin build`."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-whitelist -d "Clear all packages from the whitelist."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l blacklist -d "Set the packages on the blacklist. Packages on the blacklist are not built with a bare call to `catkin build`."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-blacklist -d "Clear all packages from the blacklist."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l build-space -d "_SPACE, -b BUILD_SPACE The path to the build space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l default-build-space -d "Use the default path to the build space ("build")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l devel-space -d "_SPACE, -d DEVEL_SPACE The path to the devel space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l default-devel-space -d "Use the default path to the devel space ("devel")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l install-space -d "_SPACE, -i INSTALL_SPACE The path to the install space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l default-install-space -d "Use the default path to the install space ("install")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l log-space -d "_SPACE, -l LOG_SPACE The path to the log space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l default-log-space -d "Use the default path to the log space ("logs")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l source-space -d "_SPACE, -s SOURCE_SPACE The path to the source space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l default-source-space -d "Use the default path to the source space ("src")"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l link-devel -d "Build products from each catkin package into isolated spaces, then symbolically link them into a merged devel space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l merge-devel -d "Build products from each catkin package into a single merged devel spaces."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l isolate-devel -d "Build products from each catkin package into isolated devel spaces."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l install -d "Causes each package to be installed to the install space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-install -d "Disables installing each package into the install space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l isolate-install -d "Install each catkin package into a separate install space."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l merge-install -d "Install each catkin package into a single merged install space."
complete -c catkin -n '__fish_seen_subcommand_from config' -s j -l jobs -d "Maximum number of build jobs to be distributed across active packages. (default is cpu count)"
complete -c catkin -n '__fish_seen_subcommand_from config'  -l jobserver -d "Use the internal GNU Make job server which will limit the number of Make jobs across all active packages."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-jobserver -d "Disable the internal GNU Make job server, and use an external one (like distcc, for example)."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l env-cache -d "Re-use cached environment variables when re-sourcing a resultspace that has been loaded at a different stage in the task."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-env-cache -d "Don't cache environment variables when re-sourcing the same resultspace."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l cmake-args -d "Arbitrary arguments which are passed to CMake. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-cmake-args -d "Pass no additional arguments to CMake."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l make-args -d "Arbitrary arguments which are passed to make. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-make-args -d "Pass no additional arguments to make (does not affect --catkin-make-args)."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l catkin-make-args -d "Arbitrary arguments which are passed to make but only for catkin packages. It collects all of following arguments until a "--" is read."
complete -c catkin -n '__fish_seen_subcommand_from config'  -l no-catkin-make-args -d "Pass no additional arguments to make for catkin packages (does not affect --make-args)."

# create
complete -c catkin -n __fish_use_subcommand -a create
complete -c catkin -n '__fish_seen_subcommand_from create' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from create' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from create'  -l profile -d "The name of a config profile to use (default: active profile)"

# env
complete -c catkin -n __fish_use_subcommand -a env
complete -c catkin -n '__fish_seen_subcommand_from env' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from env' -s i -l ignore-environment -d "Start with an empty environment."
complete -c catkin -n '__fish_seen_subcommand_from env' -s s -l stdin -d "Read environment variable definitions from stdin. Variables should be given in NAME=VALUE format."

# init
complete -c catkin -n __fish_use_subcommand -a init
complete -c catkin -n '__fish_seen_subcommand_from init' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from init' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from init'  -l reset -d "Reset (delete) all of the metadata for the given workspace."

# list
complete -c catkin -n __fish_use_subcommand -a list
complete -c catkin -n '__fish_seen_subcommand_from list' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from list' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from list'  -l profile -d "The name of a config profile to use (default: active profile)"
complete -c catkin -n '__fish_seen_subcommand_from list'  -l deps -d "Show direct dependencies of each package."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l dependencies -d "Show direct dependencies of each package."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l rdeps -d "Show recursive dependencies of each package."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l recursive-dependencies -d "Show recursive dependencies of each package."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l depends-on -d "[PKG [PKG ...]] Only show packages that directly depend on specific package(s)."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l rdepends-on -d "[PKG [PKG ...]], --recursive-depends-on [PKG [PKG ...]] Only show packages that recursively depend on specific package(s)."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l this -d "Show the package which contains the current working directory."
complete -c catkin -n '__fish_seen_subcommand_from list'  -l directory -d "[DIRECTORY [DIRECTORY ...]], -d [DIRECTORY [DIRECTORY ...]] Pass list of directories process all packages in directory"
complete -c catkin -n '__fish_seen_subcommand_from list'  -l quiet -d "Don't print out detected package warnings."
complete -c catkin -n '__fish_seen_subcommand_from list' -s u -l unformatted -d "Print list without punctuation and additional details."

# locate
complete -c catkin -n __fish_use_subcommand -a locate
complete -c catkin -n '__fish_seen_subcommand_from locate' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from locate' -s w -l workspace -d "The path to the catkin_tools workspace or a directory contained within it (default: ".")"
complete -c catkin -n '__fish_seen_subcommand_from locate'  -l profile -d "The name of a config profile to use (default: active profile)"
complete -c catkin -n '__fish_seen_subcommand_from locate' -s e -l existing-only -d "Only print paths to existing directories."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s r -l relative -d "Print relative paths instead of the absolute paths."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s q -l quiet -d "Suppress warning output."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s s -l src -d "Get the path to the source space."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s b -l build -d "Get the path to the build space."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s d -l devel -d "Get the path to the devel space."
complete -c catkin -n '__fish_seen_subcommand_from locate' -s i -l install -d "Get the path to the install space."
complete -c catkin -n '__fish_seen_subcommand_from locate'  -l this -d "Locate package containing current working directory."
complete -c catkin -n '__fish_seen_subcommand_from locate'  -l shell-verbs -d "Get the path to the shell verbs script."
complete -c catkin -n '__fish_seen_subcommand_from locate'  -l examples -d "Get the path to the examples directory."

# profile
complete -c catkin -n __fish_use_subcommand -a profile
complete -c catkin -n '__fish_seen_subcommand_from profile' -s h -l help -d "show this help message and exit"
complete -c catkin -n '__fish_seen_subcommand_from profile' -s w -l workspace -d "The path to the catkin workspace. Default: current working directory"
```

:::

今はまだ、`catkin` 決め打ちで汎用性が低いので、他のコマンドにもある程度対応できるといいなと思っています．
