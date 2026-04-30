# QA: Gas City のイベント機構 — probe / pool / reconciliation の正体

このドキュメントは、Gas City (`gc`) のオーケストレーションを駆動する **イベント機構**を、初学者がつまずきやすい 3 つの用語(`probe (scale_check)` / `pool` / `reconciliation`)を起点に整理したものです。「`gc` のイベントは常時起動エージェントでしか発生しないのか?」という疑問に最終的に答えることを目的にしています。

> 調査時点: 2026-04-30 / 対象: `main` ブランチ
> 関連ドキュメント: `.history/20260429-usage/OVERVIEW.md` §5、§6 / `.history/20260429-usage/QA/MESSAGING.md` §4 / `engdocs/contributors/reconciler-debugging.md` / `engdocs/architecture/event-bus.md` / `engdocs/design/agent-pools.md`

---

## TL;DR

| 用語 | 一行で言うと |
|---|---|
| **reconciliation** | controller が約 1 秒ごとに「設定が期待する状態」と「いま動いている状態」を突き合わせ、足りない agent を spawn し、余ってる agent を drain する循環処理 |
| **pool** | 同じ役割の agent を複数並行で動かしたいときの定義 (`[agent.pool]`)。最小数 (`min`) と最大数 (`max`) と「いま何体必要か答えるコマンド (`check`)」を持つ。常駐 / on-demand のどちらにも振れる |
| **probe (`scale_check`)** | pool が「いま何体必要か」を決めるために実行する shell コマンド。**stdout に整数 1 行を出力する**。controller がそれを `[min, max]` にクランプして desired state に反映 |
| **event bus** | システム内で起きた事実 (44 種類) の追記専用ログ。**pull (`List`) と watch (`Watch().Next()`) の併用**で消費される。event そのものは agent 起動の直接 trigger ではなく、controller が拾って reconciler を起動する間接経路 |
| **bead store と event bus の関係** | bead = state (現状の正本)、event = log (履歴 + 速報通知)。bd 内蔵の `.beads/hooks/on_*` 経由で **bead 変更が events.jsonl に流れる**。controller は `CachingStore` で両者を橋渡しし、bead で state を判断、event で「いつ判断するか」を決める(§6) |

そして本書のコア結論:

> **「event 駆動で agent が起きる」は controller を介した間接駆動である**。常時起動の user-facing エージェント (mayor 等) は不要。controller プロセスだけが必須で、controller が reconciliation を回し、その中で event の影響(order trigger 評価・bead routing 状況)を取り込んで agent を spawn / drain する。state の真実は **bead store** にあり、event bus はその **観測 + 速報通知 + キャッシュ反映** を担う二重化された冗長設計になっている。

---

## 1. つまずきやすい 3 用語の正体

### 1.1 reconciliation とは

**「設定が期待する状態」と「いま動いている状態」を比較し、差分を埋めるためのアクションを取る循環処理**のことです。

Kubernetes の controller pattern や Terraform の plan/apply と同じ考え方です:

| ツール | desired (期待) | current (現実) | 差分を埋めるアクション |
|---|---|---|---|
| Kubernetes ReplicaSet | 「Pod を 3 個動かしたい」 | 実際に動いている Pod 一覧 | Pod を作成 / 削除 |
| Terraform | `.tf` ファイルが宣言する infra | クラウド側の実状態 | リソースを create / update / destroy |
| **Gas City controller** | **`pack.toml` + `scale_check` の結果が期待する agent 一覧** | **bead store に保存された session bead 群 + 実際の tmux ペイン** | **agent session を spawn / wake / drain / close / kill** |

Gas City での実装は `cmd/gc/session_reconciler.go:264` の `reconcileSessionBeadsTraced()` が中心で、デフォルト 1 秒間隔の tick (`cmd/gc/city_runtime.go:503-535`) で繰り返されます。詳細は §2。

### 1.2 pool とは

**同じ役割の agent を弾力的に 0〜N 体動かしたい**ときに使う、`[[agent]]` の中の `[agent.pool]` サブテーブルです。**`[[pool]]` という独立セクションではありません**。

```toml
[[agent]]
name = "polecat"
prompt_template = "agents/polecat/prompt.template.md"

[agent.pool]
min = 0          # 最小数。0 なら不要時は何も動かない
max = 5          # 最大数。1 を超えると polecat-1, polecat-2, ... と命名
check = "bd ready --metadata-field gc.routed_to=polecat --unassigned --json | jq length"
```

**pool agent は常駐するのか?**

→ **`min` の値しだい**:

| `min` | 挙動 |
|---|---|
| `0` | 常駐なし。仕事が来た瞬間に spawn し、終われば自然に exit |
| `1` 以上 | その数までは常駐。超過分は需要に応じて spawn |
| `min == max` の固定 | 常時 `max` 体が動き続ける |

詳細は §3。

### 1.3 probe (`scale_check`) とは

**pool が「いま自分は何体必要か」を controller に答えるための shell コマンド**です。controller がこの shell を `sh -c` で起動し、**stdout に書かれた整数 1 行**を読み取って desired state に反映します。

```toml
[agent.pool]
check = "bd ready --metadata-field gc.routed_to=polecat --unassigned --json | jq length"
#       ↑ controller が sh -c でこれを実行 → stdout に "3" → 「3 体必要」
```

