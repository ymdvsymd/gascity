# Gas City — トラブルシューティング

**生成日:** 2026-04-29
**対象バージョン:** gascity v1.0.0+

何かおかしいときの最初の一手は **`gc doctor`**。city の構造、設定、依存ツール、ランタイムの健全性を一括チェックして問題箇所を出してくれる。

```bash
gc doctor
gc doctor --verbose   # 詳細
gc doctor --fix       # 自動修復を試みる
```

それでも分からない場合は、以下のセクションでよくある原因に当たる。

---

## インストール・PATH

### `gc: command not found`

バイナリは入っているが PATH に通っていない。

- Homebrew の場合: `$(brew --prefix)/bin` が `$PATH` に入っているか `which gc`、`echo $PATH` で確認する。
- 直接ダウンロードした場合: `~/.local/bin` か `/usr/local/bin` に置く。
  ```bash
  install -m 755 gc ~/.local/bin/gc
  ```
- fish / nushell など非標準 shell では設定ファイルが違う。`config.fish` や nushell の env を見直す。

### `gc version` が `git commit` を起動する（Oh My Zsh の `gc` エイリアス）

**現象:** `gc version` を打つと `Enumerating objects...` のように git のメッセージが流れる。

**原因:** Oh My Zsh の `git` プラグインが `gc` を `git commit --verbose` のエイリアスにしている。

**対処（一時的）:** `command` で alias をバイパス。
```bash
command gc version
```

**対処（恒久）:** `~/.zshrc` の Oh My Zsh ロード**後**に unalias を入れる。
```bash
source "$ZSH/oh-my-zsh.sh"
unalias gc 2>/dev/null
```

または `~/.oh-my-zsh/custom/gascity.zsh` に同じ行を書く（custom 配下は本体プラグインの後にロードされる）。

`unalias` を `source "$ZSH/oh-my-zsh.sh"` の前に書くと、後から git プラグインが alias を再生成して効かなくなる点に注意。

---

## 依存ツール

### `dolt` のバージョンが古い

```bash
dolt version
```

Gas City は dolt 1.86.1 以降を要求する。古いと managed dolt の起動が失敗する。

```bash
brew upgrade dolt   # macOS / Linux Homebrew
```

または GitHub releases から最新を取得して PATH に置く。

### `bd` のバージョンが古い

```bash
bd version
```

1.0.0 以降が必要。`brew upgrade beads` または GitHub releases から取得。

### macOS で `flock` がない

macOS は `flock` を同梱しない。Homebrew で入れる:

```bash
brew install flock
```

flock を使いたくない場合は、file backend に切り替えれば不要:

```bash
export GC_BEADS=file
```

または `city.toml` に:

```toml
[beads]
provider = "file"
```

### dolt / bd / flock を入れたくない

すべての beads 機能はファイルベースで代替できる。チュートリアルや短期検証ならこれで十分:

```bash
export GC_BEADS=file
```

`gc init` 時から file backend で始めるなら、生成された `city.toml` に上記の `[beads]` セクションを追記する。

---

## city / supervisor 関連

### `gc init` が「supervisor service の登録に失敗」

launchd / systemd の権限問題。`gc service list` で既存サービスが残っていないか確認する。`~/Library/LaunchAgents/com.gascity.supervisor.plist`（macOS）または `~/.config/systemd/user/gascity-supervisor.service`（Linux）が存在する場合、消してから再 `gc init`。

### `gc status` で controller PID が出ない

controller プロセスが死んでいる、または supervisor が登録だけして起動していない。

```bash
gc supervisor status
gc supervisor start
gc start <city-path>
```

それでも起動しない場合は `gc doctor --verbose` でランタイム診断。`.gc/logs/` 配下の controller ログも確認する。

### city が壊れている / `.gc/` 配下が破損

最終手段としてレジストリから外し、`.gc/` を消して `gc init` し直す。

```bash
gc unregister <city-path>
rm -rf <city-path>/.gc
gc init <city-path>
```

ただし dolt 管理の bead データ（`.gc/dolt/`）も消えるので、必要なら先に `gc bd export` 等で取り出しておく。

---

## bead / sling / hook 関連

### `gc sling` 後にエージェントが何もしない

**よくある原因と確認順序:**

