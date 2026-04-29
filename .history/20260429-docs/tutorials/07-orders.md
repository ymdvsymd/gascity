---
title: Tutorial 07 - Orders
sidebarTitle: 07 - Orders
description: トリガー条件（cooldown、cron スケジュール、シェルチェック、event）を使って formula やスクリプトを自動実行するスケジュール設定。
---

formula は作業 _がどう見えるか_ を記述します。orders は _いつ_ それが起きるかを記述します。
order はトリガー条件とアクション（formula またはシェルスクリプト）をペアにし、controller
が自動的にそれらのトリガーを評価します。トリガーが開けば order が発火します。人間による
dispatch は不要です。

`gc start` を実行すると _controller_ が起動します — 30 秒ごと（_tick_）に目を覚まし、
都市の状態をチェックして行動するバックグラウンドプロセスです。tick ごとに行うことの
ひとつが、order を実行可能にするトリガーの評価です。この定期的なチェックが orders を
動かしている仕組みです。

[Tutorial 06](/tutorials/06-beads) の続きから始めます。`my-city` が agents と
formulas の設定済みで動作しているはずです。

`gc sling` で formula を手動 dispatch しているなら、orders は次のステップです。手動
dispatch を、スケジュールやイベントに応じて都市が自分でやってくれるものに変えます。

## シンプルな order

orders は都市のトップレベルの `orders/` ディレクトリに置かれ、`formulas/` や `agents/`
と並びます。各 order はそのディレクトリ内のフラットな `*.toml` ファイルです。

```
orders/
  review-check.toml
  dep-update.toml
formulas/
  pancakes.toml
  review.toml
```

[Tutorial 04](/tutorials/04-mail) の `review` formula を 5 分ごとに dispatch する最小限の
order を以下に示します。

```toml
# orders/review-check.toml
[order]
description = "Check for PRs that need review"
formula = "review"
trigger = "cooldown"
interval = "5m"
pool = "worker"
```

`pool` フィールドは作業の送り先を controller に伝えます。_pool_ は作業キューを共有する
1 つ以上の agent からなる名前付きグループです — agents の章で簡単に紹介しました。order
が発火すると、controller は formula から wisp を作成し、指定された pool にルーティング
します。pool 内の任意の agent がそれを拾えます。

controller は tick ごとにトリガー条件を評価します。前回実行から 5 分経過すると、`review`
formula を wisp としてインスタンス化し、`worker` pool にルーティングします。order の
名前はファイルベース名（`review-check.toml` → `review-check`）から決まり、TOML 内の
何かから取られるわけではありません。

orders は都市起動時、および controller が config を再読み込みしたときに発見されます。
都市がすでに orders ディレクトリを監視している場合、何かを再起動する必要はありません。

## orders を確認する

orders をいくつか定義したら、controller が見ているもの — どんな orders があり、トリガー
がどう見え、いずれかが due かどうか — を見たくなるはずです。3 つのコマンドがそのビューを
提供します。

`gc order list` は都市内の有効な orders をすべて表示します — 一度も発火していないもの
も含みます。

```shell
~/my-city
$ gc order list
NAME            TYPE     TRIGGER      INTERVAL/SCHED  TARGET
review-check    formula  cooldown  5m              worker
dep-update      formula  cooldown  1h              worker
release-notes   formula  cooldown  24h             worker
```

`TARGET` 列は order がルーティングする pool です（TOML 上のフィールドは依然として
`pool`）。

完全な定義を確認するには次のようにします。

```shell
~/my-city
$ gc order show review-check
Order:  review-check
Description: Check for PRs that need review
Formula:     review
Trigger:        cooldown
Interval:    5m
Target:      worker
Source:      /Users/you/my-city/orders/review-check.toml
```

今 due な orders を確認するには次のようにします。

```shell
~/my-city
$ gc order check
NAME            TRIGGER      DUE  REASON
review-check    cooldown  yes  never run
dep-update      cooldown  no   cooldown: 14m remaining
release-notes   cooldown  no   cooldown: 18h remaining
```

## order を手動で実行する

任意の order はトリガーを迂回して手動で起動できます。

```shell
~/my-city
$ gc order run review-check
Order "review-check" executed: wisp mc-2xz → gc.routed_to=worker
```

exec 形式の order の場合、出力は単純で `Order "<name>" executed (exec)` となります。

