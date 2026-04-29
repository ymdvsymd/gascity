---
title: "Exec Beads Provider"
---

Gas City の bead store は work units (tasks、messages、molecules、convoys) のための汎用永続化基盤です。現在 2 つの provider があります: `bd` (Dolt 上で動く `bd` CLI を呼び出す) と `file` (チュートリアル用の JSON 永続化)。本ドキュメントでは 3 つ目の provider である `exec` を設計します。`exec` は各 store 操作を user 提供のスクリプトに委譲するもので、exec session provider と同じパターンです。

## 動機

`bd` provider は Gas City を特定の技術スタックに結合します: Dolt SQL データベースをラップする Go の `bd` CLI です。ユーザーは次のようなものを望むかもしれません。

- **beads_rust (`br`)** — SQLite + JSONL ハイブリッドで、性能特性が異なり、JVM/Dolt 依存がない
- **カスタムなバックアップ意味論** — S3 スナップショット、git コミット、その他の永続化戦略をトリガーする bead 操作
- **代替データベース** — PostgreSQL、SQLite、フラットファイル、または CLI 経由でアクセス可能な任意のストレージバックエンド

Exec beads provider は bead store をプラガブルな境界にします。レイヤリングが正しければ、ユーザーは設定の 1 行を変更するだけで Gas City を独自実装に向けられます。

## 現在のアーキテクチャ

### Store インターフェース (9 メソッド)

`internal/beads/beads.go` は SDK の bead 永続化契約である `Store` インターフェースを定義しています。

```go
type Store interface {
    Create(b Bead) (Bead, error)       // 新しい bead を永続化 → ID, Status, CreatedAt を埋める
    Get(id string) (Bead, error)       // ID で取得
    Update(id string, opts UpdateOpts) error  // フィールドを変更 (Description, ParentID, Labels)
    Close(id string) error             // status を "closed" に設定
    List() ([]Bead, error)             // すべての bead
    Ready() ([]Bead, error)            // すべての open な bead
    Children(parentID string) ([]Bead, error)  // 一致する ParentID を持つ bead
    SetMetadata(id, key, value string) error   // bead 上の key-value メタデータ
    MolCook(formula, title string, vars []string) (string, error)  // molecule をインスタンス化
}
```

### 3 つの実装

| Provider | バッキング | 利用箇所 |
|----------|---------|---------|
| `BdStore` | `bd` CLI → Dolt SQL | 本番 (デフォルト) |
| `FileStore` | JSON ファイル、MemStore をラップ | チュートリアル、軽量セットアップ |
| `MemStore` | インメモリマップ | ユニットテスト |

### BdStore 専用メソッド (Store インターフェース外)

BdStore は他のサブシステムが `*BdStore` を介して直接利用するメソッドを公開しています。

| メソッド | 利用箇所 | 目的 |
|--------|---------|---------|
| `Init(prefix)` | `cmd/gc/beads_provider_lifecycle.go` | `.beads/` データベースを初期化 |
| `ConfigSet(key, value)` | `cmd/gc/beads_provider_lifecycle.go` | bd 設定を設定 |
| `ListByLabel(label, limit)` | `cmd/gc/cmd_order.go` | label で bead を query (order history、cursors) |
| `Purge(beadsDir, dryRun)` | `cmd/gc/wisp_gc.go` と admin フロー | 閉じた ephemeral bead を削除 |
| `SetPurgeRunner(fn)` | テストのみ | テスト注入 |

### Provider の選択

`cmd/gc/providers.go` はランタイム時に bead store を選択します。

```go
func beadsProvider(cityPath string) string {
    if v := os.Getenv("GC_BEADS"); v != "" {
        return v
    }
    cfg, err := config.Load(fsys.OSFS{}, filepath.Join(cityPath, "city.toml"))
    if err == nil && cfg.Beads.Provider != "" {
        return cfg.Beads.Provider
    }
    return "bd"
}
```

優先順位: `GC_BEADS` 環境変数 → `city.toml [beads].provider` → `"bd"`。

設定:
```toml
[beads]
provider = "bd"    # または "file"、または "exec:/path/to/script"
```

## 変更が必要なもの

### 1. ListByLabel を Store インターフェースに昇格

`ListByLabel` は order サブシステムで次の用途に使われています。

- **Order history** — order に対するすべての wisp をリスト
- **Last run time** — order の最新 wisp を見つける
- **Event cursor** — order wisp 全体で最大の `seq:` label を見つける

これは bd 固有の機能ではなく、コアな query パターンです。任意の bead store は label でフィルタできます。インターフェースに含めるべきです。

