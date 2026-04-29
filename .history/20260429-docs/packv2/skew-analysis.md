# 仕様 vs. 実装スキュー分析 — 現行 Pack/City v2 望ましい状態

> 2026-04-12 に `docs/reference/config.md`（リリースブランチの Go 構造体由来の as-built）と調整済みの pack v2 仕様を比較して生成。**現行 Pack/City v2 の望ましい状態** — 理想的な最終状態ではなく、このリリース波で出荷すべきもの — を反映するため、フィールドごとのウォークスルーで改訂しました。

## カラーキー

| 色 | 意味 |
|-------|---------|
| 🟢 | リリースブランチで実装済み |
| 🔴 | リリースブランチで未実装 |
| 🟡 | NYI — 現行ロールアウトの計画にあり |
| 🔵 | NYI — 後の波 |

## フィールド配置の権威

### city.toml のみ（pack.toml では不正）

- `[[rigs]]` とすべての rig サブフィールド
- `[[patches.rigs]]`
- `[beads]`、`[session]`、`[mail]`、`[events]`、`[dolt]`
- `[daemon]`、`[orders]`、`[api]`
- `[chat_sessions]`、`[session_sleep]`、`[convergence]`
- `[[service]]`（pack が後の波で service を定義できるかは #657 で追跡）
- `max_active_sessions`（city 全体、現在は `[workspace]` 上）

### pack.toml のみ（city.toml では不正）

- `[pack]`（name、version、schema、requires_gc）
- `[imports]`
- `[defaults.rig.imports.<binding>]`

### 両方で正当（マージ時 city が勝つ）

- `[agent_defaults]`
- `[providers]`
- `[[named_session]]`
- `[[patches.agent]]`
- `[[patches.providers]]`

---

## 警告レベル

- **大声警告（loud warning）** — schema 2 の city に対して `gc start` / `gc config` のたびに送出されます。これらは V1 サーフェスで、ユーザーが新規コンテンツを書くべきではありません。
- **ささやき警告（soft warning）** — 1 度だけ送出されます。フィールドは受け入れられますが非推奨です。
- **ハードエラー（hard error）** — フィールド値が拒否されます。
- **サイレントに受け入れる（accept silently）** — このロールアウトでは警告なし。リリース後の非推奨化のために追跡されます。

**ファストフォロー（4 月 21 日のローンチ前）:** 以下のすべての soft/loud 警告のための非推奨警告インフラを実装します。

---

## City（トップレベル struct）

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟢 | `include` | []string、フラグメントをマージ | **維持。** フラグメントのみ（`-f` パス）。フラグメントが `[imports]`、`includes`、または `pack.toml` への参照を含む場合 → ハードエラー。 |
| 🟢 | `workspace` | 必須ブロック | **コンテナとして維持。** このロールアウト後に非推奨（#600）。サブフィールドは個別に下で扱う。 |
| 🟡 | `packs` | map[string]PackSource | **schema 2 で大声警告。** V1 メカニズム、`[imports]` + `packs.lock` を使用。 |
| 🟡 | `agent` | []Agent、必須 | **schema 2 で大声警告。** schema 2 では必須ではない — エージェントは `agents/<name>/` から発見される。 |
| 🟢 | `imports` | map[string]Import | **維持。** V2 メカニズム、動作中。 |
| 🟢 | `named_session` | []NamedSession | **維持。** pack.toml と city.toml の両方で正当、city が勝つ。 |
| 🟢 | `rigs` | []Rig | **city.toml に維持。** |
| 🟢 | `patches` | Patches | **維持。** `[[patches.agent]]` と `[[patches.providers]]` は両方で正当、city が勝つ。`[[patches.rigs]]` は city.toml のみ。 |
| 🟢 | `agent_defaults` | AgentDefaults | **維持。** pack.toml と city.toml の両方で正当、city が勝つ。サーフェスはそのまま（この波では拡張しない）。 |
| 🟢 | `providers` | map[string]ProviderSpec | **維持。** 両方で正当、city が勝つ。 |
| 🟡 | `formulas` | FormulasConfig | 下記の `[formulas].dir` を参照。 |
| 🟢 | `beads` | BeadsConfig | **city.toml に維持。** |
| 🟢 | `session` | SessionConfig | **city.toml に維持。** |
| 🟢 | `mail` | MailConfig | **city.toml に維持。** |
| 🟢 | `events` | EventsConfig | **city.toml に維持。** |
| 🟢 | `dolt` | DoltConfig | **city.toml に維持。** |
| 🟢 | `daemon` | DaemonConfig | **city.toml に維持。** |
| 🟢 | `orders` | OrdersConfig | **city.toml に維持。** |
| 🟢 | `api` | APIConfig | **city.toml に維持。** |
| 🟢 | `chat_sessions` | ChatSessionsConfig | **city.toml に維持。** |
| 🟢 | `session_sleep` | SessionSleepConfig | **city.toml に維持。** |
| 🟢 | `convergence` | ConvergenceConfig | **city.toml に維持。** |
| 🟢 | `service` | []Service | **city.toml に維持。** pack 定義の services は延期（#657）。 |

