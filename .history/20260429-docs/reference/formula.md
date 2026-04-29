---
title: Formula Files
description: Gas City の formula ファイルの構造と配置。
---

Gas City は PackV2 の formula レイヤーから formula ファイルを解決し、勝った formula ファイルを [`ResolveFormulas`](https://github.com/gastownhall/gascity/blob/main/cmd/gc/formula_resolve.go) を使って `.beads/formulas/` にステージします。

formula のインスタンス化は CLI またはストアインターフェースを介して行われます:

- `gc formula cook <name>` は molecule を作成します（各ステップは bead として実体化されます）
- `gc sling <target> <name> --formula` は wisp を作成します（軽量で一時的）
- `Store.MolCook(formula, title, vars)` はプログラム的に molecule または wisp を作成します
- `Store.MolCookOn(formula, beadID, title, vars)` は既存の bead に molecule をアタッチします

## 最小の formula

```toml
formula = "pancakes"
description = "Make pancakes"
version = 1

[[steps]]
id = "dry"
title = "Mix dry ingredients"
description = "Combine the flour, sugar, and baking powder."

[[steps]]
id = "wet"
title = "Mix wet ingredients"
description = "Combine eggs, milk, and butter."

[[steps]]
id = "cook"
title = "Cook pancakes"
description = "Cook on medium heat."
needs = ["dry", "wet"]
```

## よく使うトップレベルキー

| キー | 型 | 用途 |
|---|---|---|
| `formula` | string | `gc formula cook`、`gc sling --formula`、`Store.MolCook*` で使われる一意の formula 名 |
| `description` | string | 人間が読める説明 |
| `version` | integer | 任意の formula バージョンマーカー |
| `extends` | []string | 合成元となる任意の親 formula |

## ステップフィールド

各 `[[steps]]` エントリは、インスタンス化された molecule 内の 1 つの task bead を表します。

| キー | 型 | 用途 |
|---|---|---|
| `id` | string | ステップ識別子。formula 内で一意 |
| `title` | string | 短いステップタイトル |
| `description` | string | エージェントに表示されるステップの指示 |
| `needs` | []string | このステップが ready になる前に完了している必要があるステップ ID |
| `condition` | string | 等価式（`{{var}} == value` または `!=`）。false のときステップは除外される |
| `children` | []step | ネストされたサブステップ。親はコンテナ依存として動作する |
| `loop` | object | 静的なループ展開: コンパイル時に `count` 回繰り返す |
| `check` | object | ランタイムリトライ: 各試行後に `check` スクリプトを実行し `max_attempts` まで再試行 |
| `timeout` | duration string | このステップの `check` スクリプトのデフォルトタイムアウト。`check.check.timeout` が優先される |

## 変数置換

formula の説明では `{{key}}` プレースホルダを使えます。変数は formula のインスタンス化時に `key=value` ペアとして供給します。例えば:

```bash
gc sling worker deploy --formula --var env=prod
```

## convergence 固有のフィールド

convergence は [`internal/convergence/formula.go`](https://github.com/gastownhall/gascity/blob/main/internal/convergence/formula.go) で定義された formula のサブセットを使います。

| キー | 型 | 用途 |
|---|---|---|
| `convergence` | bool | convergence ループでは `true` でなければならない |
| `required_vars` | []string | 作成時に供給する必要がある変数 |
| `evaluate_prompt` | string | コントローラが注入する evaluate ステップ用の任意のプロンプトファイル |

## formula はどこから来るか

PackV2 の formula 検出は規約ベースです:

- pack の再利用可能な formula は `formulas/` に置かれる
- city pack 自身の `formulas/` レイヤーは、インポートされた pack の formula より優先される
- rig レベルのインポートは rig 固有の formula を提供できる
- インポートされた pack の formula は、解決時に pack の出自を保持する

`[formulas].dir` や `[[rigs]].formulas_dir` などのレガシーフィールドは、移行互換性のため設定スキーマに残ることがあります。新しい pack は TOML で formula ディレクトリを宣言する代わりに、PackV2 の `formulas/` ディレクトリ規約を使うべきです。

現在の formula 解決の挙動については Architecture: Formulas & Molecules (`engdocs/architecture/formulas`) を参照してください。
