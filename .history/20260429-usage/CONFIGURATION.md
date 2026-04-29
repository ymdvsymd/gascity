# Gas City — 設定ガイド

**生成日:** 2026-04-29
**対象バージョン:** gascity v1.0.0+

Gas City の設定は 4 つの TOML ファイルに分かれている。それぞれ責務が明確に分離されており、混同すると意図と違う場所に書いてしまいやすい。設定変更は `gc reload` で controller に再読み込みさせる（restart 不要）。

---

## 設定ファイル

| スコープ | パス | 役割 | git 管理 |
|---------|------|------|---------|
| **pack** | `<city>/pack.toml` | 再利用可能な定義レイヤ。agent と named_session、import、コマンド定義 | ✅ |
| **city** | `<city>/city.toml` | このマシン上のデプロイ設定。workspace、rigs、provider 既定、override | ✅ |
| **agent** | `<city>/agents/<name>/agent.toml` | agent ごとの override（dir / provider / option_defaults / hooks） | ✅ |
| **site (machine-local)** | `<city>/.gc/site.toml` | マシン固有の binding。workspace_name と rig.path | ❌（gitignore） |

設計の意図は「ポータブルな部分（pack）と環境依存（city / site）を切り離す」こと。pack だけを別マシンや別チームへ持っていけば、その先で違う rig パスに紐付けて動かせる。

優先度は「下位レイヤを上位が override する」モデル:

```
pack (lowest) → city → city patches → rig imports → rig patches → agent.toml (highest)
```

`gc config explain` で「ある値がどのレイヤから来たか」を provenance 付きで確認できる。

---

## pack.toml の主要セクション

### `[pack]`

```toml
[pack]
name = "my-city"        # 必須
schema = 2              # 必須。Pack V2 が現行
version = "0.1.0"       # 任意。pack の semver
```

`schema = 2` 必須。古い `schema = 1` のスタイルはサポート終了。

### `[[agent]]`

エージェントを定義する。複数並べる。

```toml
[[agent]]
name = "mayor"                                       # 必須
prompt_template = "agents/mayor/prompt.template.md"  # プロンプトファイル
dir = "my-project"                                   # 任意。rig 名（rig スコープ）
provider = "codex"                                   # 任意。city デフォルトを上書き
nudge = "Check your hook and mail, then act."        # 任意
idle_timeout = "4h"                                  # 任意
work_dir = ".gc/worktrees/my-project/mayor"          # 任意。tmux の cwd
overlay_dir = "agents/mayor/overlay"                 # 任意。プロンプトに追加するファイル群
session_setup = ["tmux ..."]                         # 任意。tmux 起動後に実行する補助コマンド
session_setup_script = "scripts/setup.sh"            # 任意。スクリプトで一気に
pre_start = ["packs/.../worktree-setup.sh ..."]      # 任意。session 起動前に実行
install_agent_hooks = ["claude"]                     # 任意。agent ごとに hook を仕込む
append_fragments = ["safety-rules"]                  # 任意。global_fragments を上書き
```

`dir` を指定しないものは city スコープ（city 全体に存在する 1 体）。指定すると rig スコープになり、当該 rig 用の polecat / crew として複製される。

### `[[named_session]]`

「常駐 / オンデマンド / 手動起動」のいずれかを指定する。crew はここに `mode = "always"` で書く。

```toml
[[named_session]]
template = "mayor"     # agent 名
mode = "always"        # always / on_demand / manual
```

### `[imports.<name>]`

外部 pack を取り込む。

```toml
[imports.gastown]
source = "github.com/example/gastown-pack@main"

[imports.dolt]
source = "./packs/dolt"   # 相対パス

[imports.swarm]
source = "github.com/example/swarm-pack"
ref = "v1.2.0"            # 任意のタグ / branch / commit
```

### `[patches]` / `[[patches.agent]]`

import した pack の中身を書き換えたいときに使う。

```toml
[[patches.agent]]
name = "gastown.mayor"
provider = "codex"
idle_timeout = "2h"
```

### `[[doctor]]` / `[[command]]` / `[[global]]`

| セクション | 役割 |
|----------|------|
| `[[doctor]]` | `gc doctor` から呼ばれるカスタムチェック |
| `[[command]]` | pack 提供の `gc <name>` トップレベルコマンド |
| `[[global]]` | プロンプトテンプレートで `{{global "name"}}` から参照する fragment |

---

## city.toml の主要セクション

### `[workspace]`

```toml
[workspace]
name = "my-city"                                # 任意。表示名
provider = "claude"                             # 既定 provider
includes = ["packs/swarm"]                      # 任意。レガシー include。新規には imports を使う
install_agent_hooks = ["claude"]                # 全 agent 向け hook 設定
global_fragments = ["operational-awareness"]    # 全 prompt に注入される fragment
```

### `[[rigs]]`

city に属する rig を宣言。machine-local パスは `.gc/site.toml` に分離される。

