---
title: Docs Workspace
description: Gas City の Mintlify ソースファイルとコントリビュータ向けドキュメント。
---

このディレクトリは Gas City ドキュメントサイトのソース・オブ・トゥルースです。

- Mintlify の設定は `docs.json` にあります。
- 公開ドキュメントのトップページは [`index.mdx`](/index) です。
- ダウンロード可能な仕様は `schema/` 配下にあります（supervisor OpenAPI、`gc events` JSONL、`city.toml` JSON Schema）。
- ローカルプレビューはリポジトリのルートから `./mint.sh dev` で実行できます（Mintlify は現時点で Node 22/24 LTS を要求し、Node 25+ には対応していません）。
- リンクチェックは `make check-docs` で実行します。
