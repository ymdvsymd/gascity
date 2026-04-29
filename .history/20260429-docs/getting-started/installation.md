---
title: インストール
description: Homebrew、リリースの tarball、またはソースから Gas City をインストールします。
---

## どの方法を使うべきか？

| 方法 | 適した用途 | 依存をインストール？ | 自動アップグレード？ |
|--------|----------|----------------|----------------|
| [Homebrew](#homebrew-recommended) | macOS / Linux の日常使用 | はい（6 つすべて） | `brew upgrade` |
| [直接ダウンロード](#direct-download) | CI、コンテナ、エアギャップホスト | いいえ | 手動 |
| [ソースビルド](#build-from-source) | コントリビュータ、最先端 | いいえ | 手動 |

**ほとんどのユーザーは Homebrew を使うべきです。** すべてのランタイム依存関係を自動でインストールし、`gc` を PATH に保ちます。Homebrew が使えない場合（CI イメージ、Docker レイヤ、パッケージマネージャのないマシン）には直接ダウンロードを選んでください。未リリースの変更が必要な場合や貢献を予定している場合はソースを選んでください。

## 前提条件

Gas City は小さなランタイムツールセットを必要とします。Homebrew はそのすべてをインストールします。他の方法では手動インストールが必要です。

| ツール | 必須 | 最低バージョン | macOS | Linux | 注記 |
|------|----------|-------------|-------|-------|-------|
| tmux | はい | — | `brew install tmux` | `apt install tmux` | セッション管理 |
| jq | はい | — | `brew install jq` | `apt install jq` | JSON 処理 |
| git | はい | — | （組み込み） | （組み込み） | バージョン管理 |
| dolt | はい | 1.86.1 | `brew install dolt` | [releases](https://github.com/dolthub/dolt/releases) | Beads データプレーン |
| bd (Beads CLI) | はい | 1.0.0 | `brew install beads` | [releases](https://github.com/gastownhall/beads/releases) | Issue 追跡 |
| flock | はい | — | `brew install flock` | （util-linux 経由で組み込み） | ファイルロック |
| Go 1.25+ | ソースのみ | 1.25 | `brew install go` | [golang.org](https://go.dev/dl/) | コンパイラ |
| make | ソースのみ | — | （組み込み） | `apt install make`（または `build-essential`） | `make install` を駆動 |

CI がピンしている正確なバージョンは [`deps.env`](https://github.com/gastownhall/gascity/blob/main/deps.env) にあります。

## Homebrew（推奨）

```bash
brew install gastownhall/gascity/gascity
```

これは `gastownhall/gascity` formula を tap し、対応する `gc` リリースアセットをダウンロードし、6 つのランタイム依存関係（tmux、jq、git、dolt、flock、beads）をすべてインストールします。

Gas City が homebrew-core に受理されると、通常のインストールパスは `brew install gascity` になります。`gastownhall/gascity` tap は緊急アップデート用に残ります。

インストールを検証します:

```bash
gc version
```

<Warning>
Oh My Zsh の `git` プラグインを使用している場合、`gc` は既に `git commit --verbose` のエイリアスになっている可能性があります。エイリアスをバイパスするには `command gc version` を 1 度実行してください。永続的な修正には、`~/.zshrc` の Oh My Zsh ロード後に `unalias gc 2>/dev/null` を追加するか、`~/.oh-my-zsh/custom/gascity.zsh` のようなファイルにその行を置いてください。
</Warning>

### Homebrew でのアップグレード

```bash
brew update
brew upgrade gascity
```

アップグレード後、supervisor が新しいバイナリを取り込むよう、稼働中の city を再起動してください:

```bash
gc service restart     # launchd/systemd サービスを再起動
```

`gc start` は呼び出すたびにサービスファイルを自動再生成するので、`brew upgrade` の後に `gc start` を実行すれば常にテンプレートの変更が取り込まれます（[v0.13.3 リリースノート](https://github.com/gastownhall/gascity/releases/tag/v0.13.3) を参照）。

### Homebrew でのアンインストール

```bash
gc stop <city-path>                        # まず稼働中の city を停止
brew uninstall gascity
brew untap gastownhall/gascity             # tap を削除
```

## 直接ダウンロード

リリース tarball はタグ付きバージョンごとに発行されます。サポートされるプラットフォーム:

| OS | アーキテクチャ | アーカイブ名 |
|----|-------------|--------------|
| macOS (darwin) | Apple Silicon (arm64) | `gascity_VERSION_darwin_arm64.tar.gz` |
| macOS (darwin) | Intel (amd64) | `gascity_VERSION_darwin_amd64.tar.gz` |
| Linux | x86_64 (amd64) | `gascity_VERSION_linux_amd64.tar.gz` |
| Linux | ARM (arm64) | `gascity_VERSION_linux_arm64.tar.gz` |

### ダウンロードしてインストール

```bash
# 必要なバージョンを設定（https://github.com/gastownhall/gascity/releases を確認）
VERSION=1.0.0

# プラットフォームを検出
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "$ARCH" in
  x86_64)         ARCH=amd64 ;;
  aarch64|arm64)  ARCH=arm64 ;;
esac

# ダウンロードして展開
curl -fsSLO "https://github.com/gastownhall/gascity/releases/download/v${VERSION}/gascity_${VERSION}_${OS}_${ARCH}.tar.gz"
tar -xzf "gascity_${VERSION}_${OS}_${ARCH}.tar.gz"

# PATH 上のディレクトリに移動
sudo install -m 755 gc /usr/local/bin/gc

# 検証
gc version
```

### リリース成果物の検証

Homebrew は formula からリリースのチェックサムを自動的に検証します。直接ダウンロードの場合、インストール前にアーカイブを検証します:

```bash
ARCHIVE="gascity_${VERSION}_${OS}_${ARCH}.tar.gz"
CHECKSUMS="gascity_${VERSION}_checksums.txt"

curl -fsSLO "https://github.com/gastownhall/gascity/releases/download/v${VERSION}/${CHECKSUMS}"
grep "  ${ARCHIVE}$" "${CHECKSUMS}" > "${ARCHIVE}.sha256"

if command -v sha256sum >/dev/null 2>&1; then
  sha256sum -c "${ARCHIVE}.sha256"
else
  shasum -a 256 -c "${ARCHIVE}.sha256"
fi
```

リリースアーカイブは GitHub artifact attestation も付けて発行されます。GitHub CLI がインストールされている場合、ダウンロードしたアーカイブを `gastownhall/gascity` リポジトリに対して検証できます:

```bash
gh attestation verify "${ARCHIVE}" --repo gastownhall/gascity
```

各リリースには SPDX SBOM アセットも含まれます:

```bash
curl -fsSLO "https://github.com/gastownhall/gascity/releases/download/v${VERSION}/gascity-v${VERSION}.spdx.json"
```

### 直接ダウンロードインストールのアップグレード

新しいバージョン番号で上記のダウンロード手順を繰り返します。`gc` バイナリは単一の静的ファイルなので、上書きしても安全です。

<Tip>
直接ダウンロードを使用する場合、[前提条件](#prerequisites)を別途インストールする必要があります。Homebrew はこれを自動的に処理します。
</Tip>

## ソースからビルド

`make` と Go 1.25+（`go.mod` でピン）が必要です。

```bash
git clone https://github.com/gastownhall/gascity.git
cd gascity
make install        # ビルドして $(GOPATH)/bin/gc にインストール
gc version
```

グローバルインストールせずにビルドするには:

```bash
make build          # リポジトリルートに bin/gc を出力
./bin/gc version
```

macOS では、`make build` がバイナリを自動的にアドホック署名します（`codesign -s -`）。

### コントリビュータセットアップ

ビルド後、開発用ツールチェインと pre-commit フックをインストールします:

```bash
make setup
make check          # fmt、lint、vet、ユニットテストを実行
```

完全なコントリビュータワークフローは [CONTRIBUTING.md](https://github.com/gastownhall/gascity/blob/main/CONTRIBUTING.md) を参照してください。

## インストールの検証

インストール方法に関わらず、すべてが動作することを確認します:

```bash
gc version          # インストールされたバージョンとコミットを出力
```

これが Gas City ではなく `git commit` を実行する場合、シェルに `gc` エイリアスが設定されています。このチェックには `command gc version` を使い、永続的な修正方法は[トラブルシューティング](/getting-started/troubleshooting#oh-my-zsh-git-plugin-hides-gc)を参照してください。

次に、最初の city を作成します:

```bash
gc init ~/my-city
cd ~/my-city
```

`gc init` は city を supervisor に登録し、自動的に起動します。完全なウォークスルーについては[クイックスタート](/getting-started/quickstart)を参照してください。

## ドキュメントプレビュー

ドキュメントサイトは [Mintlify](https://mintlify.com) を使用しています。リポジトリルートからローカルでプレビューします:

```bash
./mint.sh dev
```

または、サーバーを起動せずにリンクチェックのみを実行します:

```bash
make check-docs
```