返り値は `[min, max]` にクランプされます。`check` を省略すると、内部でデフォルトの「自分宛にルーティングされた未着手 bead を数える」shell が自動生成されます (`internal/config/config.go:1925-1929`)。詳細は §4。

---

## 2. Controller の reconciliation ループ

### 2.1 比較する 2 つの状態

```mermaid
flowchart LR
    subgraph Desired["DesiredState<br/>(期待する状態)"]
        D1["pack.toml / city.toml の<br/>[[agent]] 定義"]
        D2["scale_check の返り値<br/>(各 pool ごと)"]
        D3["named session の demand"]
    end

    subgraph Current["Current State<br/>(現実の状態)"]
        C1["beads ストアの<br/>session bead 群"]
        C2["tmux ペインの<br/>実プロセス"]
    end

    Reconciler["reconciler<br/>cmd/gc/session_reconciler.go"]

    Desired --> Reconciler
    Current --> Reconciler

    Reconciler --> A1["spawn<br/>(足りない agent を起動)"]
    Reconciler --> A2["wake<br/>(寝てる agent を起こす)"]
    Reconciler --> A3["drain<br/>(余ってる agent を落とす)"]
    Reconciler --> A4["kill<br/>(orphan / suspended を強制終了)"]
    Reconciler --> A5["close<br/>(drain 済み bead を閉じる)"]
```

`buildDesiredStateWithSessionBeads()` (`cmd/gc/build_desired_state.go:182`) が DesiredState を構築し、`reconcileSessionBeadsTraced()` (`cmd/gc/session_reconciler.go:264`) が現実と突き合わせます。

### 2.2 5 つのアクション

| アクション | 意味 | 発動条件 |
|---|---|---|
| **spawn** | 新しい session bead を作って provider プロセス起動 | desired にあって bead が無い |
| **wake** | 既存 bead に対応する provider が眠っているのを起こす | desired にあって bead はあるが asleep |
| **drain** | provider に Ctrl-C を送って graceful shutdown | desired から外れたが provider は生きている |
| **kill** | provider プロセスを強制終了 | orphan / suspended のドレインで必要なとき |
| **close** | bead に終了マークを付けて削除 | drain ack 済み or 仕事が無い |

これらは `session_reconciler.go:1041-1144` の `executePlannedStartsTraced()` 系で実行されます。`runtime.Provider.Start()` が最終的に tmux 上で provider プロセスを spawn します (`internal/session/chat.go:361`)。

### 2.3 3 つのフェーズ

reconciliation は毎 tick で次の 3 段を順に走ります:

```
Phase 0: timer heal      期限切れの遷移タイマーをリセット
                         重複 named session を整理
                         (session_reconciler.go:296-315)
                              │
                              ▼
Phase 1: forward pass    desired state に従ってトポロジカル順で各セッション処理
                         spawn / wake / drain の候補をリスト化
                         (session_reconciler.go:329-1138)
                              │
                              ▼
Phase 2: drain progress  進行中の drain を一歩進める
                         idle probe 管理
                         (session_reconciler.go:1146-1151)
```

### 2.4 trigger は時間と event の混合

reconciler は次の条件のいずれかで起動します (`cmd/gc/city_runtime.go:503-535`):

| trigger | 内容 |
|---|---|
| **patrol tick** | デフォルト 1 秒ごと (`patrol_interval`) |
| **poke** | 外部から `pokeCh` に通知が来たとき (即時) |
| **control socket command** | `gc reload` / `gc converge` などの CLI 命令 |
| **config reload** | `pack.toml` / `city.toml` 変更検知 |
| **crash adoption** | beads にあるが provider プロセスが消えてる session を発見したとき |

つまり「定期的にチェック」と「必要なときに即座に起動」の両方で動きます。

---

## 3. Pool の正体 — 弾性ワーカーの居場所

### 3.1 TOML 構文 (`[agent.pool]` サブテーブル)

```toml
[[agent]]
name = "polecat"
prompt_template = "agents/polecat/prompt.template.md"
work_query = "bd ready --metadata-field gc.routed_to=polecat"

[agent.pool]
min = 0
max = 5
check = "bd ready --metadata-field gc.routed_to=polecat --unassigned --json | jq length"
```

`[[pool]]` という独立セクションは存在しないので注意。pool は **agent の属性**です(`internal/config/config.go:290-320`)。

### 3.2 min / max とライフサイクル

```mermaid
stateDiagram-v2
    [*] --> idle: city 起動
    idle --> spawning: scale_check が N>0 を返す<br/>(または bead が routed_to で到着)
    spawning --> active: provider プロセス起動成功<br/>nudge poller 起動
    active --> active: 仕事を取って実行<br/>(work_query で bead を発見)
    active --> draining: scale_check の N が下がる<br/>(または idle_timeout 到達)
    draining --> [*]: 現在の bead を完了して exit
    active --> [*]: crash → 次 tick で reconciler が判定
```

`min=0, max=5` の polecat なら、需要が無いときは 0 体、5 件の仕事が来れば 5 体まで並列で動き、終われば自然に消えます。

### 3.3 min を 1 以上にするユースケース