新しい order をテストしたり、もうすぐ due になる作業をキックオフしたりするのに便利です。

## トリガーの種類

トリガーは order を ticking させるものです。order が _いつ_ 発火するかを制御します。
トリガーには 5 種類あります。

### Cooldown

最も一般的なトリガーです。名前はクールダウンタイマーの考えから来ています — order が
発火したあと、再度発火できるまで決められた間隔だけクールダウンする必要があります。

```toml
[order]
description = "Check for stale feature branches"
formula = "stale-branches"
trigger = "cooldown"
interval = "5m"
pool = "worker"
```

order が一度も実行されていなければ、最初の tick で即座に発火します。それ以降は前回
実行から `interval` が経過するまで待ちます。interval は Go の duration 文字列です —
`30s`、`5m`、`1h`、`24h`。

### Cron

Unix cron job のように絶対的なスケジュールで発火します。

```toml
[order]
description = "Generate release notes from yesterday's merges"
formula = "release-notes"
trigger = "cron"
schedule = "0 3 * * *"
pool = "worker"
```

schedule は 5 フィールドの cron 表現です。分、時、日、月、曜日。この例は毎日午前 3:00
に発火します。フィールドは `*`（任意）、整数、カンマ区切り値（1 日と 15 日なら `1,15`）
をサポートします。

cooldown との違いは、cooldown は前回実行を _基準にした_ 相対的な発火（「5 分ごと」）で
あるのに対し、cron は _絶対的な_ 時刻で発火することです（「毎日午前 3 時に」）。
cooldown はドリフトします — 前回実行が 3:02 なら次は 3:07 です。cron は毎日同じ
壁時計時刻にヒットします。

cron トリガーは 1 分間に高々 1 回しか発火しません — 同じ分のうちに既に実行済みなら、
次のマッチを待ちます。

### Condition

シェルコマンドが exit 0 で終了したときに発火します。

```toml
[order]
description = "Deploy when the flag file appears"
formula = "deploy"
trigger = "condition"
check = "test -f /tmp/deploy-flag"
pool = "worker"
```

controller は tick ごとに `sh -c "<check>"` を 10 秒のタイムアウトで実行します。コマンドが
exit 0 で終了すれば order が発火します。それ以外の終了コードでは発火しません。これは
動的・外部のトリガー用です — ファイルを確認したり、エンドポイントに ping したり、
データベースをクエリしたり。

注意点が 1 つ。チェックはトリガー評価中に同期的に実行されます。遅いチェックは、その
tick で後続の orders の評価を遅らせます。チェックは高速に保ってください。

### Event

システムイベントに反応して発火します。

```toml
[order]
description = "Check if all PR reviews are done and merge is ready"
formula = "merge-ready"
trigger = "event"
on = "bead.closed"
pool = "worker"
```

これは `bead.closed` イベントが event bus に現れたときに発火します。event トリガーは
カーソルベースの追跡を使い、発火ごとにシーケンスマーカーを進めるので、同じイベントが
2 回処理されることはありません。

### Manual

自動では発火しません。`gc order run` でしかトリガーされません。

```toml
[order]
description = "Full test suite — expensive, run only when needed"
formula = "full-test-suite"
trigger = "manual"
pool = "worker"
```

manual な orders は `gc order check` には現れません（自動では due にならないので、
チェックする対象がありません）。`gc order list` には現れます。

## formula 形式の order と exec 形式の order

これまでの例ではすべて formula をアクションとして使ってきました。しかし orders は
シェルスクリプトを直接実行することもできます。

```toml
[order]
description = "Delete branches already merged to main"
trigger = "cooldown"
interval = "5m"
exec = "scripts/prune-merged.sh"
```

exec 形式の order は controller 上でスクリプトを実行します — agent も LLM も wisp も
ありません。純粋に機械的な操作（ブランチの剪定、linter の実行、ディスク使用量の確認、
agent を介在させると無駄になるあらゆるもの）に適しています。

ルール：

- 各 order は `formula` か `exec` のいずれか一方を持ち、両方を持つことはできません。
- exec 形式は `pool` を持てません — ルーティングする agent パイプラインがないからです。
- スクリプトは環境変数 `ORDER_DIR` を受け取り、order ファイルが置かれているディレクトリを
  指します。pack 由来の orders は `PACK_DIR` も受け取ります。

