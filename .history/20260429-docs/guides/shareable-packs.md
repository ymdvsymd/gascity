---
title: "共有可能な Packs"
description: PackV2 Gas City pack を作成、インポート、カスタマイズする。
---

pack は、エージェント、プロンプトテンプレート、provider、formula、order、command、doctor チェック、overlay、skill、その他の再利用可能なアセットからなる、可搬性のある挙動定義です。city は、ルート pack に `city.toml` デプロイメントファイルとマシンローカルの `.gc/` バインディングを加えたものです。

PackV2 は 3 つの懸念事項を分離します:

- `pack.toml` と pack ディレクトリがシステムが何であるかを定義します。
- `city.toml` がこのデプロイメントがどう走るかを定義します。
- `.gc/` が `gc` によって管理されるローカル site binding とランタイム状態を保存します。

レガシーの `includes`、`[packs.*]`、`[[agent]]` の例は移行互換性のために依然としてロードされる場合がありますが、新しいドキュメントと新しい pack は PackV2 imports と `agents/<name>/` ディレクトリを使うべきです。

## Pack レイアウト

pack 構造は convention ベースです。標準ディレクトリは名前でロードされ、不透明なヘルパーファイルは `assets/` 配下に置かれます。

```text
code-review-pack/
├── pack.toml
├── agents/
│   └── reviewer/
│       ├── agent.toml
│       └── prompt.template.md
├── formulas/
│   └── review-change.toml
├── orders/
│   └── nightly-review.toml
├── commands/
│   └── status/
│       ├── help.md
│       └── run.sh
├── doctor/
│   └── check-review-tools/
│       └── run.sh
├── overlay/
├── skills/
├── mcp/
├── template-fragments/
└── assets/
    └── scripts/
        └── setup-reviewer.sh
```

## 最小の `pack.toml`

pack のメタデータと imports は `pack.toml` に存在します。エージェント定義は `[[agent]]` テーブルではなく `agents/<name>/` に存在します。

```toml
[pack]
name = "code-review"
schema = 2
version = "1.0.0"

[agent_defaults]
provider = "claude"
scope = "rig"
```

`schema = 2` は現行の PackV2 形式です。`[agent_defaults]` は、エージェント自身の `agent.toml` がフィールドをオーバーライドしない限り、`agents/` から発見されたエージェントに適用されます。

## エージェントディレクトリ

最小のエージェントは、プロンプトを持つディレクトリだけです:

```text
agents/reviewer/
└── prompt.template.md
```

pack デフォルトと異なるフィールドには `agent.toml` を使用します:

```toml
# agents/reviewer/agent.toml
scope = "rig"
nudge = "Check your hook, review the assigned change, and leave findings."
idle_timeout = "30m"
min_active_sessions = 0
max_active_sessions = 3
pre_start = ["{{.ConfigDir}}/assets/scripts/setup-reviewer.sh {{.RigRoot}}"]
```

プロンプトファイルの発見では `prompt.template.md` が優先されます。`prompt.md` と `prompt.md.tmpl` は互換性のために受け入れられます。

## Imports

pack は名前付き imports で他の pack を合成します。imports は出自を保持するので、消費者は `gastown.polecat` と `review.polecat` を区別できます。

```toml
[imports.maintenance]
source = "../maintenance"
export = true
```

ローカル imports はインポート元の pack からの相対パスを使用します。リモート imports は source とバージョン制約を使用します:

```toml
[imports.gastown]
source = "github.com/gastownhall/gastown"
version = "^1.2"
```

imports はデフォルトで推移的です。import が pack に対して内部的で消費者に見えるべきでない場合のみ `transitive = false` を設定します。

## City での使用

city はルート pack レベルで pack をインポートし、デプロイメント詳細を `city.toml` で宣言します。

```toml
# pack.toml
[pack]
name = "bright-lights"
schema = 2

[imports.gastown]
source = "./assets/gastown"

[imports.review]
source = "./assets/code-review"
```

```toml
# city.toml
[beads]
provider = "bd"

[[rigs]]
name = "backend"
max_active_sessions = 4
default_sling_target = "backend/gastown.polecat"
```

マシンローカルの rig パスは `gc` によって管理される site binding です:

```bash
gc rig add ~/src/backend --name backend
```

## Rig レベルの Imports

ある rig だけが pack のエージェントや formula を受け取るべき場合は、rig レベルの imports を使用します。

```toml
[[rigs]]
name = "backend"

[rigs.imports.gastown]
source = "./assets/gastown"

[rigs.imports.review]
source = "./assets/code-review"
```

rig レベルの imports は `backend/gastown.polecat` や `backend/review.reviewer` のような rig スコープのアイデンティティを作成します。

## 名前付きセッション

pack は現在の作業から独立して存在すべきセッションを宣言できます。

```toml
[[named_session]]
template = "mayor"
scope = "city"
mode = "always"

[[named_session]]
template = "polecat"
scope = "rig"
mode = "on_demand"
```

`template` は同じ pack のエージェント名、または必要に応じて imported された修飾名です。

## インポートしたエージェントのカスタマイズ

patches を使うと、インポートしたエージェントを再定義せずに変更できます。

```toml
[[patches.agent]]
name = "gastown.mayor"
provider = "codex"
idle_timeout = "2h"

[patches.agent.env]
GC_MODE = "coordination"
```

rig 固有のカスタマイズには、rig の下で patch します:

```toml
[[rigs]]
name = "backend"

[[rigs.patches]]
agent = "gastown.polecat"
provider = "gemini"

[rigs.patches.pool]
max = 8
```

## Formula と Order ファイル

Formula ファイルは `formulas/` に、order ファイルは `orders/` に配置します。PackV2 pack には `[formulas].dir` 宣言は必要ありません。

```text
formulas/
└── review-change.toml

orders/
└── nightly-review.toml
```

複数の pack が同じ formula 名を提供する場合、インポート元の pack がインポート先の pack に勝ちます。rig レベルの imports は、その rig に対して city レベルの formula をオーバーライドできます。

## 互換性ノート

ローダは、移行および古い city のサポートのために、いくつかの V1 フィールドを依然として公開しています:

- `workspace.includes`
- `[[rigs]].includes`
- `[packs.*]`
- `[[agent]]`
- `[formulas].dir`

これらを移行サーフェスとして扱ってください。新しい共有可能な pack は PackV2 を使用すべきです: `schema = 2`、`[imports.*]`、`agents/<name>/`、慣例の `formulas/`、カスタマイズ用の patches。