`examples/` 配下の pack を全件確認すると、`min ≥ 1` を設定しているのは **swarm pack の coder だけ**です。それ以外(gastown polecat、gastown dog、swarm dog、lifecycle polecat、hyperscale worker)はすべて `min = 0` の純オンデマンド運用になっています。つまり「pool = 常駐ゼロ」が標準であり、`min` を上げるのは明確な理由があるときに限られます。

> **TOML の実フィールド名について**: 本書では `[agent.pool]` 内の `min` / `max` という設計ドキュメント (`engdocs/design/agent-pools.md`) の表記で説明していますが、実装済みの `examples/` の `agent.toml` では agent のトップレベルに `min_active_sessions` / `max_active_sessions` と書きます。意味は同じです。
>
> ```toml
> # 実際の examples/swarm/packs/swarm/agents/coder/agent.toml
> scope = "rig"
> nudge = "Run gc bd ready --unassigned, check mail, then claim and work on a task."
> idle_timeout = "2h"
> min_active_sessions = 1
> max_active_sessions = 5
> ```

#### ① コールドスタートのレイテンシを避ける

provider プロセス起動 (Claude Code が立ち上がる)・prompt のレンダリング・nudge poller goroutine の起動を合計すると数秒〜十数秒かかります。`min = 0` だと仕事が来てからこの起動時間が体感に響きます。**`min = 1` にしておけば、最初の仕事が来た瞬間に既に動いている agent が即 claim できる**ので、レスポンスが大きく改善します。

#### ② provider 側の prompt cache を温存する

Anthropic の prompt cache は **5 分 TTL**。仕事がスパースに(例: 10 分おきに 1 件)来るワークロードで `min = 0` だと、毎回 cold start で cache miss が発生し、毎回フルプロンプトの料金がかかります。**`min = 1` にすると agent プロセスが生き続けるので、5 分以内に次の仕事が来れば cache hit**。コストとレイテンシの両方を稼げます。

#### ③ GUPP 原則の保証 — 「届いた仕事はすぐ走る」

緊急 alert 系の formula(例: `mol-dog-doctor.toml` の dolt サーバ異常検知 → ESCALATION)が cron で発火したとき、`min = 0` だと「agent が起動するまで」の数秒〜十数秒の空白が発生します。**ミリ秒〜秒単位の遅延が許容できないロール**では `min = 1` で常時待機させる必要があります。

#### ④ 並列度を固定して保証する (`min == max`)

```toml
min_active_sessions = 3
max_active_sessions = 3
```

これで **常時きっちり 3 体**が動き続けます。テスト・デモ・ベンチマークで「並列度を決定論的にしたい」場合や、外部 API のレートリミットに合わせて「同時実行は必ず N 並列」と固定したい場合に使います。scale_check による弾力性が不要なケース。

#### ⑤ broadcast 文化の pack で「常時 listening」を保証する

`examples/swarm/packs/swarm/agents/coder/agent.toml` がまさにこのケースです。swarm pack はフラット型・broadcast 文化(MESSAGING.md §5.3)で、coder どうしが `Claiming:` `Done with:` `Conflict in:` を `--all` で投げ合います。**誰かが常に listening していないと broadcast が機能しない**ため、coder は `min_active_sessions = 1` で「最低 1 人は起きている」ことを保証しています。

階層型の Gas Town pack の polecat には逆にこの要請がありません(escalation を受ける mayor が常駐 named session として別途存在するため)。だから polecat は `min = 0` で良い。

#### min を上げるコストとトレードオフ

| 観点 | 影響 |
|---|---|
| API トークン消費 | agent が idle でも provider セッションは生きているので、SessionStart や定期 hook で僅かに消費する |
| メモリ / tmux ペイン | 1 ペイン分常時占有。20 体常駐させるとそれなりのリソース |
| `idle_timeout` との関係 | `min ≥ 1` の最低数までは idle_timeout に達しても drain されない(min が hard floor になる) |
| reconciliation の動作 | min が高いと「需要 0 でも min まで戻す」spawn が走り、log が増える |

#### 判断フロー

```mermaid
flowchart TD
    Q1{仕事は秒単位で<br/>「いつ来るか分からない」?}
    Q2{起動レイテンシは<br/>許容できる?}
    Q3{仕事の頻度は?<br/>または broadcast 文化?}

    Q1 -->|YES| Q2
    Q1 -->|NO| Q3

    Q2 -->|YES| A1["min = 0<br/>(デフォルト推奨)"]
    Q2 -->|NO| A2["min = 1<br/>1 体だけ常駐"]

    Q3 -->|頻繁・broadcast| A3["min = 1〜N<br/>並列度を固定したいなら<br/>min = max"]
    Q3 -->|偶発的| A1
```

「とりあえず動かしてみて、レイテンシや cache miss が問題になったら min を上げる」が実践的なアプローチです。最初から `min ≥ 1` にする必要があるのは broadcast pack の coder のような **常時 listening が役割の本質に組み込まれているケース**だけです。

### 3.4 複数インスタンスの命名規則

`max > 1` なら controller が自動で接尾辞を付けます:

```
polecat-1, polecat-2, polecat-3, polecat-4, polecat-5
```

全インスタンスは **同じ prompt template・同じ work_query** を共有します。役割分担はなく、bead が `bd ready` で見える状態であれば、どの polecat-N が拾っても良い設計です(自然なロードバランシング)。

