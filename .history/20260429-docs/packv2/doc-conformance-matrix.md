# Pack/City v2 適合マトリクス

このドキュメントは、整理された pack/city v.next ドキュメントを、現在の Pack/City v2 ロールアウト向けの実行可能な適合計画に変換します。

目的は完全な設計を再述することではありません。3 つの実用的な質問に答えることが目的です:

1. 今 CI でブロックすべき挙動は何か
2. 警告配管が整い次第スイートに加えるべき挙動は何か
3. ドキュメント化された意図ではあるが、まだリリースブロッカーとして扱うべきでない挙動は何か

## 権威の順序

スイートが何を主張すべきかを決めるとき、以下の順序でソースを使います:

1. [skew-analysis.md](skew-analysis.md) — 現在の望ましい状態に対するリリースゲーティング台帳
2. [migrating-to-pack-vnext.md](../guides/migrating-to-pack-vnext.md) — 移行ターゲットの挙動。ただし `skew-analysis.md` がそのサーフェスを欠落、延期、非ゲーティングとマークしていない場合に限る
3. [doc-agent-v2.md](doc-agent-v2.md) — プロンプト、テンプレート、フラグメント、プロンプト関連パッチの挙動
4. [doc-pack-v2.md](doc-pack-v2.md) と [doc-directory-conventions.md](doc-directory-conventions.md) — 補助的な設計とディレクトリのガイダンス。有用だが `skew-analysis.md` を覆すことは許されない
5. [TESTING.md](../../TESTING.md) — どのテストティアを使うか

設計ドキュメントが理想的な v.next サーフェスを記述しても、`skew-analysis.md` がそれを欠落または延期とマークしている場合は、CI ゲートではなくトラックされている作業としてマトリクスに残します。

## テストティアのマッピング

| ティア | 用途 |
|---|---|
| Unit / package テスト | discovery、merge order、パス解決、テンプレートのゲーティング、警告分類 |
| Testscript (`cmd/gc/testdata/*.txtar`) | ユーザーから見える移行、コマンドの成否、警告テキスト、書き換えられたレイアウト |
| Docsync | チュートリアル向けのコマンド例を testscript のカバレッジと同期させる |
| Integration | 実際の外部インフラが必要な場合のみ。pack/city スキーマの適合のデフォルトティアではない |

## 今 CI でゲートする

これらは決着していて、十分実装されているため、今 CI でブロックします。

