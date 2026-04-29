---
title: Tutorial 06 - Beads
sidebarTitle: 06 - Beads
description: セッション、mail、formula、convoy の基盤となる universal な work primitive を理解し、work item を直接クエリ・操作する方法を学ぶ。
---

ここまで読み進めてきた方は、知らないうちに beads を作成してきました。
セッションを開始したとき、それは bead を作成しました。mail を送ったとき、bead。
formula を cook したとき、beads。sling が wisp を dispatch したとき、bead。

beads は Gas City における universal な work primitive です。追跡可能なものはすべて
（task、message、session、molecule、convoy）store の中の bead です。このチュートリアル
ではその裏側にある仕組みを明らかにします。

[Tutorial 03](/tutorials/03-sessions) の続きから始めます。`my-city` が
`my-project` を rig として動作しており、`mayor` と `reviewer` の agent
（および対応する prompt）が用意されているはずです。

```shell
~/my-city
$ cat pack.toml
[pack]
name = "my-city"
schema = 2

[[agent]]
name = "mayor"
prompt_template = "agents/mayor/prompt.template.md"

[[named_session]]
template = "mayor"
mode = "always"

~/my-city
$ cat city.toml
[workspace]
provider = "claude"

[[rigs]]
name = "my-project"

~/my-city
$ cat agents/reviewer/agent.toml
dir = "my-project"
provider = "codex"
```

対応する prompt ファイルは `agents/<name>/prompt.template.md` 配下に配置されます。
machine-local な workspace identity と rig binding は `.gc/site.toml` に置かれます。

```toml
workspace_name = "my-city"

[[rig]]
name = "my-project"
path = "/Users/csells/my-project"
```

beads はシステムの根幹です。これからプランを beads に変換し、polecats が並列で
実行できる形にしていく作業を crew と一緒に進めることになります。

## bead とは何か

bead は ID、タイトル、ステータス、タイプを持つ作業単位です。beads を直接操作するには
`bd` ツールを使います。

```shell
~/my-city
$ bd list
○ mc-194 ● P2 pancakes
├── ○ mc-194.3 ● P2 Combine wet and dry
├── ○ mc-194.4 ● P2 Cook the pancakes
└── ○ mc-194.5 ● P2 Serve
○ mc-a4l ● P2 Refactor auth module
○ mc-d4g ● P2 Sprint 42
○ mc-io4 ● P2 mayor
○ mc-xp7 ● P2 Update API docs

Status: ○ open  ◐ in_progress  ● blocked  ✓ closed  ❄ deferred
```

デフォルトでは `bd list` はツリー形式で描画され、親 bead がその子をグループ化します。
先頭の記号が bead のステータスで、続いて ID、優先度（`P2`）、タイトルが表示されます。
1 階層のフラットなリストにするには `--flat`、closed な beads を含めるには `--all` を渡します。

すべての bead には次の属性があります。

- **ID** — 都市または rig 名から派生した 2 文字のプレフィックスを持つ一意の識別子
  （例: 都市名 "my-city" なら `mc-194`、rig 名 "my-app" なら `ma-12`）
- **Title** — 人間が読める名前
- **Status** — `open`、`in_progress`、`blocked`、`deferred`、`closed` のいずれか
- **Type** — bead の種類

## bead の type

type は bead が何を表すかを決めます。

| Type         | 内容                              | 作成元                                    |
| ------------ | -------------------------------- | ----------------------------------------- |
| **task**     | 作業単位                          | `bd create`、formula のステップ           |
| **message**  | エージェント間 mail               | `gc mail send`                            |
| **session**  | 実行中の agent session            | `gc session new`                          |
| **molecule** | 永続化された formula インスタンス | `gc formula cook`                         |
| **wisp**     | 一時的な formula インスタンス     | `gc sling --formula`                      |
| **convoy**   | 関連する beads をまとめる入れ物   | `gc convoy create`、sling による自動作成 |

type システムは意図的にシンプルです。Gas City は task、message、session ごとに別々の
ストレージを持ちません — それらはすべて、type ラベルだけが異なる beads です。これが
システムの組み合わせ可能性を生み出します。同じ store、同じクエリインターフェース、同じ
依存モデルがすべての対象に対して機能します。

## bead を作成する

ほとんどの beads は間接的に作成されます。

