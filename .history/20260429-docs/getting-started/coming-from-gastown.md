---
title: Coming from Gas Town
description: Gas Town の概念を Gas City のプリミティブに最速で翻訳する方法。
---

Gas City は Gas Town から抽出された SDK です。生産性を最速で得る方法は、Town の role ツリーを 1 対 1 で移植しようとするのをやめ、代わりに Town の概念を Gas City のプリミティブにマップすることです。

- agents
- beads
- events
- config
- prompt templates
- orders、formulas、waits、mail、sling などの派生メカニズム

Gas Town でシステムを構築したことがあれば、Gas City が解決しようとしている運用上の問題はすでに分かっているはずです。主な変化はロジックがどこに存在するかです。

## 核となる転換

Gas Town は role 分類とファイルシステムレイアウトを中心に形作られています。Gas City は小さなプリミティブセットと設定を中心に形作られています。

Gas Town では次のように考えるのが普通です。

- mayor、deacon、witness、refinery、polecat、crew、dog
- `~/gt/...` ディレクトリレイアウト
- 名前付きオーケストレーション機能としての plugin と convoy
- role 固有のマネージャと cwd 由来の identity

Gas City のデフォルトメンタルモデルは次のとおりであるべきです。

- 再利用可能な動作は `pack.toml` と pack ディレクトリにある
- deployment の選択は `city.toml` にある
- マシンローカルな bindings と runtime state は `.gc/` にある
- 永続的な work item はすべて bead
- agent は汎用的; role は prompts、formulas、orders、設定から生まれる
- controller が SDK インフラ動作を所有する
- ディレクトリは実装詳細であり、アーキテクチャではない

これがオンボーディング上の最大の違いです。Gas City は「コマンドが renamed された Gas Town」ではありません。Gas Town を表現できる、より低レベルなオーケストレーションツールキットです。

## 概念マップ

| Gas Town の概念 | Gas City の概念 | あなたにとって何が変わるか |
|---|---|---|
| Town config + rig config + role ホーム | PackV2: `pack.toml`、`city.toml`、`agents/`、`.gc/` | 定義、deployment、マシンローカル状態が、role 固有のディレクトリやマネージャに散在するのではなく分離されています。 |
| Mayor、deacon、witness、refinery、polecat、crew、dog | 設定された agent | Gas City は Go コード内に焼き付けられた role 名を持ちません。これらは pack 規約であり、SDK プリミティブではありません。 |
| Plugin | Order | Exec order は agent session なしで shell を直接実行します。Formula order は agent 駆動の作業をインスタンス化します。「コマンドを実行する plugin」を考えていたなら、まず exec order から始めてください。 |
| Convoy | Convoy bead と sling/formulas | Convoy は依然として bead に支えられた作業のグルーピングですが、オーケストレーションを得るために使わなければならない特別な convoy ランタイム層はありません。 |
| Dog | 通常はまず order、ときどきスケーラブルな session 設定 | Gas Town では dog は名前付きインフラヘルパーです。Gas City では、その作業の多くは LLM session が不要なので exec order としてよりクリーンになります。 |
| Deacon の watchdog ロジック | Controller と supervisor | ヘルス patrol、order dispatch、wisp GC、reconciliation は controller の責任であり、role agent の責任ではありません。 |
| Witness のライフサイクルロジック | waits、formulas、session スケール設定、controller の wake/sleep の上に構築された pack 動作 | SDK はメカニズムを提供します。Pack が witness role をモデル化するかどうかを決めます。 |
| ハードタイプとしての crew と polecat | 永続 session とスケーラブルな session 設定 | 「crew」と「polecat」は運用スタイルです。Gas City は agent 設定と session 動作のみを知っています。 |
| `~/gt` 配下のディレクトリツリー | identity スコープ用の `dir` と session の cwd 用の `work_dir` | アーキテクチャをパスにエンコードしないでください。Identity は設定とメタデータに保ちます。Role が本当にファイルシステム隔離を必要とする場合のみ `work_dir` を使います。 |
| Role 固有の起動ファイルとローカル設定ディレクトリ | prompt templates、overlays、provider hooks、`pre_start`、`session_setup`、`gc prime` | 起動の整形は、role がディスク上のどこに存在するかから推論されるのではなく、明示的で provider 認識的になります。 |
| パス由来の identity | 明示的な agent identity、rig スコープ、env、bead メタデータ | cwd が agent が誰であるかを暗示すると仮定するコードや prompt を移植するのは避けてください。 |
| Town ワークフロー内の formula runner | Gas City での formula 解決 + バックエンド所有の実行 | Gas City は formula を解決して dispatch しますが、本格的な多段実行は依然としてバックエンド依存です。`bd` が本番のパスです。 |

