# V2 Loader & Pack Composition — 設計

> **ステータス:** [doc-pack-v2.md](doc-pack-v2.md)（[gastownhall/gascity#360](https://github.com/gastownhall/gascity/issues/360)）で提案された v.next ローダーの設計説明です。
> 現行のリリースブランチローダーの動作と、ここで取り扱う v.next 設計のコンパニオンドキュメントとなります。
> 両者を並べて読むことで差分を確認できます。

## 概念的な概観

V2 ではロードに関する考え方を 5 つのアイデアで再構築しています。これらはすべて V1 では欠落しているか、不十分な部分です。

1. **city は pack である。** 構成のルートは `pack.toml`（*定義*）に加え、コンパニオンとなる `city.toml`（*デプロイメントプラン*）と `.gc/`（マシンごとの *site binding*）です。`city.toml` を削除しても、残ったものは正当な、インポート可能な pack となります。ローダーのルートケースは「pack をロードしてからその上にデプロイメントファイルを重ねる」という形になります。

2. **import が include を置き換える。** pack は、テキストの連結ではなく、名前付きバインディング（`[imports.gastown]`）を通じて他の pack を構成します。各 import は、構成後にも残る恒久的な名前を持ちます。ロード後、`gastown.mayor` は実在する、アドレス可能なものになります — `mayor` という名前を持つ単なる `Agent` ではありません。import はバージョン管理され、エイリアス可能であり、デフォルトで推移的です（`transitive = false` でオプトアウト可能）。

3. **規約が構造を定義する。** pack のファイルシステムレイアウト *そのもの* が宣言です。`agents/foo/` が存在するなら、`foo` という名前のエージェントが存在することになります。`formulas/bar.formula.toml` が存在するなら、`bar` という名前の formula が存在することになります。ローダーは明示的な `[[agent]]` や `[[formula]].path` 宣言を読むのではなく、標準ディレクトリを走査することでコンテンツを発見します。

4. **pack は自己完結している。** pack の推移的閉包は、そのディレクトリツリーと宣言された import の集合です。pack ディレクトリ外を解決するパスはロード時エラーとなります。これにより、V1 の pack にはなかった可搬性が pack に与えられます。

5. **定義 / デプロイメント / site binding が物理的に分離されている。** `pack.toml` は定義を担います。`city.toml` はデプロイメント（rig、サブストレート、キャパシティ）を担います。`.gc/` は site binding（パス、プレフィックス、suspend フラグ、マシンローカルの認証情報）を担います。ローダーは 3 つすべてから読み取りますが、それらを混同しません。`gc rig add` のようなコマンドは、チェックインされた TOML ではなく `.gc/` に書き込みます。

最初の skills/MCP のスライスでは、ローダーは現在の city pack の `skills/` および `mcp/` カタログのみを発見します。インポートされた pack のカタログは後続のウェーブで対応します。

出力は依然としてフラット化された `City` 値と `Provenance` ですが、内部モデルでは衝突をロード順で解決するのではなく、すべてに渡って qualified name を保持します。

## トップレベルのエントリーポイント

唯一の公開エントリーポイントは概念的には変わりませんが、入力がより広範になります。単一の TOML ファイルではなく city ディレクトリです。

```go
// internal/config/config.go (proposed)
func LoadCity(
    fs fsys.FS,
    cityDir string,
    extraIncludes ...string,
) (*City, *Provenance, error)
```

`cityDir` は `pack.toml` と（オプションで）`city.toml` および `.gc/` を含むディレクトリです。`extraIncludes` は引き続き CLI で指定されるフラグメントパスを意味し、`-f` との互換性、およびシステムフラグメントを注入する必要のある `gc init` フローのために残されています。

`Provenance` は V1 の監査証跡を拡張し、ソースファイルに加えて *qualified name* と *import バインディング* を追跡します。

```go
type Provenance struct {
    Root          string
    Sources       []string
    Agents        map[string]string  // qualified name → file
    Imports       map[string]ImportProvenance  // binding name → import details
    Rigs          map[string]string
    SiteBindings  map[string]string  // .gc/-sourced fields
    Workspace     map[string]string
    Warnings      []string
}

type ImportProvenance struct {
    BindingName  string  // 例: "gastown" またはエイリアス "gs"
    PackName     string  // インポートされた pack の pack.name
    Source       string  // 解決された source 文字列
    Version      string  // 解決されたバージョン（semver または "local"）
    Commit       string  // リモート import の場合の解決済みコミットハッシュ
    Exported     bool    // 親による再エクスポート
    Path         string  // フェッチ後のディスク上の場所
}
```

## コアデータ構造

### `City`

```go
type City struct {
    // ルート pack — この city が「何であるか」。
    Pack         Pack

    // デプロイメントファイル — この city が「どう動くか」。
    Deployment   Deployment

    // Site binding — マシンローカルのアタッチメント。
    SiteBinding  SiteBinding

    // 構成済みビュー（派生）。
    Agents             []Agent
    NamedSessions      []NamedSession
    Providers          map[string]ProviderSpec
    FormulaLayers      FormulaLayers
    OverlayLayers      OverlayLayers
    Patches            Patches             // 構成中に適用される

    // rig ごとの構成済みビュー。
    Rigs               []Rig

    // 解決済みの import グラフ（インスペクション / gc コマンド用）。
    ImportGraph        ImportGraph

    // 派生されたアイデンティティ。
    ResolvedWorkspaceName string  // SiteBinding または Pack.Meta.Name のフォールバック
}
```

`Pack`、`Deployment`、`SiteBinding` は 3 つのディスク上の入力です。それ以外はすべて構成中に派生します。

### `Pack`

`pack.toml` の内容です。

```go
type Pack struct {
    Meta             PackMeta
    Imports          map[string]Import
    DefaultRig       DefaultRigPolicy   // [defaults.rig.imports.<binding>]
    AgentDefaults    AgentDefaults      // [agent_defaults]
    Providers        map[string]ProviderSpec
    NamedSessions    []NamedSession
    Patches          Patches
}

type PackMeta struct {
    Name        string
    Version     string
    Schema      int
    RequiresGc  string
    Description string
}
```

注目すべきは存在しないものです。`[[agent]]`、`[[formula]]`、`[[order]]`、`[[script]]`、`overlay_dir`、`prompt_template`、`formulas_dir`、`scripts_dir`。これらすべてがディレクトリ走査に置き換わります。

### `Import`

```go
type Import struct {
    Source   string  // ./packs/x, github.com/org/x, など
    Version  string  // semver 制約。ローカルパスでは空
    Export   bool    // 親への再エクスポート
    // ロード時に解決される項目:
    Path         string  // ディスク上の場所
    ResolvedVer  string  // コミットハッシュまたは "local"
    Pack         *Pack   // ロードされた pack のメタデータ
}
```

`Source` は V1 の include と同じ 3 つの形式（ローカル、リモート git、github tree URL）を受け付けますが、*意味* が異なります。import はインライン挿入ではなく、名前付きバインディングです。

### `Deployment` (city.toml)

```go
type Deployment struct {
    Beads        BeadsConfig
    Session      SessionConfig
    Mail         MailConfig
    Events       EventsConfig
    Daemon       DaemonConfig
    Orders       OrdersConfig
    API          APIConfig

    Rigs         []DeploymentRig
}

type DeploymentRig struct {
    Name              string
    Imports           map[string]Import  // [rigs.imports.X]
    Patches           Patches
    MaxActiveSessions int
    DefaultSlingTarget string
    SessionSleep      DurationConfig
    // ...その他のデプロイメントノブ
}
```

`city.toml` から特に取り除かれたもの: アイデンティティ（`workspace.name`）、site binding（`rig.path`、`rig.prefix`、`rig.suspended`）、または `[pack]` の内容。

### `SiteBinding` (`.gc/`)

```go
type SiteBinding struct {
    WorkspaceName    string
    WorkspacePrefix  string

    RigBindings      map[string]RigBinding   // by rig name
    LocalConfig      map[string]string       // api.bind, dolt.host, など
}

type RigBinding struct {
    Name      string
    Path      string
    Prefix    string
    Suspended bool
}
```

`SiteBinding` は `.gc/` ファイルから読み込まれますが、**ローダーが書き込むことは決してありません**。変更はコマンド（`gc init`、`gc rig add`、`gc rig suspend` など）から行われます。ローダーは `.gc/` を読み取り専用として扱います。

### `Agent`

`Agent` は V1 のフィールドの大部分を保持しますが、構成に関連するアイデンティティが変化します。

| フィールド | V1 | V2 |
|---|---|---|
| `Name` | 単純な名前 | 単純な名前（例: `mayor`） |
| `Dir` | rig プレフィックスまたは空 | rig プレフィックスまたは空 |
| `BindingName` | — | このエージェントが属する `[imports.X]` ブロックの名前（city pack 自身の場合は `""`） |
| `PackName` | — | エージェントが属する pack の `pack.name` |
| `QualifiedName()` | `Dir/Name` | `Dir/BindingName.Name`（`BindingName == ""` または曖昧でない場合は簡略化） |

`BindingName` は、ランタイム全体を通して `gastown.mayor` を実在のアイデンティティとしてアドレス可能にするものです。これは import 展開時に設定され、エージェントとともに永遠に保持されます。

### `Rig`

```go
type Rig struct {
    // city.toml [[rigs]] から
    Name              string
    Imports           map[string]Import
    Patches           Patches
    MaxActiveSessions int
    DefaultSlingTarget string

    // .gc/ site binding から
    Path       string
    Prefix     string
    Suspended  bool
    Bound      bool   // .gc/ にこの rig のバインディングがある場合は true

    // 派生
    Agents          []Agent
    FormulaLayers   FormulaLayers
    ImportGraph     ImportGraph
}
```

rig は今や *2 段階のオブジェクト* になりました。`city.toml` で宣言され（構造的）、`.gc/` でバインドされる（マシンローカル）ものです。宣言済みだが未バインドの rig は有効な状態です。ローダーはこれを生成しますが、`gc start` は警告を出してバインドを促します。

### `ImportGraph`

V1 にはない新しいトップレベル構造です。`gc deps`、`gc why <agent>`、`gc upgrade` のようなコマンドが、構成を再実行することなく「これはどこから来たのか?」に答えられるよう、解決済みの import DAG を記録します。

```go
type ImportGraph struct {
    Root  *ImportNode
    All   map[string]*ImportNode  // qualified なバインディングパス別、例: "gastown" や "gastown.maintenance"
}

type ImportNode struct {
    Binding   string       // フラットなバインディング名（例: "gastown"）
    Pack      *Pack
    Source    string
    Version   string
    Commit    string
    Exported  bool
    Children  []*ImportNode
}
// 注: 再エクスポートされた名前は FLATTEN されます。gastown が
// maintenance の "dog" エージェントを再エクスポートしている場合、city からは
// "gastown.maintenance.dog" ではなく "gastown.dog" として見えます。
// ImportGraph はツーリング（gc why）のために完全なツリーを保持しますが、
// アドレス可能な名前は再エクスポートする pack のバインディングを使用します。
```

## Pack ファイル

pack は、ルートに `pack.toml` を持ち、以下の標準サブディレクトリのいずれかを持つディレクトリです。**ディレクトリが存在すれば、その内容はロードされます** — TOML 宣言は不要です。

```
my-pack/
├── pack.toml              # メタデータ、imports、agent defaults、patches
├── agents/                # エージェント定義（エージェントごとに 1 ディレクトリ）
│   └── mayor/
│       ├── agent.toml
│       ├── prompt.md
│       ├── overlay/       # エージェントごとの overlay
│       ├── skills/        # エージェントごとの skill
│       └── mcp/           # エージェントごとの MCP 定義
├── formulas/              # *.toml の formula ファイル
├── orders/                # *.toml の order ファイル
├── commands/              # pack 提供の CLI コマンド（エントリーごとのディレクトリ）
│   └── status/
│       ├── command.toml   # 任意。デフォルトでは不十分な場合のみ
│       ├── run.sh         # デフォルトのエントリーポイント
│       └── help.md        # デフォルトのヘルプファイル
├── doctor/                # 診断チェック（commands と並列）
│   └── git-clean/
│       ├── run.sh
│       └── help.md
├── patches/               # インポートされたエージェント用のプロンプト置換
├── overlay/               # pack 全体の overlay ファイル
├── skills/                # 現在の city pack の skill
├── mcp/                   # 現在の city pack の MCP サーバ定義
├── template-fragments/    # プロンプトテンプレートのフラグメント
└── assets/                # 不透明な pack 所有のファイル（規約による発見対象外）
```

トップレベルは制御されています — 標準名は認識され、未知のものはエラーとなります。`assets/` だけが唯一の不透明バケットです。`scripts/` ディレクトリは存在せず、スクリプトはそれを使うマニフェストの隣か `assets/` 配下に置きます。詳細は[doc-directory-conventions.md](doc-directory-conventions.md)および[doc-commands.md](doc-commands.md)を参照してください。

`pack.toml` はメタデータ、import、エージェントのデフォルトを持ちますが、エージェントのリストは*持ちません*。

```toml
[pack]
name        = "gastown"
version     = "1.2.0"
schema      = 2
requires_gc = ">=0.20"

[imports.maintenance]
source  = "../maintenance"
export  = false

[imports.util]
source  = "github.com/org/util"
version = "^1.4"

[defaults.rig.imports.gastown]
source = "./packs/gastown"

[agent_defaults]
provider = "claude"
scope    = "rig"

[providers.claude]
model = "claude-sonnet-4"
```

## 規約ベースのロード

V1 から V2 への最大のワークロードシフトは、pack ごとのコンテンツが*どのように*発見されるかです。V1 は `pack.toml` から明示的な宣言を読みます。V2 は標準サブディレクトリを走査します。

| コンテンツの種類 | V1 のソース | V2 のソース |
|---|---|---|
| エージェント | `[[agent]]` テーブル | `agents/<name>/` ディレクトリ |
| エージェントプロンプト | `prompt_template = "prompts/x.md"` | `agents/<name>/prompt.md` |
| エージェントごとの overlay | `overlay_dir = "overlay/x"` | `agents/<name>/overlay/` |
| pack 全体の overlay | `overlay_dir = "overlay/default"` | `overlay/` ディレクトリ |
| Formula | `[[formula]].path` + ディレクトリスキャン | `formulas/*.toml` を直接 |
| Order | formula 内 | `orders/*.toml`（トップレベル、規約により発見） |
| スクリプト | `scripts_dir = "scripts"` | **廃止。** スクリプトはそれを使うマニフェストの隣（`commands/<id>/run.sh`、`agents/<name>/`）または `assets/` 配下に存在 |
| Skill | n/a | 現在の city pack の `skills/` ディレクトリ + `agents/<name>/skills/`（エージェントごと）。インポートされた pack のカタログは後続 |
| MCP 定義 | n/a | 現在の city pack の `mcp/` ディレクトリ + `agents/<name>/mcp/`（エージェントごと）。インポートされた pack のカタログは後続 |
| テンプレートフラグメント | インライン文字列 | `template-fragments/`（pack 全体）+ `agents/<name>/template-fragments/`（エージェントごと） |
| Command | pack.toml の `[[commands]]` | `commands/<id>/` ディレクトリ（規約ベース。任意で `command.toml` マニフェスト） |
| Doctor チェック | n/a | `doctor/<id>/` ディレクトリ（規約ベース。任意で `doctor.toml` マニフェスト） |
| 不透明な assets | 散在 | `assets/` ディレクトリ（ローダーから不透明、明示的なパス参照を通じてのみ到達可能） |

pack のトップレベルは**制御された表面**です。標準ディレクトリ名は明示的に認識され、未知のトップレベルディレクトリはエラーとなります。`assets/` だけが唯一の不透明な逃げ道です。完全なレイアウト仕様は[doc-directory-conventions.md](doc-directory-conventions.md)を参照してください。

走査は浅く予測可能です。各ディレクトリには既知のスキーマ（ファイル拡張子またはエントリごとの `agent.toml`）があります。スキーマに合致しないものは警告となり、サイレントに無視されません。

エージェントディレクトリ内の `agent.toml` は、以前は `[[agent]]` にあったエージェントごとのフィールド（provider、session lifecycle、work_query など）を保持します — パスフィールドは暗黙的になったため除きます。

## Pack 参照形式

import は V1 の include と同じ 3 つの参照形式をサポートしますが、バージョン管理されているため、パーサーはより厳格で、解決方法も実質的に異なります。

```toml
# 1. ローカルパス（バージョン制約なし）
[imports.maint]
source = "../maintenance"

# 2. リモート git、semver 制約
[imports.gastown]
source  = "github.com/gastownhall/gastown"
version = "^1.2"

# 3. city の assets/ ディレクトリ内のローカル pack
[imports.helper]
source = "./assets/local/helper"
```

ローカルパスはバージョンを取れ*ません*。リモートソースはバージョンを取る*必要があります*（またはコミットに固定）。ローダーは曖昧さを拒否します。

実際のフェッチ / キャッシュメカニズムはローダーではなく `gc import` ([doc-packman.md](doc-packman.md)) が所有します。ローダーは import が既に隠しキャッシュ（`~/.gc/cache/repos/<sha256(url+commit)>/`）配下のローカルディレクトリに解決されていることを前提とし、ロックファイル（`packs.lock`）を読んでどのコミットを使うかを判断します。

これは関心の分離における大きな変化です。V1 ではローダー自身が git リポジトリをクローンします。V2 では、その責務は `gc import` に移り、ローダーは純粋なリーダーになります。

## ロックファイルの利用

ローダーは `gc import install` が生成するロックファイルを読み取りますが、書き込みません。`packs.lock` は config/import 管理の一部であり、ローダーの関心事ではありません — ローダーは構成済みの config が正しいことを前提とします（[#583](https://github.com/gastownhall/gascity/issues/583)）。`pack.toml` の各 `[imports.X]` ブロック（または `city.toml` の `[rigs.imports.X]`）は、ロックファイル内の `[packs.X]` ブロックとペアになります。

```toml
# pack.toml
[imports.gastown]
source  = "github.com/gastownhall/gastown"
version = "^1.2"
```

```toml
# packs.lock
[packs.gastown]
source  = "github.com/gastownhall/gastown"
commit  = "abc123..."
version = "1.4.2"
parent  = "(root)"
```

宣言された各 import について、ローダーは対応する `[packs.X]` レコードを参照し、対応する sha256 キー配下のキャッシュディレクトリを見つけ、処理を続行します。一致が存在しない、またはキャッシュエントリが欠落している場合、それはロード時エラーとなり、ユーザーに `gc import install` を実行するよう伝えます。

`parent` フィールドは、この pack をグラフに導入したのが誰かを記録します（直接 import の場合は `(root)`、推移的 import の場合は別のバインディング名）。

## 構成パイプライン

新しいパイプラインはこの順序で実行されます。V1 の 14 ステップと並列比較するために番号を付けています。

### 1. city の場所を特定する

`cityDir` を解決し、`pack.toml`（必須）、`city.toml`（city には必須、city 以外の pack ロードでは不在）、`.gc/`（任意）を見つけます。

### 2. ルート pack をパースする

`pack.toml` を `parsePack()` で `Pack` にデコードします。「このフィールドは設定されたか?」の判断のため、V1 と同じ TOML メタデータ収集を行います。

### 3. デプロイメントファイルをパースする

`city.toml` を `Deployment` にデコードします。2 つのファイルは独立してパースされます — ローダーはサイレントにマージしません。

### 4. site binding を読み取る

`.gc/` を走査して `SiteBinding` を埋めます。読み取り専用です。

### 5. provenance と import グラフを初期化する

新しい `Provenance`。ルート pack を `Root` とする空の `ImportGraph`。

### 6. CLI フラグメントを適用する

`extraIncludes` は後方互換性のため引き続き尊重されますが、`pack.toml` 相当のコンテンツのみを対象とします。各フラグメントはロードされ、V1 と同じセクションごとのルール（スライスは連結、マップは深くマージ、スカラーは last-writer-wins と警告）でメモリ内の `Pack` に折り込まれます。

システム pack はここでも、起動契約のどこにおいてもインジェクトされなくなりました。import 構成はユーザーが宣言した `[imports.<binding>]` エントリから開始されます。

### 7. ルート pack の自己完結性を検証する

ルート pack のディレクトリツリーを走査します。`pack.toml` から解決されたパスが pack ディレクトリを脱出するものはハードエラーとなります。これは V1 にはない新しい「推移的閉包」チェックです。

### 8. 直接 import を解決する

`Pack.Imports` の各エントリについて以下を行います。

1. 対応する `[packs.X]` ロックレコードを参照する。欠落している場合はエラー。
2. ディスク上のパス（sha256 キャッシュディレクトリ）を解決する。
3. インポートされた pack の `pack.toml` をパースする。
4. インポートされた pack が自己完結しているかを検証する。
5. インポートされた pack の `pack.name` がロックレコードと一致するかを検証する（または、バインディング名がエイリアスとなる場合は警告）。
6. `ImportNode` を作成し、ルートの子として接続する。

### 9. 宣言された import のみを許可する

起動契約には、ローダーが所有する暗黙の import ステージは存在しません。import グラフは以下のみで構成されます。

- ルート city が宣言した直接 import
- インポートされた pack が宣言した推移的 import

city が pack に依存している場合、その依存はオーサリングされた config のどこかで宣言され、`gc import install` によって事前にマテリアライズされている必要があります。

### 10. 推移的 import を解決する

import DAG を深さ優先で走査します。インポートされた各 pack について、その `[imports.X]` を**ルート city の単一のロックファイル**に対して再帰的に解決します（pack ごとのロックではなく、ルートのロックには推移的グラフ全体が含まれます）。`export = true` のフラグが立った import は、親の親に対して可視としてマークします。

DAG はツリーである必要があります（サイクルはエラー）。各ノードは以下を保持します。
- ルートから見たバインディング名（qualified パス、`gastown.maintenance`）。
- インポートする pack 内の元のバインディング名（`maintenance`）。
- export フラグ。
- 解決済みのバージョン、ソース、コミット。

### 11. city pack のエージェントを構成する

ルートから非 rig import を通じて到達可能な各 pack（つまり、city スコープから可視であり、推移的な再エクスポートを含む）について以下を行います。

1. pack の `agents/` ディレクトリを走査する。
2. 各 `agents/<name>/` サブディレクトリについて、`agent.toml`（または欠落していればデフォルト）をパースし、`prompt.md`、`overlay/` などをロードする。
3. 各エージェントに `BindingName`（ルートから見える qualified パス）と `PackName` をスタンプする。
4. `scope` でフィルタする。`scope="city"` とスコープ指定なしのエージェントを保持し、`scope="rig"` を除外する。
5. その pack の `[agent_defaults]` デフォルトを自身のエージェントに適用する。
6. `City.Agents` に追加する。

city pack 自身は最後に処理されるため、フォールバック解決を必要とせずに import に対して勝ちます。

### 12. 名前衝突を処理する

V2 の衝突ルールは V1 よりも厳格でシンプルです。

- **単一の pack 内:** 不可能。
- **city pack vs. import:** city pack が常に勝つ。**警告がデフォルトで出力される**。シャドウイングが意図的な場合、ユーザーは import ごとに `[imports.X] shadow = "silent"` で抑制できる。
- **2 つの import が同じ単純名を定義:** **エラーではない**。両方のエージェントが存在し、両方が qualified name でアドレス可能（`gastown.mayor`、`swarm.mayor`）。単純名 `mayor` は曖昧になり、それ以外（formula、sling target）からの参照は qualify する必要がある。
- **曖昧な単純名への単純名参照:** 構成時ではなく*参照側*でエラー。

これが qualified name の核心的な利点です。衝突は構成層でのエラーではなく、参照層での解決問題になります。

### 13. patch を適用する

`pack.Patches`（ルートおよびすべてのインポートされた pack）と `deployment.Rigs[].Patches`（rig 固有）が、構成済みエージェントセットに対して適用されます。ターゲット指定は qualified name で行えるようになりました。`[[patches]]` は `gastown.mayor` を直接ターゲットにできます。曖昧でない場合は単純名のターゲット指定も依然として機能します。

インポートされた pack からの patch は、*それらが持ち込んだ*エージェントにスコープされます。pack は自身が定義していないエージェントに patch を適用できません。

### 14. rig エージェントを構成する

`Deployment.Rigs` で宣言された各 rig について以下を行います。

1. rig の `Imports`（`[rigs.imports.X]` からの rig スコープ import）を読み取る。
2. ステップ 8 と同じ方法（ロックファイルベース）で各 import を解決する。
3. インポートされた各 pack の `agents/` を走査し、`scope="rig"` フィルタでロードする。
4. エージェントに `Dir = rig.Name` *および* `BindingName` をスタンプする。
5. rig レベルの patch を適用する。
6. この rig の formula レイヤを計算する（ステップ 16 を参照）。

### 15. pack グローバルを適用する

各 pack は `[global]` コンテンツを宣言できます（現在は `session_live`）。`global_fragments` は V2 で削除され、明示的な `{{ template }}` インクルージョンを伴う `template-fragments/` に置き換わりました。pack グローバルは以下に適用されます。

- city pack の `[global]`: すべての city スコープエージェントに適用。
- インポートされた pack の `[global]`: *その pack から来た* エージェント（またはその再エクスポート）にのみ適用。これは V1 に対する*修正*です。V1 では pack グローバルがすべてのエージェントに無差別に適用されていました。
- rig import の `[global]`: その rig のエージェントセットにスコープされる。

### 16. formula と asset レイヤを計算する

優先度の低いものから高いものへとレイヤ化されます。

1. インポートされた pack の formula（import 宣言順）。
2. city pack 自身の `formulas/`。
3. rig レベルでインポートされた pack の formula（import 宣言順）。
4. （「rig ローカル」レイヤはなし — rig は `formulas_dir` を持たなくなりました。rig 固有の formula が必要な場合は、rig スコープのローカル pack を宣言してください。）

「インポートする pack が常にそのインポートに勝つ」というルールは保持されます。overlay、skill、mcp、template-fragment についても同じレイヤリングスキームが適用されます。最初のスライスでは、そのレイヤリングは現在の city pack 内でのみ適用されます。インポートされた pack の skill/MCP カタログは後続です。

**注意:** V2 には `ScriptLayers` はありません。`scripts/` ディレクトリは廃止されました。スクリプトはそれを使うマニフェスト（`commands/<id>/`、`doctor/<id>/`、`agents/<name>/`）の隣か、`assets/` 配下に存在します。

### 17. 暗黙のエージェントを注入する（組み込みプロバイダ）

V1 と同じです。**設定済みのプロバイダ**（city の `[providers]` エントリと、`workspace.provider` に一致する組み込みプロバイダ、加えて有効な場合は control-dispatcher）に対してのみ暗黙のエージェントを作成します。すべての組み込みプロバイダが暗黙のエージェントを取得するわけではなく、city が明示的に設定または参照したものだけが対象です。このロジックは V1 から変更ありません。

### 18. agent defaults を適用する

V1 のステップ 11 と同じです。city pack からの `[agent_defaults]` デフォルトは、オーバーライドしないすべてのエージェントに適用されます。インポートされた pack の `[agent_defaults]` デフォルトは、その pack 自身のエージェントにのみ適用されます（既にステップ 11 で処理済み）。

### 19. site state をバインドする

宣言された各 rig について、`SiteBinding` でそのバインディングを参照します。`Path`、`Prefix`、`Suspended` を埋め、`Bound = true` を設定します。バインドされていない rig は `Bound = false` と警告を取得します。

ワークスペースアイデンティティについては、`City.ResolvedWorkspaceName = SiteBinding.WorkspaceName`（バインディングが存在しない場合は `Pack.Meta.Name` にフォールバック、警告付き）。

### 20. 検証する

V1 と同じ形の 3 つのパスを行います。

1. **named session** — テンプレート参照は構成済みビューに存在するエージェント（qualified または曖昧でない）を指している必要がある。
2. **duration** — すべての duration 文字列がパースされる。
3. **セマンティクス** — pool 設定、work_query / sling_query の整合性、エージェントスコープ vs rig 可用性、*さらに新しい V2 チェック*。
   - すべての `pack.requires` が満たされている。
   - どの pack のパスもそのディレクトリを脱出しない。
   - インポートされた各 pack の `pack.name` がロックレコードと一致する（またはエイリアスされている場合はバインディング名と一致）。
   - 曖昧な単純名へのすべての参照が qualify されている。
   - import グラフにサイクルがない。

### 21. namepool をロードする

V1 のステップ 14 と同じ。変更なし。

### 22. 戻す

`(City, Provenance, nil)` を返します。`City` は `ImportGraph`、`Pack`、`Deployment`、`SiteBinding` に加え、構成済みのエージェントセットを保持します。

## 衝突と優先度 — V1→V2 の差分

| 関心事 | V1 | V2 |
|---|---|---|
| 2 つの pack が `mayor` を定義 | エラーまたはフォールバック解決 | 両方が `gastown.mayor` と `swarm.mayor` として存在。単純な `mayor` は曖昧 |
| city と pack の両方が `mayor` を定義 | prepend 順序により city が勝つ | city が明示的に勝つ。任意で警告 |
| エージェントの `fallback = true` | ソフトオーバーライド可能なデフォルトに使用 | **削除。** qualified name + 明示的な優先度により不要に |
| プロバイダの衝突 | フィールドごとの深いマージと警告 | 同じ |
| ワークスペースフィールドの衝突 | フィールドごとのマージと警告 | N/A — ワークスペースアイデンティティは `.gc/` に移動 |
| `[packs.X]` の pack 名衝突 | last-writer-wins と警告 | ロックファイルが正規。同じ pack を異なるバージョンで複数 import するとエラー |
| ターゲットが見つからない patch | エラー | エラー（qualified name 対応） |
| pack ディレクトリを脱出するパス | 許可 | ハードエラー |
| 循環 import | N/A（include はフラット） | ハードエラー |

`fallback = true` は最も注目すべき犠牲者です。V1 では、システム pack がデフォルトのエージェントを提供し、ユーザー pack がサイレントにオーバーライドできるようにする方法として存在しました。V2 では、qualified name と明示的なシャドウイングにより同じ効果が得られます。システム pack が `system.mayor` を提供し、ユーザー pack が `mine.mayor` を提供し、city pack がどちらを参照するかを選択します。曖昧さがないため、サイレントなオーバーライドの必要はありません。

## プロバイダ解決

プロバイダ解決は**ハイブリッドフラットモデル**を使用します。1 つのグローバルな `providers` マップに、インポートされた pack がフィールドごとの深いマージで貢献し、city pack が常に勝つというモデルです。

1. **プロバイダの名前空間はフラット。** pack ごとの名前空間ではなく、1 つのグローバル `providers` マップがあります。インポートされた pack の `[providers.claude]` は、V1 と同じフィールドごとの深いマージセマンティクス（スカラーフィールド: オーバーライド + 警告、スライスフィールド: 置換、マップフィールド: 加算）を使ってグローバルマップにマージされます。city pack の `[providers.claude]` は常にインポートされた pack の定義をシャドウします。

2. **エージェントの `provider = "claude"` はマージされた結果に解決。** qualified なプロバイダ参照は不要です。解決チェーンは: `agent.StartCommand`（エスケープハッチ） → `agent.Provider` → マージされたグローバル `providers[name]` → 組み込みプリセット → PATH を介した自動検出。

3. **組み込みプロバイダリストは変更なし。** 同じ正規名（claude、codex、gemini、cursor、copilot、amp、opencode、auggie、pi、omp）。

## ローダーが呼び出される場所

V1 と同じ呼び出し元（`cmd_start.go`、`cmd_config.go`、`cmd_agent.go`、`cmd_init.go`）ですが、関数シグネチャが `LoadWithIncludes(fs, "city.toml", -f...)` から `LoadCity(fs, cityDir, -f...)` に変わります。呼び出し元は引数を 1 度更新するだけです。

## アトミック書き込みと git の安全性

ローダーは git をクローンしなくなりました。アトミック書き込みと git-env-blacklist のメカニズムは `internal/config/` から `gc import`（`~/.gc/cache/repos/` 配下のキャッシュを所有）に移動します。ローダーは純粋なリーダーになり、新しい I/O 表面は獲得しません。

ロックファイルの*読み取り*はローダーが行う唯一の新しい I/O であり、読み取り専用です。

## マイグレーションストーリー

V1 で動作している city は、V2 がロードする前に変換する必要があります。ハードカットオーバー: `gc doctor` が V1 のパターンを検出し、`gc doctor --fix` が安全な機械的変換を処理します。`gc import migrate` はもはや主要な公開パスではありません。

マイグレーションは実装順序に合わせて 2 ステップに分割されます。

### ステップ 1: pack/city の再構築（先行リリース）

1. **`includes` → `[imports]`。** 各 `workspace.includes` エントリについて以下を行う:
   - ローカルパス: `[imports.<basename>]` を `source = path` で合成。
   - git バックエンドソース: `[imports.<repo-name>]` を `source` と `version` で合成。semver タグはバージョン制約になる。タグなしの git ソースは正確な SHA で固定される（`version = "sha:<commit>"`）。
2. **`workspace.name` → `.gc/`。** 既存のディレクトリに対して `gc init` を実行し、ワークスペース名とプレフィックスで `.gc/` を埋める。
3. **`rig.path`、`rig.prefix`、`rig.suspended` → `.gc/`。** 各 rig について、`.gc/rigs/<name>.toml` 配下にバインディングファイルを書き込む。
4. **`workspace.default_rig_includes` → `[defaults.rig.imports.<binding>]`。** `includes` → `[imports]` と同じマッピング。
5. **`fallback = true` のエージェント。** フィールドを削除し、以前にフォールバックシャドウイングに依存していて手動の曖昧さ解消が必要なエージェントについてユーザーに警告する。

このステップでは pack.toml の `[[agent]]` テーブルは引き続き機能します。

### ステップ 2: agent-as-directory（同じリリースでリリース）

6. **`[[agent]]` テーブル → `agents/<name>/` ディレクトリ。** 各 `[[agent]]` ブロックについて、非パスフィールドを持つ `agents/<name>/agent.toml` を作成し、`prompt_template` の内容を `prompt.md` に移動し、`overlay_dir` の内容を `overlay/` に移動する。

マイグレーションは一般的なケースでは機械的で、動作する V2 city を生成します。エッジケース（以前にフォールバックでマスクされていた名前の衝突、semver タグのないレガシー git ソース、その他の安全でない書き換え）は、ユーザーがレビューするか手動で対応する必要のある警告を出力します。

## まとめ: 完全パイプラインを 1 つのリストで

1. city を特定する（`pack.toml` + `city.toml` + `.gc/`）。
2. ルート `pack.toml` を `Pack` にパースする。
3. `city.toml` を `Deployment` にパースする。
4. `.gc/` を `SiteBinding` に読み取る（読み取り専用）。
5. `Provenance` と `ImportGraph` を初期化する。
6. CLI フラグメントを `Pack` に適用する（セクションごとのマージ）。
7. ルート pack の自己完結性を検証する（パス脱出なし）。
8. ロックファイルに対して直接 import を解決する → キャッシュディレクトリ。
9. 宣言された import のみをグラフに許可する。
10. `export = true` を尊重しながら推移的 import を DFS で解決する。
11. インポートされた pack と city pack から city スコープのエージェントを構成する（qualified name）。
12. 曖昧な単純名を検出する。記録するがエラーにしない。
13. 構成済みセットに対して patch を適用する（qualified name 対応）。
14. 各 rig について: rig import を解決し、rig エージェントを構成し、rig patch を適用する。
15. pack グローバルを適用する（発生元 pack のエージェントにスコープ）。
16. formula / overlay / skill / mcp / template-fragment レイヤを計算する（スクリプトレイヤなし — スクリプトはエントリローカル）。
17. 組み込みプロバイダ用の暗黙のエージェントを注入する。
18. `[agent_defaults]` デフォルトを適用する。
19. site state をバインドする（rig パス、ワークスペース名）。
20. 検証する（named session、duration、セマンティクス + V2 固有のチェック）。
21. namepool をロードする。
22. `(City, Provenance, nil)` を返す。

`Provenance` と `ImportGraph` は処理全体を通じて蓄積され、すべてのコマンドが構成を再実行することなく「これはどこから来たのか?」に答えるための監査証跡を提供します。
