# `gc reload` 設計

ステータス: GitHub issue `#787` 向けの作業中設計

## 課題

Gas City には、city を再起動せずに現在の city に有効な config を再読み込みさせる、ユーザー向けのコマンドがありません。ファイル監視が config や pack の編集を取りこぼした場合、ドキュメント化された唯一の復旧手段は `gc restart` ですが、これはアクティブなセッションを破棄し、進行中の作業を中断させます。

## ゴール

- 現在の city に対するユーザー向け `gc reload [path]` コマンドを追加する。
- standalone と supervisor 管理の両方の city で動作させる。
- 第二の reload 実装を導入するのではなく、既存の config-dirty reload パスを再利用する。
- ユーザーが reload tick が 1 回正常に完了したかを把握できるよう、デフォルトを同期動作にする。
- config 適用後の既存のランタイムセマンティクス（通常の config drift ルールによるセッションごとの再起動を含む）を維持する。
- CLI の挙動、テスト、トレースデータが安定するよう、構造化された結果と警告を公開する。

## 非ゴール

- トップレベルの `gc poke` は追加しない。
- `POST /v0/city/reload` のような新しい HTTP API ミューテーションは追加しない。
- reload 時のリモート pack フェッチによる lockfile 更新は行わない。
- 適用後のランタイム副作用に対する特別なトランザクショナルロールバックは行わない。
- この変更で新しいトラブルシューティングガイドは追加しない。

## ユーザー向け契約

### コマンド

```text
gc reload [path] [--async] [--timeout <duration>]
```

- `[path]` は省略可能で、既存の city コマンドの解決に従います。
- `--async` は、現在の city コントローラが reload リクエストを受理した時点でリターンします。
- `--timeout` は同期モードのみに適用され、厳密に正の値である必要があり、デフォルトは `5m` です。
- `--async --timeout ...` は `--timeout` が明示的に設定されている場合は不正です。実装には Cobra のフラグ変更検出を使用するため、デフォルトの `5m` のせいで素の `gc reload --async` が不正にはなりません。

### スコープ

- `gc reload` は厳密に 1 つの city（解決された現在の city）を対象とします。
- standalone で動作している city でも supervisor 配下の city でも動作します。両方のトポロジが同じ city ごとのコントローラソケットを公開しているからです。
- これはライブのランタイム操作です。city コントローラが動作していない場合、コマンドは失敗します。

### 成功と失敗のセマンティクス

同期モードは、最初の reload 処理 tick のみを待ちます。

| Outcome | 終了コード | Stdout/Stderr の契約 |
| --- | --- | --- |
| `applied` | `0` | stdout: `Config reloaded: ... (rev <short>)` |
| `no_change` | `0` | stdout: `No config changes detected.` |
| `accepted` (`--async`) | `0` | stdout: `Reload requested.` |
| `failed` | `1` | stderr: 具体的な config/load/fetch エラー |
| `busy` | `1` | stderr: コントローラが多忙で reload を受理できない |
| `timeout` | `1` | stderr: 待機予算切れ。reload は後で完了する可能性あり |

警告は致命的でない適用後の問題です。警告ありで同期成功した場合:

- stdout に主な成功行を表示
- stderr には警告 1 件につき 1 行を `gc reload: warning: ...` として表示

終了コードの根拠:

- `gc reload` は、成功時 `0`、失敗時 `1` という既存の `gc` CLI の慣例を維持します。
- より細かな結果の区別は、構造化されたコントローラ応答とトレース/テレメトリで提供され、この機能のために新しい CLI 終了コード分類を作ることはしません。
- スクリプトは人間向けの `message` テキストを解析してはなりません。CLI レイヤで成功/失敗の分岐を行うか、将来のマシン向け統合ではコントローラプロトコルを使うべきです。

### Config 境界

reload の失敗は config-load 境界で定義されます:

- リモート pack のフェッチ
- パース/ロード
- バリデーション
- `workspace.name` の不一致
- 新しくロードされた config に必要な bead のライフサイクルセットアップ

これらのいずれかが失敗した場合、reload は非ゼロを返し、古いライブの config が有効のまま残ります。

config が正常に適用された後、それ以降のランタイム実行の問題はロールバックのトリガーではなく警告として扱われます。例:

- provider 切り替えのセットアップ失敗
- rig のバリデーションエラー
- formula やスクリプトの解決エラー
- service の reload エラー
- standalone city の bead-store 更新エラー