```go
type Store interface {
    // ... 既存の 9 メソッド ...

    // ListByLabel は完全一致する label 文字列にマッチする bead を返します。
    // Limit は最大結果数を制御します (0 = 無制限)。結果は新しいものから順に並びます。
    ListByLabel(label string, limit int) ([]Bead, error)
}
```

**影響:** MemStore と FileStore に `ListByLabel` の実装が必要です (既存データに対する自明なフィルタ)。

### 2. Admin 操作は Store インターフェース外に保つ

`Init`、`ConfigSet`、`Purge`、`SetPurgeRunner` はライフサイクル/admin 操作であり、bead CRUD ではありません。これらは provider 実装に属し、SDK インターフェースには属しません。Exec beads provider はこれらをオプション操作として扱います (exit 2 = 未サポート)。

### 3. Exec beads provider を追加

新しいパッケージ: `internal/beads/exec/` (`internal/runtime/exec/` を踏襲)。

## Exec Beads プロトコル

### 呼び出し規約

```
<script> <operation> [args...]
```

データは stdin (JSON)、結果は stdout (JSON) です。Session exec provider のパターンと完全に同一です。

### 終了コード

| コード | 意味 |
|------|---------|
| 0 | 成功 |
| 1 | 失敗 (stderr にエラーメッセージ) |
| 2 | 不明な操作 (成功として扱われる — 前方互換) |

### 操作

#### コア Store 操作 (10 メソッド)

| 操作 | 呼び出し | Stdin | Stdout |
|-----------|-----------|-------|--------|
| `create` | `script create` | Bead JSON | Bead JSON (ID、status、created_at 付き) |
| `get` | `script get <id>` | — | Bead JSON |
| `update` | `script update <id>` | UpdateOpts JSON | — |
| `close` | `script close <id>` | — | — |
| `list` | `script list` | — | Bead JSON 配列 |
| `ready` | `script ready` | — | Bead JSON 配列 |
| `children` | `script children <parent-id>` | — | Bead JSON 配列 |
| `set-metadata` | `script set-metadata <id> <key>` | stdin の値 | — |
| `mol-cook` | `script mol-cook` | MolCookRequest JSON | root bead ID (プレーンテキスト) |
| `list-by-label` | `script list-by-label <label> <limit>` | — | Bead JSON 配列 |

#### Admin 操作 (オプション)

| 操作 | 呼び出し | Stdin | Stdout |
|-----------|-----------|-------|--------|
| `init` | `script init <dir> <prefix>` | — | — |
| `config-set` | `script config-set <key> <value>` | — | — |
| `purge` | `script purge <beads-dir>` | PurgeOpts JSON | PurgeResult JSON |

Admin 操作をサポートしないスクリプトは exit 2 (不明な操作) を返します。Gas City はこれを成功として扱います — admin 操作は `gc init` と `gc dolt sync` の間にのみ呼び出され、通常運用中は呼び出されません。

#### ライフサイクル操作 (オプション)

| 操作 | 呼び出し | Stdin | Stdout | 目的 |
|-----------|-----------|-------|--------|---------|
| `ensure-ready` | `script ensure-ready` | — | — | バッキング service を使用可能にする |
| `start` | `script start` | — | — | バックオフ/ヘルス追跡付きの拡張 start |
| `stop` | `script stop` | — | — | グレースフルシャットダウン付きの拡張 stop |
| `shutdown` | `script shutdown` | — | — | レガシーのグレースフル stop |
| `init` | `script init <dir> <prefix>` | — | — | ディレクトリの初回セットアップ |
| `health` | `script health` | — | — | provider ヘルスをチェック (probe のみ、副作用なし) |
| `recover` | `script recover` | — | — | 失敗後の stop、再起動、ヘルス検証 |
| `probe` | `script probe` | — | — | バッキング service が利用可能かをチェック (exit 0 = はい、2 = 未起動) |

これらの操作は `gc start` と `gc stop` から、bead store のバッキング service を管理するために呼び出されます — Docker Compose がデータベースコンテナを起動・停止するのと類似しています。これらは便宜上の操作であり、Store インターフェースの契約には含まれません。

終了コード意味論は他の操作と同じ規約に従います: 0 = 成功、1 = エラー、2 = 不要/未起動。バッキング service を持たないスクリプト (例: 組み込み SQLite データベースを使う `br`) は、すべてのライフサイクル操作で exit 2 を返します。

`health` 操作は読み取り専用 probe です — 復旧や再起動を試みては**いけません**。SDK はヘルス失敗時に別途 `recover` を呼び出します。`probe` 操作は `gc init` 中に bead 初期化を今進めるか `gc start` まで延期するかを決めるために使う軽量な可用性チェックです。

