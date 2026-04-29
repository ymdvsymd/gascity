---
title: "PackV2: Gas City の新しいパッケージシステム"
description: 既存の Gas City 0.14.0 の city または pack を PackV2 のスキーマとディレクトリ規約に移行する方法
---

> [!IMPORTANT]
> このドキュメントは Gas City v0.15.0 のプレリリースロールアウトを記述しています。
> PackV2 のいくつかの表面はまだ活発に開発中です。以下のリリースゲート付きの注意事項は
> "As of release v0.15.0, ..." の形式を取ります。

このガイドは、0.14.0 の PackV1 の世界から、0.14.1 で初めて登場し 0.15.0 のウェーブで完成しつつある PackV2 モデルへ移行するための実践的なコンパニオンです。

PackV2 は、city や pack の挙動を記述する方法における複数の問題に対処するためのイニシアチブでした。以下のものが大きく絡み合っていました。
- バージョン管理、共有、多様な文脈で使用できる pack または city の定義。
- マシン固有のプロジェクトディレクトリを city にどう接続するかというデプロイメント設定。
- Gas City がユーザーから不透明に管理する必要のあるランタイム情報。


0.14.0 以前では、city はある意味 pack でしたが、ある意味そうではありませんでした。PackV2 はそれを整理します。

0.14.1 から、Gas City は PackV2 モデルをサポートします。city は pack と同じように定義されますが、追加の `city.toml` を持ちます。

マイグレーションには 2 つのステップがあります。

1. 移植可能な定義（エージェント、formula など）を `pack.toml` および pack 所有のさまざまなディレクトリ（agents、formulas など）に移動する
2. デプロイメント情報（rig など）のみを `city.toml` に残す

3 つ目のレイヤとして `.gc/` がありますが、これは site binding とランタイム状態です。モデル上は重要ですが、ほとんどがユーザーのマイグレーション作業ではないため、このガイドは `pack.toml`、`city.toml`、および pack ディレクトリツリーに焦点を当てます。

公開マイグレーションフローの目標は、`gc doctor` を実行し、安全な機械的書き換えのために `gc doctor --fix` を実行し、結果を確認するために再度 `gc doctor` を実行することです。古い city の中には移行されるまでハードに壊れるものがあるかもしれません。これは v0.15.0 リリース時点で意図された挙動です。

> **現在のロールアウトに関する注意:** doctor 主導の修正スライスは、Skills/MCP、infix、rig パスのスライスとは別にリリースされます。その修正作業がブランチに存在しない間は、`gc import migrate` がトランジショナルなコマンド表面として依然存在することがあります。ターゲットモデルは `gc doctor` の後に `gc doctor --fix` です。

> **コマンド所有権に関する注意:** 現在の製品では、`gc import` は組み込みの Go CLI 表面です。古い bootstrap-pack の実験はレガシーな互換性材料であり、PackV2 のターゲット実装モデルではありません。