警告付きの `applied` 応答は、新しい有効な config がライブだが、適用後のランタイム更新の 1 つ以上が完全に収束しなかったことを意味します。たとえば、セッションの provider 切り替えが失敗した場合、reload 自体は成功しても古い provider が有効のまま残る可能性があります。完全な収束と劣化した成功を区別する必要があるオペレータやスクリプトは、構造化応答の `warnings` 配列、または CLI 出力の stderr 警告行を確認しなければなりません。

### 既存のランタイム挙動の維持

`gc reload` は:

- city/コントローラを再起動しない
- 既存のルールで reconciler が config drift を検出した場合、通常のセッションごとの再起動が発生することはある
- 既存の service-manager の reload 挙動を維持する
- コントローラが動作している限り、city が suspend 中でも動作する

### リモート pack の挙動

reload は、既存のコントローラ reload と起動パスと同じ完全ロードのセマンティクスを使用して、有効な config を再計算します:

- 設定されたリモート pack は config ロード前にフェッチされる場合がある
- フェッチ失敗はハードな reload 失敗となる
- `pack.lock` は変更されない

## アーキテクチャ

## CLI レイヤ

`cmd/gc` にトップレベルの `reload` コマンドを追加します:

- 既存の city コマンドのルールで city を解決
- `--async`/`--timeout` をバリデート
- city ごとのコントローラソケットに構造化された `reload` リクエストを送信
- 応答メッセージと警告を人間向け出力に整形
- コントローラ接続失敗時には、可能な限りリッチな supervisor の既知状態を表面化する:
  - supervisor 配下で city が起動中（現在のフェーズを含む）
  - city が起動失敗してバックオフ中（直近のエラーを含む）
  - city が動作していない
  - リッチな状態がない場合の汎用的な「コントローラ利用不可」フォールバック

## コントローラソケットプロトコル

コントローラには既に、config を dirty マークしてイベントループを起こす、ファイア&フォーゲットの `reload` ソケットコマンドがあります。本機能では、構造化されたバリアントを追加することで reload 制御を強化します:

```text
reload:<json>
```

### リクエスト

```json
{
  "wait": true,
  "timeout": "5m"
}
```

セマンティクス:

- `wait=true` は同期モードを意味する
- `wait=false` は非同期モードを意味する
- `timeout` は同期モードでは必須、非同期モードでは内部的に無視される

### レスポンス

```json
{
  "outcome": "applied|no_change|accepted|failed|busy|timeout",
  "message": "human readable summary",
  "revision": "full-config-revision-when-known",
  "warnings": ["normalized warning", "..."],
  "error": "specific failure string when outcome=failed"
}
```

注意点:

- `message` はコントローラが作成する正規のテキストで、CLI は基本的にそれを中継します。
- `revision` は `applied` と `no_change` の場合に含まれます。
- `accepted` は非同期受理の場合のみ使用されます。
- `busy` は、コントローラが受理ウィンドウ内にリクエストを登録できない場合、同期・非同期両モードで返される可能性があります。
- `warnings` は、出現順に正規化されたユーザー向け文字列です。

既存の素の `reload` コマンドは、内部呼び出し元やテスト向けの互換性ファイア&フォーゲットパスとして残ります。`gc reload` は構造化された `reload:<json>` コマンドを使用します。

## イベントループ統合

手動 reload は既存の config-dirty tick パスを再利用します。

実装の形:

1. コントローラ/ランタイムの配線にバッファなしの `reloadReqCh` を追加し、リクエストがイベントループに直接渡されるか、受理タイムアウト内に拒否されるようにします。
2. ソケットハンドラはバリデートし、`reloadReqCh` への reload リクエストの enqueue を試みます。
3. 受理は単なるチャネル enqueue ではなく、イベントループへの登録として定義されます。
4. ソケットハンドラは、イベントループがリクエストを受信して登録するまで最大 `5s` 待ちます。
5. イベントループがリクエストを消費したとき:
   - 別の手動 reload が既にアクティブなら `busy` を返す
   - そうでなければ、リクエストをアクティブな手動 reload として記録する
   - その後 `dirty=true` を設定する
   - その後 reconciler ループを起こす
   - その後リクエストを accepted として ack する
6. 登録後の次の tick が、ファイル監視で既に使われている同じ `dirty`-gated config reload パスを通じて reload を処理します。
7. その tick が reload の結果を解決すると、アクティブな手動リクエストを完了させ、スロットをクリアします。

重要な順序ルール:

- 受理された手動 reload は、既に実行中の tick に決して紐付けられません。
- アクティブな手動リクエストの登録は `dirty=true` の前に発生し、それは poke の前に発生します。
- reload の結果は、手動登録後に最初に開始された tick に紐付けられます。
- イベントループが `5s` 以内にリクエストを登録できない場合、呼び出し元は `busy` を受け取ります。