- `gc session new my-project/reviewer` で session bead が作成される
- `gc mail send mayor "Subject" "Body"` で message bead が作成される
- `gc formula cook review` で molecule + step beads が作成される
- `gc sling mayor review --formula` で wisp bead + convoy が作成される

ただし `bd` を使って手動で作成することもできます。

```shell
~/my-city
$ bd create "Fix the login bug"
✓ Created issue: mc-ykp — Fix the login bug
  Priority: P2
  Status: open

$ bd create "Refactor auth module" --type feature
✓ Created issue: mc-a4l — Refactor auth module
  Priority: P2
  Status: open
```

## bead のライフサイクル

beads は少数の状態を遷移します。

```
open → in_progress → closed
```

- **open** — まだ着手されていない作業。hooks 経由で agent から発見可能。
- **in_progress** — agent によって claim され、進行中。
- **closed** — 完了。
- **blocked** — open な `blocks` 依存を持つ。自動で設定される。
- **deferred** — 明示的に指定日まで snooze されている。

日常的な利用では、**open / in_progress / closed** の 3 つを使います。
`blocked` と `deferred` はシステムが管理する派生状態です。

```shell
~/my-city
$ bd close mc-ykp
✓ Closed mc-ykp — Fix the login bug: Closed

$ bd list --status open --flat
○ mc-a4l [● P2] [feature] - Refactor auth module
○ mc-xp7 [● P2] [task]    - Update API docs
```

フラグは `--status` であることに注意してください（`--state` は state 次元用の
別コマンドです）。

## 実行状態としての beads

bead store は事実上、システム全体の実行状態です。実行中のすべてのセッション、流通中の
すべてのメッセージ、進行中のすべての formula ステップ — どれもステータスを持つ bead
です。今この都市が何をしているか知りたければ、store にクエリを投げます。出力は都市内で
何がアクティブかによります。例えば次のようになります。

```shell
~/my-city
$ bd list --status in_progress --flat
◐ mc-io4 [● P2] [session] - mayor
```

これにより、agent のセッションを使い捨てプロセスとして作業実行に利用できます。
作業はメモリに保持されたり実行中プロセスで追跡されたりせず、store に永続化されます。
agent が死んでも beads は open のまま残ります。agent が再起動すれば、その hooks が
同じ作業を発見し、中断したところから再開します。都市全体が停止して再起動した場合も、
bead store が「何が起きていて、何がまだ必要か」の真実の根拠となります。

本章の残りでは、beads がどう整理され、ルーティングされ、グループ化され、agent に
発見されるかを詳しく見ていきます。

## ラベル

labels は beads の整理とルーティングに使われます。

```shell
~/my-city
$ bd label add mc-a4l priority:high
✓ Added label 'priority:high' to mc-a4l

$ bd label add mc-a4l frontend
✓ Added label 'frontend' to mc-a4l

$ bd list --label priority:high --flat
○ mc-a4l [● P2] [feature] - Refactor auth module
```

`bd label add` は 1 回の呼び出しでラベルを 1 つだけ受け取ります。複数を付けたい場合は
1 つずつ適用します。

Gas City で特別な意味を持つラベルがいくつかあります。

- **`gc:session`** — session beads を示す
- **`gc:message`** — mail beads を示す
- **`thread:<id>`** — mail メッセージを会話としてグループ化する
- **`read`** — メッセージが既読であることを示す

独自の整理用に任意のラベルを追加できます。

## メタデータ

beads は構造化された状態のために任意のキー・値メタデータを保持します。

```shell
~/my-city
$ bd update mc-a4l --set-metadata branch=feature/auth --set-metadata reviewer=sky
✓ Updated issue: mc-a4l — Refactor auth module
```

メタデータはセッション追跡（`session_name`、`alias`）、ルーティング（`gc.routed_to`）、
マージ戦略、formula 参照などに内部的に利用されます。タイトルや説明を変えずに bead に
情報を付けたいときに自由に使えます。削除には `--unset-metadata <key>` を使います。

## 依存関係

beads は他の beads に依存できます。formula で既に見たように、ステップが
`needs = ["design"]` を宣言すると、それは blocking な依存関係です。design bead が
close するまで、その step bead は開始できません。Gas City は中央スケジューラを持たず
に順序を強制します。それぞれの bead は何を待っているかを知っており、agents は
ready な作業しか見ません。

