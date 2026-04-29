---
title: "Exec Session Provider"
---

Gas City の exec session プロバイダは、各 `runtime.Provider` の操作をユーザーが提供するスクリプトに委譲します。これにより、Go コードを書かずに任意のターミナルマルチプレクサやプロセスマネージャを session バックエンドとして使えます。

## 使い方

`GC_SESSION` 環境変数を `exec:<script>` に設定します:

```bash
# 絶対パス
export GC_SESSION=exec:/path/to/gc-session-screen

# PATH ルックアップ
export GC_SESSION=exec:gc-session-screen
```

## 呼び出し規約

スクリプトは操作名を最初の引数として受け取ります:

```
<script> <operation> <session-name> [args...]
```

シェル経由ではなく、スクリプトを直接 exec します。

## 終了コード

| コード | 意味 |
|------|---------|
| 0 | 成功 |
| 1 | 失敗（stderr にエラーメッセージを含む） |
| 2 | 不明な操作（成功として扱う — 前方互換性） |

終了コード 2 は前方互換性のメカニズムです。Gas City が将来新しい操作を追加すると、古いスクリプトは exit 2 を返し、プロバイダはこれを no-op の成功として扱います。スクリプトは関心のある操作のみを実装すれば十分です。

## 操作

| 操作 | 呼び出し | 標準入力 | 標準出力 |
|-----------|-----------|-------|--------|
| `start` | `script start <name>` | JSON config | — |
| `stop` | `script stop <name>` | — | — |
| `interrupt` | `script interrupt <name>` | — | — |
| `is-running` | `script is-running <name>` | — | `true` または `false` |
| `attach` | `script attach <name>` | tty パススルー | tty パススルー |
| `process-alive` | `script process-alive <name>` | プロセス名（1 行に 1 つ） | `true` または `false` |
| `nudge` | `script nudge <name>` | メッセージテキスト | — |
| `set-meta` | `script set-meta <name> <key>` | 値を stdin で | — |
| `get-meta` | `script get-meta <name> <key>` | — | 値（空 = 未設定） |
| `remove-meta` | `script remove-meta <name> <key>` | — | — |
| `peek` | `script peek <name> <lines>` | — | キャプチャされたテキスト |
| `list-running` | `script list-running <prefix>` | — | 1 行に 1 つの名前 |
| `get-last-activity` | `script get-last-activity <name>` | — | RFC3339 または空 |

### Start Config（stdin の JSON）

`start` 操作は stdin で JSON オブジェクトを受け取ります:

```json
{
  "work_dir": "/path/to/working/directory",
  "command": "claude --dangerously-skip-permissions",
  "env": {"GC_AGENT": "mayor", "GC_CITY": "/home/user/bright-lights"},
  "process_names": ["claude", "node"],
  "nudge": "initial prompt text",
  "pre_start": ["mkdir -p /workspace", "git clone repo /workspace"]
}
```

すべてのフィールドは任意です（空のときは省略されます）。

### スタートアップヒント

JSON config には、tmux プロバイダがマルチステップの起動オーケストレーションに使うフィールドが含まれます。exec プロバイダ自体は fire-and-forget で、`script start` を呼んだらすぐ戻ります。スクリプトはこれらのヒントを処理してもよいし、無視してもかまいません:

- **`process_names`** — tmux アダプタは、エージェントを「起動完了」とみなす前に、これらのプロセス名が session のプロセスツリーに現れるのをポーリングします（30 秒タイムアウト）。スクリプトは session 作成後にバックエンドのプロセスツリーをポーリングして実装するか、subprocess プロバイダのように fire-and-forget で無視できます。

- **`nudge`** — エージェントが ready になった後に tmux アダプタが session に入力するテキスト。インタラクティブ入力をサポートするスクリプトは、これを `start` 内で処理する（session 作成後にテキストを入力する）か、`start` 戻り後に gc が呼ぶ別個の `nudge` 操作に任せることができます。