## たいてい綺麗にマップされるもの

### Role は pack agent になる

Gas Town で新しい role を追加する場面では、Gas City での動きは通常次のとおりです。

1. ローカルの `city.toml` から始める
2. すでに問題のほとんどを解決する pack があれば include する
3. ローカルの動作変更だけが必要なら、刻印された agent を上書きする
4. 共有デフォルトを全員のために変える場合のみ pack を編集する
5. ワークフロー自動化が必要な場合、agent の周りに formula や order を追加する

これにより role 動作を SDK にハードコードするのではなく設定に保ちつつ、初日の一般的なワークフローをローカルかつ漸進的に感じさせます。

### City pack と `city.toml` から始める

これが採用すべき主な初日の習慣です。

ほとんどの Gas Town ユーザーは、import された共有 pack を編集するのではなく、root city pack と `city.toml` から始めるべきです。分割は次のとおりです。

- `pack.toml` は再利用可能な pack を import し、city 固有の動作を定義する
- `agents/<name>/` は city 所有の名前付き agent を定義する
- `city.toml` は rig、substrate、scale などの deployment 選択を宣言する
- `.gc/` はローカル rig パスなどの site bindings を保存する

その変更がその pack の全 consumer にとっての新しい再利用可能なデフォルトになるべきときに、pack 編集に手を伸ばします。

### Plugin は order になる

これが最も重要な実用的翻訳です。

Gas Town で「何かをスケジュールどおり、イベント時、または条件が真のときに自動実行すべき」というアイデアなら、おそらく order が欲しいはずです。

- 作業が単に shell や controller 側のロジックなら **exec order** を使う。
- 作業が agent 駆動のワークフローをインスタンス化すべきなら **formula order** を使う。

これが多くの Town「plugin」直感に対するクリーンな置き換えです。Exec order は特に重要です — prompt なし、session なし、追加の role agent なしで非 agent コマンドを実行できるからです。

### Convoy は依然として bead 形

Gas Town では convoy で考えるよう教えられました。そのメンタルモデルは依然として転移しますが、実装境界が異なります。

Gas City では:

- convoy は依然として bead に支えられたグルーピングと系統
- `gc sling` はルーティングの一部として convoy 構造を作成できる
- formulas、orders、waits がその bead グラフの周りに構成される

なので、作業を追跡するための convoy のメンタルモデルは保ちつつも、bead と dispatch を超えた特別なオーケストレーションサブシステムが必要だと仮定しないでください。

### Crew と polecat は運用モード

Gas Town ではこれらはファーストクラスの worker タイプのように感じられます。Gas City では、これらは規約として考えるのが最善です。

- **crew**: 人間が推論することを想定した永続的な名前付き agent
- **polecat**: スケーラブルまたは一時的な session、しばしば専用 worktree を持つ

その区別は本物で有用ですが、SDK はそれを強制しません。Pack はその規約を採用、緩和、置換できます。

## Gas City が意図的に異なる場所

### Controller がインフラ動作を所有