## Workspace サブフィールド

| ステータス | フィールド | As-built | 現行ロールアウト方針 | 後の宛先 |
|--------|-------|----------|--------------------|-----------------------|
| 🟢 | `name` | 任意の string | **移行済み。** 新規 `gc init` はマシンローカルアイデンティティを `.gc/site.toml` に書き込む。`gc doctor --fix` がレガシー値を `city.toml` から移行。ランタイムは登録された alias（supervisor 管理フロー）、次に site binding / レガシー config、最後にベース名の順に解決。 | `.gc/` site binding（#600） |
| 🟢 | `prefix` | string | **移行済み。** 新規 `gc init` はマシンローカルプレフィックスを `.gc/site.toml` に書き込む。`gc doctor --fix` がレガシー値を `city.toml` から移行。 | `.gc/` site binding（#600） |
| 🟡 | `provider` | string | **ささやき警告。** "代わりに `[agent_defaults] provider = ...` を使用。" | pack.toml の `[agent_defaults]` |
| 🟡 | `start_command` | string | **ささやき警告。** "代わりに `agent.toml` 内のエージェントごとの `start_command` を使用。" | エージェントごとの `agent.toml` |
| 🟡 | `suspended` | Boolean | **ささやき警告。** "代わりに `gc suspend`/`gc resume` を使用。" | `.gc/` site binding |
| 🟢 | `max_active_sessions` | Integer | **そのまま維持。** デプロイメントキャパシティ。 | `[workspace]` 解体時に city.toml のトップレベルフィールドへ |
| 🟢 | `session_template` | string | **そのまま維持。** デプロイメント。 | `[workspace]` 解体時に `[session]` へ |
| 🟡 | `install_agent_hooks` | []string | **ささやき警告。** "代わりに `[agent_defaults]` を使用。" | pack.toml の `[agent_defaults]` |
| 🟡 | `global_fragments` | []string | **ささやき警告。** "代わりに `[agent_defaults] append_fragments` または明示的な `{{ template }}` を使用。" | 削除（template-fragments で置換） |
| 🟡 | `includes` | []string | **schema 2 で大声警告。** V1 合成、`[imports]` を使用。 | 削除 |
| 🟡 | `default_rig_includes` | []string | **schema 2 で大声警告。** pack.toml の `[defaults.rig.imports.<binding>]` を使用。 | 削除 |

## エージェントフィールド

このロールアウトでは、`[[agent]]` は schema 2 で大声警告を受けます。下記のエージェントフィールドは、`agents/<name>/` 内の `agent.toml` で何が正当かを記述します。

### Convention で置換（TOML フィールドなし）

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟢 | `name` | 必須 string | **convention で置換。** ディレクトリ名がアイデンティティ。 |
| 🟢 | `prompt_template` | パス string | **convention で置換。** エージェントディレクトリ内の `prompt.template.md` または `prompt.md`。 |
| 🟢 | `overlay_dir` | パス string | **convention で置換。** `agents/<name>/overlay/` + pack 全体の `overlay/`。 |
| 🟢 | `namepool` | パス string | **convention で置換。** `agents/<name>/namepool.txt`。 |

### V1 残骸

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟡 | `dir` | string | **削除。** Rig スコーピングは import binding で処理。 |
| 🟡 | `inject_fragments` | []string | **schema 2 で大声警告。** `append_fragments` または明示的な `{{ template }}` を使用。 |
| 🟡 | `fallback` | Boolean | **schema 2 で大声警告。** 修飾名 + 明示的な優先順位を使用。 |

### agent.toml で正当

その他のすべてのエージェントフィールドは `agent.toml` で正当です。`[agent_defaults]` サーフェスはこの波ではそのまま（拡張なし）。