- **`pre_start`** — session が作成される **前** にターゲットファイルシステムで実行するシェルコマンドの配列。エージェント起動前に存在しなければならないディレクトリ準備、worktree 作成、その他のセットアップに使います。スクリプトは、tmux session を作成する前にターゲット環境で各コマンドを実行すべきです。非致命的: コマンドが失敗したら stderr に警告するが、start を中断しない。

- **`session_setup`** — session が作成され ready になった後、戻る前にターゲットファイルシステムで実行するシェルコマンドの配列。スクリプトは session 環境内で各コマンドを実行すべきです（例: K8s では `kubectl exec -- sh -c '<cmd>'`、Docker では `docker exec -- sh -c '<cmd>'`、ローカルプロバイダでは素の `sh -c '<cmd>'`）。非致命的: コマンドが失敗したら stderr に警告するが、start を中断しない。

- **`session_setup_script`** — `session_setup` コマンドの後に実行されるコントローラファイルシステム上のスクリプトへのパス。リモートプロバイダ（K8s、Docker）の場合、ローカルでファイルを読み、その内容を session にパイプします（例: `kubectl exec -i -- sh < script`）。ローカルプロバイダの場合は `sh -c` で直接実行します。`session_setup` と同様に非致命的です。

JSON に含まれ **ない** フィールド（gc 内部で、exec プロトコルの一部ではない）:

- `ready_prompt_prefix` — 準備完了検出のためのプロンプトプレフィックス（gc は `start` 戻り後に `peek` でポーリングする）
- `ready_delay_ms` — 固定遅延フォールバック（gc は `start` 戻り後にスリープする）
- `emits_permission_warning` — bypass-permissions ダイアログの処理
- `fingerprint_extra` — config 変更検出メタデータ

区別: 準備完了ポーリングと遅延は *呼び出し元* の責任です。session セットアップコマンドは *スクリプト* の責任です — それらはコントローラではなくターゲットファイルシステムで実行されます。

### 規約

- **stdin で値を渡す**: `set-meta`、`nudge`、`start` はシェルクォーティングと引数長制限を避けるため、stdin でデータを渡します。
- **stdout で結果を返す**: `is-running`、`process-alive` は `true`/`false` を返します。`get-meta` は値を返すか、未設定なら空を返します。`list-running` は 1 行に 1 つの名前を返します。
- **冪等な stop**: `stop` は session が存在しなくても成功（exit 0）しなければなりません。
- **best-effort な interrupt/nudge**: session が存在しなくても 0 を返します。
- **空 = サポート外**: `get-last-activity` が空の stdout を返すのは、バックエンドがアクティビティ追跡をサポートしないことを意味します（Go ではゼロ時刻）。

## 独自スクリプトの作成

1. テンプレートとして `contrib/session-scripts/gc-session-screen` から始めます。
2. バックエンドがサポートする操作を実装します。
3. サポートしない操作には exit 2 を返します。
4. `GC_SESSION=exec:./your-script gc start <city>` でテストします。

### 最小スクリプト（start/stop/is-running のみ）

```bash
#!/bin/sh
op="$1"
name="$2"
case "$op" in
  start)     cat > /dev/null; my-mux new "$name" ;;
  stop)      my-mux kill "$name" 2>/dev/null; exit 0 ;;
  is-running) my-mux list | grep -q "^${name}$" && echo true || echo false ;;
  *)         exit 2 ;;
esac
```

## 環境変数

スクリプトは `GC_EXEC_STATE_DIR`（設定されていれば）をサイドカー状態ファイル（メタデータ、ラッパー）のディレクトリとして使えます。設定されていない場合、スクリプトは `$TMPDIR` または `/tmp` 配下の合理的なデフォルトを使うべきです。

## 同梱スクリプト

メンテナンスされている実装については `contrib/session-scripts/` を参照してください:

- **gc-session-screen** — GNU screen バックエンド。依存関係: `screen`、`jq`、`bash`。
