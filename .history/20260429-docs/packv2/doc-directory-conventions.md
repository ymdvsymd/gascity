# Pack Structure v.next

**GitHub Issue:** TBD

タイトル: `feat: Pack structure v.next — pack と city の原則と標準ディレクトリレイアウト`

これは [doc-pack-v2.md](doc-pack-v2.md)、[doc-agent-v2.md](doc-agent-v2.md)、および [doc-commands.md](doc-commands.md) の補足文書です。

> **同期について:** このファイルが信頼できる情報源 (source of truth) です。GitHub issue が作成された場合は、まずここを編集し、その後 `---BEGIN ISSUE---` から `---END ISSUE---` までのセクションで issue 本文を更新してください。

---BEGIN ISSUE---

## 課題

V2 設計は規約ベースの構造へと進み続けていますが、次の両方を説明する一つのドキュメントが依然として存在しません。

1. **なぜ** pack を特定の方法で構造化すべきか
2. **どの**標準トップレベルディレクトリが何を意味するか

現状、ストーリーは複数のドキュメントに散らばっています。

- `doc-pack-v2.md` は cities-as-packs と imports を説明
- `doc-agent-v2.md` は agent ディレクトリを説明
- `doc-commands.md` は command と doctor の方向性を説明
- `doc-loader-v2.md` は規約ベース discovery を前提

これにより 6 つの問題が生じます。

1. **構造が明示されず暗黙になっている。** 「新しいディレクトリ規約」と言い続けていますが、唯一の正典的なリファレンスがありません。
2. **原則とメカニズムが分離している。** pack root に持たせるべき性質を先に述べないと、提案されたディレクトリを評価しづらくなります。
3. **関連する提案がドリフトする可能性がある。** agents、formulas、commands、doctor、overlays、skills、assets はすべて一つの pack 構造に揃えたがります。
4. **Loader の設計が未明示。** loader が規約で content を発見するなら、それらの規約の安定したマップが必要です。
5. **Pack 著者には一つの参照先がない。** Pack 著者は「pack の root に何が存在しうるか」を一箇所で答えられるべきです。
6. **制御された構造と不透明な assets を明確に分離していない。** 明示的な原則がなければ、pack root は場当たり的なフォルダの構造化されていないバケットになりがちです。

## ゴール

このノートは、まず原則レベルで V2 pack の構造を定義し、続いてすべての標準サブディレクトリを順に説明します。

次の問いに答えることを目指します。

- なぜ pack の root は意図的に制御されているのか
- city が通常の pack とどう関係するか
- どのトップレベルディレクトリが標準か
- loader が規約で何を発見するか
- pack 所有の不透明なファイルがどこに置かれるか
- pack ローカルパスがどう振る舞うべきか

## 設計原則

### 1. Pack はポータブルな定義の単位

Pack は import 可能、バージョン管理可能、ポータブルであるべき対象です。

つまり、pack root には次が含まれるべきです。

- 定義そのもの
- その定義が依存するファイル
- 定義が外部 content を必要とするときの他 pack への imports

それは pack 境界の外側にある周囲の兄弟ディレクトリや未文書化のファイルシステム規約に依存すべきではありません。

### 2. City は pack + deployment + site binding

City ディレクトリには 3 層あります。

- `pack.toml` と pack 所有ディレクトリ
  - 定義
- `city.toml`
  - deployment
- `.gc/`
  - site binding と runtime state

`city.toml` と `.gc/` を削除しても、残るものは依然として有効な pack であるべきです。

### 3. Pack の root は制御された表面である

Pack のトップレベルは、開かれたガラクタ入れではなく、意図的に設計されているべきです。

つまり以下のとおりです。

- 標準のトップレベル名は明示的に認識される
- 未知のトップレベルディレクトリはエラーであるべき
- 任意の追加ファイルは、一つのよく知られた不透明ディレクトリの下に置かれるべき

これにより、pack ローカルの柔軟性を禁止せずに強い構造が得られます。

### 4. 規約は実際に役立つ場合にパス配線を置き換えるべき