### ワイヤフォーマット

#### Bead JSON

ワイヤフォーマットは `beads.Bead` の JSON タグに一致します — `bd` がすでに生成しているのと同じ形です。

```json
{
  "id": "WP-42",
  "title": "digest wisp",
  "status": "open",
  "type": "task",
  "created_at": "2026-02-27T10:00:00Z",
  "assignee": "",
  "parent_id": "",
  "ref": "",
  "needs": [],
  "description": "",
  "labels": ["order-run:digest", "pool:dog"]
}
```

JSON から省略されたフィールドはゼロ値として扱われます。`create` 入力の `id` フィールドは無視されます (スクリプトが ID を割り当てます)。

#### Create リクエスト

```json
{
  "title": "my task",
  "type": "task",
  "labels": ["pool:dog"],
  "parent_id": "WP-1"
}
```

#### UpdateOpts JSON

```json
{
  "description": "updated description",
  "parent_id": "WP-1",
  "labels": ["new-label"]
}
```

Null/不在のフィールドは適用されません。`labels` は追加します (置換しません)。

#### MolCookRequest JSON

```json
{
  "formula": "mol-digest",
  "title": "digest run",
  "vars": ["key=value"]
}
```

Stdout: root bead ID をプレーンテキストで (例: `WP-42\n`)。

#### PurgeOpts JSON

```json
{
  "dry_run": true
}
```

#### PurgeResult JSON

```json
{
  "purged_count": 5
}
```

### 規約

- **ミューテーションは stdin に JSON** — description、title、label 値での shell クォート問題を回避
- **読み取りは stdout に JSON** — bd の `--json` 出力と一貫
- **シンプルな結果はプレーンテキスト** — `mol-cook` は ID のみを返す
- **結果なしは空配列** — `list`、`ready`、`children`、`list-by-label` は `[]` を返し、null は決して返さない
- **冪等な close** — すでに閉じた bead を閉じると exit 0 を返す
- **ErrNotFound → exit 1** — `get`、`update`、`close`、`set-metadata` で未知の ID は stderr にエラーを出して exit 1

### Status のマッピング

Gas City の `beads.Store` 表面は SDK の 3 状態語彙を使用します: `open`、`in_progress`、`closed`。より豊富な status 集合を公開するバックエンドは、それらをこの 3 つの値にマップする必要があります。例えば組み込みの `BdStore` は bd の `blocked`、`review`、`testing` 状態を `open` にマップします。空の status も `open` として扱われます。

## 実装計画

### パッケージ構造

```
internal/beads/exec/
├── exec.go          # Store インターフェースを実装する ExecStore
├── exec_test.go     # フェイクスクリプトでのユニットテスト
└── json.go          # ワイヤフォーマット型 (session/exec/json.go と同様)
```

### ExecStore

```go
// ExecStore は beads.Store を実装し、各操作を fork/exec 経由で
// user 提供のスクリプトに委譲します。
type ExecStore struct {
    script  string
    timeout time.Duration
}

func NewExecStore(script string) *ExecStore {
    return &ExecStore{script: script, timeout: 30 * time.Second}
}
```

`run` メソッドは `session/exec` のパターンを完全に踏襲します。

```go
func (s *ExecStore) run(stdinData []byte, args ...string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), s.timeout)
    defer cancel()
    cmd := exec.CommandContext(ctx, s.script, args...)
    cmd.WaitDelay = 2 * time.Second
    // ... session exec と同じ exit code 2 の処理 ...
}
```

### Provider 選択の更新

`cmd/gc/providers.go` に exec ケースを追加します。

```go
func newBeadStore(cityPath, cmdName string, stderr io.Writer) (beads.Store, int) {
    provider := beadsProvider(cityPath)
    if strings.HasPrefix(provider, "exec:") {
        script := strings.TrimPrefix(provider, "exec:")
        return beadsexec.NewExecStore(script), 0
    }
    switch provider {
    case "file":
        // ... 既存 ...
    default:
        // ... 既存の bd ...
    }
}
```

### 設定の更新

```toml
[beads]
provider = "exec:/path/to/gc-beads-br"
```

または環境変数経由で:

```bash
export GC_BEADS=exec:gc-beads-br
```

## 依存マップ: SDK プリミティブと Provider 操作

この表は、Gas City の各サブシステムが必要とする bead store 操作をマップします。これがレイヤリング検証の方法です: 「Uses」列のすべての操作が Store インターフェース (または exec プロトコル) にあれば、サブシステムはどの provider でも動作します。

