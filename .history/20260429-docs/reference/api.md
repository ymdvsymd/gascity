---
title: Supervisor REST API
description: `gc` supervisor が公開する型付き HTTP + SSE コントロールプレーン。
---

`gc` supervisor は、OpenAPI 3.1 ドキュメントで記述された単一の型付き HTTP コントロールプレーンを公開します。CLI が行うすべてのことは、サードパーティクライアントでも実行できます。隠されたサーフェスはありません。

## 仕様のダウンロード

- **<a href="/schema/openapi.txt" download="openapi.json">openapi.json をダウンロード</a>** — 公式の契約。Stoplight、Postman、Swagger UI などの OpenAPI 対応ツールに取り込んで、操作を対話的にブラウズできます。
- **<a href="/schema/events.txt" download="events.json">events.json をダウンロード</a>** — `gc events` の JSONL 行スキーマ。`openapi.json` の DTO コンポーネントを参照しているため、API がソースオブトゥルースであり続けます。

## エンドポイントファミリ

仕様が完全なリファレンスです。サーフェスの簡単な要約:

- **Cities.** `GET /v0/cities`、`POST /v0/city`、`GET /v0/city/{cityName}`、`GET /v0/city/{cityName}/status`、`GET /v0/city/{cityName}/readiness`、`POST /v0/city/{cityName}/stop`。
- **Health & readiness.** `GET /health`、`GET /v0/readiness`、`GET /v0/provider-readiness`。
- **Agents.** `/v0/city/{cityName}/agents` 配下の `GET/POST/DELETE`、および SSE `/v0/city/{cityName}/agents/{agent}/output/stream`。
- **Beads（作業単位）.** `/v0/city/{cityName}/beads` 配下の CRUD、クエリ + hook 操作、依存関係、ラベル。
- **Sessions.** `/v0/city/{cityName}/sessions` 配下の CRUD、submit、prompt、resume、interaction response、transcript、SSE ストリーム。
- **Mail、convoy、order、formula、molecule、participants、transcript、adapter.** 外部メッセージングおよびオーケストレーションのサーフェス。操作ごとの形は仕様を参照してください。
- **Event bus.** supervisor スコープの `GET /v0/events` + `GET /v0/events/stream`、および city スコープの `GET /v0/city/{cityName}/events` + `GET /v0/city/{cityName}/events/stream`。
- **Config と pack.** `/v0/city/{cityName}/config` および `/v0/city/{cityName}/packs` 配下の city ごとの config と pack メタデータ。

## リクエストとレスポンスのヘッダ

すべての操作のヘッダ契約は OpenAPI 仕様に記載されています。リクエストヘッダが必須の場合や、レスポンスヘッダが約束されている場合、仕様がそれを記述します。すべての API クライアントが知っておくべき横断的な 2 つのヘッダ:

- **`X-GC-Request`**（リクエストヘッダ、すべてのミューテーションで必須）。すべての POST、PUT、PATCH、DELETE で必須の anti-CSRF トークンです。空でない値であれば何でも受け入れられます。サーバーがチェックするのはヘッダの存在です。これがないリクエストは `403 csrf: X-GC-Request header required on mutation endpoints` で拒否されます。同一オリジンポリシーを利用しているため、クロスオリジンの攻撃者は偽造リクエストにこのヘッダを設定できません。生成された Go と TypeScript のクライアントはこのヘッダを自動的に設定するため、生の HTTP クライアントだけが覚えておく必要があります。
- **`X-GC-Request-Id`**（レスポンスヘッダ、すべてのレスポンス）。サーバーがログ相関のために割り当てる、レスポンスごとの不透明な識別子です。すべてのレスポンス — 成功でもエラーでも — がこのヘッダを持ちます。仕様では `components.headers.X-GC-Request-Id` への `$ref` 経由で宣言されています。サーバーログを追跡できるよう、バグレポートにはこの値を含めてください。

SSE ストリーム操作は、最初のイベントフレームの前に追加のランタイムステータスヘッダを送出します:

- **`stream-agent-output` / `stream-agent-output-qualified`**: `GC-Agent-Status` — エージェントが動作しておらず、ライブ出力ではなくセッションログからトランスクリプトを再生している場合は `stopped` に設定されます。
- **`stream-session`**: `GC-Session-State`（例: `active`、`closed`）と `GC-Session-Status`（セッションの基底プロセスが動作していない場合は `stopped`）。

各ヘッダのスキーマは、その操作の `responses.200.headers` 内で仕様にドキュメント化されています。

## エラー