標準ディレクトリが存在する場合、loader は TOML での追加のパス配線なしにその意味を理解すべきです。

これは V2 における次のシフトの核心です。

- `prompt_template = "..."`
- `overlay_dir = "..."`
- `scripts_dir = "..."`
- すでに標準的な場所にある content への明示的なファイルパスのリスト

しかし、規約が、本当はただの不透明な assets であるファイルのために偽の意味的バケットを発明することを強いるべきではありません。

特に V2 には標準のトップレベルディレクトリとして `scripts/` は不要です。スクリプトファイルは次のいずれかに置かれるべきです。

- それを使う manifest やファイルの隣
- 一般的な不透明ヘルパーである場合は `assets/` の下

### 5. Root city pack と import された pack は同じ見た目にする

合成の root であるという理由だけで、root が別個のファイルシステムモデルを持つべきではありません。

City pack と import された pack は同じ pack 所有のディレクトリ構造を共有すべきです。City はそこに次を追加するだけです。

- `city.toml`
- `.gc/`

それ以外はすべて同じ意味を持つべきです。

最初の skills/MCP スライスについては、その「同じ意味」は現在の city pack のみに適用されます。import された pack の skills と MCP catalogs は後の話です。

### 6. Pack は境界では厳格、内部では柔軟

Pack 内部では、著者は自然にファイルを整理できるべきです。

Pack 境界をまたぐ場合のルールは厳格であるべきです。

- 参照は同じ pack 内のどこを指してもよい
- 参照は `..` を含む相対パスを使ってよい
- 正規化後、解決されたパスは pack root の内側に留まらなければならない
- Pack root を抜け出すことはエラー

Imports が pack 境界をまたぐ唯一の意図された仕組みです。

### 7. 不透明な assets には一つのホームが必要

Gas City が規約で解釈しない任意のファイルを pack 著者が置けるディレクトリが一つ必要です。

そのディレクトリは次のとおりであるべきです。

- 明確に名付けられている
- 標準
- 唯一の不透明トップレベル asset バケット

現在推奨される名前は `assets/` です。

## 標準レイアウト

### 最小 pack

```text
my-pack/
└── pack.toml
```

Pack に agents、formulas、commands、doctor checks、その他の pack 所有 assets がない場合、これは有効です。

### 典型的な pack

```text
my-pack/
├── pack.toml
├── agents/
├── formulas/
├── orders/
├── commands/
├── doctor/
├── patches/
├── overlay/
├── skills/
├── mcp/
├── template-fragments/
└── assets/
```

すべてのディレクトリが必須ではありません。標準ディレクトリが存在する場合、loader はその役割を理解します。

### 典型的な city

```text
my-city/
├── pack.toml
├── city.toml
├── .gc/
├── agents/
├── formulas/
├── orders/
├── commands/
├── doctor/
├── patches/
├── overlay/
├── skills/
├── mcp/
├── template-fragments/
└── assets/
```

Root city pack は import される pack と同じ pack 所有構造を使います。

## Root のファイル

### `pack.toml`

Pack の定義 root です。

次のような pack レベルの宣言的設定を保持することが期待されます。

- pack メタデータと identity
- imports
- providers
- agent defaults
- 名前付き sessions
- patches
- 発見されたファイルとして表現するよりも適切な、その他の pack レベル宣言的設定

規約で置き換えられるパス宣言の集積場所になるべきではありません。

#### `pack.toml` に属するもの

現在の V2 の方向性を踏まえると、`pack.toml` はより狭く、より宣言的になりつつあります。

次のようなものを含むべきです。

- pack メタデータ
  - `[pack]`
  - name、version、schema、その他の真の pack レベル identity または互換性フィールド
- imports
  - `[imports.<binding>]`
  - source、version 制約、import/re-export ポリシー
- providers
  - `[providers.*]`
- agent defaults
  - `[agent_defaults]`
  - 共有 defaults であって、個別の agent 定義ではない
- 名前付き sessions
  - `[[named_session]]`
- patches
  - 発見された content にまたがって適用される pack レベル変更ルール
