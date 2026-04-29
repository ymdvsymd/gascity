# Pack/City Model v.next

**GitHub Issue:** [gastownhall/gascity#360](https://github.com/gastownhall/gascity/issues/360) ([#159](https://github.com/gastownhall/gascity/issues/159) を置き換える)

タイトル: `feat: Pack/City Model v.next — cities as packs, import model, managed state`

agent 定義の再構築を扱う [doc-agent-v2.md](doc-agent-v2.md) ([gastownhall/gascity#356](https://github.com/gastownhall/gascity/issues/356)) と対をなします。

> **同期の維持:** このファイルが真実の根拠です。更新時はここを編集し、issue 本文を `gh issue edit 360 --repo gastownhall/gascity --body-file <(sed -n '/^---BEGIN ISSUE---$/,/^---END ISSUE---$/{ /^---/d; p; }' issues/doc-pack-v2.md)` で更新してください。

---BEGIN ISSUE---

## 課題

現行モデルは、本来分離されるべき 3 つの関心事を絡み合わせています。すなわち、ポータブルな **definition**（agents、providers、formulas）、チームの **deployment** に関する決定（rigs、substrates、capacity）、そしてマシンごとの **site binding**（パス、prefix、suspended フラグ）です。city.toml はこの 3 つすべてを抱えており、pack は最初の 1 つを持ちますが composition の段階で消えてしまいます。`.gc/` は明確な層として存在しません。これにより一連の問題が生まれます。

1. **city は pack のような存在だが pack として構造化されていない。** city レベルの内容は階層的な解決に参加し pack の内容を上書きしますが、pack のように composition、共有、import ができません。composition の単位は 1 つに統一すべきです — city 定義は単に pack であるべきです。

2. **include の意味が弱すぎる。** `includes` は qualified な identity を持たないまま pack の内容を city にダンプします。衝突は読み込み順序に依存します。aliasing も version pinning も明示的な衝突処理もありません。「gastown の mayor」と言えるような、永続的な identity を持つ named import が欲しいのです。

3. **convention と declaration が衝突する。** formulas はディレクトリで発見され、prompts は明示的な TOML パスが必要で、scripts はまた別途発見されます。convention で構造を定義したい — ディレクトリが存在すればその内容を読み込む、というように。

4. **pack が self-contained でない。** 内容は pack の境界外のパスを参照できます。transitive closure は強制されません。pack を完全にポータブルにしたい — ディレクトリツリーと宣言された import だけ、それ以外はなし、というように。

5. **managed state の置き場が明確でない。** `workspace.name`、`rig.path`、運用上のトグルが、共有可能な定義と並んでチェックインされた TOML に書かれています。クリーンに分離したい。`pack.toml` は *definition*（この city が何であるか）、`city.toml` は *deployment plan*（チームで共有される、それをどう動かすかという決定）、`.gc/` は *site binding*（deployment を特定のファイルシステムに結びつける、マシンローカルの状態）。

この提案は、pack 変更が要求する範囲を超えた `.gc/` の内部構造、パッケージレジストリや暗黙の import 表面、既存 city 向けの機械的な移行 UX については扱いません。古い city は移行されるまで完全に動かなくなる可能性があります。公の移行パスは `gc doctor` のあとに `gc doctor --fix` を実行することです。

## 提案する変更

### Cities

city は pack に deployment ファイル `city.toml` が伴ったものです。`city.toml` を削除すれば、残りは有効でポータブルな pack になります。

city 定義の構造は pack のそれと同一です（agents、formulas、prompts、scripts）。city の構造を定義する `pack.toml` ファイルも含みます。

deployment に関する決定（rigs、substrates、capacity）は `city.toml` に置かれます。Site binding（パス、prefix、運用状態）は `.gc/` に置かれます。（以下の例では、companion 提案 [#356](https://github.com/gastownhall/gascity/issues/356) の agent-as-directory モデルを使います。両提案は同じ breaking wave で同時にリリースされます。実装中はまず pack/city の再構築が `[[agent]]` 構文を保ったまま着地し、agent-as-directory はその上に第 2 ステップとして重なります。）

```
my-city/
├── pack.toml              # この city が「何であるか」（ポータブルな definition）
├── agents/                # agent 定義（convention で発見）
├── formulas/              # formula 定義（convention で発見）
├── orders/                # order 定義（convention で発見）
├── commands/              # pack 提供の CLI コマンド
├── doctor/                # 診断チェックスクリプト
├── patches/               # imported agent の prompt 置き換え
├── overlay/               # pack 全体の overlay ファイル
├── skills/                # 現 city pack の skills カタログ（imported pack のカタログは後ほど）
├── mcp/                   # 現 city pack の MCP server 定義（imported pack のカタログは後ほど）
├── template-fragments/    # prompt template フラグメント
├── assets/                # 不透明な pack 所有ファイル（convention で発見しない）

├── city.toml              # この city が「どう deploy されるか」（チーム共有）
└── .gc/                   # site binding（マシンローカル、gitignore 対象）
```

pack のトップレベルは **制御された表面** です — 標準ディレクトリ名のみが明示的に認識され、未知のトップレベルディレクトリはエラーです。任意のファイルは `assets/` 配下に置きます。レイアウト仕様の詳細は [doc-directory-conventions.md](doc-directory-conventions.md) を参照してください。

skills/MCP の最初のスライスでは、`skills/` と `mcp/` のカタログに寄与するのは現在の city pack のみです。imported pack のカタログは後のウェーブです。

埋め込み pack（必要な場合）は `assets/` 配下に置かれ、明示的な import パスで参照されます。

```toml
[imports.maintenance]
source = "./assets/maintenance"
```

city の `pack.toml` は *この city が何であるか* を定義するすべてを含みます。imports は下の専用セクションで扱います — 現時点では、pack composition は city.toml ではなくここで宣言されると理解しておけば十分です。

```toml
# pack.toml (the city pack)

[pack]
name = "my-city"
version = "0.1.0"

[imports.gastown]
source = "./assets/gastown"

[imports.maint]
source = "./assets/maintenance"

# Pack 全体の agent デフォルト — 個別 agent は agents/ ディレクトリで定義
# city.toml の city レベル [agent_defaults] でこれらを上書き可能。
[agent_defaults]
provider = "claude"

[[named_session]]
template = "mayor"
mode = "always"

# Provider 設定 — model、permission など。
[providers.claude]
model = "claude-sonnet-4-20250514"
```

city の deployment ファイル `city.toml` は、*この city をどう動かすか* についてチームが合意した内容を含みます。

```toml
# city.toml (チーム共有の deployment — identity フィールドなし)

[beads]
provider = "dolt"

[[rigs]]
name = "api-server"
max_active_sessions = 4
default_sling_target = "api-server/polecat"
session_sleep = { idle = "10m" }

[[rigs]]
name = "frontend"
max_active_sessions = 2
```

site binding（rig パス、suspended フラグ、prefix）は `gc` コマンドで管理され、`.gc/` に保存されます。

```
gc rig add ~/src/api-server --name api-server
gc rig add ~/src/frontend --name frontend
```

#### city pack と通常の pack の違いは何か

ほとんどありません。

1. **city pack は composition のルートです。** 他から import される必要はありません。
2. **city pack には deployment を記述する `city.toml` が伴います。** 通常の pack は `pack.toml` だけです。
3. **rig を持つのは city だけです。** rig は city.toml で宣言され、pack には書きません。

それ以外（agents、named sessions、providers、formulas、prompts、scripts、overlays、imports、patches）は同じように動きます。

> **設計原則:** city ディレクトリから `city.toml` を削除しても、残りは別の city が import 可能な、有効な pack になる。

#### 名前、prefix、生成

`gc init` と `gc rig add` は、デフォルトで名前と prefix を生成します。ユーザーは `--name` と `--prefix` で上書きできます（典型的には衝突解決のため）。`gc init` は今、選んだマシンローカルの workspace 名/prefix を `.gc/site.toml` に書き込みます。`pack.toml` はポータブルな definition identity を保ちます。

`gc register` は city の登録名を明示的に設定するための `--name` を受け付けます。選ばれた名前はマシンローカルの supervisor registry に保存され、`city.toml` には書き戻されません。`--name` を省略すると、`gc register` は現在の effective な city identity（site-bound な workspace 名があればそれ、なければ legacy `workspace.name`、それもなければディレクトリベース名）を使い、その値を registry に保存します。`gc register` は `city.toml` も `pack.toml` も書き換えません。([#602](https://github.com/gastownhall/gascity/issues/602))

名前と prefix はどちらも `gc` で管理されます。authoritative なコピーは `.gc/` にあります。名前は人間向けのラベル、prefix は名前から派生して bead ID に焼き込まれます。どちらも作成後に気軽に変更すべきではありません。

#### リネーム

リネームは TOML ファイルを編集するのではなく、`gc rig rename`（または `gc workspace rename`）で行います。リネームコマンドは city.toml の名前、`.gc/` の name-to-prefix マッピングを更新し、必要に応じて新しい prefix を選びます（`--prefix` 経由）。prefix の移行が必要な場合（既存の beads が古い prefix を使っている）、コマンドが対応します。

city.toml の rig 名と managed state に不一致を `gc` が検出すると、起動をブロックし、リネームコマンドで解消するようユーザーに伝えます。

#### `pack.name` と workspace identity の関係

`pack.name` は definition の identity です — 「この pack の名前は gastown だ」。これは `pack.toml` に置かれ、ポータブルで、import されると pack と一緒に旅をします。

`workspace.name` と `workspace.prefix` は、現在は legacy 互換フィールドです。新規の `gc init` はマシンローカルの identity を `.gc/site.toml` に書き、`gc doctor --fix` は legacy 値を `city.toml` から移行します。`gc register` は supervisor registry を登録 identity のマシンローカル真実の根拠として扱います。明示的な `--name` エイリアスは site-bound や legacy の workspace identity と異なってもよく、ランタイムの supervisor 管理フローはその登録エイリアスを優先します。

長期的な方向性は変わりません。ポータブル identity は `pack.name`、deployment plan は `city.toml`、マシンローカルの命名/binding は `.gc/` 配下の site binding に置く、という整理です。

フィールド単位の移行表は付録にあります。

### Import モデル

現行の `includes` 機構には 3 つの問題があります。pack が composition 後に identity を失う（「gastown の mayor」と言えない）、衝突は明示的な処理なしに読み込み順序で解決される、推移的依存が見えない（pack に sub-include を加えると、その pack を使うすべての city で表示される agents がサイレントに変わる）。imports はこれら全部を解決します。それぞれの composed pack に永続的な名前を与え、明示的な衝突解決を要求し、デフォルトでクローズドにすることでです。

pack は **imports** で他の pack を composition します。include ではありません。import は他の pack への named binding を作ります。

#### 具体例

`gastown` という pack が agents と formulas を定義しているとします。

```toml
# assets/gastown/pack.toml
[pack]
name = "gastown"
version = "1.2.0"

[agent_defaults]
provider = "claude"
scope = "rig"
```

```
assets/gastown/
├── pack.toml
├── agents/
│   ├── mayor/
│   │   ├── agent.toml     # scope = "city"
│   │   └── prompt.md
│   └── polecat/
│       └── prompt.md
├── formulas/
│   ├── mol-polecat-work.toml
│   └── mol-idea-to-plan.toml
└── assets/
    └── worktree-setup.sh
```

city pack はそれを import します。

```toml
# pack.toml (city pack)
[pack]
name = "my-city"

[imports.gastown]
source = "./assets/gastown"
```

import 後、agents は曖昧でないときは bare な名前（`mayor`、`polecat`）で、曖昧解決が必要なときは qualified な名前（`gastown.mayor`、`gastown.polecat`）で利用できます。

#### Aliasing

binding 名は pack 名と一致する必要はありません。

```toml
[imports.gs]
source = "./assets/gastown"
```

これで `gs.mayor` と `gs.polecat` が qualified 名として利用できます。

#### Version 制約

リモート import は semver 制約を使います。

```toml
[imports.gastown]
source = "github.com/gastownhall/gastown"
version = "^1.2"
```

ローカルパス import には version 制約がありません。

リモート import の解決済みバージョンは lock ファイル（`packs.lock`、形式は [doc-packman.md](doc-packman.md) が所有）に記録されます。loader は lock ファイルを読み、各 import がどのコミットに解決され、`~/.gc/cache/repos/` 配下のどのディレクトリにあるかを見つけます。loader 自身は git clone も欠落状態の自己修復もしません。その責任は `gc import install` にあります。lock エントリやキャッシュエントリの欠落は読み込み時エラーとなり、ユーザーに `gc import install` を実行するよう伝えます。

#### 推移的 import と export

デフォルトでは、imports は **推移的** です。`gastown` が内部で `maintenance` を import していれば、`gastown` を import した人にも自動的に `maintenance` の内容が来ます。これが一般的なケースです — pack が依存を要求するなら、コンシューマもそれを必要とします。

pack は特定の import について `transitive = false` で推移解決を抑制できます。

```toml
# assets/gastown/pack.toml
[imports.maintenance]
source = "../maintenance"
transitive = false
```

これは珍しい設定です — 「自分の用途で import しているが、私の pack のコンシューマには見せない」という意味です。典型的なユースケースは内部ツールやテスト専用依存です。

pack は import した pack を明示的に re-export して、その内容を re-export する pack の名前空間で利用可能にできます。

```toml
# assets/gastown/pack.toml
[imports.maintenance]
source = "../maintenance"
export = true
```

`export = true` を付けると、maintenance の agents は gastown の名前空間にフラット化されて現れます。`gastown.dog` であって `gastown.maintenance.dog` ではありません。Re-export は不透明 — コンシューマは `dog` が内部的に `maintenance` から来たことを知る必要はありません。Provenance はツール用に import グラフで追跡されますが（`gc why dog`）、アドレス可能な名前は推移的なパスではなく re-export する pack の binding です。

#### Lock ファイルモデル

ルート city の lock ファイル（`packs.lock`）は推移 import グラフ全体のすべての pack を記録します。imported pack は **独自の** lock ファイルを持ちません。`gc import install` だけがこのファイルを bootstrap または修復するコマンドです。`packs.lock` がないときは宣言されたグラフを解決して書き出し、`packs.lock` があるときはコミット済み状態からキャッシュを復元します。通常の load/start/config フローは純粋なリーダーのままです。lock ファイル形式は [doc-packman.md](doc-packman.md) を参照してください。

#### ライフサイクルの動詞

現在は部分的に混同されている、4 つの異なる操作です。

| 操作 | 動詞 | 内容 |
|---|---|---|
| city の内容を定義する | `gc init`（ファイル作成）または手動編集 | pack.toml、city.toml、ディレクトリ構造を作成 |
| インストール済みの imports を検証する | `gc import check` | 宣言された imports、`packs.lock`、ローカルキャッシュ状態をフェッチや変更なしで確認 |
| city の packs をインストールする | `gc import install` | `packs.lock` を bootstrap または修復し、すべての imports をキャッシュに materialize |
| city を controller に登録する | `gc register` | city を `.gc/` に bind し、controller に存在を伝える |
| city のランタイムを開始する | `gc start` | controller が登録済みの city を活性化 |

`gc start` は未実行なら `gc register` を含意します（zero-config を維持）。`gc register` は city を活性化前にステージしたいワークフロー向けの明示的 binding ステップです。

#### Rig レベルの imports

rig は city のコンセプトです。pack は rig を知りません。Rig レベルの imports は city.toml にあります。

```toml
# city.toml
[[rigs]]
name = "api-server"

[rigs.imports.gastown]
source = "./assets/gastown"

[rigs.imports.custom]
source = "./assets/api-tools"
```

Rig レベルの imports は rig スコープの agents を生み出します。`api-server/gastown.polecat` のように。City レベルの imports は city スコープの agents を生み出します。`gastown.mayor` のように。

#### 新しい rig のデフォルト imports

現在の `workspace.default_rig_includes` は新しい rig 向けの `[defaults.rig.imports.<binding>]` エントリになります。

```toml
# pack.toml
[defaults.rig.imports.gastown]
source = "./assets/gastown"
```

`gc rig add` で新しい rig を作成するとき、ユーザーが imports を指定しなければ、これらのデフォルトが使われます。

### Convention に基づく構造

pack のファイルシステムレイアウトはその宣言です。トップレベルは **制御された** ものです — 標準名は認識され、未知のトップレベルディレクトリはエラー、任意のファイルは `assets/` 配下に置かれます。

```
my-pack/
├── pack.toml              # メタデータ、imports、agent デフォルト、patches
├── agents/                # agent 定義（convention で発見）
├── formulas/              # *.toml formula ファイル（convention で発見）
├── orders/                # *.toml order ファイル（convention で発見）
├── commands/              # pack 提供の CLI コマンド
├── doctor/                # 診断チェックスクリプト
├── patches/               # imported agent の prompt 置き換え
├── overlay/               # pack 全体の overlay ファイル
├── skills/                # 現 city pack の skills カタログ（imported pack のカタログは後ほど）
├── mcp/                   # 現 city pack の MCP server 定義（imported pack のカタログは後ほど）
├── template-fragments/    # prompt template フラグメント
└── assets/                # 不透明な pack 所有ファイル（convention で発見しない）
```

**convention が置き換えるもの:**

| 現行の機構 | convention での置き換え |
|---|---|
| pack.toml の `[[agent]]` テーブル | `agents/<name>/` ディレクトリが存在 → agent が存在 |
| `prompt_template = "prompts/mayor.md"` | `agents/<name>/prompt.md` |
| `[[formula]].path` | `formulas/` にファイルがある → それは formula |
| `overlay_dir = "overlay/default"` | `overlay/` + `agents/<name>/overlay/` |
| `scripts_dir = "scripts"` | 廃止。スクリプトはそれを使う manifest の隣（`commands/<id>/run.sh`、`agents/<name>/`）か `assets/` 配下にあります |
| `[formulas].dir` | 廃止。`formulas/` は固定された convention で、設定可能なパスではありません |

ルール: **標準ディレクトリが存在すれば、その内容が読み込まれる。** `assets/` だけが例外で、存在はしますが loader からは不透明で、明示的なパス参照でのみ到達可能です。

完全なディレクトリレイアウト仕様、設計原則、pack-local パスの挙動ルールについては [doc-directory-conventions.md](doc-directory-conventions.md) を参照してください。

#### Formula のレイヤリング

複数の pack が import されるとき、formulas は優先度（低→高）でレイヤリングされます。

1. Imported pack の formulas（import 宣言順）
2. city pack 自身の `formulas/`
3. Rig レベルの imported pack formulas（import 宣言順）

import する側の pack は常に import される側に勝ちます。

### Pack identity と qualified 名

composition 後、すべての agent、formula、prompt は pack の出自を保持します。

#### Qualified 名のフォーマット

- `gastown.mayor` — gastown import からの mayor agent
- `swarm.coder` — swarm import からの coder agent
- `librarian` — city pack 自身の agent（修飾不要）
- `api-server/gastown.polecat` — pack 出自付きの rig スコープ

`/<name>` は city スコープ版を明示的にターゲットします。`/mayor` は任意のコンテキストから「city スコープの mayor」を意味します。これはファイルシステムの絶対パスのセマンティクス（先頭スラッシュ = ルートから）に倣っています。

#### 修飾が必要なとき

bare 名は曖昧でないときに動作します。修飾は 2 つの imported pack が同じ agent 名をエクスポートしたときにのみ必要です。city pack 自身の agents は決して曖昧になりません — 常に勝ちます。

同じ bare 名を定義する 2 つの imports は composition 時のエラーには **なりません**。両方の agent が存在し、qualified 名でアドレス可能です。エラーは *参照する* 側に移ります。曖昧な bare 名を使う formula、sling target、named-session テンプレートは修飾しなければなりません。これは V1 includes に対する named imports の中心的な利点です — 衝突は読み込み時の失敗ではなく解決の問題になります。

#### Pack のグローバルスコーピング

`[global].session_live` のような pack 全体のコンテンツは、同じ pack（または re-export）から来た agents にだけ適用されます。V1 では pack グローバルは composed city のすべての agents に無差別に適用されました。V2 ではこれが修正され、imported pack が所有しない agents に session 状態をサイレントに注入できなくなります。（V2 では `global_fragments` は廃止 — 明示的な `{{ template }}` インクルージョン付きの `template-fragments/` で置き換えられます。）

#### `fallback = true` の廃止

V1 では agents に `fallback = true` フラグがあり、システム pack がデフォルトを提供してユーザー pack がサイレントに上書きすることを許していました。V2 ではこのフラグが完全に廃止されます。Qualified 名と明示的な precedence（city pack は常に imports に勝つ）が、サイレント shadowing という落とし穴なしに同じユースケースをカバーします。

### 推移閉包

pack は self-contained です。その推移閉包はディレクトリツリーと宣言された imports です。

- pack.toml のすべてのパスは pack ディレクトリ相対で解決されます。`../` でのエスケープは不可。
- imports が外部コンテンツを参照する唯一の機構です。
- `gc` は pack の self-containment を検証します。pack ディレクトリを抜け出す解決パスはエラーです。

### Site binding (`.gc/`)

マシンごとの状態は `.gc/` に置かれ、`gc` コマンドで管理されます。

| カテゴリ | 例 | 設定するもの |
|---|---|---|
| **Identity binding** | workspace 名、workspace prefix | `gc init`、`gc config set` |
| **Rig binding** | rig パス、rig prefix | `gc rig add` |
| **運用トグル** | rig suspended フラグ | `gc rig suspend/resume` |
| **マシンローカル設定** | api.bind、session.socket、dolt.host | `gc config set` |
| **ランタイム状態** | sessions、beads、caches、logs、sockets | `gc` ランタイム |

ルール: **チェックインされた TOML ファイルにあるなら、それは definition か deployment。`.gc/` にあるなら、それは site binding。** グレーゾーンはありません。

> **現行のロールアウト:** workspace identity（`workspace.name`、`workspace.prefix`）と `rig.path` は今 `.gc/site.toml` に置かれます。loader は読み込み時に site binding を `city.toml` の上に重ね、移行中は legacy で書かれた値も読み、`gc doctor --fix` が legacy フィールドを `.gc/site.toml` に移行します。`rig.prefix` と `rig.suspended` は当面 `city.toml` に残ります。

現行の Phase A / Phase B の分割（パス抽出と post-15.0 のマルチ city rig 共有）については [doc-rig-binding-phases.md](doc-rig-binding-phases.md) も参照してください。

#### Rig のライフサイクル

rig は 2 段階のライフサイクルを持ちます。

1. **Declared** — `[[rigs]]` エントリが city.toml に存在する（チーム共有の構造）
2. **Bound** — パス binding が `.gc/` に存在する（マシンローカルの結びつけ）

declared だが bind されていない rig は有効な状態です。`gc start` は bind されていない rig について警告し、bind するかを尋ねます。これは、あるチームメイトが rig を city.toml に追加してコミットし、他のチームメイトが pull 後にローカルパスで bind するワークフローをサポートします。

## 検討した代替案

- **includes を残し、修飾を追加する。** composition 意味の弱さを解決しません。Includes は本質的にテキスト挿入であり、モジュール composition ではありません。
- **すべての設定を 1 ファイルに置く。** pack をポータブルにする definition/deployment 分離を失います。
- **3 つのファイル（pack.toml、city.toml、city.local.toml）。** 3 つ目は不要 — マシンローカル状態は `.gc/` に属し、コマンドで管理され、手動編集されません。
- **rig 上に `formulas_dir` を残す。** 「pack が composition の唯一の単位」という原則を壊します。rig 固有のローカル pack で同じことを一貫して達成できます。

## 影響範囲とインパクト

- **破壊的変更:** `includes` は `[imports]` に置き換え。`[[agent]]` テーブルは `agents/` ディレクトリに移動。`workspace.name` は `.gc/` に移動。`fallback = true` は廃止（qualified 名 + 明示的 precedence で置き換え）。Pack グローバルは city 全体に適用されるのではなく、出自 pack にスコープされる。
- **新概念:** Aliasing、versioning、デフォルト推移 imports、フラット化された re-export、単一ルートロックファイル（`packs.lock`）を持つ Import モデル。Lock ファイル消費（loader はリーダー、`gc import` が bootstrap、修復、キャッシュ materialize を所有）。Shadow 警告。ライフサイクル動詞の分離（define / install / register / start）。
- **設定の分割:** 現行の city.toml は pack.toml（definition）+ city.toml（deployment）+ `.gc/`（site binding）に分割。
- **Convention:** ファイルシステムレイアウトがほとんどの TOML パス宣言を置き換え。
- **移行:** ハードな切り替え。`gc doctor` が V1 パターンを検出し、`gc doctor --fix` が安全な機械的変換を扱います。`gc import migrate` はもはや主要な公の経路ではありません。1 リリースの非推奨警告ののち、V2 loader は V1 形状を拒否します。

## 解決済みの問い

オリジナル提案からの問いで決着したもの:

- **登録の動詞:** Binding 風。`gc register` は city を controller に bind します。`gc start` は未実行なら register を含意します（zero-config を維持）。上記「ライフサイクルの動詞」を参照。
- **Re-export の命名:** **フラット化。** `gastown.dog` であって `gastown.maintenance.dog` ではありません。上記「推移的 import と export」を参照。
- **Shadow 警告:** **デフォルトで警告**、import ごとのオプトアウト（`[imports.X] shadow = "silent"`）が意図的な shadowing 用にあります。Shadowing は imported pack から agent を「無効化」する有効な方法ですが、偶発的な衝突は可視化すべきです。
- **Rig 固有の formula 上書き:** **rig-local pack。** 「pack が composition の唯一の単位」という原則と一貫しています。city はデフォルト解決が異なる別の rig にすぎません — city-rig は他のすべての rig からクエリされますが、rig-rig は city からクエリされません。Rig-local pack で formula 上書きを一貫して達成します。
- **`packs/` ディレクトリ:** **完全に廃止。** V2 には `packs/` ディレクトリはありません。トップレベルの pack 構造は制御されており、埋め込み pack は `assets/` 配下に置かれ、明示的な import パスで参照されます。[doc-directory-conventions.md](doc-directory-conventions.md) を参照してください。
- **rig vs. city スコープの曖昧解決:** **city スコープ用の `/<name>`。** `/mayor` は「city スコープの mayor」を意味します。ファイルシステムの絶対パスのセマンティクスに倣います。
- **SHA pinning:** サポート。`[imports.X]` で `version = "sha:<full sha>"`。[doc-packman.md](doc-packman.md) に文書化されています。
- **推移 imports のデフォルト:** **デフォルトで推移的。** `transitive = false` は珍しいケース向けのオプトアウトです。
- **Alias の伝播:** **どこにでも伝播。** ローカルハンドルがランタイム identity そのもの — sessions、beads、log line すべてが alias を使います。upstream の `[pack].name` はローカルハンドルのフォールバックデフォルトですが、ユーザーが alias を付けていればランタイムでは現れません。
- **imports をまたぐ provider 解決:** **ハイブリッド（フラットな名前空間、pack は deep merge で寄与、city が勝つ）。** 1 つのグローバル `providers` マップ。Imported pack の `[providers.X]` ブロックがマージインされます。City レベルの `[providers.X]` は常に shadow します。bare な `provider = "claude"` はマージ後の結果に解決されます。
- **Settings JSON のマージ:** 設定 JSON 限定で **deep merge**。他のすべての設定（TOML キー、リスト）は last-writer-wins。この非対称は意図的で、VS Code とほとんどのエコシステムの慣習に一致します。

## 原則

- **city は単に別の rig** で、解決のデフォルトが異なるだけ。city-rig は他のすべての rig からクエリされ、rig-rig は city からクエリされない。「city は X、rig は Y」という条件分岐ロジックの多くはこのフレーミングに収束します。
- **pack は composition の唯一の単位。** すべては pack を通して composition される — formulas、agents、providers、scripts。第 2 の機構はありません。
- **convention が構造を定義する。** 標準ディレクトリが存在すれば、その内容が読み込まれる。TOML 宣言は不要。
- **definition / deployment / site binding は物理的に分離されている。** pack.toml / city.toml / `.gc/` — グレーゾーンはなし。

## ランタイム状態の保管

v1 はランタイム状態を `.gc/` と `~/.gc/` ファイルに保存します。これらの一部を beads に移すべきかは、beads DB がローカル専用かチーム共有かに依存する未解決の問いです。チーム共有なら `~/.gc/cities.toml` のようなマシンごとの状態を beads に置くと、開発者間で誤って同期されてしまいます。決定: v1 ではファイルのままとし、ローカル vs 共有 beads の問題が決着したら再検討します。これは v.next の refactor で、ユーザーから見える変化はありません。

## 未解決の問い

この提案の command/doctor 表面については未解決はありません。`commands/<name>/run.sh` と `doctor/<name>/run.sh` は決着済みの convention パスで、残る manifest 対称性の作業は command 固有のドキュメントと issue バックログで追跡されます。

## 付録: フィールド配置リファレンス

### 配置のテスト

- **Definition** (pack.toml) — 誰かがこの pack を import したら、このフィールドは付いてきて意味を持つか？
- **Deployment** (city.toml) — チームメイトがこの値を共有するが、同じ pack の別 deployment では共有しないか？
- **Site binding** (`.gc/`) — マシンごと、派生、運用、または永続的な副作用を持つか？

### Identity

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[pack].name` | **yes** | | | pack identity は definition |
| `[pack].version` | **yes** | | | pack version は definition |
| Workspace 名 | | | **yes** | 登録時に `pack.name` から派生 |

### Composition

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[imports]` | **yes** | | | この city を構成する pack |
| `[defaults.rig.imports.<binding>]` | **yes** | | | 新しい rig 向けデフォルト imports |

### Agents と sessions

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| Agent 定義 | **yes** | | | 振る舞いの definition |
| `[[named_session]]` | **yes** | | | 振る舞いの definition |
| `[agent_defaults]` のデフォルト | **yes** | **yes** | | pack.toml で pack 全体、city.toml で city レベルの上書き |
| `[patches]` | **yes** | | | definition レベルの修正 |

### Providers

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[providers]` の model/デフォルト | **yes** | | | 振る舞いの definition |
| Provider のクレデンシャル/エンドポイント | | | **yes** | 開発者ごと |

### Rigs

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[[rigs]].name` | | **yes** | | 構造的な deployment 設定 |
| `[[rigs]].path` | | | **yes** | マシンローカル binding |
| `[[rigs]].prefix` | | | **yes** | 派生で、bead ID に焼き込まれる |
| `[[rigs]].suspended` | | | **yes** | 運用トグル |
| `[[rigs]].imports` | | **yes** | | チーム共有の rig composition |
| `[[rigs]].patches` | | **yes** | | deployment 固有のカスタマイズ |
| `[[rigs]].max_active_sessions` | | **yes** | | deployment の容量 |

### ランタイム substrates

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[beads]`、`[session]`、`[events]` | | **yes** | | substrate 選択は deployment |
| `[session].socket` | | | **yes** | マシンローカルな tmux 状態 |

### インフラ

| フィールド | pack.toml | city.toml | `.gc/` | 根拠 |
|---|---|---|---|---|
| `[api].port` | | **yes** | | チームのデフォルト |
| `[api].bind` | | | **yes** | マシンローカルなネットワーク |
| `[daemon]`、`[orders]`、`[convergence]` | | **yes** | | deployment の振る舞い |

---END ISSUE---