Gas Town では一部のオーケストレーション動作が特定の role を介して仲介されます。Gas City では controller が次のようなインフラ操作の正典的な所有者です。

- 望ましい session を実行中の session に reconcile
- session スケーリング
- order 評価
- ヘルス patrol
- wisp ガベージコレクション

何かが根本的に SDK インフラなら、別の deacon ライクな role 動作を発明するのではなく、controller のパスに置くことを優先してください。

### ファイルシステムレイアウトはアーキテクチャではない

Gas Town はディレクトリをシステム契約の一部として使います。Gas City はそうしないようにしています。

現在の経験則は次のとおりです。

- `dir` を agent のスコープと identity コンテキストを運ぶために使う
- session が他の場所で実行する必要があるときに `work_dir` を使う
- 永続的なハンドオフ状態には bead メタデータを使う

別の `work_dir` を使うべき正当な理由:

- role が repo を変更し、隔離された worktree が必要
- provider のスクラッチファイルが他の role と衝突する
- role が正典的な rig root から独立した永続サンドボックスを必要とする

不適切な理由:

- 「Gas Town にはこの role 用の別フォルダがあった」

### Role は SDK 法ではなく例

Gastown pack は依然としておなじみの role を出荷していますが、それは Gas City 内部の型システムではなく、運用モデルの一例です。

これはシステムを変更するときに重要です。

- 新しい動作を追加するということは、通常 pack、formula、order、prompt の編集を意味する
- 通常、新しいハードコードされた role を SDK に追加することではない

これはバグではなく機能です。

2 種類の変更を分けて考えるのも価値があります。

- **ローカル city 変更**: `city.toml` の編集、rig オーバーライドの追加、patches の追加、city 固有の agent の追加
- **共有プロダクト変更**: 全員のためのより良いデフォルトが欲しいので pack を編集

オンボーディング作業のほとんどは最初のカテゴリにあるべきです。

## 一般的な翻訳パターン

### 「新しい dog が必要」

最初に問います:

- これは exec order にできるか?

できるなら、order を優先してください。これにより agent スロットを消費せずに trigger ロジック、history、controller 所有権が得られます。

タスクが本当に長寿命の session、リッチな対話的コンテキスト、繰り返しの agent 判断を必要とする場合のみ、dog ライクなスケーラブル session 設定に手を伸ばしてください。

### 「witness ライクなライフサイクルマネージャが必要」

どの部分が次のいずれかを問います。

- controller インフラ
- bead 状態遷移
- formula ロジック
- prompt ガイダンス

最初のカテゴリのみが Go SDK インフラに属します。残りは通常 pack で良く生きます。

### 「別の特別なディレクトリツリーが必要」

通常は不要です。

次から始めます。

- rig からの正典的な repo root
- repo を変更するか provider ファイル隔離が必要な role に対してのみ隔離された `work_dir`
- cwd 推論ではなく明示的な env とメタデータ

### 「agent なしで何かを実行する必要がある」

plugin、ヘルパー role、隠れた session を発明する前に exec order を使ってください。

これが多くの古い Town 自動化タスクに対する Gas City の直接の答えです。

## PackV2 における一般的な Gastown オーバーライド

Gastown pack を使っている場合、これらが最も一般的なローカル変更です。

### Rig を登録する

Gastown pack を root pack で import し、`city.toml` で rig を bind して `gc rig add` を実行します。

```toml
# pack.toml
[pack]
name = "my-city"
schema = 2

[imports.gastown]
source = "./assets/gastown"
```

```toml
# city.toml
[[rigs]]
name = "myproject"

[rigs.imports.gastown]
source = "./assets/gastown"
```

```bash
gc rig add /path/to/myproject --name myproject
```

### スケーラブルな polecat session を増減する

これは「この rig の polecat を増やしたい/減らしたい」に対する最もクリーンな答えです。