### 3.5 Named Session vs Pool Agent

| 観点 | Named Session | Pool Agent |
|---|---|---|
| 用途 | 単一インスタンスの常駐 (mayor, deacon) | 弾性ワーカー (polecat, coder) |
| TOML | `[[agent]]` のみ (pool 設定なし) または `[[named_session]]` | `[[agent]] + [agent.pool]` |
| 同名インスタンス数 | 1 | `min`〜`max` (動的) |
| 命名 | `mayor` (単数) | `polecat-1`, `polecat-2`, ... |
| 常駐できる? | YES (デフォルト) | YES (`min ≥ 1` のとき) |
| on-demand 可? | YES (`mode = "on_demand"`) | YES (`min = 0` のとき) |

「常駐 vs オンデマンド」は agent の種類ではなく、**設定値の選択**です。

### 3.6 pool は「キュー」を持っているか?

**持っていません**。Gas City で「待ち行列」に相当するのは **bead store そのもの**です:

```
bead を作る人 (人間 / agent / formula / order)
        │
        ▼
bead に Metadata.gc.routed_to = "polecat" を貼る
        │
        ▼
bead store に永続化  ← これが「キュー」の実体
        │
        ▼
polecat-N が work_query で発見して claim
```

pool 側にメモリ上のキューがあるわけではなく、**永続層の bead 群を pool が pull する**形です。これが「pool は in-memory queue を持たない」設計の意味で、NDI(Nondeterministic Idempotence)の実装上の根拠になっています。

---

## 4. probe (`scale_check`) — pool に何体必要かを答える仕組み

### 4.1 scale_check は shell コマンド + stdout 整数

`scale_check` は **任意の shell コマンド**で、controller が `sh -c` で実行します(`cmd/gc/pool.go:85` `shellScaleCheck()`)。**返り値は stdout に書かれた整数 1 行**で、これを `strconv.Atoi()` でパースして desired count に使います(`cmd/gc/pool.go:161-174` `parseScaleCheckCount()`)。

controller が毎 tick で行う処理を擬似コードで書くと:

```text
// 毎 tick (デフォルト 1 秒) に controller が実行
for each pool in cityConfig.pools:
    output = run_shell(pool.scale_check)        // sh -c で起動し stdout を取得
    n = parse_int(trim(output))
    desired = clamp(n, pool.min, pool.max)
    desiredState[pool.name] = desired
```

実行例:

```bash
$ sh -c 'bd ready --metadata-field gc.routed_to=polecat --unassigned --json | jq length'
3
$ echo $?
0
```

→ controller は「polecat は 3 体必要」と解釈し、`min=0, max=5` の範囲に既に収まっているので desired を `polecat: 3` にする。

### 4.2 デフォルトの scale_check (自動生成)

`check` を書かない場合、Gas City は次のような shell を内部生成します(`internal/config/config.go:1925-1929`):

```bash
ready=$(bd ready --metadata-field gc.routed_to=<agent-name> --unassigned --json | jq 'length')
molecules=$(bd list --metadata-field gc.routed_to=<agent-name> --status=open --type=molecule --no-assignee --json | jq 'length')
echo "$(( ${ready:-0} + ${molecules:-0} ))" || echo 0
```

つまり「自分宛にルーティングされた未着手 bead と未割り当ての molecule の数」を出力します。多くの場合、これだけで十分動きます。

### 4.3 失敗時の挙動

| 失敗ケース | 挙動 |
|---|---|
| stdout が空 | `min_active_sessions` (新規需要モードなら 0) を採用 |
| stdout が整数でない | 同上 |
| stdout が負数 | 同上 |
| 180 秒タイムアウト | エラーとして同上 |

stderr にエラーが記録されますが、controller 自体は停止しません(`cmd/gc/pool.go:120-159`)。**probe の失敗は city 全体を落とさない**設計です。

### 4.4 実行頻度・実行コンテキスト

- 実行主体: controller プロセス内の `evaluatePendingPools()` goroutine (`cmd/gc/build_desired_state.go:59-130`)
- 頻度: 毎 reconciliation tick (デフォルト 1 秒)
- 並列度: pool ごとに並列実行(セマフォで上限制御)
- working dir: pool の `work_dir`
- 環境変数: controller の query prefix を継承

---

## 5. event bus — 観測ログとしての役割

### 5.1 KnownEventTypes は 44 種、カテゴリで整理

`internal/events/events.go:71-88` に定数で定義された全 event 種別を、カテゴリ別に整理:

| カテゴリ | 主な event | 主な発行者 |
|---|---|---|
| **Session lifecycle** | `session.woke` `session.stopped` `session.crashed` `session.draining` `session.idle_killed` `session.suspended` `session.quarantined` `session.updated` | supervisor |
| **Bead operations** | `bead.created` `bead.closed` `bead.updated` | controller |
| **Mail** | `mail.sent` `mail.read` `mail.archived` `mail.replied` `mail.deleted` 等 | API / hook |
| **Convoy / Order** | `convoy.created` `convoy.closed` `order.fired` `order.completed` `order.failed` | controller |
| **Controller** | `controller.started` `controller.stopped` | CLI |
| **City** | `city.created` `city.ready` `city.suspended` `city.resumed` 等 | supervisor |
| **Provider / Worker** | `provider.swapped` `worker.operation` | controller / worker |
| **External messaging** | `extmsg.bound` `extmsg.inbound` `extmsg.outbound` 等 | API |