| 領域 | 必須の挙動 | 推奨ティア | 現在の実装シーム |
|---|---|---|---|
| Root composition | `pack.toml` と `city.toml` を別々の成果物として扱うのではなく、一緒に composition される | Unit + testscript | `internal/config/compose.go` |
| Pack imports | `pack.toml` 内の `[imports.<binding>]` がインポートされたコンテンツを解決して composition する | Unit + testscript | `internal/config/pack.go` |
| Import target taxonomy | `source` が公開ロケータフィールドの唯一であり、`gc import add` は解決ターゲットを plain ディレクトリ、tagged git、untagged git、または無効な pack ターゲットに分類し、それに応じて `version` を合成する（`none`、デフォルト semver、または `sha:`） | Unit + testscript | `cmd/gc/cmd_import.go`、`internal/packman/resolve.go` |
| Rig imports | `city.toml` 内の `[rigs.imports.<binding>]` がターゲット rig に対して解決される | Unit + testscript | `internal/config/pack.go`、`internal/config/compose.go` |
| Agent discovery | `agents/<name>/` が `[[agent]]` を必要とせずにエージェントを作る | Unit | `internal/config/agent_discovery.go` |
| 現在のランタイムプロバイダの解決 | このリリース波で凍結することを許容する実装済みランタイムチェーンのみをゲートする: `agent.start_command` のエスケープハッチ、次に `agent.provider`、次に `workspace.provider`、次に自動検出。`workspace.start_command` はプロバイダなしのエスケープハッチのみ。`skew-analysis.md` の置換/廃止の方向はこの行の一部として扱わない。 | Unit | `internal/config/resolve.go` |
| Provider preset の merge と lookup | インポートされた pack のプロバイダは city のプロバイダマップに加算的に merge され、city/local プロバイダは名前衝突時にインポートされたものをシャドウし、プロバイダルックアップはサポートされている場合に city オーバーライドを builtin 上にレイヤーする | Unit | `internal/config/pack.go`、`internal/config/resolve.go` |
| Prompt の命名 | `prompt.md` は不活性な markdown であり、`prompt.template.md` はテンプレート処理を有効にする | Unit + testscript | `internal/config/agent_discovery.go`、`cmd/gc/prompt.go` |
| Overlay discovery | pack 全体の `overlay/` とエージェントローカルの `agents/<name>/overlay/` が規約で検出される | Unit | `internal/config/agent_discovery.go`、`internal/overlay/overlay.go` |
| Provider オーバーレイのフィルタ | 有効なプロバイダ用の `per-provider/<provider>/` の内容のみが実体化される | Unit | `internal/overlay/overlay.go` |
| Namepool 規約 | `agents/<name>/namepool.txt` が規約で検出される | Unit | `internal/config/agent_discovery.go` |
| Template fragments | `template-fragments/` と `agents/<name>/template-fragments/` が検出され、テンプレートプロンプトにレンダリングされる | Unit + testscript | `cmd/gc/prompt.go` |
| エージェントローカルの auto-append ブリッジ | エージェント上で宣言された `append_fragments` は `.template.md` プロンプトのみに適用され、プレーンな `.md` プロンプトには何もしない | Unit + testscript | `cmd/gc/prompt.go` |
| `[agent_defaults]` の auto-append ブリッジ | `[agent_defaults].append_fragments` は `.template.md` プロンプトのみで composition と auto-append を行う | Unit + testscript | `internal/config/compose.go`、`cmd/gc/prompt.go` |
| Agent defaults のレイヤー化 | `[agent_defaults]` は `pack.toml` と `city.toml` の両方で合法で、merge では city が勝つ。ランタイム継承は実装が今日実際に適用するフィールドのみゲートする | Unit | `internal/config/compose.go`、`internal/config/config.go` |
| 修飾されたパッチターゲティング | インポートされたエージェントは `[[patches.agent]]` で修飾名でターゲットできる | Unit | `internal/config/patch.go` |
| Patch prompt template のゲーティング | 明示的にパッチされた `prompt_template` のパスは、エージェントプロンプトファイルと同じ `.template.` ルールに従う: `.template.md` はレンダリングされ、プレーンな `.md` は不活性なまま | Unit | `internal/config/patch.go`、`cmd/gc/prompt.go` |
| Formula のファイル名 | PR2 の formula ファイルはフラットな `formulas/<name>.toml` ファイル名を現在の真実のサーフェスとして使う | Unit + testscript | `cmd/gc/system_formulas.go`、`internal/citylayout/layout.go` |
| Orders discovery | トップレベルの `orders/` の検出が規約で動く | Unit | `internal/orders/discovery.go` |
| Commands discovery | デフォルトの `commands/<name>/run.sh` 検出パスが動く。最終マニフェスト形状はゲーティング対象外のまま | Unit + testscript | `internal/config/command_discovery.go` |
| Doctor discovery | デフォルトの `doctor/<name>/run.sh` 検出パスが動く | Unit + testscript | `internal/config/doctor_discovery.go` |
| レガシー移行の書き換え | `gc doctor` がレガシーな Pack/City v1 利用をインベントリし、`gc doctor --fix` がエージェントディレクトリ、prompt/overlay/namepool の移動、import 指向の composition の安全な機械的書き換えを実行する。レガシーなリモート `workspace.includes` はハードブレークの移行イシューであり、ランタイム互換ターゲットではない。 | Testscript | `cmd/gc/doctor_v2_checks.go`、移行修正パスは未定 |
| Registration の命名 | `gc register --name` は選択されたマシンローカルのエイリアスを supervisor レジストリに保存し、`city.toml` を変更しない。素の `gc register` は有効な city アイデンティティ（site バインディング/レガシー設定/basename）を使い、その値を `city.toml` にバックフィルせずレジストリに保存する | Unit | `cmd/gc/cmd_register.go`、`cmd/gc/cmd_supervisor_city.go`、`internal/supervisor/registry.go` |

