# Gas City — コマンドリファレンス

**生成日:** 2026-04-29
**対象バージョン:** gascity v1.0.0+

`gc` はトップレベル CLI で、Cobra ベース。`gc help <command>` で各コマンドの詳細ヘルプが見られる。すべてのコマンドは「現在のディレクトリから city を発見する」ロジックを共通で持ち、`--city <path>` または `--rig <name|path>` で明示できる。`GC_CITY` / `GC_CITY_PATH` / `GC_RIG` 環境変数も同じ目的に使える。

---

## コマンド一覧（俯瞰）

| グループ | 主要コマンド | 用途 |
|---------|------------|------|
| ライフサイクル | `init` `start` `stop` `restart` `reload` `status` `suspend` `resume` | city の作成・起動・停止 |
| supervisor / service | `supervisor` `service` `register` `unregister` `cities` | マシン全体の管理プロセス |
| ワークルーティング | `sling` `hook` `prime` `handoff` `bd` `beads` `wait` | bead 作成と配送 |
| エージェント | `agent` `rig` `pack` `import` | 構成定義 |
| セッション | `session` | tmux の会話セッション管理 |
| 通信 | `mail` `nudge` `event` `events` | エージェント間メッセージ・イベント |
| フォーミュラ・オーダー | `formula` `order` `convoy` `wisp` `converge` | 宣言的ワークフロー |
| 設定・診断 | `config` `doctor` `gen-doc` `graph` `trace` | inspect・健全性 |
| 拡張・統合 | `skill` `mcp` `shell` `dashboard` `build-image` `runtime` `migrate` | provider 機能との接続 |

以下、グループごとに詳細。

---

## ライフサイクル管理

### `gc init [path]`

新しい city ディレクトリをスキャフォールドし、supervisor に登録して controller を起動する。

主要フラグ:

| フラグ | 説明 | デフォルト |
|-------|------|-----------|
| `--provider <name>` | 使う provider を非対話で指定（`claude`/`codex`/`gemini`/`cursor`/`copilot`/`amp`/`opencode`/`auggie`/`pi`/`omp`/`custom`） | 対話で選ぶ |
| `--template <name>` | `minimal` / `gastown` / `custom` のスキャフォールドテンプレート | `minimal` |
| `--name <name>` | city 名（省略時はディレクトリ名） | basename |

例:

```bash
gc init ~/my-city --provider claude
gc init ~/team-orchestrator --template gastown --provider codex
```

このコマンドは `pack.toml`、`city.toml`、`agents/<name>/prompt.template.md` などを書き、`gc register` 相当の登録と `gc start` 相当の起動まで自動で行う。

### `gc start [path]`

既存 city を supervisor 配下で起動する。`gc init` 直後はすでに走っているので普段は使わない。停止後に再度立ち上げるとき、または `gc stop` 後に再起動するとき。

### `gc stop [path]`

city のすべてのエージェントセッションを停止する。`.gc/site.toml` などの永続状態は保持される。

### `gc restart [path]`

city 全体を再起動。設定変更を controller に反映させたいが `gc reload` では足りない場合に使う。

`gc restart <rig-name>` の形で rig 内のエージェントだけ再起動することもできる。

### `gc reload [path]`

controller を再起動せずに `city.toml`・`pack.toml`・`agents/*/agent.toml` を再読み込みさせる。エラーがあれば古い構成を保持したまま失敗を報告する。日常的な構成変更はこちらで十分。

### `gc status [name]`

city または rig の状態を表示。controller の PID、登録済みエージェントの一覧、active / suspended セッション数を出す。

```bash
gc status              # cwd から自動解決
gc status my-project   # 特定の rig
```

### `gc suspend [path]` / `gc resume [path]`

city 全体を一時停止する。`suspend` 状態の city は controller が新しいセッションを起動せず、既存セッションも休止する。

### `gc register [path]` / `gc unregister [path]`

supervisor に city を登録 / 抹消する。`gc init` は内部で register を呼ぶので普段は意識しない。手動で city を移動したり、レジストリから外したいときに使う。

### `gc cities`

このマシンに登録されているすべての city の一覧を出す。`gc cities list` も同義。

