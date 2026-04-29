# 一貫性監査: ディレクトリ規約と TOML 構造 — 廃止済み

> **状態: 廃止済み。** すべての所見は正典の仕様ドキュメントに統合されました:
> - Orders → トップレベル `orders/`（doc-directory-conventions.md、doc-pack-v2.md、doc-loader-v2.md）
> - `[formulas].dir` → 固定規約（doc-directory-conventions.md、doc-pack-v2.md）
> - Pack/city のエージェントデフォルトは `[agent_defaults]` を使う（doc-agent-v2.md）
> - `overlay/`（単数形）→ 実装クリーンアップ
> - フォールバックエージェント → 議論不要（V2 でフォールバックは削除）
> - Doctor/commands → 規約ベースのディレクトリ（doc-commands.md、doc-directory-conventions.md）
>
> このファイルは歴史的な参照のためにのみ残されています。下記の歴史的な内容は、
> 廃止された監査が古い暫定エイリアスを教え続けないように、現行の `[agent_defaults]`
> 用語を使うよう軽く正規化されています。

# 元の監査（歴史的）

**GitHub Issue:** *(未提出)*

タイトル: `feat: Consistency audit — directory conventions and TOML structure`

ビッグロックの再設計（pack/city モデル [#360](https://github.com/gastownhall/gascity/issues/360)、エージェント定義 [#356](https://github.com/gastownhall/gascity/issues/356)）は別途トラックされます。このイシューは、#356 のエージェントを構ディレクトリとするモデルが採用されることを前提に、それ以外のすべてを一貫性の観点から監査します。

---BEGIN ISSUE---

## コンテキスト

Gas City には名前付き定義のための 3 つのモデルがあります:

1. **TOML ファイル内のテーブル** — 定義はインライン TOML（`[[named_session]]`、`[providers.claude]`）
2. **名前付きファイルのディレクトリ** — 定義ごとに 1 ファイル、名前はファイル名から（`formulas/pancakes.formula.toml`）
3. **ディレクトリのディレクトリ** — 複数の関連ファイルを持つ複雑なエンティティ（`agents/mayor/`）

エージェント再設計（#356）はエージェントをモデル 1 からモデル 3 に移します。この監査は他のすべての定義タイプをチェックします: 正しいモデルを使っているか、一貫しているか。

## 全インベントリ

Gas City のすべてのユーザー宣言可能な定義と、その現在のパターン:

### Singleton TOML ブロック (city.toml)

これらは単一インスタンスの設定です。純粋な TOML が正しいパターン — 問題なし。

| ブロック | 用途 |
|---|---|
| `[workspace]` | City メタデータ、デフォルトプロバイダ |
| `[daemon]` | パトロール間隔、再起動ポリシー、wisp GC |
| `[beads]` | Bead ストアプロバイダ |
| `[session]` | Session プロバイダ、タイムアウト、K8s/ACP サブ設定 |
| `[mail]` | Mail プロバイダ |
| `[events]` | Events プロバイダ |
| `[dolt]` | Dolt のホスト/ポート |
| `[api]` | API のポート、バインドアドレス |
| `[chat_sessions]` | アイドルタイムアウト |
| `[session_sleep]` | session クラスごとの sleep ポリシー |
| `[convergence]` | エージェント/総量ごとの最大 convergence |
| `[orders]` | スキップリスト、最大タイムアウト |
| `[agent_defaults]` | モデル、wake モード、オーバーレイデフォルト |
| `[formulas]` | Formula ディレクトリパス |

**1 つの不整合:** `[formulas].dir` は設定可能だが、他の規約ベースのディレクトリ（`scripts/`、`commands/`、`doctor/`）はすべて固定名で検出される。`formulas/` も固定規約にすべきか?

### Singleton TOML ブロック (pack.toml)

| ブロック | 用途 |
|---|---|
| `[pack]` | Pack 名、スキーマバージョン、要件 |
| `[global]` | Pack 全体の session_live コマンド |
| `[agent_defaults]` (v.next) | Pack 全体のエージェントデフォルト |

問題なし — これらはメタデータです。

### TOML の配列テーブルとして宣言されるコレクション

| 定義 | TOML | ファイルもある? | パターン |
|---|---|---|---|
| **Agents** | `[[agent]]` | prompt、overlay、namepool | ハイブリッド — **#356 で再設計** |
| **Named sessions** | `[[named_session]]` | なし | 純粋 TOML — そのまま OK、エージェントを名前で参照する |
| **Rigs** | `[[rigs]]` | 外部プロジェクトディレクトリ | ハイブリッド (TOML + パスバインディング) — **#360 で対処** |
| **Services** | `[[service]]` | `.gc/` のランタイム状態 | 純粋 TOML — そのまま OK |
| **Providers** | `[providers.<name>]` | なし | 純粋 TOML — そのまま OK |
| **Patches** | `[[patches.agent]]` など | `patches/` の任意のプロンプトファイル | ハイブリッド — **#356 で対処** |

### 規約ベースのディレクトリ

これらはディレクトリをスキャンして検出される。TOML 宣言は不要。

| ディレクトリ | ファイルパターン | アイデンティティの由来 | 一貫している? |
|---|---|---|---|
| `agents/` (v.next) | `<name>/prompt.md` | ディレクトリ名 | はい — #356 |
| `formulas/` | `<name>.formula.toml` | ファイル名 | **はい** |
| `orders/` | `<name>/order.toml` | **ディレクトリ名** | **いいえ — 下記参照** |
| `scripts/` | `<path>.sh` | パス | はい |
| `prompts/` | `<name>.md.tmpl` | ファイル名 | `agents/<name>/prompt.md` で置換中 |
| `overlays/` | ディレクトリツリー | N/A（丸ごとコピー） | エージェントごと + プロバイダごとで置換中 |
| `namepools/` | `<name>.txt` | ファイル名 | `agents/<name>/namepool.txt` で置換中 |
| `template-fragments/` (v.next) | `<name>.md.tmpl` | ファイル名 | はい — #356 |
| `skills/` (v.next) | `<name>/SKILL.md` | ディレクトリ名 | はい — #356 |
| `mcp/` (v.next) | `<name>.toml` | ファイル名 | はい — #356 |
| `commands/` | 下記参照 | | **ハイブリッド — 下記参照** |
| `doctor/` | 下記参照 | | **ハイブリッド — 下記参照** |

## 発見された問題

### 1. Orders: 構造もロケーションも誤り

**1 つの問題に 2 つ含まれる。**

Orders は `formulas/orders/<name>/order.toml` を使う — order ごとのサブディレクトリに 1 つのファイル。コードベースのどの order ディレクトリにも `order.toml` 以外は何もない。一方、formulas はフラットファイルを使う: `pancakes.formula.toml`。

加えて、orders は `formulas/orders/` またはトップレベルの `orders/` に置けるが、city のみが両方をサポートし、pack は `formulas/orders/` のみをサポートする。

Orders は formula ではない — formula を *参照* する。ディスパッチをスケジュールし、formula はワークフローを定義する。

**提案:**
- city と pack の両方でトップレベル `orders/` に標準化する
- フラットファイルを採用する: `orders/<name>.order.toml`（formula 規約と一致）
- `formulas/orders/` のネストは廃止する

### 2. Doctor チェックと commands: 当面は触らない

Doctor チェック (`[[doctor]]`) と commands (`[[commands]]`) はどちらもハイブリッド — TOML メタデータ + スクリプトファイル参照。両方とも純粋規約（モデル 2）にできるが、未解決の問いがある: 2 つの定義が異なる引数で同じスクリプトを共有できるか? 今日はそうしないが、パターンとしては妥当（例: `check-provider.sh --provider claude` vs. `check-provider.sh --provider codex`）。共有が必要なら TOML がそれを表現する正しい場所だ。

これらは今日モデル 1 で動いている。共有スクリプトの問いが解決したら再訪する。

### 4. `[formulas].dir` は仲間外れ

すべての規約ベースのディレクトリは固定名で検出される: `scripts/`、`commands/`、`doctor/`、`overlays/`、`agents/`（v.next）。しかし `formulas/` だけは `[formulas].dir` で設定可能なパスを持つ。

**提案:** `formulas/` を固定規約にする。誰かが formula を別の場所に置きたければ、それのために pack と imports がある。

### 5. Gastown にデッドな `overlay/` ディレクトリがある

Gastown pack には `overlay/`（単数）と `overlays/`（複数）の両方がある。pack.toml で参照されているのは `overlays/` のみ。`overlay/` ディレクトリは未使用に見えるが、`embed.go` 経由で埋め込まれている。

**提案:** Gastown pack から `overlay/` を削除し、embed.go を更新する。

### 6. フォールバックエージェント: プロンプト要件が不整合

Dolt pack のフォールバック dog エージェントは `prompt_template` を持たない。Maintenance pack のフォールバック dog は持っている。両方とも `fallback = true`。プロンプトが必須か、任意か、フォールバックエージェントで異なる意味を持つかが不明確。

**提案:** 明確化してドキュメント化する。フォールバックではプロンプトが任意なら（合理的 — 単に継承するかも）、それを明示する。

## サマリー

名前付き定義のための 3 つのモデル:

| モデル | 使う場面 | 例 |
|---|---|---|
| **1. TOML テーブル** | Singleton 設定、軽量宣言、スクリプトを共有しうる定義 | `[daemon]`、`[[named_session]]`、`[[doctor]]`、`[[commands]]` |
| **2. 名前付きファイルのディレクトリ** | 各定義が 1 ファイルになるコレクション | `formulas/<name>.formula.toml`、`orders/<name>.order.toml` |
| **3. ディレクトリのディレクトリ** | 複数の関連ファイルを持つ複雑なエンティティ | `agents/<name>/`、`skills/<name>/` |

**変更点:**
- Orders はモデル 3（order ごとのディレクトリ）からモデル 2（フラットファイル）に、`formulas/orders/` からトップレベル `orders/` に移動する
- Agents はモデル 1 からモデル 3 へ（#356）
- `[formulas].dir` は固定規約になる
- Doctor チェックと commands はモデル 1 のまま（共有スクリプトパターンが現れたら再訪する）

---END ISSUE---
