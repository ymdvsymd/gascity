# Gas City — トラブルシューティング

**生成日:** 2026-05-13
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

### `gc city status` や `gc hook` が大規模 bead 集合で固まる / 「loading session snapshot timed out after …」が stderr に出る

**現象:** 10,000 件級の bead を抱える city で `gc city status` や `gc hook` を打つと応答が極端に遅い、あるいは stderr に次のような警告が混ざる。

```
gc status: loading session snapshot timed out after 3s; continuing with runtime-only status
```

（実装は `cmd/gc/cmd_citystatus.go:196` で `statusSessionSnapshotTimeout` 3 秒超過時に上記を出す。PR の解説では「partial snapshot due to deadline」という言い回しで紹介されたが、ユーザに見える実際の文言は上記。）

**原因:** これらのコマンドは city 全体を 1 回 scan して状態を集める設計だが、bead 数が桁違いに増えると I/O が膨らんで budget を超える。PR #2005（2026-05-13 マージ）で `cmd_citystatus.go` / `cmd_hook.go` / `doctor_routed_to_checks.go` / `doctor_session_model.go` に bounded scan が入り、budget 超過時は途中までの状態（runtime 観測だけ）で返り、上記の timeout 警告を stderr に出すようになった（ハングではなく早期返却）。

**対処:**

- 警告が出ている時点で得られた結果は「完全ではないが runtime から観測できた範囲」と読む。bead 由来の情報（active sessions / suspended の集計など）は欠落しうる。
- 古い bead を整理する: `bd compact` で長期 closed bead を semantic summary に畳む、`bd close` し忘れている残骸を一括 close する。
- 必要な情報が一部しか出ない場合は `bd ready --limit N` のように対象を絞った beads CLI を直接使う。
- `statusObservationTimeout`（750ms）と `statusSessionSnapshotTimeout`（3s）は実装側の変数。大規模 city で恒常的に詰まる場合は `gc trace` 採取の上、`engdocs/contributors/reconciler-debugging.md` に従って upstream に報告。

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

### 設定を反映しただけなのに session が drain される

**現象:** `gc reload` を打ったあと、走っていた session が closing → 再起動になり、作業途中の状態が一度切れる。

**原因:** デフォルトの `gc reload` は config drift を検出すると該当 session を drain する。これは「pack や agent.toml が変わったら確実に新設定で再起動させたい」想定の挙動。

**対処:** `.gc/settings.json` の細かな変更など、走っている作業を止めずに反映したい場合は `gc reload --soft` を使う。drift があっても session を drain せず、controller が新設定を取り込んでから次の tick で自然に揃える。なお、`assigned` な仕事を持っている pool session は、2026-04-29 以降は通常 reload でも drain を保留して別 tick へ送られる。

### pool 内のセッションが二重に同じ仕事をつかむ（dual-claim）

**現象:** `[pool]` を持つエージェントで、2 つの session が同じ bead に同時に着手しているように見える。`bd show <id>` の `assignee` がフラップする。

**原因:** 2026-04-29 以前は alias 経由でセッションを参照したときに identity 解決が一致せず、同一セッションが二重に claim されることがあった。

**対処:** 2026-04-29 以降の `gc` バイナリでは alias 認識の identity 解決が共通化され、この症状は出にくくなっている。古い city を使い続けているなら `brew upgrade gascity` + `gc service restart` で新しい reconciler に切り替える。発生中の dual-claim を解消するには、片方の session を `gc session kill` してから `bd update <id> --unassign` で reset し、改めて pool が拾うのを待つ。

### `named-always` のセッションが起動しない

**現象:** `pack.toml` や `city.toml` で `[[named_session]] mode = "always"` を指定したのに、対応する session が立ち上がらない。`bd ready` には仕事があるのに wake されない。

**原因:** controller が保持する `configured_named_identity` が何らかの理由で消えた場合、reconciler の以前のバージョンは wake をスキップしていた。

**対処:** 2026-04-29 以降は missing 時にも wake するようになった。古い city を使い続けているなら `brew upgrade gascity` + `gc service restart` で controller を入れ替える。それでも起きない場合は `gc config show | grep -A3 named_session` で設定が正しく反映されているか、`gc trace` で reconciler の判断を確認する。

### `resume_flag` が空で provider start が失敗する

**現象:** session の起動ログに「provider start returned without producing a session」のような表示が出て、最初の起動が空振る。tail に何も残らない。

**原因:** 一部の non-Claude provider で `resume_flag` が空のまま resume 動作に入ってしまうと、CLI が「resume すべき履歴がない」と判断して即時 exit する。

**対処:** 2026-04-29 以降は controller が「空 resume の空振り」を検知し、fresh start で 1 回 retry する。古い挙動が残っているなら `gc upgrade` 後に再起動。`provider profile` 側で resume_flag が供給されているかは [CONFIGURATION.md#provider-プロファイルと-resume-挙動](./CONFIGURATION.md#provider-プロファイルと-resume-挙動) を参照。

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

### Claude Code の chain agent が数分 idle すると応答待ちで停滞する

**現象:** mayor のような常駐エージェントが 3 分前後アイドル状態のまま動かなくなる。`gc session peek` を覗くと `[USER]` から「Are you still there?」「Provide a summary」といった away summary 系の質問が来ていて、エージェントがそこで止まっている。

**原因:** Claude Code には idle 時に away summary を要求する挙動があり、これが chain で連結したエージェント間で「次のターンが回らない」状態を生んでいた。

**対処:** 2026-04-29 以降の Gas City が `gc init` で配る `internal/hooks/config/claude.json` には `awaySummaryEnabled: false` が含まれており、この挙動を最初から抑止する。古い city でこの挙動が残っているなら、新テンプレを取り込む（`gc init` で新規生成された設定の `claude.json` をコピーする、または対応する Claude Code 設定キーを `false` に明示する）。

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

### Q: `bd dolt push` でリモートに送ろうとしたら archive mode に落ちた

A: managed dolt の remote が未設定の状態で JSONL export を呼ぶと、2026-04-29 以降は archive mode（ローカルに JSONL を残すだけ）へ自動切替する。以前は silent escalation で「成功したように見えて実は何もしていない」挙動だったが、現在は明示的に検出して報告する。リモートに送りたい場合は `bd dolt remote add <name> <url>` で送り先を構成してから再実行する。export のみで十分なら追加対応は不要で、`.beads/issues.jsonl` を git で運ぶ運用に切り替えれば良い。

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
