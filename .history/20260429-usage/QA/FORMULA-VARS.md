# QA: formula から formula へどうやって変数を渡すか

このドキュメントは、Gas City の `formula` 同士でどうやって変数（`vars`）を受け渡しするか、という観点で実装ベースに整理したものです。最後にローカル実機で end-to-end の動作検証も行ったので、その結果も併記します。

> 調査時点: 2026-05-13 / 対象: `main` ブランチ HEAD
> 検証環境: `/Users/to.watanabe/workspace/my-city`, `/Users/to.watanabe/workspace/my-project`
> 関連実装: `internal/formula/`, `internal/dispatch/fanout.go`, `internal/sling/sling.go`（rig-scoped 既定値）, `internal/config/config.go`（`Rig.FormulaVars`）

---

## TL;DR

- **formula = TOML ファイル1個**。step ではない。`formula = "name"` は formula の名前。
- 受け取り側 formula は **`[vars]` セクションで「受け取り口」を宣言**するだけ。値の出どころ（CLI / 親 formula / runtime output / rig 既定値）は問わない。
- 値の渡し方は **4 経路**：
  1. **CLI 直接**（cook/sling 時の `--var key=value`）
  2. **親 → 子の静的バインド**（`extends` / `compose.expand` / `expand_vars`）
  3. **`on_complete` ランタイム動的バインド**（親ステップの `output.<field>` を `for_each` で回し、各 item の値を子 formula の vars に注入する fan-out）
  4. **rig レベルの既定値**（`[[rigs]]` の `formula_vars` で rig ごとに var の既定を置く / 2026-05-13 追加, #1974）
- 最終的な precedence（高い順）は **`--var` > `rig.formula_vars` > routing-injected（`issue` / `rig_name` / `base_branch` ほか） > 親側の `expand_vars` / `on_complete.vars` > formula 自身の `[vars.<name>].default`**。
- 「**並列に存在する別 formula の vars を直接ルックアップする構文**」（例: `{{other_formula.var}}`）は無い。値は必ず親→子 / 出力→入力 / 環境→子 の方向に流れる。

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

## 3. 値の4経路

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

### 経路4: rig レベルの既定値（`[[rigs]] formula_vars`）

city.toml の rig 定義に `formula_vars` を置くと、その rig で sling された formula は名前一致する var の既定値を自動で受け取る。CLI で `--var` を渡せばそちらが勝つので、「この rig では既定でこの値」という rig 固有の前提を `pack` 側の formula を触らずに表現できる。

```toml
[[rigs]]
name = "billing-api"
path = "/Users/me/work/billing-api"

[rigs.formula_vars]
target_branch  = "release/2026-Q2"   # この rig だけ別 branch に向ける
checklist      = "billing-internal"  # rig 固有のレビュー観点
test_command   = "pnpm test:ci"
build_command  = "pnpm build"
```

呼ばれる formula は普段通り `[vars]` で受け取るだけでよい。caller を意識しない原則（§7）は崩れない。

```toml
formula = "mol-polecat-base"
version = 1

[vars]
[vars.test_command]
default = ""             # 未指定なら skip
[vars.target_branch]
default = "main"
```

- 値が決まる時点：sling 時の静的バインド
- 出どころ：`[[rigs]] formula_vars`
- 実装：`internal/config/config.go:511-516` の `Rig.FormulaVars`、`internal/sling/sling.go:914-984` の `BuildSlingFormulaVars` / `mergeRigFormulaVars`。
- 適用範囲：sling 経由で formula を起動した場合に有効。`gc formula cook` は CLI 直接（経路1）と formula 内の `default` のみが効く点に注意。

> **実例**: pack 同梱の `internal/bootstrap/packs/core/formulas/mol-polecat-base.toml` は冒頭で「`setup_command` / `typecheck_command` / `test_command` / `lint_command` / `build_command` は rig `formula_vars` から取る、空文字なら skip」と明示している。pack 側で「ここは rig 固有」と宣言し、city 側で `[rigs.<name>.formula_vars]` を埋める形に落ち着くと運用が楽。

### 3.5 最終的な precedence

`internal/sling/sling.go:914` の `BuildSlingFormulaVars` がまさにこの順で重ねる：

| 順位 | 出どころ | 上書きされ得るか |
|---|---|---|
| 1（最強） | `--var key=value`（経路1） | されない |
| 2 | `rig.formula_vars`（経路4） | 1 がある時のみ無視 |
| 3 | sling が注入する routing 値（`issue` / `rig_name` / `binding_name` / `base_branch` / `target_branch`） | 1 がある時のみ無視 |
| 4 | 親 formula 側の `expand_vars` / `on_complete.vars`（経路2 / 経路3） | 上位が同名キーを既に積んでいれば負ける |
| 5（最弱） | 子 formula 自身の `[vars.<name>].default` | 上位の誰かが積んでいれば負ける |

> **注意**: `gc formula cook` は経路1（`--var`）と経路5（formula default）の組み合わせしか効かない。rig-scoped 既定値や sling の routing 値を試したい場合は `gc sling` を経由する必要がある。

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