```toml
# city.toml
[[rigs]]
name = "myproject"

[rigs.imports.gastown]
source = "./assets/gastown"

[[rigs.patches]]
agent = "gastown.polecat"

[rigs.patches.pool]
max = 10
```

### 1 つの rig の polecat の provider を変更する

```toml
# city.toml
[[rigs]]
name = "myproject"

[rigs.imports.gastown]
source = "./assets/gastown"

[[rigs.patches]]
agent = "gastown.polecat"
provider = "codex"
```

これを同じオーバーライドブロックで session スケールオーバーライド、env、prompt 変更、hook 変更と組み合わせられます。

### City スコープの Gastown agent を変更する

`mayor`、`deacon`、`boot` のような city スコープ agent は patches で簡単に調整できます。

```toml
[[patches.agent]]
name = "gastown.mayor"
provider = "codex"
idle_timeout = "2h"
```

ターゲットがすでに具体的な city スコープ agent のときは patches を使います。ターゲットが rig ごとに刻印される pack agent の場合は `[[rigs.patches]]` を使います。

### 名前付き crew agent を追加する

Crew は通常 city 固有なので、共有 Gastown pack ではなく root city pack に属することが多いです。

```text
agents/wolf/
├── agent.toml
└── prompt.template.md
```

```toml
# agents/wolf/agent.toml
scope = "rig"
nudge = "Check your hook and mail, then act accordingly."
work_dir = ".gc/worktrees/myproject/crew/wolf"
idle_timeout = "4h"
```

これにより共有 pack を汎用的に保ちつつ、city が名前付きの長寿命 worker を持てます。

### Pack を fork せずに prompt、overlay、timeout を変更する

これが rig オーバーライドの目的です。

```toml
# city.toml
[[rigs]]
name = "myproject"

[rigs.imports.gastown]
source = "./assets/gastown"

[[rigs.patches]]
agent = "gastown.refinery"
idle_timeout = "4h"
```

Prompt や overlay の置き換えには、共有 pack をその場で編集するのではなく、root city pack から import された agent を patch してください。

その変更が複数の city 横断で広く有用と判明したときに、pack に移動すべきです。

## `gt` -> `gc` コマンドマップ

これは最も近いマッチのマップであり、両 CLI が同一のアーキテクチャだという主張ではありません。

2 つのルールが大いに役立ちます。

- 古い `gt` コマンドがオーケストレーション、session、ルーティング、hooks、ランタイム動作についてだったなら、最も近いホームは通常 `gc`
- 古い `gt` コマンドが本当は bead CRUD や bead コンテンツについてだったなら、最も近いホームはしばしば依然として `bd` で `gc` ではない

