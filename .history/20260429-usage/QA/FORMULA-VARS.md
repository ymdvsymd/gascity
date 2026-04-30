# QA: formula から formula へどうやって変数を渡すか

このドキュメントは、Gas City の `formula` 同士でどうやって変数（`vars`）を受け渡しするか、という観点で実装ベースに整理したものです。最後にローカル実機で end-to-end の動作検証も行ったので、その結果も併記します。

> 調査時点: 2026-04-30 / 対象: `main` ブランチ HEAD
> 検証環境: `/Users/to.watanabe/workspace/my-city`, `/Users/to.watanabe/workspace/my-project`
> 関連実装: `internal/formula/`, `internal/dispatch/fanout.go`

---

## TL;DR

- **formula = TOML ファイル1個**。step ではない。`formula = "name"` は formula の名前。
- 受け取り側 formula は **`[vars]` セクションで「受け取り口」を宣言**するだけ。値の出どころ（CLI / 親 formula / runtime output）は問わない。
- 値の渡し方は **3 経路**：
  1. **CLI 直接**（cook/sling 時の `--var key=value`）
  2. **親 → 子の静的バインド**（`extends` / `compose.expand` / `expand_vars`）
  3. **`on_complete` ランタイム動的バインド**（親ステップの `output.<field>` を `for_each` で回し、各 item の値を子 formula の vars に注入する fan-out）
- 「**並列に存在する別 formula の vars を直接ルックアップする構文**」（例: `{{other_formula.var}}`）は無い。値は必ず親→子 / 出力→入力 の方向に流れる。

---

## 1. 用語整理（よく混乱するところ）

| 概念 | 実体 |
|---|---|
| **formula** | TOML ファイル1個（テンプレート、レシピ） |
| **step** | formula の中の1工程（cook 後は1つの bead になる） |
| **molecule** | formula を `cook` で実体化したもの（root bead + 子 step beads の集合） |
| **wisp** | エフェメラルな molecule |

`mol-` プレフィックスや `exp-` プレフィックスは慣習で、SDK 自体は名前に意味を持たせていない（**ZERO hardcoded roles** の原則）。

formula の TOML 構造：

```
mol-foo.toml
├── formula = "mol-foo"     # formula の名前
├── version = 1
├── description = "..."
├── [vars]                   # この formula のスコープの変数
├── [[steps]]                # 中身は複数 step（type=workflow のとき）
└── [compose]                # 他 formula との合成ルール
```

`type = "expansion"` のときは `[[steps]]` ではなく `[[template]]` を使う（後述）。

---

## 2. 受け取り側 formula の書き方

普通の formula で OK。`[vars]` セクションで「受け取り口」を宣言するだけ。受け取り専用の特別な型は無い。

```toml
formula = "child-formula"
version = 1

[vars]
[vars.issue]
description = "受け取りたい変数の説明"
required = true                  # 呼び出し側が必ず渡す必要あり

[vars.base_branch]
default = "main"                 # 呼び出し側が省略したらこの値

[vars.priority]
enum = ["low", "normal", "high"] # バリデーション
default = "normal"

[[steps]]
id = "do-thing"
description = "{{issue}} を {{base_branch}} で処理"  # ← {{...}} で参照
```

### `[vars.<name>]` で書ける属性

| 属性 | 意味 |
|---|---|
| `description` | ドキュメント用の説明 |
| `default` | 未指定時の値 |
| `required` | true なら呼び出し側が必ず指定（default と排他） |
| `enum` | 許される値のリスト |
| `pattern` | 正規表現バリデーション |
| `type` | `string`（default） / `int` / `bool` |

短縮記法：`name = "world"` だけ書けば `default = "world"` と同じ。

### バリデーション規則

- `required: true` と `default` は排他（`Validate()` で弾かれる）。
- 値が指定されず default も無い変数を本文で参照すると、**placeholder（`{{name}}` という文字列そのもの）が残る**。tutorial の流儀は「default を入れるか required にする」。
- 変数置換は **cook/sling 時の遅延束縛**。compile / show 段階では `{{...}}` のまま残る（`docs/tutorials/05-formulas.md`）。

参考実例：`internal/bootstrap/packs/core/formulas/mol-polecat-base.toml`（受け取り側）と `mol-polecat-commit.toml`（`extends = ["mol-polecat-base"]` で継承する側）。

---

## 3. 値の3経路

### 経路1: CLI から直接

```bash
gc formula cook child-formula --var issue=bd-42 --var base_branch=develop
gc sling some-agent --var ...
```

- 値が決まる時点：cook/sling 時の静的バインド
- 出どころ：人間が打鍵

### 経路2: 親 → 子（静的バインド）

ある formula を別の formula から `expand` / `compose.expand` / `map` / `hooks` / `extends` で取り込むとき、`vars` / `expand_vars` で上書きできる。親の変数を子へ流したいなら、親側で `{{parent_var}}` を文字列展開して渡す形になる。