| サブシステム | Layer | 利用 (Store インターフェース) | 利用 (*BdStore のみ) |
|-----------|-------|----------------------|---------------------|
| Dispatch (sling) | L3 | Create, Get, Update, Close, MolCook | — |
| Task loop | L2 | Ready, Get, Update, Close | — |
| Molecules | L2 | Create, Children, Update, Close, MolCook | — |
| Messaging | L2 | Create (type=message), List | — |
| Order check | L3 | — | ListByLabel (→ 昇格) |
| Order run | L3 | MolCook | ListByLabel (→ 昇格) |
| Order history | L3 | — | ListByLabel (→ 昇格) |
| Health patrol | L2 | Ready, SetMetadata | — |
| Convoy | L3 | Create, Children, Close, Update | — |
| Rig init | L0 | — | Init, ConfigSet |
| Dolt sync | L0 | — | Purge |
| Event cursor | L3 | — | ListByLabel (→ 昇格) |

**ListByLabel を昇格した後:** `Init`、`ConfigSet`、`Purge` のみが Store インターフェース外に残ります。これらはすべて `gc init` と `gc dolt sync` 中に呼び出される admin/ライフサイクル操作で、通常の agent 作業ループ中ではありません。Exec プロトコルはこれらをオプション操作として扱います (exit 2)。

## beads_rust (br) ギャップ分析

