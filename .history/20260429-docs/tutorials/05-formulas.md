---
title: Tutorial 05 - Formulas
sidebarTitle: 05 - Formulas
description: ステップ、依存関係、変数、制御フローを備えた宣言的なワークフローテンプレートを書き、agent に dispatch する。
---

ここまでは agent に作業を 1 つずつ渡してきました — `gc sling my-agent "do this thing"`。
これでも動きますが、実際のワークフローには複数のステップとそれらの間の依存関係があります。
このチュートリアルでは複数ステップのワークフローを _formula_ として定義し、1 単位として
dispatch する方法を紹介します。

Gas City のような agent オーケストレーションエンジンが存在する主な理由のひとつは、
人間やシェルスクリプトが「適切なタイミングで適切なプロンプト」を流し続けなくても、
さまざまな作業を協調させることです。Gas City では、起こしたいことをすべて _formula_ に
書き留めておき、それを agent に渡して言いつけどおり進めてもらいます。

formula は実行すべきステップを記述しますが、_完全な_ 手順書ではありません。人生の多くの
ことと同じく、順番にやらなければならないこともあれば、並列に進められることも多くあります。

formula は、依存関係、変数、オプションの制御フローを備えたステップ集合を記述する TOML
ファイルです。formula を実行するには、他の作業と同様に `gc sling` で agent に投げます。

## シンプルな formula

formula ファイルは `.toml` 拡張子を使い、都市の `formulas/` ディレクトリに置かれます。
お試しに、そこにパンケーキのレシピを書いてみましょう。

```shell
~/my-city
$ cat > formulas/pancakes.toml << 'EOF'
formula = "pancakes"
description = "Make pancakes from scratch"

[[steps]]
id = "dry"
title = "Mix dry ingredients"
description = "Combine flour, sugar, baking powder, salt in a large bowl."

[[steps]]
id = "wet"
title = "Mix wet ingredients"
description = "Whisk eggs, milk, and melted butter together."

[[steps]]
id = "combine"
title = "Combine wet and dry"
description = "Fold wet ingredients into dry. Do not overmix."
needs = ["dry", "wet"]

[[steps]]
id = "cook"
title = "Cook the pancakes"
description = "Heat griddle to 375F. Pour 1/4 cup batter per pancake."
needs = ["combine"]

[[steps]]
id = "serve"
title = "Serve"
description = "Stack pancakes on a plate with butter and syrup."
needs = ["cook"]
EOF
```

`needs` フィールドは兄弟ステップ間の依存関係を宣言します。

- `dry` と `wet` は並列に実行できる
- `combine` は `dry` と `wet` の両方が完了してから実行される
- `cook` は `combine` を待つ
- `serve` は `cook` を待つ

これらのステップがすべて完了すると、formula は完了です。

`needs` 宣言がなければ、すべてが任意のタイミングで起きうるので、おいしいパンケーキの山
ではなく散らかったキッチンが残ってしまいます。

## formula を確認する

`formulas` ディレクトリには多くの formula ファイルが入っています。`ls` するか、
`gc` に列挙させることもできます。

```shell
~/my-city
$ gc formula list
cooking
mol-do-work
mol-polecat-base
mol-polecat-commit
mol-scoped-work
pancakes
```

特定の formula のコンパイル済みレシピを見るには次のようにします。

```shell
~/my-city
$ gc formula show pancakes
Formula: pancakes
Description: Make pancakes from scratch

Steps (5):
  ├── pancakes.dry: Mix dry ingredients
  ├── pancakes.wet: Mix wet ingredients
  ├── pancakes.combine: Combine wet and dry [needs: pancakes.dry, pancakes.wet]
  ├── pancakes.cook: Cook the pancakes [needs: pancakes.combine]
  └── pancakes.serve: Serve [needs: pancakes.cook]
```

`gc formula show` はステップと依存関係を整理して formula を _コンパイル_ し、表示します。
この場合、`(5)` の数は描画されたワークフローの 5 つのレシピステップと一致します。