```toml
[[rigs]]
name = "my-project"

[rigs.imports.dev_ops]
source = "./packs/dev-ops"

[[rigs.patches]]
agent = "dev_ops.tester"
provider = "codex"
```

### `[beads]`

bead の永続化先を切り替える。

```toml
[beads]
provider = "bd"           # bd（既定） / file / exec:<command>
```

`bd` は dolt + bd CLI、`file` は JSON ファイル、`exec` は任意のスクリプト。実験用には `file` がコンパクト。

### `[daemon]`

controller の挙動を調整する。

```toml
[daemon]
patrol_interval = "30s"     # tick 間隔
max_restarts = 5            # 同 session の連続再起動上限
restart_window = "1h"       # 上限のリセット時間
shutdown_timeout = "5s"     # 停止時の猶予
formula_v2 = true           # graph.v2 formula を有効化
```

### `[orders]`

order 共通の挙動。

```toml
[orders]
max_timeout = "120s"            # 全 order の上限
skip = ["nightly-bench"]        # 名前で除外

[[orders.overrides]]
name = "test-suite"
rig = "frontend"            # 特定 rig のみ
interval = "1m"
pool = "worker"
```

### `[chat_sessions]` / `[session_sleep]` / `[convergence]`

```toml
[chat_sessions]
timeout = "30m"             # チャット session の inactive 上限

[session_sleep]
idle_timeout = "1h"         # session が sleep に落ちるまで

[convergence]
default_max_iterations = 5
default_check_timeout = "30s"
```

### `[api]`

HTTP コントロールプレーン（dashboard が使う）。

```toml
[api]
enabled = true
listen = "127.0.0.1:8742"
auth_token_file = ".gc/api-token"
```

### `[mail]` / `[events]`

```toml
[mail]
retention_days = 30
default_from = "human"

[events]
retention_lines = 100000
critical_buffer = 1000
```

### `[dolt]`

managed dolt の挙動。`.gc/dolt-config.yaml` 経由で実体化される。

```toml
[dolt]
listener_backlog = 1024
connection_timeout = "30s"
log_level = "info"
```

### `[formulas]`

```toml
[formulas]
default_check_timeout = "30s"
default_attempts = 1
```

### `[[agent]]` / `[[named_session]]` / `[[patches.agent]]`

pack.toml と同じ構造を city.toml にも書ける。`agent.toml` で書くと per-agent override、`city.toml` で書くと city 全体の override。

---

## agent.toml の主要項目

`agents/<name>/agent.toml` は agent 単位の override。pack.toml の `[[agent]]` と同じフィールドが書ける。よく使う:

```toml
dir = "my-project"
provider = "codex"
nudge = "..."
idle_timeout = "4h"

[option_defaults]
model = "sonnet"
permission_mode = "plan"

[pool]
min = 1
max = 5
scale_check = "scripts/scale-check.sh"

install_agent_hooks = ["claude"]
work_dir = ".gc/worktrees/my-project/reviewer"
```

`[pool]` を書くとそのエージェントは pool として複数 session を持てる（polecat スケーリング）。`scale_check` は新規セッション需要を返すスクリプト（dolt v1.0.0 以降は「assigned 分は別管理。新規 session 需要のみを返す」契約）。

---

## .gc/site.toml の主要項目

マシン固有の binding。git に commit してはいけない（既定で gitignore されている）。

```toml
workspace_name = "my-city"

[[rig]]
name = "my-project"
path = "/Users/you/projects/my-project"

[[rig]]
name = "frontend"
path = "/Users/you/projects/frontend"
```

`gc rig add` がここを書く。pack や city.toml に書いた rig 名と、このマシン上の絶対パスを結びつける役目。

---

## 設定の階層と override 解決

```
1. pack defaults (lowest)
2. imported pack overlays
3. city patches    → [[patches.agent]] in pack.toml or city.toml
4. rig imports     → [rigs.imports.<name>] in city.toml
5. rig patches     → [[rigs.patches]] in city.toml
6. city.toml [[agent]] / [[named_session]]  (workspace defaults)
7. agents/<name>/agent.toml (highest)
```

「上位ほど勝つ」モデル。`gc config explain` で provenance を見ながら debug する。

---

## 環境変数

CLI と controller が読む主な環境変数を分類別に示す。`GC_*` のうち、ユーザが日常的に触るのは少なく、ほとんどは内部用。

### city / rig 解決

| 変数 | 役割 |
|------|------|
| `GC_CITY` / `GC_CITY_PATH` / `GC_CITY_ROOT` | city ディレクトリを明示 |
| `GC_RIG` | rig 名を明示 |
| `GC_DIR` | rig 配下のサブディレクトリを明示 |
| `GC_CITY_NAME` | city 名（hook が設定） |
| `GC_CITY_RUNTIME_DIR` | runtime state directory（既定: `<city>/.gc`） |

### beads / dolt provider