すべてのエラーレスポンスは RFC 9457 Problem Details ボディ（`application/problem+json`）です。エラータイプは仕様の `components.schemas.ErrorModel` の下にドキュメント化されています。`detail` フィールドは短い `code: ` プレフィックスを伴い（例: `pending_interaction: ...`、`conflict: ...`、`not_found: ...`、`read_only: ...`）、クライアントは型付きエラーの enum を必要とせずに意味的なコードでパターンマッチできます。ボディフィールドのバリデーションエラー（例: 必須の string が空で投稿された場合）は、操作によって `422 Unprocessable Entity` または `400 Bad Request` として返ります。Problem Details ボディの `errors` 配列がどのフィールドが失敗したかを示します。

## ストリーミング

SSE エンドポイントは `Content-Type: text/event-stream` を設定し、型付きの `event:` フレームを送出します。仕様は各イベントのペイロードスキーマを操作ごとの `responses.200.content.text/event-stream` エントリの下に記述します。クライアントはサーバーがサポートしている場所では標準の SSE 再接続プロトコル（`Last-Event-ID` ヘッダ）に従ってください。イベントバスストリーム（`/v0/events/stream`）は最後に受信したインデックスから再生します。

致命的なセットアップエラーは、ストリームの 200 ヘッダがコミットされる*前*に通常の Problem Details レスポンスとして返され、すぐに閉じる 200 ストリームとして返されることはありません。たとえば、`GET /v0/events/stream` は、稼働中の city にイベント provider が登録されていない場合、`detail: "no_providers: ..."` を持つ `503 application/problem+json` を返します。

## city の作成（非同期）

`POST /v0/city` は**非同期**操作です。レスポンスは、city がディスクにスキャフォールドされ、supervisor に登録された時点で `202 Accepted` として返されます。遅い finalize 作業（pack のマテリアライズ、bead store の起動、formula の解決、エージェントのバリデーション）は、supervisor reconciler の次の tick で実行されます。クライアントは supervisor イベントストリームを通じて完了を観察します — ポーリングするものは何もありません。

### レスポンス

```json
{
  "ok": true,
  "name": "my-city",
  "path": "/abs/path/to/my-city"
}
```

`name` フィールドは city の解決済みランタイムアイデンティティ（`city.toml` の `workspace.name`、またはディレクトリのベース名）です。完了のためのイベントストリームをフィルタするのに使ってください。

### 完了イベント

同じ `/v0/events/stream` 上で、クライアントは（順番に）以下を見ます:

- `city.created`（`CityCreatedPayload`） — `POST` がリターンする前に scaffold ステップから送出されます。`subject` とペイロードの `name` はレスポンスの `name` と一致します。
- `city.ready`（`CityReadyPayload`） — reconciler が `prepareCityForSupervisor` を正常に完了しました。一致するイベント: `subject == name` かつ `type == "city.ready"`。
- `city.init_failed`（`CityInitFailedPayload`） — reconciler が諦めました。ペイロードの `error` フィールドが理由を記述します。これには非同期 API が同期的に失敗しない遅延依存または provider readiness のブロッカーが含まれます。

成功した `POST` ごとに `city.ready` または `city.init_failed` のいずれか 1 つだけが届きます。クライアントはどちらかを待ちます。`GET /v0/cities` や `GET /v0/city/{cityName}/readiness` のポーリングは不要です。

### POST の前後どちらでも subscribe 可能

どちらの順序でも動作します。推奨フローは:

1. `POST /v0/city` を実行し、`202` を待ちます。
2. `GET /v0/events/stream?after_cursor=0` — 最初からの再生を要求するため、`city.created`（と場合によっては `city.ready`）が subscribe 前に発生していても配信されます。
3. `subject == response.name` かつ `type ∈ {"city.ready", "city.init_failed"}` までフレームを読み取ります。

**空の supervisor でも問題ありません。** イベントストリームは、`POST` の前に city が存在しなくても動作します。`POST` は city を supervisor レジストリ（`cities.toml`）に書き込み、202 を返す前に `.gc/events.jsonl` を同期的に作成します。そのため、イベントマルチプレクサは次の `buildMultiplexer` 呼び出しで新しい city を見つけます。サブスクライバは `503 no_providers` で再試行する必要は**ありません**。202 が成功した後にそのエラーが表面化したら、それはバグです。

### エラー

- `409 conflict: city already initialized at <path>` — ターゲットディレクトリには既にスキャフォールドされた city があります。
- `422` — provider が無効、bootstrap profile が無効、その他のボディバリデーション失敗。
- `503` — ホスト上にハード依存が欠けているか、city が必要とする provider が ready ではありません。
- `500` — 予期しないスキャフォールド失敗。`X-GC-Request-Id` 相関ヘッダ経由でサーバーログを参照してください。