```shell
~/my-city
$ bd dep mc-a4l --blocks mc-xp7
✓ Added dependency: mc-a4l (Refactor auth module) blocks mc-xp7 (Update API docs)
```

これで `mc-a4l` が close するまで、`mc-xp7` はどの agent の作業クエリにも現れません。
これは formula のステップ順序付けを実現しているのと同じ仕組みで、`needs` 宣言が
step beads 間の `blocks` エッジになります。

依存関係の type は **`blocks`**（相手より先に close する必要あり）、
**`tracks`**（情報的 — 「これが気になる」）、**`related`**（緩い関連付け）、
**`parent-child`**（包含）、**`discovered-from`**（他の作業中に浮上した作業）の 5 つです。
作業の可視性に影響するのは `blocks` だけです。

beads には別の _parent-child_ 関係もあります。bead は `parent_id` を設定して
コンテナにリンクできます。これは convoys と molecules が子をグループ化する仕組みです。
違いを言うと、依存関係は順序を表現し（"A を B より先にやれ"）、parent-child は
包含を表現します（"これらの beads はこのグループに属する"）。convoy の子は互いに
依存しません。同じバッチのメンバーであるだけです。

## convoy

formula を sling したことがあれば、知らないうちに convoy を作成しています。Gas City
は dispatch された formula 作業を自動的に convoy で包みます。`bd list` では type が
`convoy` の beads として、`gc convoy list` では進捗サマリ付きで見られます。convoy が
重要になるのは関連作業のバッチを 1 単位として追跡したいとき — 「この 5 つの task は
全部終わったか？」のような問いは convoy の問いです。

任意の作業をグループ化するために手動で作成することもできます。例えば、スプリントや
デプロイとして一緒に追跡したい beads のセットなどです。

```shell
~/my-city
$ gc convoy create "Sprint 42" mc-ykp mc-a4l mc-xp7
Created convoy mc-d4g "Sprint 42" tracking 3 issue(s)
```

convoy は type が `convoy` の bead です。子 beads は `ParentID` でリンクされます —
molecules で使うのと同じ parent-child 機構を、ステップ順序付けではなくグループ化に
使っているだけです。

```shell
~/my-city
$ gc convoy status mc-d4g
Convoy:   mc-d4g
Title:    Sprint 42
Status:   open
Progress: 1/3 closed

ID      TITLE                 STATUS  ASSIGNEE
mc-ykp  Fix the login bug     closed  -
mc-a4l  Refactor auth module  open    -
mc-xp7  Update API docs       open    -
```

### 自動 close

bead が close したとき、Gas City はその親が convoy で、すべての子が close 済みかを
チェックします。条件を満たせば、convoy は自動で close されます。これは `on_close`
hook 経由でバックグラウンドで起きます — ポーリングも手動操作も不要です。

**owned** ラベルが付いた convoy は自動 close をスキップします。convoy の完了タイミング
を明示的に制御したいワークフロー向けです。

```shell
~/my-city
$ gc convoy create "Auth rewrite" --owned --target integration/auth
Created convoy mc-0ud "Auth rewrite"
```

完了したら明示的に land します。

```shell
~/my-city
$ gc convoy land mc-0ud
Landed convoy mc-0ud
```

### beads の追加と convoy のチェック

convoy 作成後に作業が増えることもあります。スプリント途中で新しいバグが浮上したり、
プラン後に依存関係が見つかったり。既存の convoy に beads を追加できます。

```shell
~/my-city
$ gc convoy add mc-d4g mc-xp7
Added mc-xp7 to convoy mc-d4g
```

convoy が自動 close されるべきだったのにされなかった場合（hook が誤発火したなど）、
手動で reconcile できます。

```shell
~/my-city
$ gc convoy check
Auto-closed convoy mc-d4g "Sprint 42"
1 convoy(s) auto-closed
```

### 取り残された作業

assignee がいない open な beads（誰かが拾うのを待ってスタックしている作業）を
convoy 内で見つけるには次のコマンドを使います。

```shell
~/my-city
$ gc convoy stranded
CONVOY  ISSUE   TITLE
mc-d4g  mc-a4l  Refactor auth module
mc-d4g  mc-xp7  Update API docs
```