---

## supervisor / service

### `gc supervisor`

マシン全体の supervisor デーモンを直接制御する。

| サブコマンド | 役割 |
|-------------|------|
| `gc supervisor start` | バックグラウンドで supervisor を起動 |
| `gc supervisor stop` | supervisor を停止 |
| `gc supervisor status` | 走っているかチェック |
| `gc supervisor reload` | 全 city を即座に再 reconcile |

### `gc service`

launchd / systemd に登録された supervisor サービスをホスト OS の用語で操作する。

| サブコマンド | 役割 |
|-------------|------|
| `gc service list` | 登録されているサービスを一覧 |
| `gc service doctor <name>` | サービスの詳細状態 |
| `gc service restart <name>` | サービスを restart |

`brew upgrade gascity` で `gc` バイナリを更新したあとは `gc service restart` するか、何かしらの `gc start` で service ファイルが再生成されるのを待つ。

---

## ワークルーティング

### `gc sling [target] <bead-or-formula-or-text>`

Gas City の中核コマンド。テキスト・既存 bead ID・formula 名のいずれかを引数にとり、対象エージェントへ仕事を投げる。

```bash
# テキストから新規 bead を作って投げる
gc sling my-project/claude "hello.py を作って"

# 既存 bead を再ルーティング
gc sling my-project/reviewer mp-ff9

# formula を wisp として発火（--formula が必須）
gc sling worker pancakes --formula

# city 内の cwd 解決を使う場合は target も省略可（既定エージェントへ）
cd ~/my-project
gc sling "Review hello.py and write review.md"
```

裏側では、bead 作成 → 対象エージェントの session 確保 / 起動 → wisp（または既存 molecule）の attach → convoy 作成 → nudge という一連の流れが走る。

主要フラグ:

| フラグ | 説明 |
|-------|------|
| `--formula` | 引数を formula 名として扱い、wisp を作る |
| `--var KEY=VALUE` | formula の変数を渡す |
| `--no-nudge` | 仕事を作るがエージェントを起こさない |
| `--rig <name>` | 明示的に rig を指定 |

### `gc hook [agent]`

エージェントが「次のターンで読むべき情報」を出力する内部コマンド。provider の `Stop` フックや `SessionStart` フックから呼び出される想定で、人間が直接打つことは少ないが、デバッグで使える。

```bash
gc hook --inject mayor   # mayor 用の hook 出力を表示
```

### `gc prime [agent-name]`

指定エージェントの「行動プロンプト」を標準出力に出す。pack で組まれている agent と、provider 既定の implicit agent（`claude`、`codex` など）の両方に対応。

```bash
gc prime                    # 既定の組み込みプロンプト
gc prime my-project/reviewer
```

`gc prime --strict` は fallback パスを禁止して explicit な失敗を強制するデバッグオプション。

### `gc handoff <subject> [message]`

handoff メールを mayor へ送り、controller が管理しているセッションを再起動する。長時間動かしっぱなしのエージェントを綺麗に交代させたいとき。

### `gc bd [bd-args...]`

`bd` を「正しい rig ディレクトリで」実行するラッパ。city / rig をまたぐと bead プレフィックスや scope が変わるため、`gc bd ready` のように打つと自動で適切な scope が選ばれる。

### `gc beads`

beads provider 自体の操作。

| サブコマンド | 役割 |
|-------------|------|
| `gc beads health` | provider が応答するか / dolt が起動しているかを診断 |

### `gc wait`

「依存が満たされるまで session を待たせる」durable wait の管理。

| サブコマンド | 役割 |
|-------------|------|
| `gc wait <session-id-or-alias>` | 依存待ちを登録 |
| `gc wait list` | 登録中の wait を一覧 |
| `gc wait inspect <wait-id>` | 詳細表示 |
| `gc wait cancel <wait-id>` | キャンセル |
| `gc wait ready <wait-id>` | 手動で ready 化（依存をスキップ） |

---

## エージェント管理

### `gc agent`

| サブコマンド | 役割 |
|-------------|------|
| `gc agent add --name <name>` | `agents/<name>/` をスキャフォールド |
| `gc agent suspend <name>` | reconciler に当該エージェントをスキップさせる |
| `gc agent resume <name>` | suspend を解除 |