### 5.2 pull 型と watch 型の併存

| 消費者 | 取得方式 | 用途 |
|---|---|---|
| controller (bead 変更通知) | **watch** (`Provider.Watch().Next()` で blocking) | `bead.created/updated/closed` を購読して `CachingStore.ApplyEvent()` + `Poke()` で reconciler 即起動。詳細は §6 |
| controller (order trigger 評価) | **pull** (`Provider.List(Filter)`) | order の `on = "<event-type>"` 条件を毎 tick で確認 |
| supervisor multiplexer | **pull** | 複数 city の event を統合 |
| CLI ユーザー (`gc events`) | **pull** | 過去 event の検索 / デバッグ |
| `gc events --follow` | **watch (内部) + SSE (外部)** | リアルタイムテイル |
| ダッシュボード | **watch (内部) + SSE (外部)** | `GET /v0/events/stream` で配信 (heartbeat 15 秒) |
| 外部スクリプト | **pull** (fork/exec) | カスタム event sink |

「SSE」は internal で `Watch()` した結果を SSE プロトコルでクライアントへ流しているだけで、event bus 内部の API は `List()` (pull) と `Watch().Next()` (blocking watch) の 2 系統に統一されています。push 型のメッセージング(購読者に直接配信)ではなく、**purchaser が引き取りに来る pull、または blocking で次を待つ watch** のいずれかです。

### 5.3 event は agent 起動の直接 trigger ではない

ここが最も誤解しやすい点です:

```
誤解:  event 発火 → agent 起動 (push 型のメッセージング)
正解:  controller が event を拾う (Watch / List) → 状態を更新 or 評価
       → reconciliation サイクルで spawn / drain を判定
```

event 自体は **agent を直接起動しない**。controller が間に入ります:

- **bead 関連 event**(`bead.created/updated/closed`): controller が `Watch()` で受け取り、`CachingStore` を更新して reconciler を `Poke()`(即起動)。reconciler が spawn 判断する
- **event 種 trigger を持つ order**: controller が毎 tick で `List(Filter)` して該当 event を探し、見つかれば wisp 生成 → pool routing → 次 tick で spawn
- **その他の event**: 観測ログとして記録されるだけ。誰も購読していなければ何の作用もない

つまり event は「速報通知」と「観測記録」を兼ねていますが、**意思決定はすべて controller の reconciler が下します**。これが「event 駆動だが state は bead」の正確な姿です。詳しくは §6(bead と event の関係)と §7(全体像)で扱います。

### 5.4 ユーザーが `gc events` で見る使い方

```bash
gc events                              # 全 event (JSON Lines)
gc events --type bead.created          # 種別フィルタ
gc events --since 1h                   # 過去 1 時間
gc events --payload-match actor=human  # payload フィルタ
gc events --follow                     # リアルタイム tail (SSE)
gc events --watch --type order.fired   # 最初の match を待つ (デフォ 30s)
gc events --seq                        # 現在の event head 位置
```

調査・デバッグ用途。`.gc/events.jsonl` ファイルが実体で、`cat` してもいいですが、`gc events` の方がフィルタしやすいです(`cmd/gc/cmd_events.go`)。

---

## 6. bead store と event bus の関係

§3〜§5 で見てきたとおり、Gas City には 2 つの永続層があります。`bead store`(dolt または file)と `event bus`(`.gc/events.jsonl`)。両者は独立しているのに、操作上は一体に見える。なぜ分かれているのか、どう連動しているのか、どちらを controller がどう使うのかを整理します。

### 6.1 別々の永続層 — state と log

| | bead store | event bus |
|---|---|---|
| 物理ファイル | `dolt` の DB or `.beads/issues.jsonl` (file mode) | `.gc/events.jsonl` |
| 性質 | **state** (今の状態を保持。CRUD 可能) | **log** (過去の事実を append-only で記録) |
| 操作 API | `bd create / show / update / close / dep / label` | `gc event emit` / `gc events` |
| 削除 | `bd close` で可能 | **不可** (immutable、Seq は monotonic) |
| 真実度 | ground truth (現状の正本) | observation (履歴・観測ログ) |
| 主な利用者 | エージェント・人間が work を扱う | controller の trigger 評価、ダッシュボード、デバッグ |

**bead = もの、event = ものに何が起きたかの記録**、と覚えると分かりやすい。`AGENTS.md` の階層不変条件:

> Beads is the universal persistence substrate **for domain state**.
> Event Bus is the universal **observation** substrate.

### 6.2 bd の hook 機構による連動

bd には **2 種類の hook 機構** があります(混同注意):

| 種別 | 何のための hook か | 設置場所 |
|---|---|---|
| **issue ライフサイクル hook** | bead の create/close/update に反応してスクリプトを起動 | `.beads/hooks/on_create` / `on_close` / `on_update` |
| **git hook** | git の commit/merge/checkout で bead を同期 | `.git/hooks/*` (`bd hooks install` で設置) |