デフォルトのタイムアウトは異なり、formula 形式は 30 秒、exec 形式は 300 秒です。

## タイムアウト

各 order はタイムアウトを設定できます。

```toml
[order]
description = "Run the linter on changed files"
formula = "lint-check"
trigger = "cooldown"
interval = "30s"
pool = "worker"
timeout = "60s"
```

formula 形式の order の場合、タイムアウトは初期 dispatch — formula のコンパイル、wisp
の作成、pool へのルーティング — をカバーします。一度 wisp が作成されて引き渡されると、
agent は自分のペースで作業します。タイムアウトは作業中の agent を中断しません。
exec 形式の場合、タイムアウトはスクリプト実行全体をカバーします — 時間切れ時にまだ
スクリプトが実行中なら、プロセスは kill されます。`city.toml` でグローバルキャップも
設定できます。

```toml
[orders]
max_timeout = "120s"
```

実効タイムアウトは order ごとのタイムアウトとグローバルキャップのうち小さい方です。

## orders の無効化とスキップ

order は自身の定義の中で無効化できます。

```toml
[order]
description = "Temporarily disabled"
formula = "nightly-bench"
trigger = "cooldown"
interval = "1m"
pool = "worker"
enabled = false
```

無効化された orders はスキャン対象から完全に除外されます — `gc order list` にも現れず、
評価もされません。

order ファイルを編集せずに、`city.toml` で名前を指定してスキップすることもできます。

```toml
[orders]
skip = ["nightly-bench", "experimental-check"]
```

pack が提供する orders のうち、自分の都市で動かしたくないものがあるときに便利です。

## 上書き

pack の order がほぼ正しいけれど、interval や pool だけ調整したい場合があります。order
ファイルをコピーして変更するのではなく、`city.toml` で上書きを使います。

```toml
[[orders.overrides]]
name = "test-suite"
interval = "1m"

[[orders.overrides]]
name = "release-notes"
pool = "mayor"
schedule = "0 6 * * *"
```

上書きできるのは `enabled`、`trigger`、`interval`, `schedule`、`check`、`on`、`pool`、
`timeout` です。上書きは order 名でマッチします — その名前の order が存在しなければ
エラーです（fail-fast、サイレントではありません）。

## order の履歴

order が発火するたびに、Gas City は order 名がラベル付けされた追跡 bead を作成します。
履歴をクエリできます。

```shell
~/my-city
$ gc order history
ORDER           BEAD     EXECUTED
review-check    mc-3hb   2026-04-08T07:36:36Z
dep-update      mc-784   2026-04-08T06:48:12Z
review-check    mc-zbd   2026-04-08T07:31:22Z
release-notes   mc-zb8   2026-04-07T13:00:01Z

~/my-city
$ gc order history review-check
ORDER           BEAD     EXECUTED
review-check    mc-3hb   2026-04-08T07:36:36Z
review-check    mc-zbd   2026-04-08T07:31:22Z
review-check    mc-9p8   2026-04-08T07:26:18Z
```

追跡 bead は dispatch goroutine が起動する _前_ に同期的に作成されます。これが、cooldown
トリガーが直後の tick で再発火するのを防ぐ仕組みです — トリガーは order が due かを
判定するときに、最近の追跡 beads をチェックします。

## 重複防止

dispatch の前に、controller は order がすでに open（未 close）の作業を持っているかを
チェックします。持っていれば、トリガーが due だと言っていても order はスキップされます。
これが積み上げを防ぎます — agent が前回の review チェックをまだ処理中なら、controller
は次を dispatch しません。

## rig スコープの orders

orders は都市レベルだけに存在するわけではありません。pack が rig に適用されると、その
pack の orders も付いてきて、その rig にスコープされて実行されます。

`dev-ops` という pack が `test-suite` order を含むとします。

```
packs/dev-ops/
  orders/
    test-suite.toml         # trigger = "cooldown", interval = "5m", pool = "worker"
  formulas/
    test-suite.toml
```

そして都市がその pack を 2 つの rig に適用します。

```toml
# city.toml
[[rigs]]
name = "my-api"

[rigs.imports.dev_ops]
source = "./packs/dev-ops"

[[rigs]]
name = "my-frontend"

[rigs.imports.dev_ops]
source = "./packs/dev-ops"
```

```toml
# .gc/site.toml
[[rig]]
name = "my-api"
path = "../my-api"

[[rig]]
name = "my-frontend"
path = "../my-frontend"
```