| 変数 | 役割 |
|------|------|
| `GC_BEADS` | `bd` / `file` / `exec:<cmd>` |
| `GC_BEADS_PREFIX` | bead ID プレフィックスを上書き |
| `GC_BEADS_SCOPE_ROOT` | bead store の scope root |
| `GC_DOLT_HOST` / `GC_DOLT_PORT` / `GC_DOLT_USER` / `GC_DOLT_PASSWORD` | managed dolt の接続情報 |
| `GC_DOLT_DATABASE` / `GC_DOLT_DATA_DIR` / `GC_DOLT_LOG_FILE` / `GC_DOLT_LOCK_FILE` | dolt 永続化先 |
| `GC_DOLT_REAL_BINARY` | dolt バイナリのパス |
| `GC_DOLT_LOGLEVEL` | `info` / `debug` 等 |
| `GC_DOLT_GC_THRESHOLD_BYTES` / `GC_DOLT_GC_DRY_RUN` / `GC_DOLT_GC_CALL_TIMEOUT_SECS` | managed dolt GC ポリシー |
| `GC_DOLT_CONFIG_FILE` / `GC_DOLT_STATE_FILE` | managed dolt の状態ファイル |
| `GC_DOLT_MANAGED_LOCAL` | managed dolt をローカルで起動するか |
| `GC_DOLT_CONCURRENT_START_READY_TIMEOUT_MS` | 起動の readiness 検出タイムアウト |

### session / agent

| 変数 | 役割 |
|------|------|
| `GC_AGENT` | hook 内で動いている agent 名 |
| `GC_ALIAS` | session alias |
| `GC_SESSION_ID` | session bead ID |
| `GC_TMUX_SOCKET` / `GC_TMUX_SESSION` | tmux ソケット/セッション名 |
| `GC_BIN` | gc バイナリのパス |
| `GC_DOCKER_IMAGE` | container runtime 用の image |
| `GC_DRAIN` / `GC_DRAIN_ACK` / `GC_DRAIN_REASON` / `GC_DRAIN_GENERATION` | drain プロトコル |
| `GC_BRANCH` | 作業 branch ヒント |
| `GC_CONTINUATION_EPOCH` | session 連続性のマーカー |

### control / supervisor

| 変数 | 役割 |
|------|------|
| `GC_CONTROL_TARGET` / `GC_CONTROL_LEGACY_TARGET` | controller 通信先 |
| `GC_BOOTSTRAP` | 初期化フェーズ識別 |
| `GC_DRIFT_RESTART` | drift 検出時の restart 強制 |
| `GC_CEILING_DIRECTORIES` | city 探索の上限ディレクトリ |
| `GC_CANONICAL_FILES_OWNED` | canonical state の所有を主張 |

### telemetry / events

| 変数 | 役割 |
|------|------|
| `GC_EVENTS` | event log のオーバーライド先 |
| `GC_OTEL_METRICS_URL` | OpenTelemetry metrics 送信先 |
| `GC_OTEL_LOGS_URL` | OpenTelemetry logs 送信先 |

最低限覚えるべきは `GC_BEADS=file`（簡易 backend）と `GC_CITY=<path>`（city 明示）の 2 つ。それ以外は debug 用途。

---

## 設定例

### 例 1: 最小構成（minimal）

`gc init --template minimal` で生成される構成:

```toml
# pack.toml
[pack]
name = "my-city"
schema = 2

[[agent]]
name = "mayor"
prompt_template = "agents/mayor/prompt.template.md"

[[named_session]]
template = "mayor"
mode = "always"
```

```toml
# city.toml
[workspace]
provider = "claude"
```

### 例 2: file backend を強制（dolt 不要）

```toml
# city.toml
[workspace]
provider = "claude"

[beads]
provider = "file"
```

### 例 3: rig ごとに provider を変える

```toml
# city.toml
[workspace]
provider = "claude"

[[rigs]]
name = "frontend"

[[rigs.patches]]
agent = "dev_ops.implementer"
provider = "codex"   # frontend だけ codex で実装させる

[[rigs]]
name = "backend"
# patches を書かなければ workspace.provider = claude が効く
```

### 例 4: gastown スタイル + 追加 crew

`examples/gastown/city.toml` を起点に、自分の crew を追加する例:

```toml
[workspace]
name = "gastown"
provider = "claude"
global_fragments = ["command-glossary", "operational-awareness"]

[imports.gastown]
source = "packs/gastown"

[daemon]
patrol_interval = "30s"
max_restarts = 5
restart_window = "1h"
shutdown_timeout = "5s"
formula_v2 = true

[[rigs]]
name = "myproject"

# 個別 crew member を追加
[[agent]]
name = "wolf"
dir = "myproject"
prompt_template = "packs/gastown/assets/prompts/crew.template.md"
nudge = "Check your hook and mail, then act accordingly."
idle_timeout = "4h"
```

---

## 関連ドキュメント

- [OVERVIEW.md](./OVERVIEW.md) — Nine Concepts と層構造
- [COMMANDS.md](./COMMANDS.md) — 設定値を参照する CLI コマンド
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — 設定が効かないときの診断