`bd hooks` サブコマンドは後者(git hook)専用なので、前者(issue ライフサイクル hook)を見落としやすいですが、bd binary 内部に `hook.path` `hook.exec` `on_create` `on_update` `on_close` の文字列が組み込まれており、bead 操作時にこれらのスクリプトを fork-exec する仕組みになっています。

Gas City は city 初期化時に `installBeadHooks()` (`cmd/gc/hooks.go:59`) で 3 つのスクリプトを `.beads/hooks/` に自動配置します:

```sh
# .beads/hooks/on_create の中身 (gc が install したもの)
#!/bin/sh
GC_BIN="${GC_BIN:-gc}"
DATA=$(cat)                                                    # bd が stdin で issue JSON 全文を渡す
title=$(echo "$DATA" | grep -o '"title":"[^"]*"' | head -1 | cut -d'"' -f4)
(
  "$GC_BIN" event emit bead.created --subject "$1" --message "$title" --payload "$DATA" >/dev/null 2>&1 || true
) </dev/null >/dev/null 2>&1 &
```

`on_close` だけは追加で `gc convoy autoclose` と `gc wisp autoclose` も呼びます(子 wisp の自動クローズ用)。

つまり **bd 自身に hook 機構があり、Gas City がそこに `gc event emit` を呼ぶスクリプトを差し込んで両層を連動させている**。両層は独立だが、bd hook が橋渡し役を担う。

### 6.3 events.jsonl は bead 変更履歴を兼ねる

`cmd/gc/hooks.go:13` の対応マップ:

| bd の操作 | hook | 発生する event | payload の中身 |
|---|---|---|---|
| `bd create` | `on_create` | `bead.created` | issue の JSON 全文 |
| `bd update` / `label` / `dep` / `assign` / `comment` 等 | `on_update` | `bead.updated` | issue の JSON 全文 |
| `bd close` | `on_close` | `bead.closed` | issue の JSON 全文(+ convoy/wisp autoclose 副作用) |

events.jsonl に書かれる 1 行の例:

```json
{"seq":42,"ts":"2026-04-30T15:23:01Z","type":"bead.created","subject":"gc-1234","message":"Fix login bug","payload":{"id":"gc-1234","title":"Fix login bug","type":"task","status":"open","assignee":"polecat","labels":[],"...":""}}
```

**payload に issue の JSON 全文** が入るので、events.jsonl だけ読み返せば「いつ・誰が・どんな bead を作った/変えた/閉じた」が時系列で完全に再生できます。監査・デバッグ・event replay に十分な情報量です。

ただし **「最新状態を引きたい」なら bead store(CachingStore 経由)を引くのが正解** で、event を全件 replay するのは重い。**状態のクエリは bead、履歴の追跡は event** という使い分けです。

#### bead 経由でない event もある

すべての event が bead に紐付くわけではありません:

| 種別 | bead と関係 |
|---|---|
| `bead.created` / `bead.closed` / `bead.updated` | ◎ ある(bd hook 経由) |
| `mail.sent` / `mail.read` / `mail.replied` 等 | ○ ある(mail も bead なので、`bead.updated` と並行で発生) |
| `order.fired` / `order.completed` | △ wisp bead 生成のとき紐付く |
| `convoy.created` / `convoy.closed` | △ convoy bead に紐付く |
| `controller.started` / `controller.stopped` | × bead と無関係(controller ライフサイクル) |
| `city.created` / `city.ready` / `city.suspended` | × city ライフサイクル |
| `session.woke` / `session.crashed` 等 | △ session bead と紐付く場合あり |
| `extmsg.inbound` / `extmsg.outbound` | × 外部メッセージング、bead に直接対応しない |

つまり **event の世界の方が広い**。bead は「永続化された作業」だけだが、event は「システム全体の出来事」を扱います。

### 6.4 controller は両方見る — 役割が違う

ここが最大の誤解ポイントです。「controller は event bus を見る」と「controller は bead を見る」は**両方正解で、用途が違います**:

```mermaid
flowchart TB
    subgraph Sources["controller のデータ源"]
        BeadStore["bead store (dolt)<br/>= state の source of truth"]
        EventBus["event bus (.gc/events.jsonl)<br/>= state 変化の通知 + 起動 trigger"]
    end

    Reconciler["reconciler<br/>session_reconciler.go"]
    Watcher["startBeadEventWatcher<br/>api_state.go:223"]
    Cache["CachingStore<br/>internal/beads/caching_store.go"]

    BeadStore -->|"List() で session bead 取得<br/>= current state"| Cache
    Cache -->|"in-memory cache 経由"| Reconciler
    EventBus -->|"Watch().Next() で<br/>bead.* を購読"| Watcher
    Watcher -->|"ApplyEvent()<br/>= キャッシュ更新"| Cache
    Watcher -->|"Poke()<br/>= reconciler 即起動"| Reconciler
    EventBus -.->|"List(Filter) で<br/>order trigger 評価"| Reconciler
```

役割の正確な分担:

| controller の処理 | 何を見るか | 実装箇所 |
|---|---|---|
| 「いま動いている session」を把握 | **bead store** (`store.List()` → CachingStore) | `cmd/gc/session_reconciler.go:1239,1591` |
| 「期待する数」を計算 | config + `scale_check` の stdout | `cmd/gc/build_desired_state.go:182` |
| 外部 bd 書き込みの即時検知 | **event bus** (`Watch().Next()`) | `cmd/gc/api_state.go:223` `startBeadEventWatcher()` |
| キャッシュ更新 | event payload を `CachingStore.ApplyEvent()` に流す | `cmd/gc/api_state.go:270` |
| reconciler を即起動 | event 受信時に `cs.Poke()` | `cmd/gc/api_state.go:274` |
| order trigger (event type) 評価 | event bus を `List(Filter)` で pull | `internal/orders/triggers.go:175-196` |

### 6.5 CachingStore が両者をブリッジする

reconciler は素の bead store ではなく **`CachingStore` 経由で bead を読みます**。これが両層をつなぐ要です:

```mermaid
flowchart LR
    BD[("bead store<br/>(dolt)")]
    EB[("event bus<br/>(events.jsonl)")]

    subgraph Cache["CachingStore (in-memory)"]
        Beads["beads: map[ID]Bead"]
        Deps["deps: map[ID][]Dep"]
        WD["watchdog reconciler<br/>(30s/60s/120s 適応的)"]
    end

    BD -->|"① Prime() で全件読込<br/>(起動時 + watchdog)"| Cache
    EB -->|"② ApplyEvent()<br/>(bead.* event ごとに個別更新)"| Cache

    Cache -->|"List(query) / Get(id)<br/>(in-memory で即応答)"| Reconciler[reconciler]
```

入力経路は 2 系統:

- **① 直接スキャン**: `Prime()` で起動時に dolt から全件読込み。watchdog reconciler が 30s/60s/120s 間隔で適応的に再スキャンして整合補正(`caching_store.go:71-77`)
- **② event 配信**: `bead.*` event の payload で個別に更新(`ApplyEvent`)

reconciler から見ると、**メモリ上の整合した view を即取れる**。整合性を保つために 2 系統の入力を持っているのが要点です。

### 6.6 なぜ二重化するのか — 3 つの理由

理屈の上では reconciler が毎 tick で bead store を直接 `List()` すれば十分のはず。なぜ event bus 経由まで作ったのか:

#### 理由 1: サブ秒の応答性

- **tick だけ**: 1 秒ごとの定期スキャン → 最大 1 秒の遅延
- **event watch + Poke**: bd hook 発火の瞬間に reconciler が起こされる → サブ秒応答

`api_state.go:223` のコメントが端的:

> startBeadEventWatcher subscribes to the event bus and feeds bead events to all CachingStore instances for **sub-second cache freshness** on agent-initiated bd mutations (bd hooks → gc event emit → this watcher → ApplyEvent).

#### 理由 2: マルチプロセス整合性

外部のエージェントが `bd create` を直接叩いたとき、controller のメモリ内 `CachingStore` は古いまま。**bd hook → event emit → controller の Watch → ApplyEvent** という経路で、別プロセスからの変更も即座に反映できる。

`caching_store.go:18` のコメント:

> External writes (agents running bd directly) are picked up via the bd hook → gc event emit → event bus path. Call ApplyEvent when the event bus delivers bead.created/updated/closed events.

#### 理由 3: 失敗からの復旧

event は best-effort で失敗を握り潰す(`>/dev/null 2>&1 || true`)。それだけ信じると整合性が崩れる。`CachingStore` は **watchdog reconciler** を持ち、定期的に bead store を直接スキャンして event 経路の取りこぼしを補正します。

つまり「**event 経路で速報を受け、bead 経路で照合する**」という二重化。これが NDI(Nondeterministic Idempotence)の実装上の現れです:

- 速報経路(event)が落ちても、watchdog が bead から復元する
- bead が真実なので、event を全部消失しても state は失われない
- 複数の独立した観察者(event Watch、tick、watchdog)が同じ bead 状態に収束する

### 6.7 設計パターンとしての位置づけ

CQRS(Command Query Responsibility Segregation)に近い構造ですが、純粋な event sourcing とは違います:

| パターン | 真実の所在 | event の役割 | Gas City との対比 |
|---|---|---|---|
| 純粋な event sourcing | event log のみ | 真実そのもの。state は導出 | 違う |
| CQRS (command/query 分離) | command 側の state | command と query の橋渡し | 近い |
| **Gas City** | **bead store** | **状態変化の通知 + キャッシュ反映 + trigger 評価** | これ |

要するに **bead が真実、event は副産物**。event を全部消しても bead さえあれば state は復元できる(逆は違う)。これにより:

- bead は indexed query で効率的に引ける(`scale_check` の `bd ready ... | jq length` が現実的なコストで動く)
- event は時系列・追記専用の特性を活かして観測 / 監査 / 即応通知に専念できる
- event の取りこぼしは watchdog で補正でき、bead が真実なので最終的な整合は守られる

> **覚え方**: 「bead は何があるか、event は何が起きたか」。controller は両方を見て、bead で state を判断し、event で「いつ判断するか」を決める。

---

## 7. 全体像 — 「event 駆動でエージェントが起きる」の正確な姿

§1〜§6 を統合すると、agent 起動の「自律フロー」は次のようになります:

