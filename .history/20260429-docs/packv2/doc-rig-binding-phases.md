# Rig バインディングフェーズ

ドキュメント状態: 移行期の真実

GitHub issues:
- [gastownhall/gascity#588](https://github.com/gastownhall/gascity/issues/588)
- [gastownhall/gascity#587](https://github.com/gastownhall/gascity/issues/587)

このノートは、rig バインディングとマルチ city リギングの現行ワーキング POR を記録します。意図的に作業を 2 つのフェーズに分けています:

- フェーズ A: 15.0 以前のパス抽出
- フェーズ B: 15.0 以降のマルチ city rig 共有

実行姿勢:

- フェーズ A は現在の 15.0 リリースブランチに属します。
- フェーズ B は現在のブランチや現在の big-test-pass ゲートには属しません。
- フェーズ B は、このブランチの直後の次のスライスではなく、15.0 以降の別個のフォローオン期間として扱ってください。

## なぜ分割が存在するのか

これらの 2 つの懸念事項は関連していますが、同じ変更ではありません:

1. `city.toml` から `rig.path` を削除する
2. 複数の city が同じディレクトリをバインドできるようにする

フェーズ A は狭い状態モデルのクリーンアップであり、15.0 のためにできるだけリスクをゼロに近づけて維持すべきです。

フェーズ B は bead ストレージとリダイレクト挙動に触れます。その作業は 15.0 のローンチパスと現在の統合ブランチの外に留めます。

## 共有アイデンティティモデル

これらの用語は両フェーズで一貫して使われます。

### City 名

- 登録時に割り当てられる
- `workspace.name` に依存しない
- 登録された city の運用ライフタイムにわたって安定
- マシンローカルの登録空間内で一意

### City プレフィックス

- グローバルに一意である必要はない
- 運用上の便宜フィールドであり、正準のマシングローバルアイデンティティではない

### Rig 名

- city ローカルなアイデンティティ
- city 内でのみ一意
- `city.toml` 内の rig 宣言とマシンローカルバインディング状態を相関させるのに使用

### Rig プレフィックス

- rig の安定した bead 名前空間
- ユーザー制御
- city 内でのみ一意
- `city.toml` に残る

## フェーズ A: `city.toml` から `rig.path` を削除する

フェーズ A は 15.0 で安全なクリーンアップです。

### ゴール

マシンローカルな rig パスバインディングのみを `city.toml` から外します。

### 何が変わるか

- `rig.path` が `city.toml` から離れる
- `rig.name` は `city.toml` に残る
- `rig.prefix` は `city.toml` に残る
- `rig.suspended` は今は `city.toml` に残る
- bead ストレージの挙動は実質的にそのまま
- マルチ city 共有 rig セマンティクスは導入されない

### バインディングキー

相関キーは:

- `(cityPath, rigName)`

これは以下の結合です:

- `city.toml` 内の rig 宣言
- マシンローカルな rig バインディング状態

### リネームのセマンティクス

ユーザーが `city.toml` 内の rig 名を手動で編集した場合、それはアイデンティティ変更です。

既存のマシンローカルバインディングは一致しなくなり、修復されるまで rig はバインドされていないものとして扱われます。

システムは意図を推測したり、サイレントにバインディングを移行したりすべきではありません。

### Doctor 契約

- `gc doctor`
  - バインディング/名前の不一致を検出して報告する
  - デフォルトでは修復しない
- `gc doctor --fix`
  - 復旧パスが曖昧でない場合に修復してよい

### 移行姿勢

これはハードブレイクです。

- `city.toml` 内のレガシー `rig.path` 互換性を維持しない
- 移行するユーザーは rig を再登録または再バインドする必要があるかもしれない

### 復旧ツール

フェーズ A で実現可能であれば、マシンローカル rig バインディング向けの明示的なインポート/エクスポートを追加します:

- `gc rig bindings export <path>`
- `gc rig bindings import <path>`

これは新しいマシンでの暗黙的なパス推測よりも望ましいです。

操作は city スコープです:

- city ルートから呼び出してよい
- またはその city に解決される任意の rig 化されたディレクトリから呼び出してよい

エクスポートされたファイルは 1 つの city とその rig パスバインディングを表し、マシングローバルなダンプではありません。

#### フェーズ A のインポート/エクスポートファイル

正確な形式は進化する可能性がありますが、想定される形は:

```toml
version = 1

[city]
path = "/Users/dbox/repos/gc/cities/backstage"
name = "backstage"

[[rigs]]
name = "api-server"
path = "/Users/dbox/src/api-server"

[[rigs]]
name = "frontend"
path = "/Users/dbox/src/frontend"
```

フェーズ A では、ファイルは city パス、city 名、または両方を運ぶことができます。実際の使用がどのフィールドが時間経過でプライマリになるかを明らかにすると予想されます。

#### インポートバリデーションのセマンティクス

インポートは何かを書く前にファイル全体をバリデートします。

- 参照される rig パスが欠落または無効な場合:
  - エラー
  - 何もバインドしない
- ファイルがターゲット `city.toml` に存在しない rig 名を参照している場合:
  - エラー
  - 何もバインドしない
- `city.toml` にインポートファイルに存在しない rig 名が含まれている場合:
  - 許可される
  - それらの rig は未バインドのまま

インポートは必要に応じて city を登録し、できる限り 1 つのトランザクションとして rig バインディングを適用すべきです。

### フェーズ A の非ゴール

- 複数 city にまたがる共有 rig ディレクトリ
- bead ストレージを city の `.gc/` 配下に移動
- リダイレクト駆動の rig ルート `.beads`
- `~/.gc/cities.toml` 内のリッチなマルチ city rig レコード

## フェーズ B: マルチ city rig 共有

フェーズ B は 15.0 以降の作業です。

意図的に現在の 15.0 ブランチの一部ではなく、現在の big-test-pass ゲートの一部でもなく、リリース安定化後の別個のフォローオン実装期間として扱われるべきです。

### ゴール

2 つ以上の city が同じディレクトリを安全にバインドできるようにします。

### Rig レジストリモデル

共有 rig ディレクトリのために、マシングローバルレコードは以下を追跡すべきです:

- `path`
- `bindings = [...]`
- 任意の `default_binding`

各バインディングは以下のタプルです:

- `city`
- `rig`

`default_binding` が存在する場合、それはリストされたバインディングのうちの 1 つでなければなりません。

形の例:

```toml
[[rigs]]
path = "/Users/dbox/src/shared-rig"

bindings = [
  { city = "/Users/dbox/repos/gc/cities/backstage", rig = "api-server" },
  { city = "/Users/dbox/repos/gc/cities/switchboard", rig = "api-server" },
]

default_binding = { city = "/Users/dbox/repos/gc/cities/backstage", rig = "api-server" }
```

### Bead ストレージモデル

実際の bead ストアは city の `.gc/` 配下に移動します。

rig ルートの `.beads` アーティファクトは、選択された city の管理対象 bead ストアを指すだけのリダイレクトシムになります。

### デフォルト切り替え

`gc rig set-default <path>` は、可能な限りアトミックに両方を更新します:

- `~/.gc/cities.toml` 内の `default_binding` レコード
- rig ルートの `.beads` リダイレクトターゲット

### フェーズ B のメモ

これは 15.0 以降に意図的に延期されています。bead 関連コードは高リスク領域であり、ローンチパスでチャーンさせるべきではないからです。

実用的な計画ルール:

- フェーズ B の真実は今書き留める
- 現在のブランチに混ぜない
- 15.0 の安定化をブロックさせない

## フィールドサマリ

### `city.toml` に残す

- `rig.name`
- `rig.prefix`
- `rig.suspended`（今のところ）
- `rig.imports`
- `rig.max_active_sessions`
- `rig.patches`
- `rig.default_sling_target`
- `rig.session_sleep`
- `rig.dolt_host`
- `rig.dolt_port`

### フェーズ A で `city.toml` から外す

- `rig.path`

### レガシーフィールド

これらは新しいバインディングモデルの一部ではありません:

- `rig.includes`
- `rig.overrides`
- `rig.formulas_dir`

これらは移行 / ハードフェイルの話に属するもので、フェーズ A の設計には含まれません。