| ステータス | フィールド | 注記 |
|--------|-------|-------|
| 🟢 | `description` | |
| 🟢 | `scope` | `"city"` または `"rig"` |
| 🟢 | `suspended` | この波では agent.toml に残る。リリース後 `.gc/` へ |
| 🟢 | `provider` | |
| 🟢 | `start_command` | |
| 🟢 | `args` | |
| 🟢 | `session` | `"acp"` トランスポートオーバーライド |
| 🟢 | `prompt_mode` | |
| 🟢 | `prompt_flag` | |
| 🟢 | `ready_delay_ms` | |
| 🟢 | `ready_prompt_prefix` | |
| 🟢 | `process_names` | |
| 🟢 | `emits_permission_warning` | |
| 🟢 | `env` | |
| 🟢 | `option_defaults` | |
| 🟢 | `resume_command` | |
| 🟢 | `wake_mode` | |
| 🟢 | `attach` | |
| 🟢 | `max_active_sessions` | |
| 🟢 | `min_active_sessions` | |
| 🟢 | `scale_check` | |
| 🟢 | `drain_timeout` | |
| 🟢 | `pre_start` | |
| 🟢 | `on_boot` | |
| 🟢 | `on_death` | |
| 🟢 | `session_setup` | |
| 🟢 | `session_setup_script` | パスは pack ルートから解決 |
| 🟢 | `session_live` | |
| 🟢 | `install_agent_hooks` | agent_defaults を上書き |
| 🟢 | `hooks_installed` | |
| 🟢 | `idle_timeout` | |
| 🟢 | `sleep_after_idle` | |
| 🟢 | `work_dir` | |
| 🟢 | `default_sling_formula` | |
| 🟢 | `depends_on` | |
| 🟢 | `nudge` | |
| 🟢 | `work_query` | |
| 🟢 | `sling_query` | |

## AgentDefaults

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟢 | `model` | 存在 | **維持。** ランタイムでまだ自動適用されない。 |
| 🟢 | `wake_mode` | 存在 | **維持。** ランタイムでまだ自動適用されない。 |
| 🟢 | `default_sling_formula` | 存在 | **維持。** ランタイムで適用される。 |
| 🟢 | `allow_overlay` | 存在 | **維持。** ランタイムでまだ自動適用されない。 |
| 🟢 | `allow_env_override` | 存在 | **維持。** ランタイムでまだ自動適用されない。 |
| 🟢 | `append_fragments` | 存在 | **維持。** global_fragments/inject_fragments の移行ブリッジ。 |

この波では `[agent_defaults]` サーフェスの拡張なし。

## FormulasConfig

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟡 | `dir` | デフォルト `"formulas"` | **存在し、`"formulas"` と等しい場合はささやき警告。** それ以外の値に設定された場合はハードエラー。`formulas/` は固定の convention。 |

## Import

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟢 | `source` | 存在 | **維持。** |
| 🟢 | `version` | 存在 | **維持。** |
| 🟢 | `export` | 存在 | **維持。** |
| 🟢 | `transitive` | 存在 | **維持。** |
| 🟢 | `shadow` | 存在 | **維持。** |

すべての Import フィールドは仕様と一致。変更不要。

## Rig

| ステータス | フィールド | As-built | 現行ロールアウト方針 | 後の宛先 |
|--------|-------|----------|--------------------|----|
| 🟢 | `name` | 必須 | **city.toml に維持。** | |
| 🟢 | `path` | 必須 | **city.toml に維持。** | `.gc/site.toml`（#588） |
| 🟢 | `prefix` | string | **city.toml に維持。** | `.gc/`（#588） |
| 🟢 | `suspended` | Boolean | **city.toml に維持。** | `.gc/`（#588） |
| 🟡 | `includes` | []string | **schema 2 で大声警告。** `[rigs.imports]` を使用。 | 削除 |
| 🟢 | `imports` | map[string]Import | **city.toml に維持。** | |
| 🟢 | `max_active_sessions` | Integer | **city.toml に維持。** | |
| 🟡 | `overrides` | []AgentOverride | **ささやき警告。** "代わりに `patches` を使用。" 両方とも受理。 | 削除 |
| 🟢 | `patches` | []AgentOverride | **city.toml に維持。** V2 名。 | |
| 🟢 | `default_sling_target` | string | **city.toml に維持。** | |
| 🟢 | `session_sleep` | SessionSleepConfig | **city.toml に維持。** | |
| 🟡 | `formulas_dir` | string | **schema 2 で大声警告。** rig スコープ import を代わりに使用。 | 削除 |
| 🟢 | `dolt_host` | string | **city.toml に維持。** | |
| 🟢 | `dolt_port` | string | **city.toml に維持。** | |

## AgentOverride / AgentPatch

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟡 | `inject_fragments` | 存在 | **大声警告。** V1 残骸。 |
| 🟡 | `inject_fragments_append` | 存在 | **大声警告。** V1 残骸。 |
| 🟢 | `prompt_template` | パス string | **この波では維持。** リリース後: `patches/` 経由の convention ベース。 |
| 🟢 | `overlay_dir` | パス string | **この波では維持。** リリース後: convention ベース。 |
| 🟢 | `dir` + `name` ターゲティング（AgentPatch） | 存在 | **この波では維持。** 修飾名ターゲティングは既に動作。 |
| 🟢 | その他のすべての override フィールド | 存在 | **維持。** |

## PackSource

| ステータス | フィールド | As-built | 現行ロールアウト方針 |
|--------|-------|----------|--------------------|
| 🟡 | （構造体全体） | 存在 | **schema 2 で大声警告。** V1 メカニズム、`[imports]` + `packs.lock` を使用。 |

