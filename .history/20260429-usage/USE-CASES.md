# Gas City — ユースケースガイド

**生成日:** 2026-04-29
**対象バージョン:** gascity v1.0.0+

---

## シナリオ一覧

| # | シナリオ | カテゴリ | 難易度 |
|---|---------|---------|--------|
| 1 | 一つ目の city を立ち上げて hello world を作らせる | Getting Started | 初級 |
| 2 | カスタムレビュアエージェントを定義する | Core Workflow | 初級 |
| 3 | mayor とポレキャットを連携させて作業を委譲する | Core Workflow | 中級 |
| 4 | formula でマルチステップワークフローを書く | Core Workflow | 中級 |
| 5 | order で定期ジョブを仕込む（cooldown / cron / event） | Automation | 中級 |
| 6 | 既存パックを取り込んで複数 rig に共通展開する | Advanced Usage | 上級 |
| 7 | handoff で長時間エージェントを綺麗に交代する | Recovery | 中級 |

---

## シナリオ 1: 一つ目の city を立ち上げて hello world を作らせる

**カテゴリ:** Getting Started
**難易度:** 初級
**使うコマンド:** [`gc init`](./COMMANDS.md#gc-init-path), [`gc rig`](./COMMANDS.md#gc-rig), [`gc sling`](./COMMANDS.md#gc-sling-target-bead-or-formula-or-text), [`bd show`](./COMMANDS.md#gc-bd-bd-args)

### 状況

これから Gas City を初めて触る。手元の Mac / Linux に `claude` または `codex` の CLI が入っており、ログイン済み。最低限の構成で「city を作る → 別フォルダを rig として登録 → エージェントに `hello.py` を作らせる」までを通しで体験したい。10分以内で結果を見たい。

### 手順

1. Homebrew で Gas City をインストールし、`gc` が動くか確認する。
   ```bash
   brew install gastownhall/gascity/gascity
   gc version
   ```

2. city を作る。`--provider` を渡すと対話プロンプトをスキップできる。
   ```bash
   gc init ~/first-city --provider claude
   cd ~/first-city
   ```

3. 作業対象になる空のプロジェクトディレクトリを用意し、git init し、rig として登録する。
   ```bash
   mkdir -p ~/hello-world
   cd ~/hello-world
   git init
   gc rig add ~/hello-world
   ```

4. rig 内から、その rig 専用の Claude エージェントへ仕事を投げる。
   ```bash
   gc sling hello-world/claude "hello.py を作って Hello World を出力させて"
   ```

5. 進行を見守る。`bd show --watch` で bead の状態が変わるたびに更新される。
   ```bash
   bd show <さっき出力された bead ID> --watch
   ```
   `STATUS` が `closed` になったら Ctrl+C で抜ける。

6. 結果を確認する。
   ```bash
   ls         # hello.py
   cat hello.py
   ```

### 期待される結果

```text
$ gc sling hello-world/claude "hello.py を作って..."
Created hw-ff9 — "hello.py を作って Hello World を出力させて"
Attached wisp hw-6yh (default formula "mol-do-work") to hw-ff9
Auto-convoy hw-4tl
Slung hw-ff9 → hello-world/claude

$ bd show hw-ff9 --watch
✓ hw-ff9 · hello.py を作って...   [● P2 · CLOSED]
Owner: -- · Assignee: claude-hw-208 · Type: task

$ cat hello.py
print("Hello, World!")
```

### ヒント

- 引数の bead ID プレフィックス（`hw-`）は、rig 名「hello-world」から派生している。`gc rig list` で確認できる。
- `gc sling` ではなく `gc session attach hello-world/claude` で直接タミーンに乗り込めるが、Claude が prompt を読み終わる前に attach すると初期 prompt が見えないことがある。最初は `gc sling` のほうが結果が分かりやすい。
- うまく動かないときは `gc doctor`、それでもダメなら [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) を参照する。

---

## シナリオ 2: カスタムレビュアエージェントを定義する

**カテゴリ:** Core Workflow
**難易度:** 初級
**使うコマンド:** [`gc agent add`](./COMMANDS.md#gc-agent), [`gc prime`](./COMMANDS.md#gc-prime-agent-name), [`gc sling`](./COMMANDS.md#gc-sling-target-bead-or-formula-or-text)

### 状況

シナリオ 1 で作った `~/first-city` と `~/hello-world` をまだ使える。`hello.py` をコードレビューする専用エージェントを `reviewer` として定義したい。Claude の代わりに Codex でレビューさせたい（複数 provider のセカンドオピニオン運用）。

### 手順

1. city ディレクトリで `gc agent add` してスキャフォールドする。
   ```bash
   cd ~/first-city
   gc agent add --name reviewer --dir hello-world
   ```
   `agents/reviewer/prompt.template.md` ができる。

2. provider 上書きを書く（city デフォルトの Claude ではなく Codex を使わせる）。
   ```bash
   cat > agents/reviewer/agent.toml <<'EOF'
   dir = "hello-world"
   provider = "codex"
   EOF
   ```

3. プロンプトテンプレートをレビュア向けに書き換える。
   ```bash
   cat > agents/reviewer/prompt.template.md <<'EOF'
   # Code Reviewer Agent
   You are an agent in a Gas City workspace. Pick up review tasks and execute them.

   ## Your tools
   - `bd ready` — see available work items
   - `bd show <id>` — see details of a work item
   - `bd close <id>` — mark work as done

   ## How to work
   1. Check available work: `bd ready`
   2. Read the target file mentioned in the bead title
   3. Write findings into review.md (bugs, security, style)
   4. Close the bead with `bd close <id>`
   EOF
   ```

4. 設定の解決を確認する。
   ```bash
   gc prime hello-world/reviewer    # 完成した prompt が出る
   gc reload                        # controller に再読込させる（必要なら）
   ```

5. レビューを依頼する。
   ```bash
   cd ~/hello-world
   gc sling hello-world/reviewer "hello.py をレビューして review.md にフィードバックを書いて"
   ```

6. 進捗を見て結果を確認する。
   ```bash
   bd ready                # 投入中の bead 確認
   gc session list --template hello-world/reviewer  # session が作られたか
   ls                      # hello.py + review.md
   cat review.md
   ```

### 期待される結果

`review.md` が生成され、Codex が出した観点別のフィードバックが書かれている。`hello.py` が一行 print なので「特に問題なし」のような短い結果になることが多い。

### ヒント

- もっと厳密にしたい場合は `agents/reviewer/agent.toml` に `option_defaults = { model = "sonnet", permission_mode = "plan" }` のように provider 固有オプションを書ける。
- `gc prime hello-world/reviewer` の出力をエディタで確認しながらプロンプトを iterate するのがやりやすい。
- 同じ rig に対してレビュア・実装者・テスタを別エージェントとして定義し、`gc sling` で振り分けると並行作業が組める。

---

## シナリオ 3: mayor とポレキャットを連携させて作業を委譲する

**カテゴリ:** Core Workflow
**難易度:** 中級
**使うコマンド:** [`gc mail`](./COMMANDS.md#gc-mail), [`gc session`](./COMMANDS.md#セッション管理), [`gc sling`](./COMMANDS.md#gc-sling-target-bead-or-formula-or-text)

### 状況

シナリオ 2 までで mayor（常駐 crew）と reviewer（rig スコープのエージェント）が揃った。mayor に依頼を投げて、mayor が判断して reviewer にスリングする「人間 → mayor → reviewer」の流れを作りたい。これは Gas City における典型的なオーケストレーションパターン。

### 手順

1. mayor 用の常駐セッションが立ち上がっているか確認する。`gc init` 直後なら自動で立っているはず。
   ```bash
   cd ~/first-city
   gc session list
   # mayor が active であること
   ```

2. mayor へメールを送る。subject と body を分けると thread として追えるようになる。
   ```bash
   gc mail send mayor -s "Code review" -m "hello-world の hello.py をレビューして、結果を review.md にまとめてください"
   ```

3. mayor を起こす。ナッジするとそのターンで mail が hook 経由で context に入る。
   ```bash
   gc session nudge mayor "メールと hook を確認して、必要なら作業を委譲してください"
   ```

4. mayor の出力を覗く。
   ```bash
   gc session peek mayor --lines 20
   # mayor が「reviewer に振りますね」と判断 → gc sling を実行する様子が見える
   ```

5. reviewer のセッションができていること、bead が進んでいることを確認する。
   ```bash
   gc session list
   bd list --status open --flat
   ```

6. 完了したら mail で結果を確認する。mayor が結果を返してくることもある（プロンプトの書き方次第）。
   ```bash
   gc mail inbox
   gc mail read <id>
   ```

### 期待される結果

`gc session peek mayor` を見ると、mayor が次のような行動をとる:

1. メールに気付く（hook 経由で `[USER]` メッセージとして入る）
2. ターゲットファイル `hello.py` を確認
3. `gc sling hello-world/reviewer "..."` を自分の判断で発行
4. （プロンプトに書いてあれば）作業の完了を mail で人間へ報告

reviewer 側は polecat として一時セッションが立ち、レビュー後に idle → 自動シャットダウンの流れをたどる。

### ヒント

- `crew`（常駐）と `polecat`（一時）の使い分けは `pack.toml` の `[[named_session]]` の有無で決まる。`mode = "always"` のものが crew。
- mayor のプロンプトに「mail を確認したら委譲先を決め、`gc sling` を使って依頼すること」と明記しておくと、判断が安定する。
- `gc session logs mayor -f` を別ターミナルで開いておくと、mayor の発言と reviewer の発言を横並びで観察できる。

---

## シナリオ 4: formula でマルチステップワークフローを書く

**カテゴリ:** Core Workflow
**難易度:** 中級
**使うコマンド:** [`gc formula list`](./COMMANDS.md#gc-formula), [`gc formula show`](./COMMANDS.md#gc-formula), [`gc formula cook`](./COMMANDS.md#gc-formula), [`gc sling --formula`](./COMMANDS.md#gc-sling-target-bead-or-formula-or-text)

### 状況

「設計 → 実装 → テスト → レビュー」の決まったフローを毎回テキストで指示するのが面倒。TOML で一度書いてしまい、`gc sling` で発火するだけにしたい。`design` の後に `implement` と `test` が並列で動き、両方終わってから `review` に進むような依存関係を表現したい。

### 手順

1. city の `formulas/` に新しい formula を書く。
   ```bash
   cd ~/first-city
   mkdir -p formulas
   cat > formulas/feature-flow.toml <<'EOF'
   formula = "feature-flow"
   description = "設計→実装→テスト→レビュー"

   [vars.title]
   description = "機能名"
   required = true

   [[steps]]
   id = "design"
   title = "{{title}} の設計を書く"

   [[steps]]
   id = "implement"
   title = "{{title}} を実装する"
   needs = ["design"]

   [[steps]]
   id = "test"
   title = "{{title}} のテストを書く"
   needs = ["design"]

   [[steps]]
   id = "review"
   title = "{{title}} の実装とテストをレビュー"
   needs = ["implement", "test"]
   EOF
   ```

2. 書いた formula が認識されているか確認する。
   ```bash
   gc formula list
   gc formula show feature-flow --var title="ログイン機能"
   ```
   `Steps (4)` と展開後のタイトルが表示されればコンパイル成功。

3. wisp として一発投入する場合（軽量、step が 1 エージェント完結）。
   ```bash
   cd ~/hello-world
   gc sling hello-world/claude feature-flow --formula --var title="ログイン機能"
   ```

4. molecule として cook して、各ステップを別エージェントに振りたい場合。
   ```bash
   cd ~/hello-world
   gc formula cook feature-flow --var title="ログイン機能"
   # Root: hw-2wx, ステップごとに hw-2wx.1, .2, ... が作られる
   gc sling hello-world/claude   hw-2wx.1   # design
   gc sling hello-world/claude   hw-2wx.2   # implement
   gc sling hello-world/reviewer hw-2wx.3   # test
   gc sling hello-world/reviewer hw-2wx.4   # review
   ```

5. 進捗を見る。
   ```bash
   bd show hw-2wx --watch
   gc convoy list
   ```

### 期待される結果

`gc formula show` がこんな感じの依存ツリーを描く:

```text
Formula: feature-flow

Variables:
  {{title}}: 機能名 (required)

Steps (4):
  ├── feature-flow.design: ログイン機能 の設計を書く
  ├── feature-flow.implement: ログイン機能 を実装する [needs: feature-flow.design]
  ├── feature-flow.test: ログイン機能 のテストを書く [needs: feature-flow.design]
  └── feature-flow.review: ログイン機能 の実装とテストをレビュー [needs: feature-flow.implement, feature-flow.test]
```

`implement` と `test` は `design` 完了後に並列で進む。`review` はその両方を待つ。

### ヒント

- 大きい formula は `[[steps.children]]` で子ステップにグループ化できる。`needs = ["backend"]` だけ書けば子の全完了を待つ。
- 条件分岐（`condition = "{{env}} == staging"`）と loop（`[steps.loop] count = 3`）と check（後段のシェル検証）も使える。詳しくは `docs/tutorials/05-formulas.md`。
- `gc sling --formula` は内部で wisp を作って 1 エージェントに渡すので、各ステップを別エージェントへ振りたい場合は `gc formula cook` + 個別 `gc sling` のほうが柔軟。

---

## シナリオ 5: order で定期ジョブを仕込む（cooldown / cron / event）

**カテゴリ:** Automation
**難易度:** 中級
**使うコマンド:** [`gc order list`](./COMMANDS.md#gc-order), [`gc order check`](./COMMANDS.md#gc-order), [`gc order run`](./COMMANDS.md#gc-order), [`gc order history`](./COMMANDS.md#gc-order)

### 状況

毎週月曜の朝にリリースノートをドラフトさせたい。同時に、PR がマージされるたび（イベント駆動）にテストを走らせたい。さらに、コードベースの lint は 30 秒おきにシェル直叩きで（agent を使わずに）回したい。これらを `order` で表現したい。

### 手順

1. city の `orders/` に order ファイルを作る。lint は exec、リリースノートは formula、merge-test は event 駆動。
   ```bash
   cd ~/first-city
   mkdir -p orders scripts
   ```

2. lint 用 exec order（30 秒 cooldown、agent 不要）。
   ```bash
   cat > orders/lint.toml <<'EOF'
   [order]
   description = "30秒ごとに変更ファイルへ lint を走らせる"
   trigger = "cooldown"
   interval = "30s"
   exec = "scripts/lint-changed.sh"
   timeout = "60s"
   EOF
   cat > scripts/lint-changed.sh <<'EOF'
   #!/usr/bin/env bash
   set -e
   echo "lint check at $(date) in $ORDER_DIR" >&2
   EOF
   chmod +x scripts/lint-changed.sh
   ```

3. 月曜 9 時のリリースノート order（cron）。先に `formulas/release-notes.toml` を用意しておく。
   ```bash
   cat > orders/release-notes.toml <<'EOF'
   [order]
   description = "週次リリースノート"
   formula = "release-notes"
   trigger = "cron"
   schedule = "0 9 * * 1"
   pool = "mayor"
   EOF
   ```

4. event 駆動の merge-ready チェック（任意の bead クローズで発火）。
   ```bash
   cat > orders/merge-ready.toml <<'EOF'
   [order]
   description = "PR が closed されたらマージ可否を点検"
   formula = "merge-ready"
   trigger = "event"
   on = "bead.closed"
   pool = "mayor"
   EOF
   ```

5. controller に再読み込みさせる（restart せずに済ませる）。
   ```bash
   gc reload
   ```

6. 状況を確認する。
   ```bash
   gc order list
   gc order check
   gc order show release-notes
   ```

7. 試しに手動で発火する。
   ```bash
   gc order run lint
   gc order history lint --limit 5
   ```

### 期待される結果

```text
$ gc order list
NAME            TYPE     TRIGGER   INTERVAL/SCHED  TARGET
lint            exec     cooldown  30s             -
release-notes   formula  cron      0 9 * * 1       mayor
merge-ready     formula  event     bead.closed     mayor

$ gc order check
NAME            TRIGGER   DUE  REASON
lint            cooldown  yes  never run
release-notes   cron      no   next fire in 3d 14h
merge-ready     event     no   no event since cursor
```

`gc order run lint` を打つと scripts/lint-changed.sh が走り、`gc order history` に履歴が残る。

### ヒント

- exec order には `pool` を書けない（agent を経由しないから）。formula order だけ `pool` を持てる。
- 同じ `lint` order を pack 経由で取り込んでいるとき、`city.toml` に `[orders] skip = ["lint"]` と書けば city レベルで無効化できる。
- パラメータだけ変えたいときは `[[orders.overrides]]` で `interval` や `pool` を上書きできる（pack の order を直接編集せずに済む）。
- 重複防止: 同じ order の前回起動 bead がまだ open なら、trigger が due でも skip される（ピックアップが詰まる事故防止）。

---

## シナリオ 6: 既存パックを取り込んで複数 rig に共通展開する

**カテゴリ:** Advanced Usage
**難易度:** 上級
**使うコマンド:** [`gc import`](./COMMANDS.md#gc-import), [`gc pack`](./COMMANDS.md#gc-pack), [`gc rig add`](./COMMANDS.md#gc-rig), [`gc config explain`](./COMMANDS.md#gc-config)

### 状況

社内に「dev-ops」pack（test-suite formula と関連 order を持つ）が共有 git リポジトリに置いてある。フロントエンドとバックエンドの 2 リポジトリ（rig）両方に同じ pack を当て、しかしフロントだけ test 周期を縮めたい。pack そのものは編集せず、city 側の override で済ませたい。

### 手順

1. city の `pack.toml` で pack を import する。GitHub URL とブランチ参照を直接書ける。
   ```toml
   # pack.toml
   [pack]
   name = "team-orchestrator"
   schema = 2

   [imports.dev_ops]
   source = "github.com/example/dev-ops-pack@main"
   ```

2. 入手と検証。
   ```bash
   gc pack fetch          # clone / update
   gc import install      # pack.toml + packs.lock を実体化
   gc import list
   gc import why dev_ops  # なぜこのインポートがあるか
   ```

3. 2つの rig を登録する。
   ```bash
   cd ~/team-orchestrator
   gc rig add ~/api
   gc rig add ~/frontend
   ```

4. `city.toml` で各 rig に pack を適用し、フロントだけ override を書く。
   ```toml
   # city.toml
   [workspace]
   provider = "claude"

   [[rigs]]
   name = "api"

   [rigs.imports.dev_ops]
   source = "github.com/example/dev-ops-pack@main"

   [[rigs]]
   name = "frontend"

   [rigs.imports.dev_ops]
   source = "github.com/example/dev-ops-pack@main"

   [[orders.overrides]]
   name = "test-suite"
   rig = "frontend"
   interval = "1m"
   ```

5. 反映と検証。
   ```bash
   gc reload
   gc order list
   # NAME        TYPE     TRIGGER   INTERVAL/SCHED  TARGET
   # test-suite  formula  cooldown  5m              worker         <- pack 既定
   # test-suite  formula  cooldown  5m              api/worker     <- api 用
   # test-suite  formula  cooldown  1m              frontend/worker  <- override
   ```

6. 設定の出処を確認する。
   ```bash
   gc config explain
   # provenance 注釈付きで「この値はどの pack のどのファイルから来たか」が分かる
   ```

### 期待される結果

`gc order list` で 3 種類の `test-suite` が並ぶ:

- city 直下の `test-suite`（pack 既定、5 分）
- `api/test-suite`（rig 適用、5 分）
- `frontend/test-suite`（rig 適用 + override、1 分）

`gc order run test-suite --rig frontend` で個別実行できる。

### ヒント

- pack は編集せずに override で吸収するのが基本。pack を編集するのは「全 city 共通に変えたい」ときだけ。
- ローカル開発中の pack は `source = "./packs/dev-ops"` と相対パスで書ける。
- `[[rigs.patches]]` を使うと、pack に含まれる agent の `provider` や `idle_timeout` を rig 単位で書き換えられる。例: `[[rigs.patches]] agent = "dev_ops.tester"` `provider = "codex"`。

---

## シナリオ 7: handoff で長時間エージェントを綺麗に交代する

**カテゴリ:** Recovery
**難易度:** 中級
**使うコマンド:** [`gc handoff`](./COMMANDS.md#gc-handoff-subject-message), [`gc session`](./COMMANDS.md#セッション管理), [`gc runtime`](./COMMANDS.md#gc-runtime)

### 状況

mayor のセッションがメモリを食って動きが鈍くなってきた。会話履歴も長すぎて Claude のコンテキスト上限に近づいている。状態を取りこぼさずに「今何をしていたか」を mail で伝言してから、controller に session を再起動させたい。

### 手順

1. 今の状態を peek で確認する。
   ```bash
   gc session peek mayor --lines 30
   ```

2. handoff 実行。subject と body を渡すと、mail として現セッションに記録され、closure メッセージとして次のセッションに引き継がれる。
   ```bash
   gc handoff "状態引き継ぎ" "ログイン機能のレビューを reviewer に振り済み。完了確認待ち。"
   ```

3. controller が session を入れ替えるのを待つ。
   ```bash
   gc session list
   # mayor の状態が closing → 新しい session が active に
   ```

4. 新しい mayor の様子を確認する。`bd ready` と `gc mail check` を実行して、引き継ぎメールを読んでいるか peek で見る。
   ```bash
   gc session peek mayor --lines 20
   ```

5. もし controller が起こしてくれない場合の手動操作（通常は不要）。
   ```bash
   gc runtime drain mayor          # graceful shutdown を依頼
   gc runtime drain-check mayor    # 状態確認
   gc runtime request-restart      # エージェント側から restart を依頼
   ```

### 期待される結果

旧 mayor は handoff 内容を mail として残し、closing 状態へ移行する。controller の次の tick で新 mayor が起動し、最初のターンで引き継ぎ mail を hook 経由で読み、引き継ぎ前と一貫した会話を続ける。

### ヒント

- handoff は「会話履歴のリセット」と「mail による状態伝達」をワンセットで行う。会話のコンテキスト圧縮テクニックよりも頑健。
- `gc session pin <id>` で「絶対に殺さない」セッションを pin できる。逆に `gc session reset` は強制リセット。
- 何もしなくても controller は idle_timeout に達したセッションを自動でシャットダウンするので、agents/<name>/agent.toml に `idle_timeout = "4h"` 等を入れて運用負荷を下げるのが定石。
- handoff が頻繁に発生するエージェントは、プロンプトテンプレートに「次の自分に何を伝えるか」を書く欄を設けておくと品質が安定する。