## city の登録解除（非同期）

`POST /v0/city/{cityName}/unregister` は、supervisor のレジストリから city を削除し、supervisor に city のコントローラを停止するよう信号を送ります。`POST /v0/city` と同様に非同期です: レスポンスは、レジストリエントリが削除され supervisor に通知された時点で `202 Accepted` として返されます。supervisor reconciler は次の tick でコントローラを停止し、完了イベントを送出します。

ディスク上の city ディレクトリは**触られません**。この操作は city を supervisor から切り離すだけです。後で再アタッチするのは単純な `gc register` です。

### レスポンス

```json
{
  "ok": true,
  "name": "my-city",
  "path": "/abs/path/to/my-city"
}
```

### 完了イベント

`/v0/events/stream` 上で、クライアントは（順番に）以下を見ます:

- `city.unregister_requested`（`CityUnregisterRequestedPayload`） — レジストリ書き込みの前にハンドラから送出され、サブスクライバが解体の開始を見られるようにします。
- `city.unregistered`（`CityUnregisteredPayload`） — city のコントローラが停止すると reconciler から送出されます。一致するイベント: `subject == name` かつ `type == "city.unregistered"`。
- `city.unregister_failed`（`CityUnregisterFailedPayload`） — コントローラがクリーンに停止しなかった場合に reconciler から送出されます。ペイロードの `error` フィールドが失敗を記述します。

成功した unregister ごとに `city.unregistered` または `city.unregister_failed` のいずれか 1 つだけが届きます。クライアントはどちらかを待ちます。

### エラー

- `404 not_found: city not registered with supervisor: <name>` — その名前のレジストリエントリがありません。
- `501` — supervisor に Initializer が組み込まれていない（テスト専用 config）。
- `500` — 予期しないレジストリ書き込み失敗。

## イベント契約

イベント API、SSE ストリーム、`gc events` は、3 つの異なるプレゼンテーションレイヤにおける同じ契約です。API がソースオブトゥルースです。

JSONL フレーミング、空出力時の挙動、ハートビート抑制、`--seq` プレーンテキストカーソルフォーマットを含む CLI 出力契約の明示については、[gc events Formats](/reference/events) を参照してください。

### City スコープ

- `GET /v0/city/{cityName}/events` は `ListBodyWireEvent` を返し、`X-GC-Index` を含みます。
- `GET /v0/city/{cityName}/events/stream` は以下を送出します:
  - `EventStreamEnvelope` を持つ `event: event`
  - `HeartbeatEvent` を持つ `event: heartbeat`
- 再開:
  - `Last-Event-ID` または `after_seq`
- city スコープでの `gc events` は 1 行に 1 つの `WireEvent` JSON オブジェクトを出力します。
- city スコープでの `gc events --watch` と `gc events --follow` は 1 行に 1 つの `EventStreamEnvelope` JSON オブジェクトを出力します。
- city スコープでの `gc events --seq` は API の `X-GC-Index` 値を出力します。

### Supervisor スコープ

- `GET /v0/events` は `WireTaggedEvent` 項目を持つ `SupervisorEventListOutputBody` を返します。
- `GET /v0/events/stream` は以下を送出します:
  - `TaggedEventStreamEnvelope` を持つ `event: tagged_event`
  - `HeartbeatEvent` を持つ `event: heartbeat`
- 再開:
  - `Last-Event-ID` または `after_cursor`
- supervisor スコープでの `gc events` は 1 行に 1 つの `WireTaggedEvent` JSON オブジェクトを出力します。
- supervisor スコープでの `gc events --watch` と `gc events --follow` は 1 行に 1 つの `TaggedEventStreamEnvelope` JSON オブジェクトを出力します。
- supervisor スコープでの `gc events --seq` は現在の合成 supervisor カーソルを出力し、`--after-cursor` で使用できます。

### トランスポートとセマンティック型

- SSE の `event:` 行はトランスポートエンベロープです: `event`、`tagged_event`、または `heartbeat`。
- セマンティックなイベント種別は JSON ペイロードの `type` フィールドです: `bead.created`、`mail.sent`、`session.woke` など。
- CLI は別個のイベントスキーマを定義しません。同じ DTO とエンベロープを JSONL としてストリームします。

## バージョニング

API は URL プレフィックス（`/v0`）でバージョニングされます。破壊的変更は新しいプレフィックスで提供されます。現在の仕様が `v0` の公式契約です。