```toml
[[steps]]
id = "do-thing"
expand = "child-formula"
expand_vars = { target = "{{parent_target}}" }   # 親の値を子へ
```

実装：`internal/formula/expand.go` の `mergeVars(expFormula, step.ExpandVars)` で子側の default に対し親の override がマージされる。

- 値が決まる時点：cook/sling 時の静的バインド
- 出どころ：親 formula の TOML に literal で書かれた値（`{{parent_var}}` 経由なら親の var）

### 経路3: `on_complete` ランタイム動的バインド（fan-out）

実行時の「あるステップの出力 JSON」を別 formula の vars に流し込める唯一の経路。`for_each: output.<field>` で配列を回し、各要素を `bond` 先 formula の var に束ねる。

```toml
[[steps]]
id = "survey-workers"
description = "Inspect rigs and emit JSON: { polecats: [{name, rig}, ...] }"

[steps.on_complete]
for_each = "output.polecats"   # 親stepが出した output 配列
bond     = "exp-arm"           # 各要素ごとに molecule を作る
parallel = true

[steps.on_complete.vars]
polecat_name = "{item.name}"   # ← runtime 値が子formula のvar に入る
rig          = "{item.rig}"
```

- `{item}` / `{item.field}` / `{index}`（0始まり）が予約語（`internal/formula/types.go:684-687`）。
- `for_each` は `output.` プレフィックス必須。
- `parallel` と `sequential` は排他。
- 実装：`internal/formula/types.go:660-698`、`internal/dispatch/fanout.go`。

- 値が決まる時点：**ランタイム**（親ステップ完了直後）
- 出どころ：親ステップが出した `output` JSON

---

## 4. 経路3 の運用要件と落とし穴

ここが今回最大の調査ポイント。実装と検証の結果、以下の3条件を満たさないと動かない。

### 4.1 `[daemon] formula_v2 = true` が必須

`city.toml` にこのフラグを足す：

```toml
[daemon]
formula_v2 = true
```

無いと `contract = "graph.v2"` を持つ formula が下記エラーで弾かれる：

```
formula "exp-survey" declares contract graph.v2 but formula_v2 is disabled;
enable [daemon] formula_v2 or remove the graph.v2 contract
```

実装：`internal/config/config.go:1216` の `DaemonConfig.FormulaV2` フラグ。

**注意：このエラーは `gc formula show` では silent に潰れる**（exit=1 だが stdout/stderr に何も出ない）別バグあり。`gc convoy control` だと正しく表示される。

### 4.2 送り側 formula は `contract = "graph.v2"` 必須

`on_complete` を使う formula は graph.v2 コントラクトを明示しないと Validate が通らない（`internal/formula/types.go:933, 974-976` の `requiresExplicitGraphContract`）。

```toml
formula = "exp-survey"
version = 1
contract = "graph.v2"   # ← 必須

[[steps]]
id = "survey-workers"
[steps.on_complete]
for_each = "output.polecats"
bond     = "exp-arm"
```

### 4.3 受け側 formula は `type = "expansion"` + `[[template]]`

`bond` 先は **`type = "expansion"` で、本文が `[[template]]`**（`[[steps]]` ではない）でないと dispatcher が弾く：

```
"exp-arm" is not an expansion formula (type=workflow)
```

正しい形：

```toml
formula = "exp-arm"
version = 1
type = "expansion"      # ← 必須

[vars]
[vars.polecat_name]
required = true
[vars.rig]
required = true

[[template]]            # ← steps ではなく template
id = "arm"
title = "Arm {polecat_name}"
description = "Deploy polecat {polecat_name} into rig {rig}."
```

### 4.4 ブレース規則（重要・間違えやすい）

| 場所 | 記法 | 何を置換 |
|---|---|---|
| `[[steps]]` 系 formula 本文 | `{{var}}` | `[vars]` の値 |
| **`[[template]]`（expansion 系）本文** | **`{var}`** | **`[vars]` の値（単ブレース）** |
| `on_complete.vars` の値側 | `{item.field}`, `{index}` | runtime の output JSON 要素 |

`internal/formula/range.go:94` の `substituteVars` が**単ブレース**で動くため、expansion template に `{{var}}` と書くと内側の `{var}` が残る（実機検証済み、後述）。

---

## 5. 実機検証（end-to-end）

`/Users/to.watanabe/workspace/my-city` で実走した。手順と結果を記録。

### 5.1 セットアップ

`city.toml`：

```toml
[workspace]
provider = "claude"

[daemon]
formula_v2 = true

[[rigs]]
name = "my-project"
path = "/Users/to.watanabe/workspace/my-project"
```

`formulas/exp-survey.toml`（送り側）：

```toml
formula = "exp-survey"
version = 1
contract = "graph.v2"

[[steps]]
id = "survey-workers"
title = "Survey workers"
description = "Inspect rigs and emit JSON: {\"polecats\":[{\"name\":...,\"rig\":...}]}"

[steps.on_complete]
for_each = "output.polecats"
bond     = "exp-arm"
parallel = true

[steps.on_complete.vars]
polecat_name = "{item.name}"
rig          = "{item.rig}"
```