`agent add` 時の主要フラグ:

| フラグ | 説明 |
|-------|------|
| `--name <name>` | エージェント名（必須） |
| `--dir <rig>` | 作業 rig（city スコープにする場合は省略） |
| `--provider <name>` | provider を上書き（city デフォルトと違う場合） |

スキャフォールド後、`agents/<name>/prompt.template.md` を編集してプロンプトを書き、必要に応じて `agents/<name>/agent.toml` で `provider` や `dir`、`option_defaults` を上書きする。

### `gc rig`

外部プロジェクトを city に紐付ける。

| サブコマンド | 役割 |
|-------------|------|
| `gc rig add <path>` | プロジェクトを rig として登録 |
| `gc rig list` | 登録済み rig を一覧 |
| `gc rig suspend [name]` | rig 配下のエージェントを停止 |
| `gc rig resume [name]` | suspend を解除 |
| `gc rig remove <name>` | rig をレジストリから外す |

`gc rig add` 時に `--name <alias>` を渡すとディレクトリ名と異なる名前を付けられる。

### `gc pack`

外部 pack ソースを city に取り込むための前段管理。

| サブコマンド | 役割 |
|-------------|------|
| `gc pack fetch` | 未取得 pack を clone、既存を update |
| `gc pack list` | キャッシュ済みリモート pack の状態を表示 |

リモート pack は通常、`pack.toml` の `[imports.<name>]` セクションで `source = "github.com/owner/repo@ref"` のように宣言する。

### `gc import`

宣言済みインポートの実体管理（pack の取り込み）。

| サブコマンド | 役割 |
|-------------|------|
| `gc import add <source>` | pack import を追加 |
| `gc import remove <name>` | 削除 |
| `gc import check` | インストール状態を検証 |
| `gc import install` | `pack.toml` と `packs.lock` を反映してインストール |
| `gc import upgrade [name]` | 制約内でアップグレード |
| `gc import list` | インポート済み pack を一覧 |
| `gc import why <name-or-source>` | なぜ import が存在するかを説明 |

---

## セッション管理

`gc session` 配下のサブコマンドで、tmux 上のライブセッションを操作する。

| サブコマンド | 役割 |
|-------------|------|
| `gc session new <template>` | template（agent 名）から新規セッションを作る |
| `gc session list` | アクティブ・suspend 中のセッションを一覧 |
| `gc session attach <id-or-alias>` | tmux にアタッチ（または resume） |
| `gc session peek <id-or-alias>` | 端末出力をスナップショットで読む |
| `gc session logs <id-or-alias>` | 会話履歴を表示。`--tail N` / `-f` 対応 |
| `gc session nudge <id-or-alias> <message...>` | 端末に文字列を流し込む |
| `gc session submit <id-or-alias> <message...>` | semantic delivery intent でメッセージ送信 |
| `gc session suspend <id-or-alias>` | 状態保存・リソース解放 |
| `gc session close <id-or-alias>` | 永久クローズ |
| `gc session rename <id-or-alias> <title>` | タイトル変更 |
| `gc session prune` | 古い suspend セッションを掃除 |
| `gc session kill <id-or-alias>` | runtime を強制終了（reconciler が再起動） |
| `gc session pin / wake / reset` | 個別の lifecycle 補助 |

`gc session logs --tail 0` で全履歴、`--tail 5` で末尾 5 件。`-f` は live tail。

`peek` は「画面に何が見えているか」、`logs` は「会話履歴」の意味で使い分ける。

---

## 通信

### `gc mail`

永続メッセージのやりとり。送信した瞬間 bead が立ち、受信側は次のターンで自動的に通知される（hook 経由）。