### Workspace とランタイム

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt install` | `gc init` | Gas City では city を作成するのに `gc init` を使います。 |
| `gt init` | `gc rig add` または `gc init` | Town の `init` と `install` は Gas City では city 作成と rig 登録に分かれています。 |
| `gt rig` | `gc rig` | ほぼ直接マッピング。 |
| `gt start` | `gc start` | マシン全体の supervisor 下で city を起動します。 |
| `gt up` | `gc start` | 同じ高レベルの意図。 |
| `gt down` | `gc stop` | 現在の city の session を停止します。 |
| `gt shutdown` | `gc stop` | 同じ意図、異なる実装モデル。 |
| `gt daemon` | `gc supervisor` | Supervisor は Gas City の正典的な長寿命ランタイムです。 |
| `gt status` | `gc status` | City 全体の概要。 |
| `gt dashboard` | `gc dashboard` | 同じ一般目的; `gc dashboard serve` も明示形として依然存在。 |
| `gt doctor` | `gc doctor` | ほぼ直接マッピング。 |
| `gt config` | `gc config` と `city.toml` 編集 | Gas City の設定はファイルファースト; `gc config` はほぼ inspect/explain。 |
| `gt disable` | `gc suspend` | 最も近い運用上のマッチは Town スタイルのシステム全体トグルではなく、city ごとの suspend。 |
| `gt enable` | `gc resume` | Suspend された city を再開。 |
| `gt uninstall` | 直接の同等物なし | Gas City には supervisor インストール/アンインストールはありますが、Town スタイルのグローバルアンインストールコマンドはありません。 |
| `gt version` | `gc version` | 直接マッピング。 |
| `gt completion` | 直接の同等物なし | Gas City は現在対応する completion コマンドを公開していません。 |
| `gt help` | `gc help` | 直接マッピング。 |
| `gt info` | `gc version`、`gc status`、docs | 単一の `gc info` コマンドはありません。 |
| `gt stale` | 直接の同等物なし | 最も近いチェックは `gc version` と `gc doctor`。 |
| `gt town` | `gc start`、`gc status`、`gc stop`、`gc supervisor` に分散 | Gas City は別の Town 名前空間を保持しません。 |

### 設定と拡張

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt git-init` | `git init` と `gc rig add` | Gas City では git リポジトリのセットアップと city 登録は別の関心事です。 |
| `gt hooks` | 設定駆動の hook インストールと `gc doctor` | Gas City は Town の hook 管理名前空間を持ちません; hook インストールは主に設定とライフサイクル駆動です。 |
| `gt plugin` | `gc order` | Plugin ライクな controller 自動化は通常 exec order や formula order になります。 |
| `gt issue` | 直接の同等物なし | 通常は意図に応じて bead メタデータや session コンテキストに置き換えられます。 |
| `gt account` | 直接の同等物なし | Provider アカウント管理は Gas City のコア CLI の外です。 |
| `gt shell` | 直接の同等物なし | Gas City は Town スタイルの shell 統合名前空間を出荷していません。 |
| `gt theme` | 直接の同等物なし | Pack スクリプトや tmux 設定が通常のパスです。 |

