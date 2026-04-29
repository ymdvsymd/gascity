# Gas City — クイックスタート

**生成日:** 2026-04-29
**対象バージョン:** gascity v1.0.0+
**所要時間:** 約 10 分（前提依存のインストール含めず約 5 分）

---

## 前提条件

- macOS、または Linux（apt 系）/ WSL2 が利用できる
- Homebrew が使える環境（macOS / Linux いずれも推奨）
- Claude Code (`claude`)、Codex (`codex`)、Gemini (`gemini`) のいずれか1つの provider CLI が PATH に通っており、ログイン済み
- 任意のディレクトリに作業フォルダを作る権限がある

> Oh My Zsh の `git` プラグインを有効にしている場合、`gc` は `git commit --verbose` のエイリアスに上書きされる。詳細と恒久対処は [TROUBLESHOOTING.md#oh-my-zsh-の-gc-エイリアス](./TROUBLESHOOTING.md#oh-my-zsh-の-gc-エイリアス) を参照。それまでは `command gc ...` で実行する。

---

## Step 1: インストール

最も簡単で再現性が高いのは Homebrew のタップ経由のインストールである。`tmux`・`jq`・`git`・`dolt`・`bd`・`flock` の依存もまとめて入る。

```bash
brew install gastownhall/gascity/gascity
gc version
```

期待される出力（バージョン番号は時期により異なる）:

```text
1.0.0
```

`gc` が `git commit` を起動した場合は Oh My Zsh のエイリアスが先勝ちしている。`command gc version` で一旦バイパスし、後述のトラブルシューティングで恒久対処する。

dolt や bd を入れたくない（チュートリアル用途で済ませたい）場合は、ファイルベースの beads provider に切り替える:

```bash
export GC_BEADS=file
```

これで以降のコマンドは dolt / bd / flock を必要としない。

---

## Step 2: city を作る

city はオーケストレーション環境一式を入れる作業ディレクトリ。`gc init` がスキャフォールドし、マシン全体の supervisor に登録し、controller を起動するまでを一括で行う。

```bash
gc init ~/bright-lights --provider claude
cd ~/bright-lights
```

`--provider` を省略すると対話プロンプトでテンプレートと provider を選ぶ画面が出る。Codex を使うなら `--provider codex`、Gemini なら `--provider gemini`。

`gc init` の終了直後、city はすでに走っている。確認:

```bash
gc cities
gc status
```

`gc status` の `Controller` 行に PID が出ていれば controller プロセスが立ち上がっている。

city ディレクトリの中身を覗くと、最低限の構成が見える:

```bash
ls
# agents  city.toml  pack.toml  ...
cat city.toml
# [workspace]
# provider = "claude"
cat pack.toml
# [pack]
# name = "bright-lights"
# schema = 2
#
# [[agent]]
# name = "mayor"
# prompt_template = "agents/mayor/prompt.template.md"
#
# [[named_session]]
# template = "mayor"
# mode = "always"
```

`mayor` という常駐エージェントが既定で1つ用意されている。これは「いつでも attach できる pack の慣習エージェント」であって、SDK レベルで特別扱いされてはいない。

---

## Step 3: rig として作業対象プロジェクトを登録

エージェントが実際に作業するのは「rig」と呼ばれる外部プロジェクトディレクトリ。git 初期化済みのフォルダを rig として city に登録する。

```bash
mkdir ~/hello-world
cd ~/hello-world
git init
gc rig add ~/hello-world
```

登録すると、その rig 専用の bead ID プレフィックス（例: `hw-` で始まる）と beads データベースが用意される。

```bash
gc rig list
```

city ディレクトリの `city.toml` に `[[rigs]]` エントリが、`.gc/site.toml` にこのマシン上での実パスバインディングがそれぞれ書かれる（自動）。

---

## Step 4: 最初のスリングを投げる

rig 内に居るとき、`<rig>/<provider>` の形で実行先を指定できる。`gc sling` は bead を作成し、対応する provider のセッションを起動し、そのセッションにタスクを渡す。

```bash
cd ~/hello-world
gc sling hello-world/claude "hello.py を作って Hello World を出力してください"
```

期待される出力（ID は実行ごとに変わる）:

```text
Created hw-ff9 — "hello.py を作って Hello World を出力してください"
Attached wisp hw-6yh (default formula "mol-do-work") to hw-ff9
Auto-convoy hw-4tl
Slung hw-ff9 → hello-world/claude
```

`hw-ff9` がメインの bead、`hw-6yh` がそれを実行する wisp（ephemeral formula 実行）、`hw-4tl` がグルーピング用の自動 convoy。

---

## Step 5: 進行を眺める

`bd show` の `--watch` オプションで bead の状態が更新されるたび表示が更新される:

```bash
bd show hw-ff9 --watch
```

`STATUS` が `closed` になれば作業完了。Ctrl+C で抜けて、結果を確認:

```bash
ls
# hello.py
cat hello.py
```

別の見方として、ライブセッションを覗くこともできる:

```bash
gc session list
gc session peek <session-id>
```

`mayor` 常駐セッションに乗り込んで対話したい場合:

```bash
gc session attach mayor
# Ctrl-b d でデタッチ（tmux 標準）
```

---

## Step 6: 後片付け（任意）

city を完全に止めたいときは:

```bash
gc stop ~/bright-lights
```

二度と使わない city はレジストリから抹消できる:

```bash
gc unregister ~/bright-lights
```

ただし dolt 管理の beads データはディレクトリの `.gc/` 配下に残る。完全に消すならディレクトリごと削除する。

---

## 次のステップ

- 日常で使うコマンドを一通り知る → [コマンドリファレンス](./COMMANDS.md)
- カスタムエージェントを定義したい → [USE-CASES.md#シナリオ-2-カスタムレビュアエージェントを定義する](./USE-CASES.md#シナリオ-2-カスタムレビュアエージェントを定義する)
- マルチステップワークフローを書きたい → [USE-CASES.md#シナリオ-4-formula-でマルチステップワークフローを書く](./USE-CASES.md#シナリオ-4-formula-でマルチステップワークフローを書く)
- 定期実行を仕込みたい → [USE-CASES.md#シナリオ-5-order-で定期ジョブを仕込む](./USE-CASES.md#シナリオ-5-order-で定期ジョブを仕込む)
- 詳細設定を理解したい → [設定ガイド](./CONFIGURATION.md)
- 動かない / 何かおかしい → [トラブルシューティング](./TROUBLESHOOTING.md)