### convoy のメタデータ

convoy はグループ化された作業の挙動を制御するメタデータを持ちます。

- **`convoy.owner`** — この convoy を管理する agent
- **`convoy.notify`** — convoy 完了時に通知する相手
- **`convoy.merge`** — PR のマージ戦略（`direct`、`mr`、`local`）
- **`target`** — 子 beads が継承するターゲットブランチ

これらは作成時にフラグで設定します。

```shell
~/my-city
$ gc convoy create "Deploy v2" --owner mayor --merge mr --target main
Created convoy mc-zk1 "Deploy v2"
```

target を後から更新することもできます。

```shell
~/my-city
$ gc convoy target mc-zk1 develop
Set target of convoy mc-zk1 to develop
```

## agent はどうやって作業を見つけるか

ここで beads がランタイムにつながります。agents は _hooks_ を通じて作業を発見します
— ターン間で実行されるシェルコマンドが、利用可能な beads をチェックします。

典型的な流れは次のとおりです。

1. 作業が作成される（`bd create`、`gc sling`、formula cook など経由）
2. 作業が agent にルーティングされる（assignee または `gc.routed_to` メタデータ経由）
3. agent の hook が _work query_ を実行し、ready な該当 beads を探す
4. 作業が見つかれば、hook が system reminder として agent のコンテキストに注入する
5. agent は作業を見て行動する（GUPP: "if you find work on your hook, you run it"）

ルーティングされた pool 作業の場合、クエリは assignee ではなくメタデータをチェックします。

```shell
~/my-city
$ bd ready --metadata-field gc.routed_to=my-project/worker --unassigned --limit=1
```

`mc-xp7` は今 `mc-a4l` によって blocked されているので、このクエリはまだ何も返しません。
それが意図したところです。blocked な作業は agent の作業クエリから見えません。`mc-a4l`
が close されたら同じクエリを再実行することで、`mc-xp7` が対象になります。

これは「pull」モデルです — 作業が push されるのではなく、agents が作業をチェックします。
シンプルで、クラッシュ耐性があり（キューに入った作業は再起動を生き延びる）、自然に
スケールします。

## bead store

beads は store に永続化されます。Gas City はいくつかのバックエンドをサポートします。

- **bd**（デフォルト）— `bd` CLI 経由の Dolt-backed データベース。フル機能で本番向け。
- **file** — ディスク上の JSON ファイル。シンプルで、チュートリアルや小規模設定に向く。
- **exec** — カスタムスクリプトに委譲。外部システムとの統合用。

`city.toml` でバックエンドを設定します。

```toml
[beads]
provider = "file"    # または "bd"（デフォルト）
```

ほとんどのユーザーにとってデフォルトで問題なく、特に意識する必要はありません。

---

普段は beads を直接操作する必要はありません。上位コマンド（`gc session`、`gc mail`、
`gc sling`、`gc formula`）が bead の作成と管理を代行します。ただし都市横断で未消化の
作業をクエリしたい、agent 用に ad-hoc な task を作りたい、formula の依存グラフを
調べたい、agent が作業を拾わない理由をデバッグしたい、といったときは `bd` を直接
使うことになります。

```shell
~/my-city
$ bd list --status open --type task --flat
○ mc-xp7 [● P2] [task] - Update API docs
○ mc-2wx.1 [● P2] [task] - Mix dry ingredients (parent: mc-2wx, blocks: mc-2wx.3)

$ bd show mc-a4l
○ mc-a4l · Refactor auth module   [● P2 · OPEN]
Owner: dbox · Type: feature
Created: 2026-04-08 · Updated: 2026-04-08

LABELS: frontend, priority:high

METADATA
  branch: feature/auth
  reviewer: sky

PARENT
  ↑ ○ mc-d4g: Sprint 42 ● P2

BLOCKS
  ← ○ mc-xp7: Update API docs ● P2

$ bd close mc-a4l
✓ Closed mc-a4l — Refactor auth module: Closed
```

beads は都市の実行状態における真実の根拠です。Gas City の他のすべて — sessions、
mail、formulas、convoys — は beads の上に構築されています。

## 次に学ぶこと

- **[Orders](/tutorials/07-orders)** — 時刻、スケジュール、条件、イベントによって
  トリガーされる formula とスクリプトの自動運転
