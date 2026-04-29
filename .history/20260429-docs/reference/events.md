---
title: gc events Formats
description: `gc events` が出力する正確なフォーマット。
---

`gc events` は supervisor のイベント API の CLI への投影です。API がソース・オブ・トゥルースですが、ユーザが OpenAPI ドキュメントからリバースエンジニアリングせずに `gc events` を消費できるよう、このページでは CLI 出力のコントラクトを明示的にドキュメント化しています。

## ソース・オブ・トゥルース

これらの CLI フォーマットは supervisor API と SSE コントラクトの投影です:

- City list API: `GET /v0/city/{cityName}/events`
- City SSE API: `GET /v0/city/{cityName}/events/stream`
- Supervisor list API: `GET /v0/events`
- Supervisor SSE API: `GET /v0/events/stream`

基底の DTO は公開された OpenAPI ドキュメントから来ます:

- `WireEvent`
- `WireTaggedEvent`
- `EventStreamEnvelope`
- `TaggedEventStreamEnvelope`
- `HeartbeatEvent`

正規の supervisor 仕様と `gc events` JSONL ラインスキーマは [Schemas](/schema) からダウンロードしてください。より広範なイベントバスの注記は [Supervisor REST API](/reference/api) を参照してください。

## 出力モード

`gc events` には 2 つの出力ファミリがあります:

- List モード: `gc events`
- Stream モード: `gc events --watch` と `gc events --follow`

例外が 1 つあります:

- Cursor モード: `gc events --seq`

### List モード

`gc events` は **JSON Lines** を stdout に書き出します。

- 各行は厳密に 1 つの JSON オブジェクトです。
- 外側の配列やラッパーオブジェクトはありません。
- マッチするものがなければ stdout は空です。

#### City スコープ

city がスコープ内にあるとき、各出力行は `GET /v0/city/{cityName}/events` からの 1 つの `WireEvent` オブジェクトです。

例:

```json
{"actor":"human","message":"hello","seq":21,"subject":"mayor","ts":"2026-04-17T15:20:52.136314-07:00","type":"mail.sent"}
```

#### Supervisor スコープ

city がスコープ内になく supervisor API が使われているとき、各出力行は `GET /v0/events` からの 1 つの `WireTaggedEvent` オブジェクトです。

例:

```json
{"actor":"human","city":"mc-city","message":"hello","seq":21,"subject":"mayor","ts":"2026-04-17T15:20:52.136314-07:00","type":"mail.sent"}
```

supervisor 形式はマージされたイベントバスが複数の city にまたがるため `city` を追加します。

### Stream モード

`gc events --watch` と `gc events --follow` も **JSON Lines** を stdout に書き出しますが、行スキーマは list モードとは異なります。

- 各行は厳密に 1 つの SSE イベント envelope を JSON としてシリアライズしたものです。
- CLI はマッチするイベント envelope のみを出力します。
- ハートビート SSE フレームは内部で消費され、stdout には **書き出されません**。
- `--watch` がマッチなしでタイムアウトした場合、stdout は空でコマンドは正常終了します。

#### City スコープ

各行は API の `event: event` SSE ペイロードに対応する 1 つの `EventStreamEnvelope` オブジェクトです。

例:

```json
{"actor":"human","message":"hello","seq":21,"subject":"mayor","ts":"2026-04-17T15:20:52.136314-07:00","type":"mail.sent"}
```

#### Supervisor スコープ

各行は API の `event: tagged_event` SSE ペイロードに対応する 1 つの `TaggedEventStreamEnvelope` オブジェクトです。

例:

```json
{"actor":"human","city":"mc-city","message":"hello","seq":21,"subject":"mayor","ts":"2026-04-17T15:20:52.136314-07:00","type":"mail.sent"}
```

### Cursor モード

`gc events --seq` は JSONL を出力 **しません**。stdout に 1 つのプレーンテキストカーソルを表示します。

#### City スコープ

値はその city のイベントログの現在の `X-GC-Index` ヘッドです。

例:

```text
21
```

#### Supervisor スコープ

値は `--after-cursor` で使われる複合 supervisor カーソルです。

例:

```text
alpha:4,beta:9,mc-city:21
```

## フィルタリングと形

以下のフラグはどのオブジェクトを出力するかをフィルタするだけです。JSON の形は変わりません:

- `--type`
- `--since`
- `--payload-match`
- `--after`
- `--after-cursor`

同じルールが list モードと stream モードの両方に適用されます。

## 機械可読スキーマ

ダウンロード可能な <a href="/schema/events.txt" download="events.json">events.json</a> スキーマは、list、watch、follow モードからの 1 行の JSON オブジェクトを検証します。フレーミングメタデータと `openapi.json` への `$ref` のみを含みます:

- City の list 行は `WireEvent` を使う。
- Supervisor の list 行は `WireTaggedEvent` を使う。
- City の stream 行は `EventStreamEnvelope` を使う。
- Supervisor の stream 行は `TaggedEventStreamEnvelope` を使う。

`gc events --seq` は JSON ではなくプレーンテキストを書き出すため、JSON Schema の対象外です。

## トランスポートとセマンティック型

stream モードでは、これらを区別してください:

- SSE の `event:` 値はトランスポート envelope 名: `event`、`tagged_event`、`heartbeat`。
- JSON オブジェクトの `type` フィールドはセマンティックなイベント型: `bead.created`、`mail.sent`、`session.woke` など。

`gc events` は JSON ペイロードと envelope を出力し、生の SSE フレームのテキストは出力しません。

## エラー

成功したイベントクエリは stdout にデータのみを書き出します。

運用上の失敗は人間可読のテキストとして stderr に書かれ、非ゼロの終了ステータスを返します。例:

- API 検出の失敗
- `--after` と `--after-cursor` のような不正なフラグの組み合わせ
- API が Problem Details として返したストリームセットアップの失敗
- 不正または復号できないストリームペイロード

## 安定性コントラクト

CLI は独立したイベント DTO を定義しません。安定性コントラクトは:

- `WireEvent`、`WireTaggedEvent`、`EventStreamEnvelope`、`TaggedEventStreamEnvelope` に対する公開 supervisor OpenAPI スキーマ
- このページの明示的な CLI フレーミングルール: list および stream モードの JSONL、`--seq` のプレーンテキスト、マッチなしの list クエリの空 stdout、stream モードのハートビート抑制