| 観点 | 経路1（CLI） | 経路2（compose） | 経路3（on_complete） | 経路4（rig 既定） |
|---|---|---|---|---|
| 値が決まる時点 | cook/sling 時 | cook/sling 時 | ランタイム | sling 時 |
| バインドの種類 | 静的 | 静的 | 動的 | 静的 |
| 出どころ | 人間 | 親 formula の TOML | 親ステップの output JSON | `[[rigs]] formula_vars` |
| 用途 | 単発実行 | 再利用可能な合成 | fan-out（N 個の子 molecule） | rig 固有の既定値を pack を変えずに与える |
| 必要な設定 | なし | なし | `formula_v2`, `graph.v2`, `expansion` | rig 側の TOML 一段（pack 触らず） |
| 受け側 formula | 通常 | 通常 | `type = "expansion"` + `[[template]]` | 通常 |

---

## 7. 重要な設計原則

4 経路すべてで、**子 formula 側のインターフェースは `[vars.<name>]` ひとつ**。子 formula は誰が値を入れたか知らない／知る必要がない。これが **Gas City の formula が「呼び出し元から疎」でいられる理由**で、再利用性の根幹。

「並列に存在する別 formula の vars を直接ルックアップする構文」（例: `{{other_formula.var}}`）が**意図的に無い**のも同じ理由。値は必ず「親→子」「出力→入力」「環境→子」の方向に流れ、formula は呼び出し元を知らない。

経路4（rig レベル既定値）も同じ原則の下にある。rig は city の物理レイアウトを表す層であって、formula 自身は rig 名を知らない。「この rig はこの var をこの値で埋める」という設定だけが、sling 時に外側から子 formula へ静的に注入される。

### 7.1 layer precedence の一貫化（2026-05-13 / #2028）

formula 解決には **layer**（city ディレクトリ直下 `formulas/` ←→ pack `imports.<name>` 由来 ←→ core pack 由来）という直交軸もある。同じ formula 名が複数 layer に存在するとき、どちらが勝つか。

これは長らく `loadFormula`（`gc formula show` / `gc formula cook` / `extends:` 連鎖が使う）と `ResolveFormulas`（`.beads/formulas/` の symlink staging が使う）で**走査方向が逆**になっており、`gc formula show` で見える内容と `.beads/formulas/` 経由で実行される内容が食い違うことがあった。

PR #2028 で `internal/formula/Resolve` / `ResolveAll` を抽出し、両者が共通プリミティブを使うように直された。**現在は last-wins（city 直下 > imports > core）で統一されている**。「city/formulas/ に同名 formula を置いて imported pack の formula を override する」というドキュメント通りの動作が `show` 経由でも保証される。within-layer の優先順位（canonical `.toml` > legacy `.formula.toml` > legacy `.formula.json`）も変わらない。

---

## 8. 参考ファイル一覧

- `internal/formula/types.go` — `Formula`, `Step`, `VarDef`, `OnCompleteSpec` の型定義
- `internal/formula/expand.go` — compose / expand 系の置換実装
- `internal/formula/range.go:94` — `substituteVars`（単ブレース置換）
- `internal/formula/resolve.go` — `Resolve` / `ResolveAll`（layer 横断の名前解決プリミティブ, #2028）
- `internal/dispatch/fanout.go` — `processFanout`, `resolveFanoutItems`
- `internal/config/config.go:511-516` — `Rig.FormulaVars`（rig レベル既定値）
- `internal/config/config.go` の `DaemonConfig.FormulaV2` — `[daemon] formula_v2` のフラグ
- `internal/sling/sling.go:914-984` — `BuildSlingFormulaVars` / `mergeRigFormulaVars`（sling 時の precedence 実装）
- `internal/bootstrap/packs/core/formulas/mol-polecat-base.toml` — `[vars]` 受け取り口と「rig `formula_vars` から取る」契約の実例
- `docs/tutorials/05-formulas.md` — vars / 遅延束縛のチュートリアル
- `cmd/gc/cmd_formula.go:84` — `gc formula show`（silent failure については §9 参照）

---

## 9. 残課題（メモ）

- **`gc formula show` の silent failure**：`formula_v2 disabled` のエラーを silent に潰す挙動について、2026-05-13 時点で対応するコミットは確認できなかった。`internal/formula/compile.go:513` で error は返されているが、上位の `cmd/gc/cmd_formula.go` 側で stderr に出ないケースが残っている可能性がある。気になる場合は `gc convoy control <bead>` 経由で同じ formula を踏むとエラー文言が前面に出る。報告は upstream 候補。
- **`#2028` で潰された別問題**：`loadFormula` と `ResolveFormulas` の layer precedence inversion は 2026-05-13 でクローズ済み（§7.1）。`city/formulas/` 直下の override が `show`/`cook` でも反映される。
- **`on_complete` を使った TOML formula の実例**：リポジトリ内には **1 件も存在しない**（grep 確認済み、2026-05-13 時点でも同じ）。型・パーサ・グラフ化・ランタイム dispatcher は揃っているが、pack 同梱の実運用例はまだ無い。
