# City/Pack Import 管理

**GitHub Issue:** TBD

タイトル: `feat: gc import — import management for schema-2 Gas City packs`

[doc-pack-v2.md](doc-pack-v2.md) ([gastownhall/gascity#360](https://github.com/gastownhall/gascity/issues/360)) の補完文書。これは pack/city モデルと、`gc import` が操作する schema-2 import サーフェスを定義しています。

> **同期の維持:** このファイルがソースオブトゥルースです。GitHub issue が作成されたら、ここで編集してから、`gh issue edit <N> --repo gastownhall/gascity --body-file <(sed -n '/^---BEGIN ISSUE---$/,/^---END ISSUE---$/{ /^---/d; p; }' issues/doc-packman.md)` で issue 本文を更新してください。

## ステータス更新 — 2026-04-19

PackV2 imports のローンチ契約は次のようになります:

- `gc import` が schema-2 のオーサリングと修復のサーフェスです。
- 公式のユーザー編集サーフェスは、root city の `pack.toml` 内の `[imports.<binding>]` です。
- `packs.lock` は完全な推移グラフのコミット済み解決アーティファクトです。
- 通常の load、start、config フローは、宣言された imports、`packs.lock`、ローカルキャッシュの純粋な読み取り側です。フェッチも自己修復もしません。
- `gc import check` は、宣言された imports、lock 状態、ローカルキャッシュ状態に対する読み取り専用のバリデーションサーフェスです。
- `gc import install` は単一の修復コマンドです。必要に応じて宣言された imports から `packs.lock` をブートストラップし、可能な場合は `packs.lock` からキャッシュ状態を復元します。
- このローンチには公開のパッケージレジストリ、ディスカバリ、暗黙 import の話はありません。

---BEGIN ISSUE---

## 課題

PackV2 のローンチには、書き下された 1 つの import 契約が必要です。それまでの設計ドキュメントは以下の点で乖離していました:

- 古いドキュメントが間違った lock ファイル名をまだ使用していたか
- ランタイムエントリーポイントが暗黙的に imports をフェッチまたは修復してよいか
- 暗黙 imports が公開モデルの一部であるか
- パッケージレジストリやディスカバリのサーフェスが `gc import` の一部であるか
- 新規クローンのブートストラップとキャッシュ修復が 1 つのコマンドパスを共有するか

このドキュメントは、PackV2 ドキュメント全体が参照すべき契約を確定させます。

## ローンチ契約

1. **`gc import` が schema-2 import 管理を所有します。** ユーザーは `gc import` と root city の `pack.toml` 内の `[imports.<binding>]` を通じて imported pack を宣言・保守します。
2. **`packs.lock` は解決済み状態の権威です。** 完全な推移グラフと、再現可能な復元のための正確な git コミットを記録します。
3. **通常の load/start/config フローは読み取り専用です。** これらは `pack.toml`、`city.toml`、`packs.lock`、ローカルキャッシュを消費します。クローン、解決、imports の書き換えはしません。
4. **`gc import install` が唯一の修復パスです。** ユーザーは新規クローン、欠落キャッシュ、lock の drift に対してこれを実行します。エラーテキストはこのコマンドを示すべきです。
5. **暗黙 imports は存在しません。** city にとって重要なすべての imported pack は明示的に宣言されます。
6. **このローンチにはパッケージレジストリやディスカバリの話はありません。** imports は公開カタログから発見されるのではなく、ソースで宣言されます。

## オーサリングサーフェス

schema-2 cities は root の `pack.toml` 内で imports を宣言します:

```toml
[pack]
name = "my-city"
version = "0.1.0"

[imports.gastown]
source = "https://github.com/gastownhall/gastown"
version = "^1.2"

[imports.helper]
source = "./assets/helper"
```

binding 名は公開設定サーフェスの一部です:

- `[imports.<binding>]` 配下の TOML キーである
- `packs.lock` で使われる名前である
- 合成されたコンテンツでユーザーが見る名前空間修飾子である

## Import ソースモデル

imports は 1 つの公開ロケータフィールド `source` を持ちます。

一般的なソース形式:

- ファイルシステムパス、例えば `./assets/helper`、`../packs/foo`、または `/abs/path/bar`
- `file://...`
- `https://...`
- `ssh://...`
- `git@...`
- 素の `github.com/org/repo`

解決ルール:

- プレーンディレクトリのターゲットはプレーンディレクトリ import のままで、バージョン選択は使用しない
- git バックのターゲットは semver 制約または明示的な `sha:<commit>` ピンを使用できる
- `gc import add` は格納するソース文字列の正規化と、呼び出し側が制約を省略した場合のデフォルト制約の選択を担う

再現可能な復元セマンティクスを必要とするリモート imports は、`packs.lock` のエントリに解決されなければなりません。ローカルパスの imports は有効なオーサリングサーフェスのままですが、コミットされたリモートロック状態の代わりにはなりません。

## ロックファイル契約

`packs.lock` は city ルートに置かれ、root city のための解決済みグラフを 1 つフラットに記録します:

```toml
[packs.gastown]
source = "https://github.com/gastownhall/gastown"
commit = "abc123..."
version = "1.4.2"
parent = "(root)"

[packs.polecat]
source = "https://github.com/gastownhall/polecat"
commit = "def456..."
version = "0.4.1"
parent = "gastown"
```

ルール:

- root city が 1 つの `packs.lock` を所有する
- imported pack 自体はロックファイルを持たない
- 直接 imports は `parent = "(root)"` を持つ
- 推移 imports は導入された binding 名を `parent` に記録する
- ローダ/ランタイムは `packs.lock` を入力としてのみ使用する

## `gc import install`

`gc import install` はブートストラップパスでもあり、修復パスでもあります。

`packs.lock` が存在し、宣言された imports を満たしている場合:

- `packs.lock` を読む
- 記録されたグラフを共有キャッシュにマテリアライズ
- キャッシュされたコンテンツが lock エントリと一致することを検証

`packs.lock` が存在しない、不完全、または宣言された imports と一致しなくなった場合:

- 宣言された `[imports.<binding>]` からグラフを解決
- 新しい `packs.lock` を書く
- 結果のグラフを共有キャッシュにマテリアライズ

通常の load/start/config パスは、この作業を自分では決して行いません。これらのエントリーポイントが lock やキャッシュ状態の欠落を検出した場合、`gc import install` を実行するよう明確なヒントとともに失敗します。

## ユーザー向けコマンドのセマンティクス

### `gc import add <source>`

- 直接の `[imports.<binding>]` エントリを追加または更新する
- 直接および推移グラフを解決する
- `packs.lock` を書き込みまたは更新する
- 必要なキャッシュエントリをマテリアライズする

### `gc import remove <binding>`

- `[imports.<binding>]` から直接 import を削除する
- 残りのグラフを再計算する
- `packs.lock` を書き直す
- city グラフの一部でなくなったキャッシュエントリを除去する

### `gc import install`

- 必要に応じて宣言された imports から `packs.lock` をブートストラップする
- 可能な場合は `packs.lock` からキャッシュ状態を復元する
- 新規クローン、壊れたキャッシュ、オフライン準備ワークフローで使用される唯一の修復パスを提供する
- 管理対象のリポジトリキャッシュエントリが `packs.lock` から drift した場合、その場で修復する。これは `$HOME/.gc/cache/repos/<key>` 内のローカル編集や追跡外ファイルを破棄する場合がある。なぜならそのディレクトリはマシン管理だから

### `gc import check`

- 宣言された imports をフェッチせずに `packs.lock` に対してバリデートする
- ロックされたキャッシュエントリとキャッシュされた pack ルートが既に存在することをバリデートする
- `gc import install` を修復パスとして指し示しつつ、stale な lock/cache の drift を報告する

### `gc import upgrade [<binding>]`

- 1 つの binding またはグラフ全体を、宣言された制約内で再解決する
- `packs.lock` を書き直す
- 更新されたキャッシュエントリをマテリアライズする

### `gc import list`

- `packs.lock` を読む
- 現在の city の直接および推移 imports を表示する

### `gc import migrate`

- 古い city レイアウト用の移行ツール
- schema-2 cities の主要な day-2 サーフェスではない

## 新規クローン、コールドスタート、オフライン挙動

### コミット済み `packs.lock` を持つ新規クローン

`gc import install` を実行します。これはコミット済みの lock からキャッシュを復元します。

### `packs.lock` のない新規クローン

`gc import install` を実行します。これは宣言された imports を解決し、`packs.lock` を書き、キャッシュを満たします。

### import 状態が欠落した通常の load/start/config

ファストフェイルし、ユーザーに `gc import install` を実行するよう伝えます。

### ミューテーションなしで import 状態を確認

`gc import check` を実行します。これはバージョン解決もフェッチもクローンもファイルの書き換えもしません。欠落した lock エントリ、欠落したキャッシュエントリ、キャッシュチェックアウトの drift、欠落したキャッシュ済み `pack.toml` ファイル、stale な lock エントリを報告します。lock/cache 状態を修復するには `gc import install` を実行します。

### オフライン実行

通常の load/start/config はネットワークフリーのままです。必要な lock やキャッシュ状態が欠けている場合、オフラインのエントリーポイントは依然として失敗します。自己修復を試みません。

## ストレージレイアウト

```text
my-city/
├── pack.toml
├── city.toml
├── packs.lock
└── .gc/
    └── cache/
        └── repos/
            ├── <sha256(normalized-clone-url+commit)>/
            └── ...
```

キャッシュは `gc import` が所有する実装の詳細です。ローダは `gc import install` が用意した解決済みディレクトリを消費します。

## 非ゴール

このローンチでは以下を定義しません:

- 暗黙 imports
- パッケージディスカバリやレジストリブラウジング
- ランタイム側のネットワークフェッチや自動修復
- city ツリーへの imports の vendoring
- `source` と binding 名を超える別個の公開アイデンティティシステム

## 移行ノート

このドキュメントは schema-2 サーフェスを記述します。古い V1 スタイルの `[packs.*]` と `workspace.includes` レイアウトは移行入力として残りますが、新しい PackV2 cities の公式オーサリング契約ではありません。変換マップについては [migrating-to-pack-vnext.md](../guides/migrating-to-pack-vnext.md) を使用してください。

---END ISSUE---