これからの数例では、これまでのチュートリアルと同じ `mayor` を引き続き使い、reviewer
以外にもう 1 つの実行ターゲットを得るために汎用 worker を追加します。

```shell
~/my-city
$ gc agent add --name worker
Scaffolded agent 'worker'

~/my-city
$ cat > agents/worker/prompt.template.md << 'EOF'
# Worker Agent
You are a general-purpose Gas City worker. Execute assigned work carefully and report the result.
EOF
```

都市はすでに `claude` をデフォルトにしているので、この city-scoped な worker にはまだ
`agent.toml` は不要です。provider、model、ディレクトリの上書きをしたくなったら追加します。

## formula をインスタンス化する

formula を書く理由は、それが実際に動くのを見たいからです。最も簡単な方法は agent に
sling することです。

```shell
~/my-city
$ gc sling mayor pancakes --formula
Slung formula "pancakes" (wisp root mc-194) → mayor
```

これは formula をコンパイルし、store に作業項目を作成し、`mayor` agent にルーティングし、
グループ化された作業を追跡する convoy を作成します。sling はライフサイクル全体を扱います。
コンパイル、インスタンス化、ルーティング、convoy 作成、そして必要に応じてターゲット agent
への nudge までです。

formula を sling した結果は **wisp** です — 軽量で一時的な bead ツリーです。store に
materialize されるのは root bead だけで、ステップはコンパイル済みレシピからインライン
で読まれます。wisps は close 後にガベージコレクションされます。多くの場合これが正しい
選択です。

複数の agent が異なるステップを独立して進めるような長寿命のワークフローでは、代わりに
**molecule** が必要です。molecule は各ステップを独立した bead として materialize し、
それぞれを個別に追跡・ルーティングできます。`gc formula cook` で molecule を作り、
個別ステップを必要な場所に sling します。

```shell
~/my-project
$ gc formula cook pancakes
Root: mp-2wx
Created: 6
pancakes -> mp-2wx
pancakes.combine -> mp-2wx.3
pancakes.cook -> mp-2wx.4
pancakes.dry -> mp-2wx.1
pancakes.serve -> mp-2wx.5
pancakes.wet -> mp-2wx.2

~/my-project
$ gc sling worker mp-2wx
Auto-convoy mp-w0n
Slung mp-2wx → worker
```

cook は、それを処理する agents が属する rig の中で実行します。これにより molecule の
bead プレフィックスが `my-project` と揃い、rig-local な worker がスコープ境界を超えずに
拾えるようになります。wisps と molecules の違いは、materialize される状態の量だけです。
wisps は軽くて速く、molecules はステップごとの可視性とルーティングを提供します。

## 変数

関数のように、formula はパラメータ化できます。パラメータは `[vars]` セクションで変数
として宣言し、formula 内のステップタイトル、説明、その他のテキストフィールドで
`{{name}}` として参照します。

すべての変数は cook または sling のタイミングで展開されます — formula 内のプレース
ホルダは、出来上がる beads の中で具体的な値になります。

最も単純なケースでは、変数はデフォルト値付きの名前です。

```toml
formula = "greeting"

[vars]
name = "world"

[[steps]]
id = "say-hello"
title = "Say hello to {{name}}"
```

```shell
~/my-city
$ gc formula cook greeting --var name="Alice"
Root: mc-8he
Created: 2
greeting -> mc-8he
greeting.say-hello -> mc-8he.1

~/my-city
$ gc formula cook greeting
Root: mc-kza
Created: 2
greeting -> mc-kza
greeting.say-hello -> mc-kza.1
```

`cook` は置換後のタイトルをエコーしません。展開をプレビューするには `gc formula show`
を使います。

```shell
~/my-city
$ gc formula show greeting --var name="Alice"
Formula: greeting

Variables:
  {{name}}:  (default=world)

Steps (2):
  └── greeting.say-hello: Say hello to Alice
```