### 作業ルーティングとワークフロー

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt sling` | `gc sling` | 精神と名前で直接マッピング。 |
| `gt handoff` | `gc handoff` | ほぼ直接マッピング。 |
| `gt convoy` | `gc convoy` | Convoy 作成と追跡でほぼ直接マッピング。 |
| `gt hook` | `gc hook` | 同じ名前、より狭い表面: `gc hook` は work-query と hook 注入動作で、Town の完全な hook マネージャではありません。 |
| `gt ready` | `bd ready` | これは city 中心より bead 中心のままです。 |
| `gt done` | 単一の直接の同等物なし | Gas City では通常これは bead クローズ、メタデータ遷移、convoy アクション、formula step です。 |
| `gt unsling` | 直接の同等物なし | 通常 `bd` と `gc sling` での bead 編集と再ルーティングに置き換えられます。 |
| `gt formula` | `gc formula list/show/cook`、`gc sling --formula`、`gc order` | `gc formula` は formula を管理 (list、show、cook)。`gc sling --formula` は wisp として dispatch。 |
| `gt mol` | `gc formula cook`、`bd mol ...` | `gc formula cook` は molecule を作成; `bd` は bead レベル操作を扱う。 |
| `gt mq` | 直接の汎用 `gc` コマンドなし | Gastown スタイルのマージキュー動作は汎用 SDK 名前空間ではなく pack と formula にあります。 |
| `gt gate` | `gc wait` | 永続的な wait が最も近い SDK 概念。 |
| `gt park` | `gc wait` | 同じ根底のアイデア: 依存や gate の周りで停止して resume。 |
| `gt resume` | `gc wait ready`、`gc session wake`、`gc mail check` | 古いアクションが parked wait か、sleep 中の session か、ハンドオフ/mail resume かによります。 |
| `gt synthesis` | 部分的: `gc converge`、formulas、convoys | 1 コマンドのパリティはありません。 |
| `gt orphans` | 直接の汎用コマンドなし | Gas City では通常 pack ロジックと witness/refinery formula と bead 検査です。 |
| `gt release` | 主に `bd` の状態編集 | 単一の `gc release` コマンドはありません。 |

### Sessions、roles、agent

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt agents` | `gc session` と `gc status` | Session 管理は Gas City で汎用的; Town 固有の agent スイッチャーではありません。 |
| `gt session` | `gc session` | 同じ広い考え、しかし polecat 固有ではありません。 |
| `gt crew` | `city.toml` の agent と `gc session` | Crew は pack 規約であり、ファーストクラスの SDK コマンドファミリーではありません。 |
| `gt polecat` | Gastown pack の `polecat` agent と `gc status` / `gc session` / `gc sling` | 専用のトップレベル SDK 名前空間はありません。 |
| `gt witness` | Gastown pack の `witness` agent と `gc session` / `gc status` | 専用のトップレベル SDK 名前空間はありません。 |
| `gt refinery` | Gastown pack の `refinery` agent と `gc session` / `gc status` | 専用のトップレベル SDK 名前空間はありません。 |
| `gt mayor` | Gastown pack の `mayor` agent と `gc session attach mayor` / `gc status` | 焼き付けられたコマンドファミリーではなく、設定された agent として管理されます。 |
| `gt deacon` | Gastown pack の `deacon` agent と `gc session`、`gc status`、controller 動作 | Gas City では deacon が行ったことの多くは controller/supervisor にあります。 |
| `gt boot` | Gastown pack の `boot` agent | 他の role agent と同じパターン。 |
| `gt dog` | 通常 `gc order`、ときどき `city.toml` のスケーラブル session 設定 | Dog ライクなヘルパーはしばしば exec order としてモデル化するほうが良いです。 |
| `gt role` | `gc config explain`、`gc session list`、prompt/設定検査 | Role はファーストクラスの SDK 概念ではありません。 |
| `gt callbacks` | 直接の同等物なし | Callback 動作はランタイム、hooks、waits、orders に折り込まれています。 |
| `gt cycle` | 直接の汎用コマンドなし | 最も近い同等物は tmux バインディングや pack 固有の session UX。 |
| `gt namepool` | 設定のみ (現在) | Gas City は設定で namepool ファイルをサポートしますが、トップレベル `gc namepool` コマンドは公開していません。 |
| `gt worktree` | `work_dir`、`pre_start`、`git worktree`、pack スクリプト | Worktree 動作は明示的な設定とスクリプト配線で、汎用的な `gc worktree` 名前空間ではありません。 |