---

## 仕様機能 — 実装状況

| ステータス | 概念 | 仕様の場所 | 注記 |
|--------|---------|--------------|-------|
| 🟢 | `[imports]` 解決 | doc-pack-v2、doc-loader-v2 | pack.go の `ExpandCityPacks` |
| 🟢 | Convention によるエージェント発見（`agents/<name>/`） | doc-agent-v2 | agent_discovery.go の `DiscoverPackAgents` |
| 🟢 | `pack.toml` の独立パース | doc-loader-v2 | compose.go が pack.toml を city.toml と並行で読む |
| 🟢 | 両ファイルでの `[agent_defaults]` | doc-pack-v2 | 合成パイプライン経由で動作 |
| 🟢 | `prompt.template.md` convention | doc-agent-v2 | agent_discovery.go が発見 |
| 🟢 | `agents/<name>/overlay/` convention | doc-agent-v2 | agent_discovery.go が発見 |
| 🟢 | `agents/<name>/namepool.txt` convention | doc-agent-v2 | agent_discovery.go が発見 |
| 🟢 | `per-provider/` overlay フィルタリング | doc-agent-v2 | overlay.go の `CopyDirForProvider` |
| 🟢 | `template-fragments/` 発見 | doc-agent-v2 | prompt.go の Pack + per-agent レベル |
| 🟢 | AgentDefaults の `append_fragments` | doc-agent-v2 | ランタイムで適用 |
| 🟢 | 修飾名 patch ターゲティング | doc-agent-v2 | patch.go の `qualifiedNameFromPatch` |
| 🟢 | Import の `shadow` フィールド | doc-pack-v2 | pack.go の Warning/silent ロジック |
| 🟢 | `orders/` トップレベル発見 | doc-directory-conventions | orders/discovery.go の `discoverFlatFiles` |
| 🟢 | `commands/` convention 発見 | doc-commands | command_discovery.go の `DiscoverPackCommands` |
| 🟢 | トップレベル `scripts/` サーフェスなし / `ScriptLayers` ランタイム shim なし | doc-directory-conventions、doc-loader-v2 | 実装済み。ランタイムは `ScriptLayers` を収集せず、`<city>/scripts` をマテリアライズしない。起動パスは古いバージョンが残した stale な symlink のみのアーティファクトを刈り取るのみ。 |
| 🔴 | `[defaults.rig.imports]` ローダサポート | doc-pack-v2 | Migrate ツールが書き込むがローダは無視 |
| 🟢 | `gc register --name` フラグ | doc-pack-v2 | 実装済み。現行ロールアウトでは選択された登録名をマシンローカル supervisor レジストリに保存し、`city.toml` を書き直さない。フラグなしの登録は、有効な city アイデンティティ（site binding / レガシー config / ベース名）を使い、選択された名前のみをレジストリに保存する。 |
| 🔴 | `patches/` ディレクトリ convention | doc-agent-v2 | 未実装 |
| 🔴 | `skills/` pack 発見 | doc-agent-v2 | 最初のスライスは現在の city pack のみ、リスト表示のみ。imported pack カタログは後 |
| 🔴 | `mcp/` TOML 抽象化 | doc-agent-v2 | 最初のスライスは現在の city pack のみ、リスト表示のみ。provider プロジェクションは後 |
| 🟢 | workspace アイデンティティ + rig パスバインディング用の `.gc/site.toml` | doc-pack-v2 | 実装済み。`workspace.name`、`workspace.prefix`、`rig.path` は site binding 状態に移行する。 |
| 🔵 | Pack/Deployment/SiteBinding 構造体の分離 | doc-loader-v2 | ローダは 1 つの City 構造体に合成 |
| 🔵 | Pack 定義の `[[service]]` | — | #657 |
| 🔵 | `[agent_defaults]` のすべてのエージェントフィールドへの拡張 | — | 後の波 |

---

## ファストフォロー成果物（マージ後、4 月 21 日のローンチ前）

1. **非推奨警告インフラ** — 上記のすべての V1 フィールドに対する大声およびささやき警告を実装。
2. **schema 2 cities での大声警告** ：`[[agent]]`、`workspace.includes`、`workspace.default_rig_includes`、`[packs]`、`rigs.includes`、`rigs.formulas_dir`、`fallback`、`inject_fragments` の使用に対して。
3. **ささやき警告** ：`workspace.provider`、`workspace.start_command`、`workspace.suspended`、`workspace.install_agent_hooks`、`workspace.global_fragments`、`rigs.overrides`、`[formulas].dir` に対して。
4. **ハードエラー** ：`[formulas].dir` が `"formulas"` 以外に設定された場合。
5. **ハードエラー** ：`include` フラグメントが `[imports]`、`includes` を含むか `pack.toml` を参照する場合。