`[vars]` で `name = "world"` と書くと、`"world"` がデフォルト値になります。`--var name`
を渡さなければデフォルトが使われます。デフォルトもなく `required` でもない変数の場合、
プレースホルダは出力に `{{name}}` というリテラルテキストとして残ります — これは通常
望ましくないので、必ずデフォルトを与えるか required にマークするのがよい習慣です。

変数にはより豊富な定義を持たせることもできます — 説明、必須フラグ、検証など。

- `description` — 人間向けの説明
- `required` — インスタンス化時に提供される必要がある
- `default` — 呼び出し側が値を渡さなかったときに使われる
- `enum` — 許可される値の集合に制限する
- `pattern` — 正規表現による検証

これらを使った、より完全な例を示します。

```toml
formula = "feature-work"

[vars.title]
description = "What this feature is about"
required = true

[vars.branch]
description = "Target branch"
default = "main"

[vars.priority]
description = "How urgent is this"
default = "normal"
enum = ["low", "normal", "high", "critical"]

[[steps]]
id = "implement"
title = "Implement {{title}}"
description = "Work on {{title}} against {{branch}} (priority: {{priority}})"
```

変数は `--var` で渡します。展開の様子は次のようになります。

```shell
~/my-city
$ gc formula cook feature-work --var title="Auth overhaul" --var branch="develop"
Root: mc-iqy
Created: 2
feature-work -> mc-iqy
feature-work.implement -> mc-iqy.1

~/my-city
$ gc formula cook feature-work --var title="Auth overhaul" --var priority="critical"
Root: mc-jrz
Created: 2
feature-work -> mc-jrz
feature-work.implement -> mc-jrz.1
```

`show` で置換済みのレシピ（および宣言された変数）をプレビューできます。

```shell
~/my-city
$ gc formula show feature-work --var title="Auth system"
Formula: feature-work

Variables:
  {{title}}: What this feature is about (required)
  {{branch}}: Target branch (default=main)
  {{priority}}: How urgent is this (default=normal)

Steps (2):
  └── feature-work.implement: Implement Auth system
```

ここで重要なのは、変数はコンパイルパイプライン全体を通してプレースホルダのまま残る
ということです。実際に beads を作成する瞬間 — `cook` または `sling` のときにのみ
置換されます。これが late binding であり、formula を異なるコンテキストで再利用可能に
します。

## 依存関係グラフ

`needs` についてはパンケーキの例で既に見ました。formula が大きくなるとさらに興味深く
なります。ステップは fan out できます — 同じ前任者に依存する複数のステップは並列で
実行されます。

```toml
[[steps]]
id = "design"
title = "Design the feature"

[[steps]]
id = "implement"
title = "Implement it"
needs = ["design"]

[[steps]]
id = "test"
title = "Test it"
needs = ["implement"]

[[steps]]
id = "review"
title = "Review the PR"
needs = ["implement"]
```

ここでは `test` と `review` の両方が `implement` を待ちますが、互いには並列に走れます。
依存関係グラフは DAG であり、循環はコンパイル時に拒否されます。

### ネストされたステップ

formula が大きくなったら、関連するステップを親の下にグループ化できます。

```toml
[[steps]]
id = "backend"
title = "Backend work"

[[steps.children]]
id = "api"
title = "Build the API"

[[steps.children]]
id = "db"
title = "Set up the database"

[[steps]]
id = "frontend"
title = "Frontend work"
needs = ["backend"]
```

親はコンテナとして機能します — `backend` のすべての子が完了するまで `frontend` は
始まりません。子はコンパイル済みレシピでは親の名前空間に入るので（`backend.api`、
`backend.db`）、ID が一意に保たれます。親があれば、`needs = ["backend"]` のように
依存先を 1 つにまとめられ、個々の子を全部リストする必要はありません。