```mermaid
sequenceDiagram
    participant World as 何かが起きる<br/>(時刻到来 / bead 作成 / mail 受信)
    participant Bus as event bus<br/>(.gc/events.jsonl)
    participant Ctl as controller<br/>(常駐デーモン)
    participant Probe as scale_check<br/>(shell コマンド)
    participant Recon as reconciler
    participant Pool as pool agent<br/>(polecat-N)

    Note over Ctl: 1秒ごとの tick (またはイベント駆動 wake)

    World->>Bus: event を append
    World->>Bus: bead を update<br/>(routed_to = polecat)

    Ctl->>Bus: 新着 event を pull
    Ctl->>Ctl: order trigger 評価<br/>(on = "..." の条件マッチ?)
    alt order が due
        Ctl->>Ctl: dispatchOne()<br/>wisp 生成 + pool routing
    end

    Ctl->>Probe: sh -c "<scale_check>"
    Probe-->>Ctl: stdout: "3"
    Ctl->>Ctl: desired[polecat] = 3<br/>(min/max にクランプ)

    Ctl->>Recon: reconcile(desired, current)
    Recon->>Recon: 差分計算<br/>(spawn/wake/drain/kill/close)
    Recon->>Pool: spawn polecat-1, polecat-2, polecat-3<br/>(runtime.Provider.Start)

    Note over Pool: 各 polecat-N が起動<br/>nudge poller 開始<br/>work_query で bead を pull
    Pool->>Bus: session.woke event を発行
    Pool->>Pool: 仕事を実行
    Pool->>Bus: bead.closed event を発行

    Note over Ctl: 仕事が無くなれば次 tick で<br/>desired = 0 となり drain される
```

ここから分かる重要な点:

1. **event bus 自体は trigger ではない**。push されない。controller が pull しないと何も起きない
2. **controller プロセスだけが「常駐必須」**。mayor のような user-facing 常駐 agent は pack 設計の選択であって、event 機構の前提ではない
3. **agent 起動は reconciliation の出力**。event はその入力の一部に過ぎない
4. **pool agent は「待機中」を必要としない**。`min=0` でも、仕事が来れば controller が起こしてくれる

つまり「**event 駆動でエージェントが起きる**」は嘘ではないが、間に **controller の reconciliation という意思決定者**が必ず挟まる、というのが正確な姿です。

---

## 8. 「常時起動」が必須なのは何か

| 主体 | 常駐必須? | 理由 |
|---|---|---|
| supervisor (launchd / systemd) | ✓ 必須 | マシン全体で 1 個。各 city の controller を起動する |
| **controller (city ごと 1 個)** | ✓ 必須 | reconciliation の主体。event を pull し agent を spawn する **唯一のプロセス** |
| named session agent (mayor / deacon) | △ pack 設計次第 | 多くの pack で常駐させているが、SDK としては必須でない |
| pool agent (polecat / coder) | ✗ 不要 | `min = 0` なら需要発生時のみ spawn |

「`gc` のイベントは常時起動エージェントでしか発生しないのか?」への最終回答:

> **No**。常時起動が必須なのは controller プロセスだけで、controller は reconciliation を通じて、停止中の agent を on-demand で起こす経路を持っています。具体的には、order の cron / cooldown / event trigger が controller の tick で評価され、wisp 生成 → pool routing → 次 tick の reconciliation で pool agent spawn、という連鎖が成立します。mayor のような常時起動 user-facing エージェントが存在しなくても、この連鎖は完結します。

これが AGENTS.md にある **SDK self-sufficiency** (「すべての SDK インフラ操作は controller だけで機能する。SDK 操作が特定のユーザー設定エージェント役割を必要としてはならない」) の実装上の根拠です。

---

## 付録: 主要参照ファイル

| ファイル | 役割 |
|---|---|
| `cmd/gc/city_runtime.go:503-535` | controller の patrol ループ (1 秒 tick) |
| `cmd/gc/build_desired_state.go:182-250` | DesiredState 構築 (config + scale_check の結果) |
| `cmd/gc/build_desired_state.go:59-130` | `evaluatePendingPools()` (pool ごとに probe 並列実行) |
| `cmd/gc/pool.go:85-87` | `shellScaleCheck()` (probe を `sh -c` で起動) |
| `cmd/gc/pool.go:161-174` | `parseScaleCheckCount()` (stdout を整数にパース) |
| `cmd/gc/session_reconciler.go:264-1154` | 3 フェーズ reconciliation 本体 |
| `cmd/gc/session_lifecycle_parallel.go:1358-1483` | spawn 候補の並列実行 |
| `internal/session/chat.go:361` | 最終的な provider プロセス spawn 地点 |
| `internal/orders/order.go:27-28` | order trigger 5 種定義 |
| `internal/orders/triggers.go:175-196` | event 種の order trigger 評価 (`checkEvent`) |
| `internal/orders/order_dispatch.go:340-373` | order 発火 → wisp 生成 → pool routing |
| `internal/events/events.go:71-88` | KnownEventTypes (44 種) |
| `internal/events/multiplexer.go` | 複数 city の event 統合 |
| `internal/config/config.go:290-320` | Pool / Agent 構造体 |
| `internal/config/config.go:1925-1929` | デフォルト scale_check の自動生成 |
| `engdocs/architecture/event-bus.md` | event bus アーキテクチャ仕様 |
| `engdocs/design/agent-pools.md` | pool 設計ドキュメント |
| `engdocs/contributors/reconciler-debugging.md` | reconciler デバッグ手法 |