## 警告配管が整ったら CI に追加する

これらは現在の Pack/City v2 の望ましい状態の一部ですが、まだ完全に信頼できない deprecation または警告のインフラに依存します。テストを今書くのが有用なら書きますが、警告サーフェスがエンドツーエンドで実装されるまで CI を fail させてはいけません。

| 領域 | 期待される挙動 | 推奨ティア |
|---|---|---|
| レガシー `[[agent]]` | スキーマ 2 移行互換のために受け入れられるが、大きな警告を出す | Testscript |
| レガシー composition | `workspace.includes`、`workspace.default_rig_includes`、`rig.includes` は大きな警告を出してユーザーに imports へ誘導する | Testscript |
| レガシーなプロンプト注入 | `global_fragments`、`inject_fragments`、`inject_fragments_append` は `append_fragments` または明示的な `{{ template }}` への deprecation 警告を出す | Testscript + unit |
| レガシーなフォールバックモデル | `fallback` は大きな警告を出し、v.next のオーサリングサーフェスの一部ではない | Testscript |
| レガシーパスの配線 | レガシーなエージェント定義上の `prompt_template`、`overlay_dir`、`namepool` は移行向けフローで警告する | Testscript |
| Workspace のソフト deprecation | `workspace.provider`、`workspace.start_command`、`workspace.install_agent_hooks` は文書化された置換パスで警告する。`workspace.name` と `workspace.prefix` は現在 `gc doctor --fix` 経由でアクティブな site バインディング移行パスを持つ | Testscript |
| Formula のディレクトリパス | `[formulas].dir = "formulas"` はソフト警告。それ以外の値は拒否される | Unit + testscript |
| Rig オーバーライドの命名 | `rig.overrides` はソフト警告で受け入れられ、`rig.patches` を推奨する | Unit + testscript |
| Fragment 専用 include | トップレベルの `include` は fragment 専用のままで、`[imports]`、include ベースの composition、`pack.toml` 参照などの pack-composition 内容を拒否する | Unit + testscript |

## トラックするがまだゲートしない

これらは実装が明示的に欠けているか、信頼できるリリースゲートにするにはまだ未確定です。

| 領域 | 現状 | 今ゲーティングしない理由 |
|---|---|---|
| `[defaults.rig.imports.<binding>]` のローダーサポート | 文書化された意図、未実装 | 移行ツールが書き出すかもしれないが、ローダーがまだそれを尊重しない |
| ランタイムプロバイダ選択を駆動する `[agent_defaults] provider` | 移行ターゲットは文書化されているが、ランタイムの挙動はゲートに足るほど揃っていない | 現在の実装はランタイムデフォルトを `workspace.provider` / `ResolveProvider` 経由で解決する。将来のルールを今ロックすると false failure を生む |
| インポートされたプロンプトの置換のための `patches/` ディレクトリ規約 | v.next ドキュメントに文書化、未実装 | 現在の実装は完全なローダー検出のパッチファイルではなく明示的なパッチフィールドに依存する |
| Pack の `skills/` 検出 | 文書化、未実装 | 最初のスライスは現在の city pack のみで一覧表示のみ。インポートされた pack のカタログはあと |
| `mcp/` TOML 抽象化 | 文書化、未実装 | skills と同じ最初のスライススコープ: 現在の city pack のみ、最初は一覧表示のみ、プロバイダ投影はあと |
| `.gc/site.toml` の site バインディング分離 | ワークスペースアイデンティティ + `rig.path` で実装済み | ローダーは `.gc/site.toml` をオーバーレイし、コマンドは site バインディングを書き、`gc doctor --fix` はレガシーな `workspace.name`、`workspace.prefix`、`rig.path` を移行する。`rig.prefix` / `rig.suspended` はこのフェーズで `city.toml` に残る |
| Doctor 最終マニフェストの対称性/形状 | 仕様未確定 | discovery は今テストできるが、最初のスイートで最終マニフェスト形状を凍結すべきでない |
| Command 衝突ルールと最終 command/doctor マニフェスト形状 | 仕様未確定 | ドキュメントは凍結された契約言語ではなく "現在優先される方向" の言語をまだ使う |
| レガシークリーンアップサーフェス | 例えば古いドキュメント/サンプル内の古い `.order.` / `.formula.` 参照や `[workspace]` の解体 | ハンドオフのクリーンアップとして残すが、現在の波の出荷ゲートとして扱わない |