同じ依存構造はフラットなステップと明示的な `needs` でも実現できます — `api` と `db` を
トップレベルにして、`frontend` に両方を `needs` として持たせるだけです。子はステップ数
が多い formula で長い `needs` リストを抱えなくて済む便利機能です。`backend` に 10 個の
子ステップがあるなら、`needs = ["backend"]` の方が `needs = ["api", "db", "schema",
"seed", "migrate", ...]` よりすっきりします。子は名前空間も提供します — 異なる親
ステップが衝突なく `test` という子を持てます。

## 制御フロー

formula のステップが非順次的、さらには非決定的な順序で実行されることがしばしばあるのは、
ここまでで明らかでしょう。`needs` フィールドは依存関係を設定し、その混沌から秩序を
作り出します。`children` フィールドは、その混沌を多数のステップにまたがって整理する
ために使えます。

ステップが実行されるかどうか、そして実行されるなら何回実行されるかを制御する構文も
いくつかあります。

### 条件

ステップは sling または cook 時に指定された変数の値に基づいて、条件付きで含めたり
除外したりできます。

```toml
[[steps]]
id = "deploy"
title = "Deploy to staging"
condition = "{{env}} == staging"
```

条件はシンプルな等価式を使います。`{{var}} == value` または `{{var}} != value`。変数が
最初に置換され、次に文字列として比較されます。複雑な式言語はありません — もっと高度な
分岐が必要なら、複数の変数と条件を異なるステップに散らしてください。

`gc formula show` で条件の効果を確認できます。

```shell
~/my-city
$ gc formula show deploy-flow --var env=dev
Steps (2):
  └── deploy-flow.build: Build

~/my-city
$ gc formula show deploy-flow --var env=staging
Steps (3):
  ├── deploy-flow.build: Build
  └── deploy-flow.deploy: Deploy to staging
```

### ループ

ステップは複数回実行されるサブステップ群をラップできます。

```toml
[[steps]]
id = "retries"
title = "Attempt deployment"

[steps.loop]
count = 3

[[steps.loop.body]]
id = "attempt"
title = "Try to deploy"
```

body は cook 時に 3 つの順次イテレーションに展開されます。

```shell
~/my-city
$ gc formula show retry-deploy
Steps (4):
  ├── retry-deploy.retries.iter1.attempt: Try to deploy
  ├── retry-deploy.retries.iter2.attempt: Try to deploy [needs: retry-deploy.retries.iter1.attempt]
  └── retry-deploy.retries.iter3.attempt: Try to deploy [needs: retry-deploy.retries.iter2.attempt]
```

各イテレーションは独自のステップとして materialize されます。途中で抜ける方法はありません
— すべてのイテレーションは事前にレシピに焼き込まれます。

### Check

formula が cook されると、条件は評価済みでループも展開済み — すべてが事前に確定しています。
しかし時には実行時の判断が必要です。このステップは実際に成功したのか？

Check は agent がステップを終えた後に検証スクリプトを実行します。スクリプトが pass すれば
そのステップは完了です。pass しなければ agent はもう一度試みます。
チェックは各試行のあと、formula がまだ実行中の間に走ります — コンパイル時の展開ではなく、
実行時のフィードバックループです。

```toml
[[steps]]
id = "implement"
title = "Implement the feature"

[steps.check]
max_attempts = 2

[steps.check.check]
mode = "exec"
path = "scripts/verify.sh"
timeout = "30s"
```

ここで何が起きるかというと、agent が「implement」に取り組み、終わったら Gas City が
`scripts/verify.sh` を実行して結果を検査します。スクリプトが 0 で終了すればステップ完了。
非ゼロで終了したら agent はもう一度挑戦します — 合計 `max_attempts` 回まで。すべての
試行が失敗するとステップは失敗します。

---

これで formula のコア — ステップを定義し、依存関係を結線し、変数でパラメータ化し、
条件・ループ・Check で実行を制御する方法 — をカバーしました。

## 次に学ぶこと

- **[Beads](/tutorials/06-beads)** — formula、session、その他すべての基礎にある
  universal な作業 primitive
- **[Orders](/tutorials/07-orders)** — 定期的な dispatch のためのスケジュール
  トリガー付き formula
