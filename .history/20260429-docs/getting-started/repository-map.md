---
title: Repository Map
description: Gas City リポジトリ内で主要サブシステムが配置されている場所。
---

## トップレベルのレイアウト

| パス | 内容 |
|---|---|
| `cmd/gc/` | CLI のエントリポイント、コントローラの組み立て、ランタイムの構築、コマンドハンドラ |
| `internal/runtime/` | ランタイムプロバイダの抽象、および tmux、subprocess、exec、ACP、K8s、ハイブリッド実装 |
| `internal/config/` | `city.toml` のスキーマ、バリデーション、合成、pack、パッチ、オーバーライド解決 |
| `internal/beads/` | work、mail、molecule、wait に使われるストアの抽象とプロバイダ実装 |
| `internal/session/` | session bead のメタデータ、wait ライフサイクルヘルパー、session アイデンティティユーティリティ |
| `internal/orders/` | 周期的ディスパッチのための order パースとスキャン |
| `internal/convergence/` | バウンドされた反復リファインメントループとゲート処理 |
| `internal/api/` | HTTP API ハンドラとリソースビュー |
| `docs/` | Mintlify ドキュメントサイト（チュートリアル、ガイド、リファレンス） |
| `engdocs/` | コントリビュータ向けのアーキテクチャ、設計ドキュメント、提案、アーカイブ |
| `examples/` | サンプル city、pack、formula、リファレンストポロジー |
| `contrib/` | ヘルパスクリプト、Dockerfile、統合サポートアセット |
| `test/` | 統合テストとサポートテストパッケージ |

## どこから始めるか

- CLI の挙動: `cmd/gc/` から始め、それが呼び出すコマンド固有のヘルパに従ってください。
- ランタイム/プロバイダ作業: `internal/runtime/runtime.go` と、変更するプロバイダパッケージから始めます。
- 設定と pack の挙動: `internal/config/config.go`、`internal/config/compose.go`、`internal/config/pack.go` から始めます。
- work のルーティングと molecule の作成: `cmd/gc/cmd_sling.go` と `internal/beads/` から始めます。
- supervisor、session、wake/sleep の挙動: `cmd/gc/`、`internal/session/`、`internal/runtime/` から始めます。

コントリビュータ向けのパッケージウォークスルーは Codebase Map (`engdocs/contributors/codebase-map`) に続きます。