### 通信と nudges

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt mail` | `gc mail` | ほぼ直接マッピング。 |
| `gt nudge` | `gc session nudge` または `gc nudge` | 特定のライブ session には `gc session nudge`、遅延配信制御には `gc nudge` を使います。 |
| `gt peek` | `gc session peek` | ほぼ直接マッピング。 |
| `gt broadcast` | 単一の直接の同等物なし | 通常はグループまたは複数の明示的ターゲットへの `gc mail send` としてモデル化されます。 |
| `gt notify` | 直接の同等物なし | 通知ポリシーはトップレベル SDK コマンドファミリーではありません。 |
| `gt dnd` | 直接の同等物なし | 最も近い動作は通常 mail やローカルワークフローポリシーにあります。 |
| `gt escalate` | 直接の同等物なし | エスカレーションは bead、mail、orders、pack 固有のワークフローでモデル化します。 |
| `gt whoami` | 直接の同等物なし | Identity は専用 CLI ではなく設定、session メタデータ、`GC_*` env で明示的です。 |

### Beads、events、診断

| `gt` | Gas City での最も近いもの | 備考 |
|---|---|---|
| `gt bead` | 主に `bd` | Bead CRUD は依然として主に bead ツールの仕事です。 |
| `gt cat` | 主に `bd` | 同じルール: bead コンテンツ検査は bead 中心。 |
| `gt show` | 主に `bd` | 詳細な bead 状態/コンテンツには bead ツールを使います。 |
| `gt close` | 主に `bd close` | 依然として bead 中心。 |
| `gt commit` | `git commit` | Gas City は Town のように commit をラップしません。 |
| `gt activity` | `gc event emit` と `gc events` | 同じ基本的なイベント/ロギング空間。 |
| `gt trail` | `gc events`、`gc session peek`、`gc session logs` | 1 コマンドのパリティはありません。 |
| `gt feed` | `gc events` | 最も近いライブシステムフィード。 |
| `gt log` | `gc events` または `gc supervisor logs` | イベント履歴かランタイムログかによります。 |
| `gt audit` | 部分的: `gc events`、`gc graph`、`bd` クエリ | 単一の audit 名前空間の同等物はありません。 |
| `gt checkpoint` | 直接の同等物なし | Session の永続性はユーザー向け checkpoint CLI ではなくランタイムと bead/session モデルにあります。 |
| `gt patrol` | 直接の同等物なし | Patrol 動作は通常 orders と formulas でモデル化されます。 |
| `gt migrate-agents` | `gc migration` | 同じ一般的な移行/アップグレードバケット。 |
| `gt prime` | `gc prime` | 直接マッピング。 |
| `gt account` | 直接の同等物なし | Provider アカウント管理は Gas City のコア CLI の外です。 |
| `gt shell` | 直接の同等物なし | Gas City は Town スタイルの shell 統合名前空間を出荷していません。 |
| `gt theme` | 直接の同等物なし | Pack スクリプトや tmux 設定が通常のパスです。 |
| `gt costs` | 直接の同等物なし | 今日対応するトップレベルのコスト会計コマンドはありません。 |
| `gt seance` | 直接の同等物なし | Gas City には resume と session メタデータがありますが、seance コマンドはありません。 |
| `gt thanks` | 直接の同等物なし | 対応するコマンドはありません。 |

### 実用的な翻訳ルール

`gt` コマンドがどこに行ったか分からない場合、次の順で問います。

1. ほぼ同じ名前で `gc` になっただけか?
2. それは本当に `bd` に留まるべき bead 操作か?
3. Gas City がその動作を設定、orders、waits、formulas、controller ロジックに移動したため、もう特別なコマンドではないか?

## 文字どおり移植すべきでないもの

これらの Gas Town の習慣は Gas City で不要な複雑さを生むことが多いです。

- 正確な `~/gt/...` ディレクトリツリー
- cwd 由来の identity
- SDK コードでの新しいハードコードされた role 名
- order で十分なときの plugin システム
- 本当は shell コマンドである作業のための特別なヘルパー agent
- label やメタデータで十分なときに beads の外で永続状態を複製する

最も一般的なアーキテクチャ的ミスは、Town の表面を import することで、Gas City のプリミティブで意図を再表現しないことです。

## ファストランプチェックリスト

すでに Gas Town を知っているなら、これが Gas City で効果的になる最短のパスです。

1. Nine Concepts Overview (`engdocs/architecture/nine-concepts`) を読む。
2. Config System docs (`engdocs/architecture/config`) を読む。
3. Orders (`engdocs/architecture/orders`) を読み、頭の中で「plugins」を「orders」に再マップする。
4. Formulas & Molecules (`engdocs/architecture/formulas`) を読み、formula は Gas City が解決するが、設定された beads バックエンドが実行することを覚えておく。
5. [examples/gastown/city.toml](https://github.com/gastownhall/gascity/blob/main/examples/gastown/city.toml) を最初に見て、次に [examples/gastown/packs/gastown/pack.toml](https://github.com/gastownhall/gascity/blob/main/examples/gastown/packs/gastown/pack.toml) を見る。City ファイルが通常のスタートポイントで、pack はその背後にある再利用可能なデフォルトを定義します。

これら 5 点を頭に入れておけば、Gas Town から Gas City へのランプはほとんど速く進みます。