1. session が立ち上がっているか。
   ```bash
   gc session list
   ```
   `creating` のまま固まっていれば provider 起動失敗の可能性。

2. provider CLI が直接実行できるか。
   ```bash
   claude --version
   codex --help
   ```
   ログイン切れや PATH 不一致を確認する。

3. hook が動いているか。
   ```bash
   gc hook --inject <agent>
   ```
   何も出力されないなら hook 未インストール。`install_agent_hooks` 設定を確認する。
   ```bash
   gc config show | grep install_agent_hooks
   ```

4. bead が見えているか。
   ```bash
   bd ready
   bd show <bead-id>
   ```
   `STATUS: open`、`assignee` が空 or 期待 agent になっているか。

5. metadata で routed_to が正しいか。
   ```bash
   bd show <bead-id> --json | jq '.metadata'
   ```
   `gc.routed_to=my-project/claude` のようになっているはず。違っていれば `gc sling` の target 指定に問題。

### bead が `blocked` のままになる

dependency が解決していない。

```bash
bd show <bead-id>
# BLOCKS / BLOCKED-BY セクションをチェック
```

依存先の bead が close されると自動で `open` に戻る。手動で外したいときは:

```bash
bd dep <bead-id> --remove-blocks <other-id>
```

### convoy が auto-close しない

通常、子が全 close になれば convoy も自動で close する。`owned` ラベルが付いていると skip される。

```bash
gc convoy check          # 強制 reconcile
gc convoy land <id>      # owned convoy を terminate
```

---

## session 関連

### tmux session が見つからない（attach 失敗）

```bash
gc session list
tmux ls                    # 直接確認
```

`gc` は GC 専用 tmux ソケットを使うことが多いので、`tmux -L gascity ls` のような形でないと出ない場合がある。`gc session list` の結果を信用するのが確実。

### 「kill -9 されたエージェント」が controller によって即座に再起動される

それは正常動作（`max_restarts` 上限まで）。同じセッションを永続的に止めたいなら:

```bash
gc agent suspend <name>
# 再開
gc agent resume <name>
```

### session のログが大きすぎて読めない

```bash
gc session logs <id> --tail 50      # 末尾 50 件
gc session logs <id> -f             # follow
```

`--tail` は表示する transcript エントリ数（v1.0.0 以降の挙動）。HTTP API 経由の `tail` パラメータは compaction セグメント単位で、CLI とは数え方が違う点に注意。

---

## order / formula 関連

### order が定刻に起動しない

```bash
gc order list                  # enabled な順序を確認
gc order check                 # 各 order の DUE 状態
gc order history <name>        # 過去の発火履歴
```

cron は最大 1 分に 1 回しか fire しない。前回起動の bead がまだ open なら trigger を満たしていても skip される（duplicate prevention）。

condition trigger は同期実行で 10 秒タイムアウト。重い check は order 全体の評価を遅らせる。

### order 名の重複（rig 単位とで混同）

city 直下と rig 配下で同じ formula 名を使うと、`gc order list` に複数行出る。`gc order run` / `show` には `--rig <name>` を付ける。

```bash
gc order show test-suite --rig frontend
gc order run  test-suite --rig frontend
```

### formula のコンパイルエラー

```bash
gc formula show <name>          # 静的解析でエラーが出る
gc formula show <name> --var KEY=VALUE
```

required な変数が抜けているか、`needs` がサイクルを作っているか、`condition` の文法ミスのいずれか。`gc config explain` ではなく `gc formula show` がコンパイル本体。

---

## hook / mail 関連

### `gc mail send` したのに相手が読まない

mail は「送ったときには届かない」設計。次のターンの hook で初めて届く。手動でターンを開かせるには nudge する:

```bash
gc session nudge <agent> "メール確認をお願いします"
```

または `gc mail check --inject` を hook が実行しているか確認する（provider 設定の問題のことが多い）。

### Claude Code の hook が動かない

`<city>/.gc/settings.json` が正しく書かれているか確認。`install_agent_hooks = ["claude"]` を `[workspace]` または該当 `[[agent]]` に書く。`gc reload` で再読み込み。

```bash
gc config show | grep -A5 install_agent_hooks
```