| サブコマンド | 役割 |
|-------------|------|
| `gc mail send [<to>] [<body>]` | 送信。`-s "subject" -m "body"` 形式も可 |
| `gc mail check [session]` | 未読数を返す。`--inject` で hook 出力 |
| `gc mail inbox [session]` | 未読メッセージ一覧 |
| `gc mail read <id>` | 既読化して内容表示 |
| `gc mail peek <id>` | 既読化せず内容表示 |
| `gc mail reply <id> [-s subject] [-m body]` | 返信 |
| `gc mail mark-read <id>` / `mark-unread <id>` | 状態マーク |
| `gc mail thread <thread-id>` | スレッド内全メッセージ |
| `gc mail count [session]` | 件数表示 |
| `gc mail delete <id>` | 削除（bead を close） |
| `gc mail archive <id>` | 既読化せずアーカイブ |

### `gc nudge`

queue されている nudge（端末への文字列送信）を管理する。`gc session nudge` がライブ送信なのに対し、こちらは deferred / dead-letter を扱う。

| サブコマンド | 役割 |
|-------------|------|
| `gc nudge status [session]` | キューと dead-letter を表示 |
| `gc nudge drain [session]` | キューにある nudge を配達 |
| `gc nudge poll [session]` | out-of-band 配達が必要な session に poll |

### `gc event` / `gc events`

| コマンド | 役割 |
|---------|------|
| `gc event emit <type>` | city の event ログにイベントを追加 |
| `gc events` | API 経由で event ストリームを取得 |
| `gc events --type <name> --since <time>` | フィルタ付き取得（フラグはコマンドの help 参照） |

---

## フォーミュラ・オーダー

### `gc formula`

| サブコマンド | 役割 |
|-------------|------|
| `gc formula list` | city 内の formula を一覧 |
| `gc formula show <name>` | コンパイル後のレシピを表示。`--var KEY=VALUE` で変数展開 |
| `gc formula cook <name>` | molecule（永続的 bead ツリー）として具体化 |

`gc formula cook` で作られた molecule は `bd show <root-id>` で見える。`gc sling` で個別ステップを別エージェントへ振ることもできる。`gc sling <target> <formula-name> --formula` の形にすると、cook ではなく wisp として一発投入になる。

### `gc order`

| サブコマンド | 役割 |
|-------------|------|
| `gc order list` | 有効な order を一覧 |
| `gc order show <name>` | 詳細表示 |
| `gc order check` | いま due な order を一覧 |
| `gc order run <name>` | trigger を無視して即座に発火 |
| `gc order history [name]` | 過去の発火履歴 |

`--rig <name>` で rig スコープの order を区別できる。同名 order が city / rig 双方にあるときは scope qualifier が必要。

### `gc convoy`

関連 bead をまとめる convoy。`gc sling --formula` は自動で convoy を作るので、明示的に作るのはスプリントや一連の PR を独立に追跡したいとき。

| サブコマンド | 役割 |
|-------------|------|
| `gc convoy create <name> [issue-ids...]` | 新規 convoy。`--owner <agent>` `--merge <strategy>` `--target <branch>` `--owned` |
| `gc convoy list` | 進捗付きで一覧 |
| `gc convoy status <id>` | 詳細状態 |
| `gc convoy add <id> <issue-id>` | 子 bead を追加 |
| `gc convoy target <id> <branch>` | target ブランチ更新 |
| `gc convoy close <id>` | 手動 close |
| `gc convoy land <id>` | owned convoy を terminate + cleanup |
| `gc convoy check` | 子が全 close の convoy を auto-close |
| `gc convoy stranded` | 仕事はあるが assignee がいない convoy を発見 |
| `gc convoy autoclose <bead-id>` | 兄弟が全 close なら親 convoy を自動 close |

`gc convoy create --owned --target integration/auth` のようにすると、`auto-close` をスキップして手動 land を待つ運用に切り替わる。

### `gc wisp` / `gc converge`

| コマンド | 役割 |
|---------|------|
| `gc wisp` | wisp の auto-close 補助。通常は controller が回す |
| `gc converge create` | bounded iterative refinement loop を作る |
| `gc converge status <bead-id>` | ループ状態 |
| `gc converge approve <bead-id>` | manual gate で承認 close |
| `gc converge iterate <bead-id>` | 強制で次回反復 |
| `gc converge stop / list / test-gate / retry` | ライフサイクル操作 |

`converge` は「何度か試行して結果を改善する」反復ワークフロー（コードレビュー → 修正 → 再レビュー）に向く。