## Import ソースのカバレッジ

これらのケースは、`doc-packman.md` の POR を実行可能なまま保つために import 重点の unit/testscript バンドルでカバーされるべきです:

- plain ディレクトリ pack へのプレーンパス => import は `version` なしで書かれる
- semver タグ付きのローカル git リポジトリへのプレーンパス => import は git ベースのソースに正規化され、デフォルトの semver 制約を取得する
- semver タグなしのローカル git リポジトリへのプレーンパス => import は git ベースのソースに正規化され、`sha:<commit>` を取得する
- semver タグ付きの `file://` ローカル git リポジトリ => デフォルトの semver 制約
- semver タグなしの `file://` ローカル git リポジトリ => デフォルトの `sha:<commit>`
- 素の `github.com/org/repo` => git ベースの import 構文として扱われる
- 無効な pack ターゲット / スキーマ不一致 => ハードエラー、import は書かれない

## 最初のフィクスチャセット

スイートの実装をすぐに開始する場合、スコープを爆発させずに自信を実質的に高める最小セットがこれです。

### Testscript

- `cmd/gc/testdata/migrate-v2.txtar` を正典の移行回帰として拡張し続ける
- `pack.toml` の imports と rig スコープの imports のために `pack-v2-imports.txtar` を追加
- 警告配管が安定したら、レガシーフィールドの警告のために `pack-v2-warnings.txtar` を追加
- 違法な `[formulas].dir` 値や fragment 以外のトップレベル include などのハードエラーのために `pack-v2-errors.txtar` を追加
- 以下の import 重点フィクスチャを拡張:
  - プレーンパスのディレクトリ import
  - プレーンパスの git ベース import
  - 素の `github.com/...` import
  - 無効な pack ターゲット

### Unit テスト

- `internal/config/compose.go`: pack + city の merge order、フィールド配置、city 勝ちセマンティクス
- `cmd/gc/cmd_import.go`: import ターゲット分類とデフォルト version 合成
- `internal/packman/resolve.go`: semver vs `sha:` のデフォルトと git ソース解決
- `internal/config/resolve.go`: ランタイムプロバイダ解決とプロバイダプリセットのルックアップ/merge 挙動
- `internal/config/agent_discovery.go`: `agents/<name>/`、prompt 命名、overlay と namepool の規約
- `cmd/gc/prompt.go`: `.template.` ゲーティング、fragment ルックアップ、エージェントローカルと `[agent_defaults]` 両方のソースに対する `append_fragments` の挙動
- `internal/overlay/overlay.go`: プロバイダフィルタリングとオーバーレイのレイヤー化
- `internal/config/patch.go`: 修飾名のパッチターゲティングとパッチされた prompt-template パスの取り扱い
- `internal/config/doctor_discovery.go`: デフォルトの doctor 検出
- `cmd/gc/system_formulas.go`: PR2 の formula/order ファイル名の真実
- `internal/orders/discovery.go`: トップレベル orders 検出

## 終了基準

以下のすべてが真であるとき、スイートはプロダクト品質を駆動するのに十分強くなります:

1. **今 CI でゲートする** のすべての行に少なくとも 1 つの自動アサーションがある
2. **警告配管が整ったら CI に追加する** のすべての行に名前付きのテストオーナーがある（一時的にスキップされていても）
3. **トラックするがまだゲートしない** のすべての行が上に移動するか、リリース判断前に明示的に再確認される
4. 移行ドキュメントと testscript フィクスチャが、サンプルの変更に同期し続ける

それが「設計メモがある」と「本物の適合スイートロードマップがある」の境界線です。