`formulas/exp-arm.toml`（受け側）：

```toml
formula = "exp-arm"
version = 1
type = "expansion"

[vars]
[vars.polecat_name]
required = true
[vars.rig]
required = true

[[template]]
id = "arm"
title = "Arm {polecat_name}"
description = "Deploy polecat {polecat_name} into rig {rig}."
```

### 5.2 実走シーケンス

```bash
# 1. cook で 4 beads が作られる（root + 3 step）
gc formula cook exp-survey
# Root: mp-me9x
# Created: 4
# exp-survey -> mp-me9x
# exp-survey.survey-workers -> mp-0sqh
# exp-survey.survey-workers-fanout -> mp-qk4o   ← 自動付与の fanout step
# exp-survey.workflow-finalize -> mp-0s5t      ← 自動付与の finalize step

# 2. agent の代わりに output JSON を直接セット
bd update mp-0sqh --set-metadata \
  'gc.output_json={"polecats":[{"name":"alice","rig":"r1"},{"name":"bob","rig":"r2"}]}'
bd update mp-0sqh --status closed

# 3. fanout step を手動駆動（通常は controller の serve loop が自動でやる）
gc convoy control mp-qk4o
# control dispatch: bead=mp-qk4o action=fanout-spawn created=2
```

### 5.3 結果

`exp-arm` molecule が **2 個展開**された（polecats 配列が 2 要素なので）：

| run | template記法 | 結果 title | 結果 description |
|---|---|---|---|
| 1（誤） | `{{polecat_name}}` | `Arm {alice}` | `Deploy polecat {alice} into rig {r1}.` |
| **2（正）** | **`{polecat_name}`** | **`Arm alice`** | **`Deploy polecat alice into rig r1.`** |

run 2 で値が完全に展開され、想定通りの bead が runtime に生まれた。

### 5.4 ランタイム実装の核心

`internal/dispatch/fanout.go:475` の `resolveFanoutItems`：

```go
raw := source.Metadata["gc.output_json"]   // ← agent が close 前にセット
// JSON parse → for_each path で配列抽出 → 各 item ごとに molecule 生成
```

つまり **agent は work 完了時に bead の `gc.output_json` メタデータに JSON 文字列をセットしてから close する**、という契約。これが「formula 間の実行時データフロー」の正体。

---

## 6. 経路間の対比

| 観点 | 経路1（CLI） | 経路2（compose） | 経路3（on_complete） |
|---|---|---|---|
| 値が決まる時点 | cook/sling 時 | cook/sling 時 | ランタイム |
| バインドの種類 | 静的 | 静的 | 動的 |
| 出どころ | 人間 | 親 formula の TOML | 親ステップの output JSON |
| 用途 | 単発実行 | 再利用可能な合成 | fan-out（N 個の子 molecule） |
| 必要な設定 | なし | なし | `formula_v2`, `graph.v2`, `expansion` |
| 受け側 formula | 通常 | 通常 | `type = "expansion"` + `[[template]]` |

---

## 7. 重要な設計原則

3経路すべてで、**子 formula 側のインターフェースは `[vars.<name>]` ひとつ**。子 formula は誰が値を入れたか知らない／知る必要がない。これが **Gas City の formula が「呼び出し元から疎」でいられる理由**で、再利用性の根幹。

「並列に存在する別 formula の vars を直接ルックアップする構文」（例: `{{other_formula.var}}`）が**意図的に無い**のも同じ理由。値は必ず「親→子」「出力→入力」の方向に流れ、formula は呼び出し元を知らない。

---

## 8. 参考ファイル一覧

- `internal/formula/types.go` — `Formula`, `Step`, `VarDef`, `OnCompleteSpec` の型定義
- `internal/formula/expand.go` — compose / expand 系の置換実装
- `internal/formula/range.go:94` — `substituteVars`（単ブレース置換）
- `internal/dispatch/fanout.go` — `processFanout`, `resolveFanoutItems`
- `internal/config/config.go:1216` — `DaemonConfig.FormulaV2`
- `internal/bootstrap/packs/core/formulas/mol-polecat-base.toml` — `[vars]` 受け取り口の実例
- `docs/tutorials/05-formulas.md` — vars / 遅延束縛のチュートリアル
- `cmd/gc/cmd_formula.go:83` — `gc formula show`（silent failure バグあり）

---

## 9. 残課題（メモ）

- `gc formula show` が `formula_v2 disabled` のエラーを silent に潰す（exit=1 だが出力なし）。`internal/formula/compile.go` 周辺の error path を要確認。upstream 報告候補。
- `on_complete` を使った TOML formula はリポジトリ内に **1 件も存在しない**（grep 確認済み）。型・パーサ・グラフ化・ランタイム dispatcher は揃っているが、実運用例はまだ無い。