---

## 設定・診断

### `gc config`

| サブコマンド | 役割 |
|-------------|------|
| `gc config show` | 解決後の city 設定を TOML で出力 |
| `gc config explain` | provenance 注釈付きで解決過程を表示 |

`gc config explain` は「この設定値はどの pack のどのファイルから来たか」が分かるので、override の挙動デバッグに有用。

### `gc doctor`

city の構造・config・依存・runtime を一括診断する。

| フラグ | 説明 |
|-------|------|
| `--verbose` | 詳細を出す |
| `--fix` | 自動修復を試みる |

新規環境のセットアップ後と、何かおかしいときの最初の一手。

### `gc gen-doc`

`docs/reference/cli.md` などのリファレンスを自動生成する内部ツール。コントリビューター向け。

### `gc graph <bead-ids|convoy-id...>`

bead 同士の依存グラフを表示。Mermaid 形式 / DOT 形式の出力に対応。

### `gc trace`

session reconciler のトレース情報を出す。`engdocs/contributors/reconciler-debugging.md` を参照しながらインシデント調査に使う。

---

## 拡張・統合

### `gc skill list`

city / pack / provider が公開している skill を一覧。Claude Code の skill 機能などとの統合点。

### `gc mcp`

provider 向けに projection された MCP サーバ設定を inspect する。

| サブコマンド | 役割 |
|-------------|------|
| `gc mcp list` | projected MCP servers を表示 |

### `gc shell`

ホスト shell に Gas City 統合フックを差し込む。

| サブコマンド | 役割 |
|-------------|------|
| `gc shell install [bash\|zsh\|fish]` | 統合スクリプトをインストール |
| `gc shell remove` | 削除 |
| `gc shell status` | 状態確認 |

### `gc dashboard serve`

Web ダッシュボード（Vite + React）を起動する。Supervisor、登録 city、active session を可視化する。

### `gc build-image [city-path]`

prebaked agent コンテナイメージをビルドする。Kubernetes runtime や CI で使うためのデプロイ補助。

### `gc runtime`

process-intrinsic な runtime 操作（drain / undrain / drain-check / drain-ack / request-restart）。エージェントが「自分を再起動してくれ」と controller にお願いするときに hook から呼ぶ。

| サブコマンド | 役割 |
|-------------|------|
| `gc runtime drain <name>` | session を graceful shutdown |
| `gc runtime undrain <name>` | drain を取り消す |
| `gc runtime drain-check [name]` | drain 中なら exit 0 |
| `gc runtime drain-ack [name]` | controller に「停めて良い」と通知 |
| `gc runtime request-restart` | 再起動を controller に依頼してブロック |

### `gc migrate`

スキーマ・データの移行スクリプト。`gc migrate run` のような形で、リリース間で必要な変換を行う。詳細は `gc migrate --help` を参照。

### `gc version`

バイナリのバージョンを表示。

---

## 内部・ローレベル

### `gc bd-store-bridge`、`gc dolt-config`、`gc dolt-state`、`gc internal`

dolt や bd を直接叩く内部コマンド群。普段の運用では使わない。`gc doctor` 経由で間接的に呼ばれる。インシデント対応で `gc dolt-state inspect-managed` などをマニュアル実行することがある。

### `gc supervisor reload`

すべての city を即座に reconcile。設定変更を一気に反映させたいときの強制発火。

---

## グローバルフラグ

| フラグ | 環境変数 | 役割 |
|-------|---------|------|
| `--city <path>` | `GC_CITY` / `GC_CITY_PATH` | city ディレクトリを明示 |
| `--rig <name\|path>` | `GC_RIG` | rig を明示 |
| `--help` / `-h` | — | コマンド毎のヘルプ |

`gc help <command>` で詳細ヘルプ、`gc <command> --help` でも同等。

---

## 関連ドキュメント

- [OVERVIEW.md](./OVERVIEW.md) — 概念モデルと全体像
- [USE-CASES.md](./USE-CASES.md) — このコマンドが実際に使われる文脈
- [CONFIGURATION.md](./CONFIGURATION.md) — 設定ファイルの構造
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — `gc doctor` の使い方を含む