- その他の pack ワイドな宣言的ポリシー
  - 真に pack 全体に適用され、ディレクトリ規約で表現するほうが良くない場合に限る

#### `pack.toml` に属さないもの

原則として、`pack.toml` は location で発見できる content をインベントリしたり配線したりすべきではありません。

つまり、次のものは保持しない方向に進むべきです。

- 個別の agent 定義
  - これらは `agents/<name>/` に置く
- prompt ファイルパス
  - `prompt.md` を使う
- overlay ディレクトリ宣言
  - `overlay/` と `agents/<name>/overlay/` を使う
- script ディレクトリ宣言
  - V2 には標準のトップレベル `scripts/` ディレクトリは存在しない
- 単純なケースの command インベントリ
  - 単純な command は `commands/<name>/run.sh` から動作するべき
- 単純なケースの doctor インベントリ
  - 単純な check は `doctor/<name>/run.sh` から動作するべき

必要に応じて pack root の下にローカルな TOML を依然として置くことができます。

- `agents/<name>/agent.toml`
- `commands/<id>/command.toml`
- `doctor/<id>/doctor.toml`

ただし、これらはエントリローカルな overlay であり、`pack.toml` を完全なファイルシステムインデックスに戻す理由ではありません。

### `city.toml`

Root city のみの deployment ファイルです。

次のようなチーム共有の deployment ポリシーを保持することが期待されます。

- rigs
- capacity
- service と substrate の決定
- deployment 指向の運用ポリシー

通常の import された pack の中には存在すべきではありません。

### `.gc/`

Root city のみのマシンローカルな site binding と runtime state です。

次を保持することが期待されます。

- workspace と rig の bindings
- caches
- sockets
- logs
- runtime state
- マシンローカル設定

これはポータブルな pack 定義の一部ではありません。

## 標準トップレベルディレクトリ

### `agents/`

規約で agents を定義します。

各直下子ディレクトリが一つの agent です。

```text
agents/
├── mayor/
│   ├── prompt.md
│   ├── agent.toml
│   ├── overlay/
│   ├── skills/
│   ├── mcp/
│   └── template-fragments/
└── polecat/
    └── prompt.md
```

このディレクトリは [doc-agent-v2.md](doc-agent-v2.md) でさらに規定されます。

### `formulas/`

規約で発見される formula 定義を保持します。

期待される内容: `*.toml` ファイル (formula ごとに 1 つ)。`.formula.` の中置は移行的なもので、削除を目指しています。`formulas/` は固定の規約 — 古い `[formulas].dir` 設定可能パスはなくなりました。

### `orders/`

規約で発見される order 定義を保持します。

期待される内容: `*.toml` ファイル (order ごとに 1 つ)。`.order.` の中置は移行的なもので、削除を目指しています。

Orders は formulas ではありません — dispatch をスケジュールするために formulas を*参照*します。トップレベルに置かれ、`formulas/` の下にネストされません。

### `patches/`

Import された agents の prompt 置換ファイルを保持します。

Patches は agent 定義とは異なります — `agents/<name>/` はあなた自身の agent を作成しますが、このディレクトリの patches は他人の agent を変更します。Patch ファイルは `pack.toml` または `city.toml` の `[[patches.agent]]` から修飾名で参照されます。

### `commands/`

Pack 提供の CLI command 定義と assets を保持します。

現在推奨される方向。

```text
commands/
├── status/
│   ├── run.sh
│   └── help.md
└── repo/
    └── sync/
        ├── run.sh
        └── help.md
```

主要なアイデア。

- ディレクトリがデフォルトの command tree を定義
- 各 command の葉は独自のローカルディレクトリを持つ
- ネストされたディレクトリはネストされた command words を意味する
- `run.sh` がデフォルトのよく知られたエントリポイント
- `help.md` は存在する場合のデフォルトのよく知られた help ファイル
- エントリローカルな scripts と help はエントリポイントの隣に置かれる
- `command.toml` はオプションで、メタデータや明示的な override が必要な場合のみ存在すべき

