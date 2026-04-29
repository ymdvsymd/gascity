# Agent Definition v.next

**GitHub Issue:** [gastownhall/gascity#356](https://github.com/gastownhall/gascity/issues/356)

タイトル: `feat: Agent Definition v.next — agents as directories`

これは pack/city モデルの再設計を扱う [doc-pack-v2.md](doc-pack-v2.md) の companion です。

> **同期の維持:** このファイルが真実の根拠です。更新時はここを編集し、issue 本文を `gh issue edit 356 --repo gastownhall/gascity --body-file <(sed -n '/^---BEGIN ISSUE---$/,/^---END ISSUE---$/{ /^---/d; p; }' issues/doc-agent-v2.md)` で更新してください。

> [!IMPORTANT]
> このドキュメントは Gas City v0.15.0 のプレリリースロールアウトを記述しています。
> いくつかの PackV2 表面はまだ活発に開発中です。リリースゲート付きの注意書きには
> 「As of release v0.15.0, ...」という形式を使います。

---BEGIN ISSUE---

## 課題

agent 定義は `[[agent]]` TOML テーブルと、別々のディレクトリツリーに散らばったファイルシステム資産（prompts、overlays、scripts）に分割されています。これにより 6 つの問題が生じます。
v
1. **identity の散逸。** agent が何であるかを理解できる単一の場所がありません。agent を追加するには city.toml の編集 *と*、複数のディレクトリ（`prompts/`、`overlay/`、`scripts/`）でのファイル作成が必要になります。

2. **不可視な prompt 注入。** 各 `.md` ファイルは密かに Go テンプレートです。フラグメントは `global_fragments` と `inject_fragments` で注入されますが、prompt ファイル自体には現れません。prompt を読んでも agent が実際に何を見ているかは分かりません。

3. **Provider のファイルがプロバイダ間で漏れる。** Overlay ファイル（`.claude/settings.json`、`CLAUDE.md`）は、agent が使う provider に関係なくすべての agent の作業ディレクトリにコピーされます。Codex agent が Claude の settings を受け取ってしまいます。

4. **skills や MCP server の置き場がない。** [Agent Skills](https://agentskills.io) 標準は 30 以上のツール（Claude Code、Codex、Gemini、Cursor、Copilot 等）に採用されていますが、Gas City には pack に skills を同梱する convention がありません。MCP server 設定はプロバイダ固有の JSON で overlay ファイルに焼き込まれ、抽象化されていません。最初のスライスでは、両表面とも現 city pack 限定で、imported pack のカタログは後ほどです。

5. **Definition と修正の混同。** 「自分の agent を定義している」のと「imported agent を調整している」の間に分離がありません。両方とも `[[agent]]` テーブルを使い、衝突解決は読み込み順序と `fallback` フラグに依存します。

6. **その場限りの資産配線。** Overlay、prompt、script はそれぞれ独自の機構（`overlay_dir`、`prompt_template`、`scripts_dir`）を持ち、一貫したパターンがありません。

## 提案する変更: agents as directories

agents は convention で定義されます。`agents/` 内のディレクトリで、少なくとも `prompt.md` ファイルを 1 つ持つものです。追加の資産はすべて agent のディレクトリ内に置かれ、オプションの `agent.toml` ファイルに任意の設定が入ります。

**最小の agent** — prompt だけで、すべてのデフォルトを継承:

```
agents/polecat/
└── prompt.md
```

**設定上書きを伴う agent:**

```
agents/mayor/
├── agent.toml         # オプション — デフォルトを上書き
└── prompt.md          # 必須 — system prompt
```

**完全に設定された agent**、agent ごとの資産付き:

```
agents/mayor/
├── agent.toml         # オプション — デフォルトを上書き
├── prompt.md          # 必須 — system prompt
├── namepool.txt       # オプション — pool セッション用の表示名
├── overlay/           # オプション — agent 固有の overlay ファイル
│   ├── AGENTS.md      # provider に依存しない instructions（全 provider にコピー）
│   └── per-provider/
│       └── claude/
├── skills/            # オプション — agent 固有の skills
├── mcp/               # オプション — agent 固有の MCP server
└── template-fragments/ # オプション — agent 固有の prompt フラグメント
```

city 全体の資産と複数の agents を含む **完全な city:**

```
my-city/
├── city.toml
├── agents/
│   ├── polecat/
│   │   └── prompt.md
│   └── mayor/
│       ├── agent.toml
│       └── prompt.md
├── overlay/                   # city 全体の overlay（全 agent）
│   ├── per-provider/
│   │   ├── claude/
│   │   │   ├── .claude/
│   │   │   │   └── settings.json
│   │   │   └── CLAUDE.md
│   │   └── codex/
│   │       └── AGENTS.md
│   └── .editorconfig          # provider 非依存（全 agent）
├── skills/                    # city 全体の skills（全 agent）
├── mcp/                       # city 全体の MCP server（全 agent）
├── template-fragments/        # city 全体の prompt template フラグメント
├── formulas/
├── orders/
├── commands/
├── doctor/
├── patches/
└── assets/
```

### city.toml: agent デフォルト

`[[agent]]` テーブルは共有デフォルト用に `[agent_defaults]` で置き換えられます。このブロックは `pack.toml`（pack 全体のポータブルなデフォルト）と `city.toml`（city レベルの deployment 上書き）の両方に現れることができ、city が pack の上に重なります。

```toml
# pack.toml — pack 全体のデフォルト
[agent_defaults]
default_sling_formula = "mol-do-work"
```

```toml
# city.toml — city レベルの上書き（オプション）
[agent_defaults]
append_fragments = ["operational-awareness"]
```

As of release v0.15.0、実際に適用されるデフォルトは依然として狭く、`default_sling_formula` と prompt 描画時の `[agent_defaults].append_fragments` です。他の `AgentDefaults` フィールドはパースされて構成されますが、ランタイムで自動継承されることはまだありません。`provider` や `scope` などの agent ごとのフィールドは依然として `agents/<name>/agent.toml` にあります。

個々の agent は自身の `agent.toml` で上書きします。

```toml
# agents/mayor/agent.toml — デフォルトと異なるところだけ
scope = "city"
max_active_sessions = 1
```

最小の agent（`prompt.md` だけのディレクトリ）はすべてのデフォルトを継承し、`agent.toml` を必要としません。

### Pool agents

Pool 動作は構造ではなく設定です。pool agent は同じ定義から複数の同時セッションを spawn する agent です — 単一セッションで処理しきれないペースで作業が来るときに役立ちます。controller は設定されたバウンド内でセッション数を需要に応じてスケールアップ・ダウンします。

```toml
# agents/polecat/agent.toml
min_active_sessions = 1
max_active_sessions = 3
```

agent のディレクトリに `namepool.txt` ファイル（1 行に 1 名前）が含まれていれば、各セッションは表示エイリアスとしてその中から名前を受け取ります — TOML フィールドは不要、`prompt.md` と同じ convention-over-configuration アプローチです。すべてのインスタンスは同じ prompt、skills、MCP server、overlay を共有します — セッションの identity と作業ディレクトリだけが異なります。

### Provider 対応の overlay

overlay は agent 起動前に作業ディレクトリに materialize されるファイルです。プロバイダ固有のファイルは `per-provider/` サブディレクトリに置かれ、agent はそのプロバイダ向けのファイルだけを受け取ります。

レイヤリング順序（あとが勝つ、ファイル衝突時）:

1. city 全体の `overlay/` — universal なファイル（`per-provider/` の外すべて）
2. city 全体の `overlay/per-provider/<provider>/` — provider にマッチ
3. agent 固有の `agents/<name>/overlay/` — universal なファイル
4. agent 固有の `agents/<name>/overlay/per-provider/<provider>/` — provider にマッチ

`<provider>` 名は Gas City のプロバイダ名（`claude`、`codex`、`cursor` 等）にマッチします。agent のプロバイダを切り替えると、適用される overlay ファイルが変わります — 手動でクリーンアップする必要はありません。

これにより、city は異なるプロバイダ向けに別々の `CLAUDE.md` と `AGENTS.md` ファイルを同梱でき、各 agent は自分のプロバイダ向けのものだけを見ます。

### Skills

skills は [Agent Skills](https://agentskills.io) オープン標準を使います。Claude Code、Codex、Gemini、Cursor、GitHub Copilot、JetBrains Junie、Goose、Roo Code を含む 30 以上のプロバイダで採用されています。

skill は `SKILL.md` ファイル（YAML frontmatter + markdown instructions）を含むディレクトリで、オプションで `scripts/`、`references/`、`assets/` のサブディレクトリを持ちます。

```
skills/code-review/
├── SKILL.md               # 必須: メタデータ + instructions
├── scripts/               # オプション: 実行可能コード
├── references/            # オプション: ドキュメント
└── assets/                # オプション: テンプレート、リソース
```

```yaml
# SKILL.md frontmatter
---
name: code-review
description: Reviews code changes for bugs, security issues, and style. Use when reviewing PRs or changed files.
---
```

skills は **プロバイダ間でポータブル** です。同じ SKILL.md は Claude Code、Codex、Gemini、その他準拠 agent で動作します。後のスライスでは、Gas City は起動時に skills を agent の作業ディレクトリ内のプロバイダ期待位置に materialize します（例: Claude Code には `.claude/skills/`、Codex には `.agents/skills/`）。

現 city pack 内では skills は city 全体または agent ごとに置けます。

```
my-city/
├── skills/                    # city 全体 — 全 agent から利用可能
│   ├── code-review/
│   │   └── SKILL.md
│   └── test-runner/
│       ├── SKILL.md
│       └── scripts/
│           └── run-tests.sh
├── agents/
│   └── polecat/
│       └── skills/            # agent 固有 — この agent のみ
│           └── polecat-workflow/
│               └── SKILL.md
```

agent は city 全体の skills + 自身の skills を受け取ります。名前衝突時は agent 固有が勝ちます。

> **最初のスライス:** skills の発見/materialize は現 city pack 限定です。imported pack の skills カタログは後ほどです。

最初の skills CLI スライスは list 専用です:

```sh
gc skill list
gc skill list --agent polecat
gc skill list --session <id>
```

#### Skill の昇格

> **後のスライス:** 最初の skills 表面は list 専用です。Promote/retain フローはここで設計メモとして記述しますが、最初の実装スライスではまだ必要ありません。

agent がセッション中（rig の作業ディレクトリ内）に skill を作成すると、その rig にローカルなまま残ります。それを city 定義に取り込むには:

```
gc skill promote code-review --to city        # city の skills/ にコピー
gc skill promote code-review --to agent mayor  # agents/mayor/skills/ にコピー
```

昇格は明示的な人間の決定です — skills は rig から city に自動的には流れません。

### MCP server

MCP（Model Context Protocol）サーバーはランタイムプロトコルで agent にツール、リソース、prompt を提供します。skills（ポータブルなファイル標準を持つ）と異なり、MCP server 設定はプロバイダ固有 — 各プロバイダが独自の settings ファイルに埋め込みます。Gas City はこれを provider 非依存の TOML フォーマットで抽象化します。

`gc mcp list` は projection 専用かつターゲット固有です:

```sh
gc mcp list --agent polecat
gc mcp list --session <id>
```

> **破壊的変更:** ターゲットフラグなしの bare `gc mcp list` は今エラーに
> なります。Projected MCP は具体的な agent またはセッションターゲットに
> 依存するので、ターゲットなしの形式には well-defined な意味がありません。
> これまで pack inventory チェックとして `gc mcp list` を実行していた自動化は
> `--agent` または `--session` に切り替える必要があります。

ターゲットが effective な MCP を持つとき、Gas City はプロバイダネイティブの
MCP 表面を GC 管理のランタイム状態として採用します。最初の採用時、既存の
プロバイダネイティブの内容は
`.gc/mcp-adopted/<provider>/<timestamp>.<ext>` にスナップショットされ、
1 行の警告が stderr に出力されるので、手書きの `.mcp.json`/`settings.json`/
`config.toml` エントリは復元できます。シンボリックリンクのターゲットは
無条件で拒否されます — 管理対象ターゲットは通常ファイルである必要があります。

各 stage-1 reconcile でのクリーンアップは、city ルート配下と **依然として
attach されているすべての** rig 配下の `.gc/mcp-managed/` を歩き、claimant が
もういない managed marker/target（`city.toml` から削除された agent、
プロバイダ変更、MCP ディレクトリ削除）を取り除きます。`city.toml` から detach
された rig は設定済みルートから到達できなくなるため、その managed marker は
残り、手動または明示的な `gc rig detach` ツール経由でクリーンアップされる必要が
あります。GC は managed なランタイム artifact をローカルの `.gitignore` にも
ベストエフォートで追加し、effective な MCP の変更はセッションのフィンガー
プリントに参加するため、影響を受けるセッションは drift で再起動します。

> **テンプレート展開と TOML エスケープ。** `.template.toml` ファイルは
> TOML パースの *前に* Go の `text/template` で展開されます。`"`、`\`、改行を
> 含む値は invalid な TOML を生み出す可能性があります — パースエラーは
> 展開後のファイルを指し、テンプレートではありません。秘密値はシンプルな
> 文字列に保つ（埋め込みクォート/バックスラッシュなし）か、Go の
> `printf "%q"` テンプレート関数で自分でエスケープして展開後の出力が
> valid な TOML になるようにしてください。

#### 定義フォーマット

MCP server は `mcp/` 内の名前付き TOML ファイルです。

```toml
# mcp/beads-health.toml
name = "beads-health"
description = "Query bead status and health metrics"
command = "scripts/mcp-beads-health.sh"
args = ["--city-root", "."]

[env]
BEADS_DB = ".beads"
```

テンプレート展開（動的なパス、クレデンシャル）が必要なときは `.template.toml` を使います。

```toml
# mcp/beads-health.template.toml
name = "beads-health"
description = "Query bead status and health metrics"
command = "assets/mcp-beads-health.sh"
args = ["--city-root", "{{.CityRoot}}"]

[env]
BEADS_DB = "{{.RigRoot}}/.beads"
```

prompts と同じ `.template.` ルールです — プレーンな `.toml` は静的、`.template.toml` は `PromptContext` 変数を伴う Go template 展開を通ります。

リモート MCP server は `command` の代わりに `url` を使います。

```toml
# mcp/sentry.template.toml — .template.toml が Go template 展開をトリガー
name = "sentry"
description = "Sentry error tracking integration"
url = "https://mcp.sentry.io/sse"

[headers]
Authorization = "Bearer {{.SENTRY_TOKEN}}"
```

#### フィールド仕様

| フィールド | 必須 | 説明 |
|---|---|---|
| `name` | はい | server 名（拡張子を除いたファイル名と一致する必要あり） |
| `description` | はい | この server が提供するもの |
| `command` | はい* | ローカル server を起動するコマンド（stdio transport） |
| `args` | いいえ | コマンドへの引数 |
| `url` | はい* | リモート server の URL（HTTP transport） |
| `headers` | いいえ | リモート server 向けの HTTP ヘッダ |
| `[env]` | いいえ | ローカル server に渡される環境変数 |

*`command` または `url` のいずれかが必須です。

#### Gas City が agent 起動時に行うこと（後のスライス）

1. この agent 向けのすべての MCP server 定義を集める（city 全体 + agent 固有）
2. `.template.toml` ファイルを template 展開する
3. `command` パスを絶対パスに解決する（スクリプトは rig にコピーされません）
4. プロバイダの設定フォーマットに注入する:
   - Claude Code: `.claude/settings.json` の `mcpServers` にマージ
   - Cursor: `.cursor/mcp.json` の `mcpServers` にマージ
   - VS Code/Copilot: VS Code 設定にマージ
   - その他: サポート範囲内でプロバイダ固有のマッピング

各 MCP server は別ファイルなので、複数の pack の MCP server がきれいにマージされます — 単一の settings ファイルでの last-writer-wins はありません。

> **後のスライス:** プロバイダ設定への projection は最初のスライスから意図的に分離されています。中立な TOML モデルとリスト可視性を、最初の実装境界として維持します。

### Prompt と template

**`.template.` インフィックスは template 処理に必須 ([#582](https://github.com/gastownhall/gascity/issues/582))。** `prompt.md` はプレーンな markdown — テンプレートエンジンは動きません。`prompt.template.md` は Go の `text/template` を通ります。「すべてが密かにテンプレート」はもうありません。

これは prompt だけでなく、すべてのファイルタイプに適用されます。ファイルが template 展開を必要とするなら、名前に `.template.` が入ります（例: `prompt.template.md`、`beads-health.template.toml`）。そうでなければ、入りません。

### Template フラグメント

フラグメントは prompt 内容の再利用可能な塊です。`.template.md` ファイルで定義された名前付き Go テンプレートです。

```markdown
{{ define "command-glossary" }}
Use `/gc-work`, `/gc-dispatch`, `/gc-agents`, `/gc-rigs`, `/gc-mail`,
or `/gc-city` to load command reference for any topic.
{{ end }}
```

フラグメントは city または pack レベルの `template-fragments/` に置かれます。

```
my-city/
├── template-fragments/
│   ├── command-glossary.template.md
│   ├── operational-awareness.template.md
│   └── tdd-discipline.template.md
├── agents/
│   ├── mayor/
│   │   └── prompt.template.md
│   └── polecat/
│       └── prompt.md
```

prompt が `.template.md` の agent はフラグメントを明示的に取り込めます。

```markdown
# Mayor

You are the mayor of this city.

{{ template "operational-awareness" . }}

---

{{ template "command-glossary" . }}
```

prompt がプレーンな `.md` の agent はフラグメントを使えません — テンプレートエンジンが動かないからです。

**これが置き換えるもの:**

| 現行の機構 | 新モデル |
|---|---|
| workspace 設定の `global_fragments` | 廃止 — 各 prompt が必要なものを明示的に include |
| agent 設定の `inject_fragments` | 廃止 — 同じ理由 |
| patch の `inject_fragments_append` | 廃止 — 同じ理由 |
| `prompts/shared/*.template.md` | city レベルの `template-fragments/*.template.md` |
| すべての `.md` ファイルが Go テンプレートを通る | `.template.md` ファイルだけが Go テンプレートを通る |

3 層の注入パイプライン（inline templates → global_fragments → inject_fragments）が 1 つに収束します。**`.template.md` ファイル内の明示的な `{{ template "name" . }}` です。** prompt ファイルが、agent が見る内容の単一の真実の根拠です。

#### Auto-append（オプトイン）

移行と利便性のため、city 全体や pack 全体のデフォルトが `[agent_defaults].append_fragments` 経由でフラグメントを auto-append できます。

```toml
# pack.toml or city.toml
[agent_defaults]
append_fragments = ["operational-awareness", "command-glossary"]
```

agent ローカルの `append_fragments` は [#671](https://github.com/gastownhall/gascity/issues/671) で追跡されるフォローアップのままで、release v0.15.0 時点ではサポート対象の移行契約には含まれていません。

`append_fragments` は `.template.md` prompt にしか効きません。プレーンな `.md` prompt は不活性です — 何も注入されず、テンプレートエンジンも動きません。

### 暗黙の agents

Gas City は設定された各プロバイダ（claude、codex、gemini 等）に対して組み込みの agent を提供し、`gc init` 直後に agent 設定なしで `gc sling claude "do something"` が即座に動くようにしています。

暗黙の agents は同じディレクトリ convention に従います。`gc` バイナリから `.gc/system/agents/` に materialize されます。

```
.gc/system/agents/
├── claude/
│   └── prompt.md
├── codex/
│   └── prompt.md
└── gemini/
    └── prompt.md
```

**Shadowing:** 同じ名前のユーザー定義 agent はシステム暗黙に勝ちます。優先度チェーン（低→高）:

1. **System 暗黙** (`.gc/system/agents/`) — 最低限、常に存在
2. **Pack 定義** (pack 内の `agents/claude/`) — システムを上書き
3. **City 定義** (city の `agents/claude/`) — pack を上書き

### Agent patches

patches は新しい agent を定義することなく、imported agent を修正します。agent 定義とは異なります — `agents/<name>/` は常にあなたの agent を作り、patches は他の誰かの agent を修正します。

**設定のみの patch** — qualified 名で agent.toml フィールドを上書き:

```toml
# city.toml
[[patches.agent]]
name = "gastown.mayor"
model = "claude-opus-4-20250514"
max_active_sessions = 2

[patches.agent.env]
REVIEW_MODE = "strict"
```

**Prompt 置き換え** — city の `patches/` ディレクトリ内のファイルにリダイレクト:

```toml
[[patches.agent]]
name = "gastown.mayor"
prompt = "gastown-mayor-prompt.md"     # patches/ からの相対
```

```
my-city/
├── city.toml
├── agents/                    # あなたの agent のみ
└── patches/                   # patch 関連のすべてのファイル
    └── gastown-mayor-prompt.md
```

主要な設計判断:
- `agents/<name>/` = 新しい agent。`[[patches.agent]]` = imported agent の修正。決して混同されません。
- patch は qualified 名（`gastown.mayor`）でターゲット指定。bare 名は曖昧でないときに動作します。
- ファイルレベル: 当面は prompt 置き換えのみ。Skills、MCP、overlay は後回し。

### Rig patches

rig patch は 1 つの rig にスコープされた agent patch です。city.toml で rig 宣言と並んで置かれます。

```toml
# city.toml
[[rigs]]
name = "api-server"

# api-server の polecat は 2 セッション; 他の rig には影響なし
[[rigs.patches]]
agent = "gastown.polecat"
max_active_sessions = 2
```

agent patches と同じフィールド、同じ qualified 命名、同じセマンティクス。違いはスコープのみです。

| 機構 | 場所 | スコープ |
|---|---|---|
| Agent patches | city.toml の `[[patches.agent]]` | 全 rig |
| Rig patches | city.toml の `[[rigs.patches]]` | 1 つの rig のみ |

**適用順序**（あとが勝つ）:

1. agent 定義（`agents/` ディレクトリから）
2. pack レベルの agent patches（pack の `[[patches.agent]]` から）
3. city レベルの agent patches（city.toml の `[[patches.agent]]` から）
4. rig patches（city.toml の `[[rigs.patches]]` から）

rig patch は 1 つの rig 向けに city レベルの patch を取り消すことができます。

## 検討した代替案

- **`[[agent]]` テーブルを残し、資産 convention を並行して追加。** identity の散逸を解決しません — 2 つの並行宣言機構は 1 つよりも悪いです。
- **プロバイダごとに別の `overlay_dir` フィールドを使う provider 固有 overlay。** 複数の pack が overlay に貢献するときに composition できません。
- **MCP 設定を生のプロバイダ JSON として overlay に同梱。** 現行アプローチ。pack をまたぐ composition ができず（settings.json での last-writer-wins）、プロバイダ間で重複します。
- **カスタム skills システムを構築する。** Agent Skills は既に 30 以上のツールに採用されています。独自構築は walled garden を作るだけです。

## 影響範囲とインパクト

- **破壊的変更:** `[[agent]]` テーブルは `agents/` ディレクトリに移動。移行ツールが必要。
- **設定:** city.toml は正規の `[agent_defaults]` デフォルトを得て、`[[agent]]` テーブルを失います。`agent.toml` は新たに agent ごと。`[agents]` は互換エイリアスとしてのみ残ります。
- **Prompts:** `.template.md` インフィックスは template 処理に必須に。`{{` を使う既存の `.md` prompt は `.template.md` への改名が必要。
- **新機能:** Skills、MCP TOML 抽象、`per-provider/` overlay、`template-fragments/` convention、`patches/` ディレクトリ。
- **命名:** 現行の `[[rigs.overrides]]` は `[[patches.agent]]` との一貫性のため `[[rigs.patches]]` に改名。
- **ドキュメント:** チュートリアルとリファレンスドキュメントの更新が必要。

## 未解決の問い

- **Skill ライフサイクル:** agent が作成した skill は自動昇格すべきか、rig ローカルにとどめるべきか、明示的な `gc skill promote` を要求すべきか? 現行の設計は明示的と言っています。
- **プロバイダ名の agent:** `agents/claude/` は `provider = "claude"` を使う必要があるか、それとも命名は単なる convention か?
- **暗黙 agents の抑制:** 「claude を provider として設定するが、暗黙の `claude` agent は欲しくない」と city はどう言うのか?
- **Patch ディレクトリ構造:** フラットな `patches/` か、ターゲット pack で名前空間化するか?
- **Patches vs. overrides の命名:** この提案ではどこでも "patches" に統一します。代替案: どこでも "overrides" に統一。重要な性質は、機構がスコープに関わらず同じであることです。

---END ISSUE---
