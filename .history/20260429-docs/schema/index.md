---
title: Schemas
description: Gas City ドキュメントと共に公開される機械可読なスキーマアーティファクト。
---

このセクションでは、ツール用に生成されたスキーマアーティファクトを公開しています。正規の JSON ファイルは `docs/schema/` に置かれており、以下のダウンロードリンクは Mint が提供するテキストミラーを使うことで、ローカルプレビューと本番の双方で動作するファイルダウンロードを提供します。

## OpenAPI 3.1

supervisor の HTTP および SSE コントロールプレーンは、生の OpenAPI ドキュメントとして公開されています:

- <a href="/schema/openapi.txt" download="openapi.json">Download <code>openapi.json</code></a>

このファイルは Swagger UI、Stoplight、Postman、クライアントジェネレータで利用してください。ライブの supervisor スキーマから再生成するには:

```bash
go run ./cmd/genspec
```

ナラティブな API 概要、エンドポイントファミリ、ワイヤレベルの注記については、[Supervisor REST API](/reference/api) ページを参照してください。

## gc events JSONL Schema

`gc events` の list/watch/follow 出力は、フィールドを重複させる代わりに OpenAPI DTO コンポーネントを参照する小さな JSON Schema として公開されています:

- <a href="/schema/events.txt" download="events.json">Download <code>events.json</code></a>

このファイルは、`gc events`、`gc events --watch`、`gc events --follow` が出力する 1 行の JSON オブジェクトを検証するために利用してください。カーソルモードは `gc events --seq` が JSONL ではなくプレーンテキストのカーソルを書き出すため、意図的に JSON Schema の対象外です。

CLI 出力コントラクトの詳細（スコープ選択、出力なし時の挙動、ハートビート抑制、カーソルフォーマットを含む）については、[gc events Formats](/reference/events) を参照してください。

## City Config JSON Schema

`city.toml` の設定スキーマも、生の JSON Schema ドキュメントとして公開されています:

- <a href="/schema/city-schema.txt" download="city-schema.json">Download <code>city-schema.json</code></a>

このファイルはバリデーション、エディタ統合、外部ツールに利用してください。再生成するには:

```bash
go run ./cmd/genschema
```