キューイングルール:

- 一度に 1 つの手動 reload リクエストのみがアクティブになり得ます
- 手動リクエストの便乗/合体の契約は追加されません
- 並行する手動リクエストは `busy` を受け取る可能性があります
- 手動リクエストは、既存の watcher 由来の dirty 状態と合体して 1 回の reload 試行になることがあり、`manual` として帰属されます

タイムアウトルール:

- enqueue タイムアウト（`5s`）と wait タイムアウト（`--timeout`、デフォルト `5m`）は別物のままです
- `timeout` は呼び出し元が待つのをやめたことを意味し、reload は後で完了する可能性があります

## ランタイム結果の捕捉

現在の reload パスは stdout/stderr に直接出力し、粗いテレメトリのみを記録します。`gc reload` をサポートするため、既存のロギング動作に加えて構造化された結果を生成するようリファクタリングします。

提案する内部結果の形:

- outcome: `applied`、`no_change`、または `failed`
- revision
- message
- warnings
- provider 変更マーカー
- error

この結果は以下に消費されます:

- アクティブな手動 reload リクエスト
- トレース記録
- テレメトリ記録

watcher 駆動の reload は、待機する CLI 呼び出し元なしで同じ reload 実装を引き続き使用します。

## 可観測性

### トレース

config-reload のトレース記録を以下を含むよう拡張します:

- `source`: `manual` または `watch`
- `warnings[]`

ルール:

- source は実際の reload 試行結果すべてに記録される
- watcher アクティビティで config が既に dirty な状態で手動 reload リクエストが受理されたら、manual が勝つ
- 結果として行われる reload は、watcher reload とそれに続く別個の手動再生ではなく、依然として 1 回の reload 試行
- tick トリガは実際の tick ソースを記録し、手動 reload は `trigger_detail="manual_reload"` で区別される

### テレメトリ

メトリクス/ロギングは低カーディナリティに保ちます。`RecordConfigReload` を以下を記録するよう拡張します:

- `status`: `ok` または `error`
- `source`: `manual` または `watch`
- `outcome`: `applied`、`no_change`、または `failed`
- `warning_count`

警告文字列の全文はテレメトリには含めず、トレースと CLI の stderr に残します。

## スコープ外のマシンインターフェース

本機能では `gc reload --json` も、パブリック HTTP reload ミューテーションも追加しません。

理由:

- この CLI のミューテーションコマンドは概ね人間向けのままにする
- 構造化されたコントローラ応答が、実装とテストに必要な型付き契約を既に提供している
- CLI JSON 契約やリモート API サーフェスの追加は、issue `#787` のスコープを超える

## ドキュメント

コマンドヘルプと生成済みの CLI リファレンスを以下のように更新します:

- `gc reload` は city/コントローラを再起動せずに現在の city を再ロードする
- 有効な config を再計算する前に、設定されたリモート pack をフェッチする場合がある
- config drift や provider の変更により必要な場合、既存のセッションごとの再起動が発生する場合がある

本機能では別個のトラブルシューティングガイドは追加しません。

## テスト

最低限のカバレッジ:

- standalone city: 同期 reload が applied
- supervisor 管理 city: 同期 reload が applied
- 同期 no-change
- 同期で無効な config の場合、古い config を維持
- 待機なしで非同期 accepted
- busy
- timeout
- コントローラ利用不可 / リッチな supervisor 状態のエラー表面化
- リモート pack のフェッチ失敗
- 手動ソース帰属（既存の watcher dirty 状態に対して manual が勝つ場合を含む）
- 警告がコントローラ応答、CLI stderr、トレースに表面化される
- ユーザーが明示的に `--timeout` を設定したときのみ `--async --timeout` のバリデーションが機能する
- ドキュメント化された成功/失敗グループに対する安定した `0`/`1` 終了コード挙動

## 実装ノート

- `gc reload` 用の新しいトップレベルコマンドファイルを追加。
- `reload` 用のコントローラリクエスト/レスポンス型とソケットハンドリングを追加。
- standalone および supervisor 管理 city のコントローラ起動を通じて `reloadReqCh` を配線。
- ランタイム reload を、watcher 駆動の既存挙動を維持しつつ構造化された結果と警告を返せるようリファクタリング。
- テレメトリとトレースのレコーダおよびそのテストを更新。
- コマンドヘルプ着地後に CLI リファレンスドキュメントを再生成。