これで都市は同じ order を rig ごとに独立して動かしています。

```shell
~/my-city
$ gc order list
NAME        TYPE     TRIGGER      INTERVAL/SCHED  TARGET
test-suite  formula  cooldown  5m              worker
test-suite  formula  cooldown  5m              my-api/worker
test-suite  formula  cooldown  5m              my-frontend/worker
```

同じ名前が 3 つ、ターゲットが 3 つ違います — 各 order を所有する rig は qualified な
ターゲット名（`my-api/worker` と `my-frontend/worker`）にエンコードされています。
特定の 1 つを操作するには `--rig` を渡します。

```shell
$ gc order show test-suite --rig my-api
$ gc order run test-suite --rig my-api
```

これらは 3 つの独立した orders です。都市レベルの `test-suite` は独自の cooldown タイマー、
追跡 beads、履歴を持ちます。`my-api` 版は別個に追跡されます — 都市レベルの order が
2 分前に発火したからといって、`my-api` の order が due かどうかには影響しません。
内部的には Gas City はそれらを _scoped name_ で区別します。`test-suite` と
`test-suite:rig:my-api` と `test-suite:rig:my-frontend` です。

pool ターゲットは自動的に qualify されます。order 定義の `pool = "worker"` は dispatch
された wisp 上では `gc.routed_to=my-api/worker` となり、都市レベルの pool ではなく
rig 自身の agents に作業をルーティングします。

## order の階層

orders は packs、rigs、自分の都市の `orders/` ディレクトリから来るので、同じ order 名が
複数の場所に存在し得ます。それが起きると、最も優先度の高い層が勝ちます。優先度の低い順
から並べると次のとおりです。

1. **City packs** — 取り込んだ pack に同梱されている orders（例: `dev-ops` pack の
   `test-suite`）
2. **City local** — 自分の都市の `orders/` ディレクトリにある orders
3. **Rig packs** — 特定の rig に適用された pack 由来の orders
4. **Rig local** — rig 自身の `orders/` ディレクトリにある orders

上位の層は、同名の order について下位層の定義を完全に置き換えます。だから `dev-ops`
pack が 5 分の cooldown で `test-suite` を定義していて、自分で 1 分の cooldown の
`orders/test-suite.toml` を作ったら、自分の方が勝ちます — pack 版は完全に無視されます。

## 組み合わせる

2 つの orders を持つ都市の例です。頻繁に走る lint チェック（exec、agent 不要）と、
週次のリリースノート（formula、agent に dispatch）です。

[Tutorial 05](/tutorials/05-formulas) のとおり `worker` agent はすでに作成済みとします。
残るのは order ファイルと、それが dispatch する formula だけです。

```toml
# orders/lint-check.toml
[order]
description = "Run the linter on changed files"
trigger = "cooldown"
interval = "30s"
exec = "scripts/lint-changed.sh"
timeout = "60s"
```

```toml
# orders/release-notes.toml
[order]
description = "Generate release notes from the week's merges"
formula = "release-notes"
trigger = "cron"
schedule = "0 9 * * 1"
pool = "worker"
```

```toml
# formulas/release-notes.toml
formula = "release-notes"

[[steps]]
id = "gather"
title = "Gather merged PRs from the last week"

[[steps]]
id = "summarize"
title = "Write release notes"
needs = ["gather"]

[[steps]]
id = "post"
title = "Post release notes to the team channel"
needs = ["summarize"]
```

```shell
~/my-city
$ gc start
City 'my-city' started

~/my-city
$ gc order list
NAME           TYPE     TRIGGER      INTERVAL/SCHED  TARGET
lint-check     exec     cooldown  30s             -
release-notes  formula  cron      0 9 * * 1       worker

~/my-city
$ gc order check
NAME           TRIGGER      DUE  REASON
lint-check     cooldown  yes  never run
release-notes  cron      no   next fire in 3d 14h
```

lint チェックは即座に発火し（never run + cooldown トリガー = due）、その後 30 秒ごとに
発火します。リリースノートは月曜午前 9 時に発火し、3 ステップの formula wisp を `worker`
pool に dispatch します。どちらも誰かが `gc sling` を打つ必要はありません。

orders は時刻、スケジュール、条件、event でゲートされた formula とスクリプトの自動運転
であり、controller が tick ごとに評価します。