これによりファイルシステムの形が CLI の形と整合し、各 command の葉にローカルな asset スコープが与えられます。

重要な分割は次のとおりです。

- ユーザー向けの command words はデフォルトでディレクトリの形から来る
- ローカルな実行可能ファイルと help ファイルはデフォルトで単純なファイル名規約を使える
- `command.toml` は必須ではなく、エスケープハッチとして引き続き利用可能

### `doctor/`

Pack 提供の doctor checks と assets を保持します。

現在推奨される方向。

```text
doctor/
├── git-clean/
│   ├── doctor.toml
│   ├── run.sh
│   └── help.md
└── binaries/
    ├── doctor.toml
    └── run.sh
```

Doctor と commands は連携して設計されるべきです。両者は構造的に兄弟関係の表面です。

- 名前付きの運用エントリ
- 小さな manifest
- 実行可能なエントリポイント
- オプションの help とローカル assets

違いは露出にあります。

- commands は `gc` command 表面に貢献する
- doctor checks は `gc doctor` に貢献する

Commands と同様に。

- `run.sh` がデフォルトのよく知られたエントリポイント
- `help.md` は存在する場合のデフォルトのよく知られた help ファイル
- 実際に check を実行する script は、特別なトップレベル `scripts/` ディレクトリに依存するのではなく、自然に manifest と並んで存在すべき

### `overlay/`

V2 overlay ルールに従って agents に適用される pack ワイドな overlay ファイルを保持します。

特定の agent に固有でない共有 overlay マテリアルにこれを使います。

Agent ごとの overlays は `agents/<name>/overlay/` に属します。

### `skills/`

現在の city pack の共有 skills を保持します。

現在の city pack に同梱され、pack と agent の合成ルールに従って利用可能となる再利用可能な skills にこれを使います。

Agent ごとの skills は `agents/<name>/skills/` に属します。

### `mcp/`

現在の city pack の MCP server 定義または関連 MCP assets を保持します。

Agent ごとの MCP assets は `agents/<name>/mcp/` に属します。

### `template-fragments/`

Pack ワイドな prompt template の fragments を保持します。

Agent ごとの template fragments は `agents/<name>/template-fragments/` に属します。

### `assets/`

Gas City が規約で解釈しない、不透明な pack 所有 assets を保持します。

これは pack root を厳格に制御しつつ、任意のファイルを許容するためのエスケープハッチです。

例。

- 相対パスで参照されるヘルパー scripts
- 静的なデータファイル
- 標準 discovery 表面に紐づかない templates
- フィクスチャとテストデータ
- 相対 import パスで明示的に参照される埋め込み pack

Gas City は `assets/` を不透明として扱うべきです。Pack 境界の内側に参照が留まることは検証してもよいですが、内部レイアウトに特別な意味を割り当てるべきではありません。

ここは、root ディレクトリモデルをシンプルに保ちつつ埋め込み pack を許容する自然な場所でもあります。例。

```text
assets/imports/maintenance/pack.toml
```

このように。

```toml
[imports.maintenance]
source = "./assets/imports/maintenance"
```

これにより埋め込みが可能になり、pack root はシンプルかつ均一に保たれます。

## Pack ローカルパスの動作

Pack ローカル参照は次の単純なルールに従うべきです。

- 相対パスは、それを宣言したファイルまたは manifest を起点に解決される

より一般的には。

- パスを受け付ける任意のフィールドは、同じ pack 内のどのファイルも指してよい
- それには標準ディレクトリ下のファイルが含まれる
- それには `assets/` 下のファイルが含まれる

例。

- command `run = "./run.sh"` は `command.toml` を起点に解決
- doctor `run = "./run.sh"` は `doctor.toml` を起点に解決
- 他の pack ローカル参照は実用的な範囲で同じローカリティルールに従うべき

`..` は許容されるべきですが、一つの厳格な制約があります。

- 正規化後、解決されたパスは依然として同じ pack root の内側に留まらなければならない

つまりこれは許容されます。

```toml
run = "../shared/run.sh"
```

