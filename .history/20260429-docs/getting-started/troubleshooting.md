---
title: Troubleshooting
description: 一般的なインストールおよびセットアップの問題と解決方法。
---

## 組み込みの doctor を実行する

`gc doctor` は city の構造、設定、依存関係、ランタイムの問題をチェックします。常に最初の一手として最適です:

```bash
gc doctor
gc doctor --verbose   # 詳細を追加表示
gc doctor --fix       # 自動修復を試みる
```

## インストール後に "command not found"

`gc` をインストールしたのにシェルから見つけられない場合、バイナリが `PATH` 上にありません。

**Homebrew** はバイナリを通常すでに PATH に含まれているディレクトリに配置します。`brew --prefix` で確認し、`$(brew --prefix)/bin` が `PATH` に含まれているか確認してください。

**直接ダウンロード** の場合、バイナリを PATH 上のディレクトリに移動またはシンボリックリンクする必要があります:

```bash
install -m 755 gc ~/.local/bin/gc   # または /usr/local/bin/gc
```

そして検証します:

```bash
which gc
gc version
```

非標準のシェル（fish、nushell）を使用している場合は、`~/.bashrc` や `~/.zshrc` ではなく、そのシェルの PATH 設定を確認してください。

## Oh My Zsh の Git プラグインが `gc` を隠す

Oh My Zsh の `git` プラグインは `gc` を `git commit --verbose` のエイリアスとして定義します。そのエイリアスがアクティブな間、`gc version`、`gc init`、`gc start` などのコマンドは Gas City バイナリではなく git を実行します。

一時的な回避策:

```bash
command gc version
command gc init ~/my-city
```

`command` はその呼び出しに対してシェルエイリアスをバイパスします。

`~/.zshrc` での恒久的な修正:

```bash
source "$ZSH/oh-my-zsh.sh"
unalias gc 2>/dev/null
```

`unalias` 行は Oh My Zsh のロード **後** に来る必要があります。`source "$ZSH/oh-my-zsh.sh"` の前に書くと、`git` プラグインがあとからエイリアスを再生成します。

Oh My Zsh は組み込みプラグインの後に `$ZSH_CUSTOM` のファイルもロードするため、これも良い代替策です:

```bash
mkdir -p ~/.oh-my-zsh/custom
printf '%s\n' 'unalias gc 2>/dev/null' > ~/.oh-my-zsh/custom/gascity.zsh
```

Oh My Zsh の git エイリアスを使わない場合は、`plugins=(...)` リストから `git` を削除することもできます。

## 前提条件の不足

`gc init` と `gc start` は必要なツールをチェックし、不足しているものを報告します。既存の city 内でより充実したチェックを行うには `gc doctor` も実行できます。

### 常に必須

| ツール | macOS | Debian / Ubuntu |
|------|-------|-----------------|
| tmux | `brew install tmux` | `apt install tmux` |
| git | `brew install git` | `apt install git` |
| jq | `brew install jq` | `apt install jq` |
| pgrep | 同梱 | `apt install procps` |
| lsof | 同梱 | `apt install lsof` |

### デフォルトの beads プロバイダ (`bd`) に必要

| ツール | 最小バージョン | macOS | Linux |
|------|-------------|-------|-------|
| dolt | 1.86.1 | `brew install dolt` | [releases](https://github.com/dolthub/dolt/releases) |
| bd | 1.0.0 | [releases](https://github.com/gastownhall/beads/releases) | [releases](https://github.com/gastownhall/beads/releases) |
| flock | -- | `brew install flock` | `apt install util-linux` |

dolt、bd、flock をインストールしたくない場合は、ファイルベースのストアに切り替えてください:

```bash
export GC_BEADS=file
```

または `city.toml` に以下を追加します:

```toml
[beads]
provider = "file"
```

ファイルプロバイダは Gas City をローカルで試す用途には十分です。`bd` プロバイダは耐久性のあるバージョン管理ストレージを提供し、実際の作業には推奨されます。

## Dolt のバージョンが古すぎる

Gas City は dolt 1.86.1 以降を要求します。バージョンを確認します:

```bash
dolt version
```

Homebrew でアップグレード（`brew upgrade dolt`）するか、[dolthub/dolt/releases](https://github.com/dolthub/dolt/releases) から新しいリリースをダウンロードしてください。

## `bd` のバージョンが古すぎる

Gas City は `bd` 1.0.0 以降を要求します。バージョンを確認します:

```bash
bd version
```

Homebrew でアップグレード（`brew upgrade beads`）するか、[gastownhall/beads/releases](https://github.com/gastownhall/beads/releases) から新しいリリースをダウンロードしてください。

## flock が見つからない (macOS)

macOS には `flock` が同梱されていません。Homebrew でインストールします:

```bash
brew install flock
```

代わりに、ファイルベースの beads プロバイダに切り替えれば（上記参照）flock 要件を完全にスキップできます。

## `gc version` が予期しない出力を表示する

`gc version` がクリーンなバージョン文字列ではなく git の進捗行（`Enumerating objects...`）を表示する場合、Gas City v0.13.4 以降にアップグレードしてください。これはリモート pack の取得が git の sideband 出力をターミナルに書き出すバグで、[PR #141](https://github.com/gastownhall/gascity/pull/141) で修正されました。

## WSL (Windows Subsystem for Linux)

Gas City は WSL 2 上の標準的な Ubuntu または Debian ディストリビューションで動作します。前提条件のインストールには、上の表の Linux 列を使ってください。tmux は動作するターミナルが必要です — Windows Terminal などの WSL 対応ターミナルエミュレータを使ってください。

## ソースからのビルドに失敗する

ソースからビルドするには `make` と Go 1.25 以降が必要です:

```bash
make --version
go version
```

`make` がない場合はインストールしてください（Debian/Ubuntu では `apt install make`、macOS では `xcode-select --install`）。Go のバージョンが古すぎる場合は [go.dev/dl](https://go.dev/dl/) またはパッケージマネージャから更新してください。続いて:

```bash
make build
./bin/gc version
```

完全なコントリビュータセットアップについては [CONTRIBUTING.md](https://github.com/gastownhall/gascity/blob/main/CONTRIBUTING.md) を参照してください。

## それでも解決しない場合

[gastownhall/gascity/issues](https://github.com/gastownhall/gascity/issues) にイシューを作成し、`gc doctor --verbose` の出力と OS/アーキテクチャを記載してください。