provider 側の `~/.claude/` 配下の settings と衝突していないかも要確認。Claude Code は `settings.json` を多層解決する。

---

## supervisor / service 関連

### `gc service restart` が必要なのはいつ？

`brew upgrade gascity` して `gc` バイナリを更新したとき。supervisor の plist / service ファイルを実体化された binary に紐付け直すため。

```bash
gc service restart
```

`gc start <city>` を打つと service ファイルが自動で再生成されるので、複数 city があるなら一つ start すれば十分。

### supervisor を完全停止したい

```bash
gc supervisor stop
gc service list      # service が無効化されたか確認
```

完全消去（再 `gc init` で復活する）:

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.gascity.supervisor.plist
rm ~/Library/LaunchAgents/com.gascity.supervisor.plist
```

Linux は `systemctl --user stop gascity-supervisor` および `disable`。

---

## dashboard 関連

### `gc dashboard serve` が起動するがブラウザに出ない

ポートとバインドアドレスを確認:

```bash
gc dashboard serve --port 8742
```

`city.toml` の `[api]` セクションに `enabled = false` が書かれていないか、`auth_token_file` のパスが書き込み可能か確認。

---

## ファイル整合性チェック

### `make dashboard-check` が失敗する（contributor 向け）

`internal/api/openapi.json`、`docs/schema/openapi.json`、`cmd/gc/dashboard/web/src/generated/` の OpenAPI 由来ファイルが同期していない。

```bash
make dashboard-check
# CI が失敗するなら詳細ログを見て、再生成スクリプトを実行
```

### `TestOpenAPISpecInSync` 失敗

`go test ./...` で OpenAPI スペックの不一致が報告された場合、`internal/api/openapi.json` を再生成して commit する。

---

## FAQ

### Q: file backend と bd backend の違いは？

A: file backend は `<city>/.gc/beads.json` に JSON で書くだけのシンプルな実装。トランザクション・並列書き込み・履歴がない。bd backend は dolt（Git ライクな分散 SQL DB）を使い、versioned な永続化と CLI 経由の高度なクエリを提供する。本格運用は bd を推奨。

### Q: pack を更新したのに `gc reload` で反映されない

A: import した pack（`packs/<name>/`）は cache されている。`gc pack fetch` で update したあと、`gc import upgrade <name>` を打つ。それでも変わらないときは `gc import remove` → `gc import install` で再インストール。

### Q: 同じ city を別マシンで動かしたい

A: `pack.toml`、`city.toml`、`agents/`、`formulas/`、`orders/` を git で同期し、新マシンで `gc init` ではなく `gc register <path>` する。`.gc/site.toml` はマシン固有なので新マシンで `gc rig add <path>` で書き直す。`gc start <path>` で起動。

### Q: provider のレート制限に当たった

A: 各 agent の `idle_timeout` を長くして頻繁な起動を抑える、`[pool]` の max を下げる、order の `interval` を伸ばす、のいずれか。複数 provider に分散したい場合は `[[rigs.patches]]` で agent ごとに provider を散らす。

### Q: ログをどこに出している？

A: city 配下 `.gc/logs/` に controller / supervisor のログ。session の会話は dolt（または file backend のフィールド）に永続化。`gc session logs <id>` でアクセスする。

### Q: city ディレクトリを移動したい

A: 単純移動は壊れる。`gc unregister <old>` → 物理移動 → `gc register <new>` の手順。rig も `.gc/site.toml` を編集するか、`gc rig remove` → `gc rig add` で更新する。

---

## それでも解決しない場合

```bash
gc doctor --verbose > doctor.log 2>&1
```

を実行し、出力と OS / アーキテクチャ情報を添えて [gastownhall/gascity/issues](https://github.com/gastownhall/gascity/issues) に報告する。`engdocs/contributors/reconciler-debugging.md` には controller / session reconciler のインシデント解析ワークフローが書かれており、`gc trace` の出力を取りながら追跡できる。

---

## 関連ドキュメント

- [OVERVIEW.md](./OVERVIEW.md) — どの層が何を担当するかの全体像
- [COMMANDS.md](./COMMANDS.md) — `gc doctor` / `gc supervisor` / `gc runtime` などの詳細
- [CONFIGURATION.md](./CONFIGURATION.md) — 設定が効かないときに見るべき場所