(pack 内に解決される場合)

これは許容されません。

```toml
run = "../../../outside.sh"
```

(pack 境界を抜け出す場合)

指針となる原則は次のとおりです。

- pack 内部では柔軟
- pack 境界では厳格

## Loader の期待

V2 loader はこれらのディレクトリを標準シグナルとして扱うべきです。

- `agents/` は「agents を発見する」を意味
- `formulas/` は「formulas を発見する」を意味
- `orders/` は「orders を発見する」を意味
- `commands/` は「command エントリを発見する」を意味
- `doctor/` は「doctor エントリを発見する」を意味
- `patches/` は「import された agents の prompt 置換ファイルをロードする」を意味
- `overlay/`、`skills/`、`mcp/`、`template-fragments/` は現在の city pack について「これらの種類の pack ワイド assets をロードする」を意味します。import された pack の catalogs は後の話です
- `assets/` は「不透明な pack 所有ファイル; 規約ベース discovery なし」を意味

Loader は規約で十分な場合、標準ディレクトリに対して明示的な TOML パス宣言を要求すべきではありません。

## Root と import された pack の動作の比較

同じ pack 所有ディレクトリは、次のすべてで同じ意味を持つべきです。

- root city pack
- 直接 import された pack
- 再エクスポートされた import 済み pack

違いは、異なるファイルシステム意味論からではなく、合成と露出のルールから生じるべきです。

これは特に次に重要です。

- commands
- doctor checks
- overlays
- skills

ディレクトリが import された pack で一つの意味を持ち、root city pack で別の意味を持つなら、V2 が取り除こうとしている非対称性を再導入している可能性が高いです。

## オープンクエスチョン

### 1. どの表面が完全に規約ベースで、どれが TOML 補助か?

最大の残存問題は、規約がどこまで及ぶかです。

完全に規約ベースの discovery の候補。

- agents
- formulas

依然として軽量な manifest メタデータが必要そうな候補。

- commands
- doctor checks

### 2. Command と doctor の最終的な manifest 形は?

現在の傾向。

- `commands/<id>/command.toml`
- `doctor/<id>/doctor.toml`
- エントリローカルな `run.sh` とオプションの `help.md`

オープンな詳細には、正確なフィールド名やオプションのメタデータが含まれます。

### 3. 未知のトップレベルディレクトリはエラーにすべきか?

現在の傾向: はい。

トップレベル pack 表面は厳格に制御されたままにすべきです。任意のファイルは `assets/` の下に置くべきです。

### 4. `assets/` を唯一の不透明トップレベルディレクトリにすべきか?

現在の傾向: はい。

複数の不透明 root を許容すると、制御された pack 構造を持つ意義が弱まります。

### 5. 標準 discovery はどこまでネスティングを許容すべきか?

次に対しては正確な walk ルールが依然として必要です。

- `formulas/`
- `commands/`
- `doctor/`

例。

- ネストされたサブディレクトリは自由に許容されるか?
- 名前は相対パスから派生するか?
- 一部のサブディレクトリは予約されるか?

## 作業ドラフト要約

現在の方向性は次のとおりです。

1. pack はポータブルな定義の単位
2. city は pack に `city.toml` と `.gc/` を加えたもの
3. トップレベル pack 表面は意図的に制御されるべき
4. 規約は実際に役立つ場合にパス配線を置き換えるべき
5. root city pack と import された pack は同じ pack 所有構造を使うべき
6. `assets/` は唯一の不透明トップレベル asset バケット
7. `agents/`、`formulas/`、`orders/`、`commands/`、`doctor/`、`patches/`、`overlay/`、`skills/`、`mcp/`、`template-fragments/`、`assets/` が標準 pack ディレクトリ
8. commands と doctor checks は現在、エントリごとのディレクトリと小さな manifest、ローカル assets に傾いている
9. パスを値とするフィールドは同じ pack 内のどこでも (`assets/` を含めて) 指してよい
10. pack ローカルパスは pack 内部では柔軟、pack 境界では厳格であるべき

---END ISSUE---