[beads_rust](https://github.com/Dicklesworthstone/beads_rust) は SQLite + JSONL を使った beads コンセプトの Rust 再実装です。Gas City の要件にどうマップするかを示します。

### サポート済み (直接マッピング)

| Store メソッド | br コマンド | 備考 |
|-------------|------------|-------|
| `Create` | `br create --json <title>` | `--type`、`--label` あり |
| `Get` | `br show --json <id>` | JSON を返す |
| `Update` | `br update --json <id>` | `--description`、`--label` あり |
| `Close` | `br close --json <id>` | 直接マッピング |
| `List` | `br list --json` | `--limit`、`--all` あり |
| `Ready` | `br ready --json` | open な bead |
| `ListByLabel` | `br list --json --label=X` | `--label` フィルタあり |

### ギャップ (スクリプトでブリッジ必要)

| Store メソッド | ギャップ | 回避策 |
|-------------|-----|------------|
| `Children(parentID)` | create で `--parent` がない | スクリプトが parent→child を sidecar または label で追跡 |
| `SetMetadata(id, key, value)` | `--set-metadata` がない | スクリプトが label (`meta:key=value`) または sidecar ファイルを使用 |
| `MolCook(formula, title, vars)` | molecule の概念がない | スクリプトが formula TOML から root bead + step bead を作成 |

### Store インターフェースで不要

| br 機能 | 関連性 |
|-----------|-----------|
| `br comment` | Store インターフェースになし — 将来の拡張になる可能性 |
| `br search` | Store インターフェースになし — 検索は List + フィルタで実施 |
| `br dep-tree` | molecule に興味深いが必須ではない |
| `br blocked` | 依存追跡を伴う Ready のサブセット |
| `br priority` | Gas City の bead モデルにない |

### 実現可能性評価

`br` をラップする `gc-beads-br` スクリプトは、**基本的な bead CRUD** に対しては実現可能です (10 操作中 7 つが直接マップ)。3 つのギャップ (Children、SetMetadata、MolCook) はスクリプトがブリッジロジックを実装する必要があります。

- **Children**: `br list --label=parent:<id>` を使用 (スクリプトが create 時に parent label を付ける)
- **SetMetadata**: `br update --label=meta:key=value` を使用 (スクリプト規約)
- **MolCook**: formula TOML をパースし、root と step bead を作成し、parent リンクを配線。これが最も困難なギャップです — スクリプトが Gas City の formula フォーマットを理解する必要があります。

より現実的なアプローチ: `MolCook` を Gas City 内の Go で実装し (すでに formula TOML を知っている)、Store インターフェースに対する `Create` + `Update` 呼び出しに分解する。これにより MolCook はスクリプトが実装すべきプリミティブではなく、**合成された操作**になります。

## 設計判断: MolCook を合成にするかプリミティブにするか

**Option A: MolCook を exec プロトコルのプリミティブとする。**
スクリプトが formulas を理解し、molecule bead ツリーを作成する必要があります。bd には `bd mol cook` があるので簡単ですが、カスタムバックエンドには困難です。

**Option B: MolCook を Go で Create + Update から合成する。**
Gas City が formula TOML を読み、`Create` で root bead を作成、`Create` で ParentID 付きの step bead を作成、`Update` で依存関係を配線します。スクリプトは CRUD プリミティブのみが必要です。

**推奨: Option B。** MolCook は*メカニズム* (Layer 2) であり、*プリミティブ*ではありません。Task Store 操作 + Config パースから合成されます。すべてのバックエンドスクリプトに formula 知識を押し込むのは Bitter Lesson に違反します — SDK が合成を扱い、スクリプトはストレージを扱うべきです。

これにより Store インターフェースは次のようになります。

```go
type Store interface {
    Create(b Bead) (Bead, error)
    Get(id string) (Bead, error)
    Update(id string, opts UpdateOpts) error
    Close(id string) error
    List() ([]Bead, error)
    Ready() ([]Bead, error)
    Children(parentID string) ([]Bead, error)
    SetMetadata(id, key, value string) error
    ListByLabel(label string, limit int) ([]Bead, error)
    MolCook(formula, title string, vars []string) (string, error)  // exec では内部で合成
}
```

Exec provider では、`MolCook` は ExecStore 自体が独自の `Create` と `Update` メソッド + formula パースを使って Go で実装します。BdStore は引き続き `bd mol cook` に委譲します。FileStore/MemStore は独自の Go 実装を持ちます。

## 移行パス

### Phase 1: インターフェース昇格 (本 PR)
1. Store に `ListByLabel(label string, limit int) ([]Bead, error)` を追加
2. MemStore と FileStore に実装 (既存データのフィルタ)
3. `cmd/gc/cmd_order.go` の関数を `*BdStore` から `Store` に変更

### Phase 2: Exec Provider
1. `internal/beads/exec/` パッケージを作成
2. すべての Store インターフェースメソッドを持つ ExecStore を実装
3. `beadsProvider()` に `exec:` プレフィックス処理を追加
4. プロトコルドキュメントを記述

### Phase 3: MolCook 分解
1. `bd mol cook` の formula→bead-tree ロジックを Go に抽出
2. Create + Update を使う合成された MolCook を ExecStore に実装
3. オプションで FileStore/MemStore にも合成された MolCook を追加

### Phase 4: リファレンススクリプト
1. beads_rust をラップする `gc-beads-br` スクリプトを記述
2. すべての Gas City 操作がエンドツーエンドで動作することを検証
3. ギャップと回避策をドキュメント化

## 比較: Session vs. Beads Exec パターン

| 観点 | Session Exec | Beads Exec |
|--------|-------------|------------|
| インターフェース | `runtime.Provider` (14+ メソッド) | `beads.Store` (10 メソッド) |
| データフォーマット | 混合 (start に JSON、その他はテキスト) | すべてのミューテーションと読み取りに JSON |
| 選択 | `GC_SESSION=exec:<script>` | `GC_BEADS=exec:<script>` |
| 設定 | N/A (環境変数のみ) | `[beads] provider = "exec:..."` |
| 前方互換 | Exit 2 = 不明な操作 | Exit 2 = 不明な操作 |
| ワイヤ型 | `startConfig` (安定したサブセット) | `beads.Bead` JSON タグ (安定) |
| タイムアウト | 30s | 30s |
| 合成操作 | なし (すべてプリミティブ) | MolCook (Create+Update から合成) |

## オープンクエスチョン

1. **`Children` は label 規約を使うべきか、それともファーストクラスの parent フィールドを使うべきか?** Label (`parent:<id>`) を使えばスクリプトはネイティブな parent サポートが不要です。しかし `bd` はネイティブな parent サポートを持っています。決定: ワイヤフォーマットで ParentID をファーストクラスのフィールドとして残し、ネイティブにサポートしないスクリプトは内部で label を使う。

2. **`ListByLabel` は複数 label (AND) をサポートすべきか?** 現在の BdStore は単一 label のみサポート。当面はシンプルに保つ — 単一 label。複数 label の query は単一 label の結果から合成可能。

3. **Exec provider の Purge 意味論。** Purge は dolt 固有 (Dolt データベースから閉じた ephemeral bead を削除する) です。Exec provider では委譲すべきか合成すべきか? 推奨: オプションとして委譲 (exit 2 = no-op)。スクリプトは独自のクリーンアップ戦略を実装できます。

## 出荷済みスクリプト

メンテナンスされている実装は `contrib/beads-scripts/` を参照してください。

- **gc-beads-br** — beads_rust (`br`) バックエンド。`br` CLI を SQLite + JSONL バッキングでラップします。依存: `br`、`jq`、`bash`。
- **gc-beads-k8s** — Kubernetes バックエンド。`kubectl exec` 経由で軽量な「beads runner」pod 内で `bd` を実行します。Pod はクラスタ内の StatefulSet として実行される Dolt に接続します。依存: `kubectl`、`jq`、`bash`。