> **スコープに関する注意:** このガイドはターゲットの PackV2 マイグレーション形態を記述しています。以下のセクションの一部は、現在のロールアウトの最初のスライスにのみ存在する表面を指しています。それが該当する場合は、ガイド内でインラインで明記し、トラッキング issue へのリンクを記載しています。リリースゲート付きの挙動については、`docs/packv2/skew-analysis.md` および `docs/packv2/doc-conformance-matrix.md` も参照してください。
>
> **最初のスライスに関する注意:** このウェーブでは、skill と MCP は現在の city pack のみが対象です。インポートされた pack のカタログとプロバイダ投影は後続のスライスです。[#588](https://github.com/gastownhall/gascity/issues/588) の `.gc/site.toml` rig パス分割は、現在マイグレーションフローの一部となっています。`rig.path` は `city.toml` から離れ、`rig.prefix` と `rig.suspended` は現在の Phase A ロールアウトでは `city.toml` に残ります。

## はじめる前に

重要な思考の転換は以下です。

- **Gas City 0.14.0** は `city.toml` と多くの明示的なパス配線を中心に据える
- **Gas City 0.14.1 以降** は `pack.toml`、名前付き import、規約ベースのディレクトリを中心に据える

クリーンなターゲット形態は以下です。

- `pack.toml`
  - 移植可能な定義、import、pack 全体のポリシー
- `city.toml`
  - この city のデプロイメント決定
- pack 所有のディレクトリ
  - agents、formulas、orders、commands、doctor チェック、overlay、skills、MCP、テンプレートフラグメント、assets

## 最初に: `city.toml` と `pack.toml` を分割する

これは最も重要なマイグレーションステップです。それ以外のすべてはこれに依存します。

新しいモデルでは、city はデプロイされた pack です。つまり、ルート city ディレクトリに独自の `pack.toml` があり、古い「すべてが `city.toml` に存在する」モデルが分解されます。

### `pack.toml` に属するもの

`pack.toml` は今や移植可能な定義の本拠地となります。

- pack のアイデンティティと互換性メタデータ
- import
- provider
- pack 全体のエージェントデフォルト
- named session
- pack レベルの patch
- その他の pack 全体の宣言的ポリシー

pack 内のすべてのファイルのレジストリであるべきではありません。規約で何かを見つけられるなら、規約を優先します。

### `city.toml` に属するもの

`city.toml` は今やデプロイメントの本拠地となります。

- rig
- rig 固有の構成と patch
- substrate の選択
- API/daemon/runtime の挙動
- キャパシティとスケジューリングのポリシー

pack の移植可能な定義が存在する場所であるべきではありません。

## 最初の具体的なステップ: include を import に移動する

ほとんどの既存 city にとって、最初に実際に行う変更は構成です。

Gas City 0.14.0 では、構成は include ベースです。PackV2 ロールアウトでは、構成は import ベースです。

### 古い city レベル include

```toml
# city.toml
[workspace]
name = "my-city"
includes = ["packs/gastown"]
```

### 新しいルート pack import

```toml
# pack.toml
[pack]
name = "my-city"

[imports.gastown]
source = "../shared/gastown"
```

主な変化は、import がローカル名（ここでは `gastown`）を取得することです。そのローカル名が、pack の他の部分がインポートされたコンテンツを参照する際に使用するものになります。

### 古い rig レベル include

```toml
# city.toml
[[rigs]]
name = "api-server"
path = "/srv/api"
includes = ["../shared/gastown"]
```

### 新しい rig レベル import

```toml
# city.toml
[[rigs]]
name = "api-server"

[rigs.imports.gastown]
source = "../shared/gastown"
```

city 全体の import には city pack の `pack.toml` を使います。pack を 1 つの rig にだけ構成すべき場合は、`city.toml` 内の rig スコープ import を使います。

リモート import の場合、import 宣言を配置した後で `gc import install` を実行します。これにより `packs.lock` の書き込みまたは修復、およびキャッシュのマテリアライズが行われます。読み取り専用の検証パスが必要な場合は `gc import check` を使います。これは欠落または古いロック/キャッシュ状態を報告し、修復のために `gc import install` を案内します。

`city.toml` に残る主なものは rig です。マイグレーションを進める際の通常のパターンは以下です。

- 移植可能な定義を `pack.toml` と pack 所有のディレクトリに移動する
- rig およびその他のデプロイメントの選択を `city.toml` に残す

## 次に: エリアごとに移行する

ルート分割が整えば、残りの作業ははるかに機械的になります。

## エージェント

エージェントはインライン TOML インベントリから出て、エージェントディレクトリに移動します。

### 古い形態

```toml
[[agent]]
name = "mayor"
prompt_template = "prompts/mayor.md"
overlay_dir = "overlays/default"
```

### 新しい形態

```text
agents/
└── mayor/
    ├── prompt.template.md
    └── agent.toml
```

`agent.toml` は、エージェントが共有デフォルトを超えるオーバーライドを必要とする場合のみ使います。

### マイグレーションの注意点

- 各 `[[agent]]` 定義を `agents/<name>/` に移動する
- テンプレート化されたプロンプト内容を `agents/<name>/prompt.template.md` に移動する
- エージェントローカルの overlay 内容を `agents/<name>/overlay/` に移動する
- 共有デフォルトは `[agent_defaults]` に保持する（pack 全体は `pack.toml`、city レベルのオーバーライドは `city.toml`）
- pack 全体の provider は `[providers.*]` に保持する

city を移行している場合、city ローカルのエージェントは依然としてルート city pack 内のエージェントです。

## Formula

Formula はほとんどの部分が既に新しい方向にフィットしています。

### 推奨される形態

```text
formulas/
└── build-review.toml
```

### マイグレーションの注意点

- formula はトップレベルの `formulas/` に置く
- formula の場所を設定可能なパス配線として扱うのをやめる
- ネストされた order を formula 空間から取り出す

## Order

Order は formula により近い形にリファクタリングされています。

整合性監査でも捉えられている現在の方向は以下です。

- order を `formulas/orders/` から取り出す
- トップレベル `orders/` に標準化する
- フラットファイル `orders/<name>.toml` を使う

### 古い形態

```text
formulas/
└── orders/
    └── nightly-sync/
        └── order.toml
```

### 新しい形態

```text
orders/
└── nightly-sync.toml
```

これにより、一貫したペアが得られます。

- `formulas/<name>.toml`
- `orders/<name>.toml`

## Command

Command は規約優先のエントリディレクトリへ向かっています。

### シンプルなケース

```text
commands/
└── status/
    └── run.sh
```

これはデフォルトの単一語コマンドには十分です。

### より豊かなケース

```text
commands/
└── repo-sync/
    ├── command.toml
    ├── run.sh
    └── help.md
```

`command.toml` は、デフォルトのマッピングでは不十分な場合のみ使います。例えば以下の場合です。

- 複数語コマンドの配置
- 拡張ルートの配置
- より豊かなメタデータ
- デフォルト以外のエントリーポイント

### マイグレーションの注意点

古い形:

```toml
[[commands]]
name = "status"
description = "Show status"
script = "commands/status.sh"
```

新しいシンプルなケース:

```text
commands/status/run.sh
```

新しいより豊かなケース:

```text
commands/repo-sync/
├── command.toml
├── run.sh
└── help.md
```

デフォルトの `commands/<name>/run.sh` 発見パスは現在のリリース表面の一部です。`command.toml` はメタデータや明示的なオーバーライドのために任意で残ります。残りの command マニフェスト対称性作業は [#668](https://github.com/gastownhall/gascity/issues/668) で追跡されています。

## Doctor チェック

Doctor チェックは command と並行して移行しています。

### シンプルなケース

```text
doctor/
└── binaries/
    └── run.sh
```

### より豊かなケース

```text
doctor/
└── git-clean/
    ├── doctor.toml
    ├── run.sh
    └── help.md
```

マイグレーションのルールは command と同じです。

- エントリーポイントを使うチェックの近くに置く
- デフォルトのマッピングでは不十分な場合のみローカル TOML を使う

デフォルトの `doctor/<name>/run.sh` 発見パスは現在のリリース表面の一部です。`doctor.toml` はメタデータや明示的なオーバーライドのために任意で残ります。残りの command/doctor マニフェスト対称性作業は [#668](https://github.com/gastownhall/gascity/issues/668) で追跡されています。

## Overlay

Overlay はグローバルなパスバケットから離れ、pack 全体のコンテンツとエージェントローカルのコンテンツの明確な分割へと向かっています。

以下を使います。

- pack 全体の overlay 素材には `overlay/`
- エージェントローカルの overlay 素材には `agents/<name>/overlay/`

古い設定が `overlay_dir = "..."` に依存している場合、マイグレーションのステップは通常、それらのファイルをいずれかの場所に再配置することです。

ローダーは `overlay/`（単数形）のみを発見します — `overlays/`（複数形）という名前のディレクトリはサイレントに無視されます。古いレイアウトやこのガイドの古いドラフトから pack ルートにそうしたディレクトリがある場合は、`overlay/` にリネームしてください。

## Skill、MCP、テンプレートフラグメント

これらはほとんどの部分で新しいディレクトリ構造に直接従います。

以下を使います。

- 現在の city pack の共有 skill には `skills/`
- 現在の city pack の共有 MCP アセットには `mcp/`
- pack 全体のプロンプトフラグメントには `template-fragments/`

そして、

- `agents/<name>/skills/`
- `agents/<name>/mcp/`
- `agents/<name>/template-fragments/`

を使うのは、アセットが特定のエージェントに属するときです。

### Skill のマテリアライズ（v0.15.1 で新規）

**Gas City 0.15.1** から、skill はリスト専用ではなくなりました。サポートされているプロバイダのすべてのエージェント（`claude`、`codex`、`gemini`、`opencode`）は、すべての city pack の skill **および** すべての bootstrap implicit-import pack の skill（例: `core`）を、セッション spawn 前にプロバイダ固有のシンクへシンボリックリンクとしてマテリアライズします。アタッチメントフィルタリングはなく、エージェントは欲しい skill を宣言しません。カタログ全体に加え、自身の `agents/<name>/skills/` ディレクトリが上に乗ります。

**シンクパス**は、エージェントのスコープルート（city スコープ）または rig パス（rig スコープ）に配置されます。

- Claude エージェント: `<scope-root>/.claude/skills/<name>`
- Codex エージェント: `<scope-root>/.codex/skills/<name>`
- Gemini エージェント: `<scope-root>/.gemini/skills/<name>`
- OpenCode エージェント: `<scope-root>/.opencode/skills/<name>`

複数プロバイダの city は、同じスコープルートに兄弟のシンクディレクトリを生成します。`copilot`、`cursor`、`pi`、`omp` エージェントは v0.15.1 ではシンクを持たず、マテリアライズされません。

名前衝突時の**優先度**:

1. エージェントローカル（`agents/<name>/skills/<foo>`）は共有に勝つ。
2. city pack（`skills/<foo>`）は bootstrap implicit import に勝つ。
3. 同じ `(scope-root, vendor)` の 2 つのエージェントは、同じエージェントローカル名を両方提供できない — `gc start` は skill 衝突エラーで失敗する。一方をリネームして修正する。

**ライフサイクル:** 追加、編集、リネーム、削除はすべて、コンテンツハッシュフィンガープリントを通じて影響を受けるエージェントを drain します。すべての supervisor tick がクリーンアップ + 再マテリアライズパスを実行するため、in-place の skill 編集は完全な再起動サイクルなしで反映されます。シンクパスにユーザーが配置したコンテンツ（自分でそこに置いた通常のファイルやディレクトリ）は保持されます。クリーンアップは、ターゲットが既知の gc 管理カタログルート配下にあるシンボリックリンクのみを削除します。

### v0.15.1 で削除 — アタッチメントリストの tombstone

v0.15.0 のアタッチメントリストフィールド — `skills`、`mcp`、`skills_append`、`mcp_append`、およびランタイムのみの `shared_skills` — は **v0.15.1 では非推奨の tombstone** です。アップグレードする city が壊れないようパースは可能ですが、materializer によって無視されます（すべてのエージェントがすべてを取得）。これらのフィールドのいずれかが存在する場合、config ロード時に 1 度だけ警告が発生します。

`city.toml` / `pack.toml` から削除して移行します。`gc doctor --fix` を実行すると自動的に取り除かれます。フィールドは **v0.16** でハードなパースエラーになります。

MCP アクティベーション（MCP 定義をエージェントのプロバイダ設定に投影する）はフォローアップとして追跡されており、v0.15.1 後に `main` にリリースされます。

## フラグメント注入のマイグレーション

古い 3 層プロンプト注入パイプラインは、明示的なテンプレート include に置き換えられます。

| 古いメカニズム | 新しいモデル |
|---|---|
| ワークスペース config の `global_fragments` | 廃止 — コンテンツを `template-fragments/` に移動し、`.template.md` プロンプト内で明示的に `{{ template "name" . }}` を使用 |
| エージェント config の `inject_fragments` | 廃止 — 同じアプローチ |
| patch の `inject_fragments_append` | 廃止 — 同じアプローチ |
| すべての `.md` ファイルが Go テンプレートを通る | `.template.md` ファイルのみが Go テンプレートを通る |

マイグレーション便宜上、`[agent_defaults].append_fragments` は、各プロンプトファイルを編集することなく、名前付きフラグメントを `.template.md` プロンプトに自動追加します。

```toml
# pack.toml or city.toml
[agent_defaults]
append_fragments = ["operational-awareness", "command-glossary"]
```

プレーンな `.md` プロンプトは不活性です — フラグメントは付加されず、テンプレートエンジンも実行されません。

> **As of release v0.15.0:** `[agent_defaults].append_fragments` は現在のリリースにおける実証済みのマイグレーションブリッジです。エージェントローカルな `append_fragments` は依然として spec/runtime のパリティギャップとして [#671](https://github.com/gastownhall/gascity/issues/671) で追跡されています。

## アセットとパス

これは 0.14.0 の多くのアドホックなパスの慣習を置き換えるポジティブなルールです。

### `assets/` はあなたのファイルの不透明な家

ファイルが Gas City が発見に使用する標準表面の一部でない場合、それは `assets/` に属します。

例:

- ヘルパースクリプト
- 静的データファイル
- フィクスチャとテストデータ
- 別の pack 内に運ばれるインポートされた pack ペイロード

### パス値のフィールド

パスを受け取るフィールドは、同じ pack 内の任意のファイルを指すことができます。

これには以下が含まれます。

- 標準ディレクトリ配下のファイル
- `assets/` 配下のファイル
- `..` を使用する相対パス

ハードな制約は以下です。

- 正規化後、パスは pack ルート内に留まる必要がある

### 例

```toml
run = "./run.sh"
help = "./help.md"
run = "../shared/run.sh"
source = "./assets/imports/maintenance"
```

## よくあるマイグレーションの落とし穴

### 「`city.toml` にまだ多くが残っている」

それは通常、定義とデプロイメントが依然として混ざっていることを意味します。

問いかけてください。

- これは移植可能な定義か?
- これはデプロイメントか?

それから以下にそれぞれ移動します。

- `pack.toml` と pack 所有のディレクトリ
- `city.toml`

### 「以前は `scripts/` に依存していた」

0.14.0 にあったからといって、`scripts/` を標準のトップレベル規約として再作成しないでください。

代わりに、

- エントリーポイントスクリプトは、それを使う command または doctor エントリの隣に置く
- 一般的な不透明なヘルパーは `assets/` 配下に置く

例えば、この古いパターン:

```text
scripts/
└── setup.sh
```

と:

```toml
session_setup_script = "scripts/setup.sh"
```

は、スクリプトがエントリローカルか一般的なヘルパーかによって、以下のいずれかになります:

```text
commands/status/run.sh
```

または:

```text
assets/scripts/setup.sh
```

### 「TOML はどこでも必要なのか?」

いいえ。

シンプルなケースは規約で動作するはずです。

- `agents/<name>/prompt.md`
- `commands/<name>/run.sh`
- `doctor/<name>/run.sh`

TOML は実際に以下が必要なときに使います。

- デフォルト
- オーバーライド
- メタデータ
- 明示的な配置


## リファレンス: Gas City 0.14.0 の `city.toml` 要素から PackV2 へ

これは古い `city.toml` スキーマの徹底的なトップレベルルックアップテーブルに加え、マイグレーション中に最も重要な qualified された行です。

> **現在のロールアウトに関する注意:** 以下の行のいくつかは、すべての進行中のブランチの正確な状態ではなく、ターゲット PackV2 の宛先を記述しています。現在の 15.0 ウェーブでは、マシンローカルなワークスペースアイデンティティ（`workspace.name`、`workspace.prefix`）と `rigs.path` は、新しく書かれたまたは移行された city では `.gc/site.toml` に存在します。`rigs.prefix` と `rigs.suspended` はこのリリースでは `city.toml` に残ります。

| 0.14.0 要素 | 何をしたか | 新しい家または対応 |
|---|---|---|
| `include` | ロード前に `city.toml` に追加の config フラグメントをマージしていた | マイグレーションの一部として削除する。実際の構成を import に移し、残った config を `pack.toml`、`city.toml`、または発見されるディレクトリに移動する。 |
| `[workspace]` | city メタデータと pack 構成を 1 か所で保持 | ルート `pack.toml`、`city.toml`、`.gc/` に分割する。 |
| `workspace.name` | ワークスペースアイデンティティ | `.gc/site.toml` に `workspace_name` として移動する。ランタイムアイデンティティは登録済みエイリアス（supervisor 管理フロー）、site binding / レガシー config、ディレクトリ basename の順に解決される。`pack.name` は移植可能な定義アイデンティティおよび init 時のデフォルトとして残る。 |
| `workspace.prefix` | ワークスペースの bead プレフィックス | `.gc/site.toml` に `workspace_prefix` として移動する。ランタイム/API 表面は存在する場合は実効的な site バインドプレフィックスを使用し、それ以外は実効的な city 名から派生する。 |
| `workspace.includes` | city レベルの pack 構成 | ルート city `pack.toml` の `[imports.*]` に移動する。 |

このロールアウトでは、生成されたスキーマ契約も変更されます。チェックインされた `city.toml` ファイルおよび下流バリデータは、ワークスペースアイデンティティが `.gc/site.toml` に移動した後は、`[workspace].name` を必須としてはいけません。

| `workspace.default_rig_includes` | 新しく追加された rig のためのデフォルト pack 構成 | 各デフォルト include をルート city `pack.toml` の `[defaults.rig.imports.<binding>]` エントリに移動する。 |
| `[providers.*]` | 名前付きプロバイダプリセット | 通常はルート city `pack.toml` の `[providers.*]` に移動する。設定が真にデプロイメント専用の場合を除く。 |
| `[packs.*]` | include で使用される名前付きリモート pack ソース | `[imports.*]` エントリに集約する。`city.toml` には独立した `[packs.*]` レジストリは存在しなくなる。 |
| `[[agent]]` | インラインエージェント定義 | `agents/<name>/` に移動し、任意で `agent.toml` を伴う。 |
| `agent.prompt_template` | エージェントプロンプトへのパス | テンプレート化されたプロンプトには `agents/<name>/prompt.template.md` に移動する。プレーンで非テンプレート化された Markdown には `prompt.md` のみを使う。 |
| `agent.overlay_dir` | overlay コンテンツへのパス | コンテンツを `agents/<name>/overlay/` または pack 全体の `overlay/` に移動する。 |
| `agent.session_setup_script` | セットアップスクリプトへのパス | パス値フィールドとして保持するが、pack ローカルファイル（通常はそれを使うものの隣か `assets/` 配下）を指すようにする。 |
| `agent.namepool` | 名前ファイルへのパス | 保持する場合は、`agents/<name>/namepool.txt` のようなエージェントローカルコンテンツへ移行する。 |
| `[[named_session]]` | 名前付き再利用可能セッション | ルート city `pack.toml` の `[[named_session]]` に移動する。 |
| `[[rigs]]` | rig デプロイメントエントリ | `city.toml` に保持する。 |
| `rigs.path` | マシンローカルなプロジェクトバインディング | Phase A の rig binding スライスでは、新しい書き込みではこれをオーサリングされた `city.toml` に永続化しない。古い city は移行されるまでこれを引き続き持つ可能性がある。 |
| `rigs.prefix` | 派生した rig プレフィックス | 現在のリリースウェーブでは `city.toml` に保持する。デプロイメント状態だが、まだ独立した site binding ストレージには抽出されていない。 |
| `rigs.suspended` | オペレーショナルトグル | 現在のリリースウェーブでは `city.toml` に保持する。移植可能な pack 定義というよりデプロイメント/ランタイム状態として残る。 |
| `rigs.includes` | rig スコープの pack 構成 | `city.toml` の rig スコープ import に移動する。 |
| `rigs.overrides` | インポートされたエージェントの rig 固有のカスタマイズ | `city.toml` の rig レベルデプロイメントカスタマイズとして保持する。 |
| `[patches]` | マージ後の修正 | pack 定義 patch を `pack.toml` に移動する。rig 固有の patch は rig とともに `city.toml` に保持する。 |
| `[beads]` | bead store バックエンドの選択 | `city.toml` に保持する。 |
| `[session]` | session substrate config | site ローカルバインディングを除き、`city.toml` に保持する。 |
| `[mail]` | mail substrate config | `city.toml` に保持する。 |
| `[events]` | events substrate config | `city.toml` に保持する。 |
| `[dolt]` | Dolt 接続のデフォルト | `city.toml` に保持する。 |
| `[formulas]` | formula ディレクトリ config | 規約を優先する。残った pack 全体の formula ポリシーがある場合のみ保持する。それ以外は削除する。 |
| `formulas.dir` | formula ディレクトリパス | 固定のトップレベル `formulas/` 規約に置き換える。 |
| `[daemon]` | controller daemon の挙動 | `city.toml` に保持する。 |
| `[orders]` | スキップリストやタイムアウトなどの order ランタイムポリシー | `city.toml` に保持する。 |
| `[api]` | API サーバーデプロイメント config | マシンローカルな bind 詳細を除き、`city.toml` に保持する。 |
| `[chat_sessions]` | チャットセッションランタイムポリシー | `city.toml` に保持する。 |
| `[session_sleep]` | sleep ポリシーのデフォルト | `city.toml` に保持する。 |
| `[convergence]` | 収束限界 | `city.toml` に保持する。 |
| `[[service]]` | ワークスペース所有のサービス宣言 | デプロイメント所有のサービスである場合は `city.toml` に保持する。 |
| `[agent_defaults]` | この city のエージェントに適用されるデフォルト | `pack.toml`（pack 全体の移植可能なデフォルト）と `city.toml`（city レベルのデプロイメントオーバーライド）の両方に存在する。city が pack の上に重なる。v0.15.0 リリース時点では、実際に適用されるデフォルトは依然として狭い: `default_sling_formula` と `[agent_defaults].append_fragments`。 |

## リファレンス: Gas City 0.14.0 の `pack.toml` 要素から PackV2 へ

これは古い共有可能 pack スキーマと、人々が持っている可能性のある過渡期の pack フィールドのルックアップテーブルです。

| 0.14.0 要素 | 何をしたか | 新しい家または対応 |
|---|---|---|
| `[pack]` | pack メタデータ | `pack.toml` に保持する。 |
| `pack.name` | pack アイデンティティ | `[pack]` に保持する。 |
| `pack.version` | pack バージョン | `[pack]` に保持する。 |
| `pack.schema` | pack スキーマバージョン | `[pack]` に保持し、必要に応じて新しいスキーマに更新する。 |
| `pack.requires_gc` | サポートされる最小 gc バージョン | `[pack]` に保持する。 |
| `pack.city_agents` | 古い pack システムの city-vs-rig スタンプヒント | マイグレーション時に再考する。新しいモデルでは、このフィールドではなくエージェントローカル定義とスコープルールを優先する。 |
| `pack.includes` | pack-to-pack 構成 | `pack.toml` の `[imports.*]` に置き換える。 |
| `pack.requires` | pack 要件 | 要件モデルが変更なしで残る場合は `[pack]` に保持する。それ以外は設計ドキュメントの現在の要件形態に移行する。 |
| `[imports.*]` | 過渡期 config の名前付き import | `pack.toml` に保持する。これが新しい構成表面。 |
| `[[agent]]` | インライン pack エージェント定義 | `agents/<name>/` に移動し、任意で `agent.toml` を伴う。 |
| `agent.prompt_template` | エージェントプロンプトファイルパス | テンプレート化されたプロンプトには `agents/<name>/prompt.template.md` に移動する。プレーンで非テンプレート化された Markdown には `prompt.md` のみを使う。 |
| `agent.overlay_dir` | エージェント overlay パス | コンテンツを `agents/<name>/overlay/` または `overlay/` に移動する。 |
| `agent.session_setup_script` | エージェントセットアップスクリプトパス | pack ローカルファイルを指すパス値フィールドとして保持する。 |
| `[[named_session]]` | pack 定義の名前付きセッション | `pack.toml` に保持する。 |
| `[[service]]` | pack 定義のサービス | 新しいモデルで pack 定義のままサービスが残る場合のみ保持する。それ以外は city 所有のサービスを `city.toml` に移動する。 |
| `[providers.*]` | pack で使用されるプロバイダプリセット | `pack.toml` に保持する。 |
| `[formulas]` | formula ディレクトリ config | 規約を優先する。ディレクトリ配線を削除し、トップレベル `formulas/` を使う。 |
| `formulas.dir` | formula ディレクトリパス | トップレベル `formulas/` に置き換える。 |
| `[patches]` | pack レベルの patch ルール | `pack.toml` に保持する。 |
| `[[doctor]]` | pack doctor インベントリ | デフォルトで `doctor/<name>/run.sh` へ移行する。必要に応じて任意で `doctor.toml` を伴う。 |
| `doctor.script` | doctor エントリーポイントへのパス | pack ローカルパスとして保持する。通常は `doctor/<name>/run.sh`。 |
| `[[commands]]` | pack コマンドインベントリ | デフォルトで `commands/<name>/run.sh` へ移行する。必要に応じて任意で `command.toml` を伴う。 |
| `commands.script` | command エントリーポイントへのパス | pack ローカルパスとして保持する。通常は `commands/<name>/run.sh`。 |
| `[global]` | pack 全体の session-live 挙動 | pack グローバル表面が設計どおりに残る場合は `pack.toml` に保持する。 |

## リファレンス: 古いトップレベルディレクトリ

このテーブルは上記の 2 つのスキーマテーブルのファイルシステムコンパニオンです。

| 古いディレクトリまたはパターン | 0.14.0 での意味 | 新しい家または対応 |
|---|---|---|
| `prompts/` | パスでアドレスされるプロンプトテンプレートの共有バケット | テンプレート化されたプロンプトには `agents/<name>/prompt.template.md` にプロンプトコンテンツを移動する。プレーンで非テンプレート化された Markdown には `prompt.md` のみを使う。 |
| `scripts/` | ヘルパーおよびエントリーポイントスクリプトの共有バケット | 標準のトップレベルディレクトリとして保持しない。エントリーポイントスクリプトはそれを使うものの隣に置き、一般的なヘルパーは `assets/` 配下に置く。 |
| `formulas/` | formula ディレクトリ。場合により TOML でパス配線される | 固定のトップレベル `formulas/` 規約として保持する。 |
| `formulas/orders/` | formula 配下のネストされた order 定義 | フラットな `*.toml` ファイルを使ってトップレベル `orders/` に移動する。 |
| `orders/` | 一部の city でのトップレベル order ディレクトリ | この場所に標準化するが、フラットな `orders/<name>.toml` ファイルを使う。 |
| `overlay/` | pack 全体の overlay バケット | トップレベル `overlay/` として保持する。エージェントローカル overlay は `agents/<name>/overlay/` 配下に存在する。 |
| `overlays/` | 古い pack やこのガイドの古いドラフトで複数形で名付けられた pack 全体の overlay バケット | `overlay/` にリネームする — ローダーは単数形のみを発見する。 |
| `namepools/` | エージェント名プールの共有バケット | 保持する場合はエージェントローカルファイルに移行する。 |
| アドホックなスクリプトを持つ `commands/` | コマンドヘルパーディレクトリと TOML 配線 | `commands/` を保持するが、`commands/<name>/run.sh` のようなエントリディレクトリとして整理する。 |
| アドホックなスクリプトを持つ `doctor/` | doctor ヘルパーディレクトリと TOML 配線 | `doctor/` を保持するが、`doctor/<name>/run.sh` のようなエントリディレクトリとして整理する。 |
| `skills/` | 新しいレイアウトの現在の city pack の skill ディレクトリ | トップレベル `skills/` として保持する。 |
| `mcp/` | 新しいレイアウトの現在の city pack の MCP ディレクトリ | トップレベル `mcp/` として保持する。 |
| `template-fragments/` | 新しいレイアウトの共有プロンプトフラグメントディレクトリ | トップレベル `template-fragments/` として保持する。 |
| `packs/` | ローカルにベンダーされた pack または bootstrap import | 標準のトップレベルディレクトリとして扱わない。不透明な埋め込み pack が必要な場合は `assets/` 配下に置き、明示的にインポートする。 |
| pack ルートの緩いヘルパーファイル | 制御された表面に混ざる任意のファイル | `README.md`、`LICENSE*`、`CONTRIBUTING.md`、`CHANGELOG*` のような標準のリポジトリドキュメントは pack ルートに保持する。その他の不透明なヘルパーは `assets/` 配下に移動する。 |

## 推奨されるマイグレーション順序

実際の city または pack について、最も実践的な順序は以下です。

1. ルート `pack.toml` を追加する
2. `workspace.includes` と `rigs.includes` を import に移動する
3. エージェント定義を `agents/` に移動する
4. order をトップレベルのフラットファイルに移動する
5. command と doctor チェックを `commands/` と `doctor/` に移動する
6. 不透明なヘルパーを `assets/` に移動する
7. `city.toml` と `pack.toml` に残ったものを上記のリファレンステーブルを使ってクリーンアップする

これにより、より小さなクリーンアップ作業に時間を費やす前に、大きな構造変更を完了できます。
