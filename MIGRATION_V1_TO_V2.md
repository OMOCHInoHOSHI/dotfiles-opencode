# V1→V2 移行ガイド（dotfiles-opencode）

OpenCode V1 で構築したこの dotfiles の設定を V2 へ移行するための実践ガイド。
2026-10-18 時点の opencode v2.0.32 環境で実際に検証済み。**このリポジトリ自体は移行完了済み**。

> 網羅的な移行手順・全フィールドの対応表は公式ガイドを参照:
> <https://opencode.ai/v2/docs/migrate-v1>

## 0. 結論（TL;DR）

| 項目 | 対応 |
|------|------|
| `~/.config/opencode/opencode.jsonc` の symlink | **従来どおり必要** |
| `AGENT.md` / `instructions.md` の symlink | **不要**（v2では機能しない。作成しない） |
| `instructions` のパス指定 | **要変更**: 相対パス → ホーム基準パス `~/dotfiles-opencode/...` |
| V1形式フィールド（`agent` / `permission`） | そのままでも動作する（v2が内部正規化）。変換推奨・本リポジトリは変換済み |
| `tui.json(c)` | `cli.json` へ自動移行（初回起動時） |
| プラグイン | V1の実装は動作しない（API刷新）。未使用なら影響なし |

## 1. V1/V2 の互換性の基本

- 設定ファイルの置き場所は V1 と同じ（`~/.config/opencode/`、プロジェクト側も同様）
- サポートされている V1 フィールドは**読み込み時にメモリ上で正規化**される。ソースファイルの書き換えは不要
- 公式に「意図的な破壊的変更」は次の3つだけ:
  1. プラグインAPI
  2. サーバーAPI（クライアント連携）
  3. ターミナル設定（`tui.json(c)` → `cli.json`。自動移行される）
- それ以外のサポート済みV1挙動は互換が意図されている

## 2. 移行手順

### 手順1: V2の確認

```bash
opencode --version
```

- V1とV2は同じ `opencode` コマンドを使い、併存しない（V2のインストーラーがV1バイナリを置き換える）
- 確認時点: `v2.0.32`

### 手順2: opencode.jsonc の symlink（従来どおり）

```bash
ls -l ~/.config/opencode/opencode.jsonc
```

期待: `-> ~/dotfiles-opencode/opencode/opencode.jsonc`

未作成なら:

```bash
ln -s ~/dotfiles-opencode/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc
```

`AGENT.md` / `instructions.md` の symlink は**作らない**（v2では読まれない）。

### 手順3: instructions をホーム基準パスに変更 ★最重要

**v2では相対パスの指示ファイルが「セッションのワークスペース基準」で解決される。**
v1時代の「設定ディレクトリ基準」ではないため、symlink 運用では読まれない。

```jsonc
// NG（v1時代の書き方。ワークスペース直下に同名ファイルがある場合しか読まれない）
"instructions": ["./AGENT.md", "./instructions.md"],

// OK（どこから起動しても読まれる）
"instructions": [
  "~/dotfiles-opencode/opencode/AGENT.md",
  "~/dotfiles-opencode/opencode/instructions.md"
],
```

補足:

- `~/` はホームディレクトリ基準で解決される（絶対パス指定でも可）
- 読み込みに失敗しても**エラーにならず黙ってスキップされる**ため、必ず手順6で確認する
- 例外: ワークスペースがプロジェクト外の場合や `OPENCODE_DISABLE_PROJECT_CONFIG=1` のときは
  グローバル設定ディレクトリ基準になる。環境依存を避けるため `~/` 指定が安全

### 手順4: ターミナル設定（cli.json）の確認

- v2のターミナル設定は `~/.config/opencode/cli.json` の1ファイルのみ（プロジェクト別は無い）
- `cli.json` が無い状態で最初にV2 TUIを起動すると、グローバルの `tui.json` 設定が自動移行される
- `$schema` は `https://opencode.ai/v2/cli.json`

```bash
ls -l ~/.config/opencode/cli.json
```

### 手順5: 認証の確認

```bash
opencode auth list
```

この環境での期待値:

- OpenCode Go: `OPENCODE_API_KEY`（environment）… `secrets_sh` 由来
  - 環境変数認証は**起動前に `common_sh` を読み込んだシェル**で起動する必要がある
- Google / OpenRouter / OpenCode Console: stored（credentialストア）

### 手順6: 反映と検証（必須）

```bash
opencode reload        # または新規セッションを起動
```

1. 実効設定の確認:

```bash
opencode api get /api/config
```

→ `instructions` に `~/dotfiles-opencode/...` が2件表示される

2. 指示ファイルが実際に読まれているかの確認（新規セッションで質問する）:

> あなたのシステム指示に「## 行動」という見出しのセクションが含まれている場合、
> その箇条書きをそのまますべて列挙してください。含まれていない場合は「なし」とだけ答えてください。

→ AGENT.md の4項目が返れば読み込み成功

3. `opencode reload` 直後は、**実行中のセッションにも** `Instructions from: ...` の更新として反映される

## 3. V2ネイティブ形式への変換（任意・本リポジトリでは実施済み）

V1形式のままでも動作するが、v2ネイティブ形式にすると設定が明確になる。
このリポジトリで関係する主な対応:

| V1 | V2 | 備考 |
|----|----|------|
| `agent` | `agents` | トップレベルのマップ名 |
| `permission`（ツール別オブジェクト） | `permissions`（順序付き配列） | `{ "action", "resource", "effect" }` |
| アクション名 `bash` / `task` / `write`・`patch` | `shell` / `subagent` / `edit` | |
| `model` + `variant` の分離指定 | `"provider/model#variant"` | 例: `"opencode-go/glm-5.3-flash#max"` |
| エージェント内 `prompt` | `system` | |
| `disable` | `disabled` | |

注意点:

- トップレベルで V1/V2 の混在は可能だが、**個々のエージェント等のエントリ内では混ぜない**
- 権限は順序付き配列で「最後にマッチしたルールが優先」。現状の並び（deny の後に対応する allow）を維持する
- `external_directory` は v2 デフォルトで許可（v1は ask）。明示的な allow はそのまま維持してよい
- `doom_loop` は v2 の現行コア権限アクションではない（エントリは残しても実効しない可能性がある）
- 本リポジトリの変換は「変換前後の正規化結果（`/api/config`）が完全一致」することを確認済み

## 4. V1残骸の棚卸し

| 場所 | 内容 | 対応 |
|------|------|------|
| `~/.config/opencode/package.json` ほか（`package-lock.json` / `bun.lock` / `node_modules` / `.gitignore`） | V1プラグイン用の依存一式（`@opencode-ai/plugin` 1.17.13） | **削除**（バックアップ: `/tmp/opencode/v1-backup-config/`。再取得可能） |
| `~/.local/share/opencode/auth.json` | V1時代の認証ファイル | **削除**。standalone起動でも `auth list` が stored を表示し、未使用と確認済み（バックアップ: `/tmp/opencode/v1-backup-data/`） |
| `~/.local/share/opencode/storage/` | V1時代のセッション保管 | **残す**（削除すると過去セッションが失われる） |
| `~/.local/share/opencode/opencode.db` 等 | V2のセッション/履歴DB | そのまま |
| `~/dotfiles-opencode/secrets_sh` | シェル用シークレット | **触らない**（Git管理外。中身は読まない運用） |

## 5. トラブルシュート

| 症状 | 確認ポイント |
|------|--------------|
| グローバル指示が効かない | `instructions` が相対パスになっていないか／`~/.config/opencode` へのsymlink運用に戻していないか。手順6の質問で確認（**無言で無視される**ため気づきにくい） |
| `CLAUDE.md` が読まれない | v2は `AGENTS.md` のみ対応（フォールバックなし）。内容を `AGENTS.md` へ移す |
| 設定変更が反映されない | `opencode reload` または新規セッション。プロジェクト設定は `.opencode/opencode.json` が直下の `opencode.json` より優先される |
| プラグインが動かない | V1プラグインは v2 では動作しない（API刷新）。v2方式（`plugins` 設定 / `.opencode/plugins/`）へ移植が必要 |
| 旧フィールドの警告が出る | `logLevel` / `server` 等は v2 に移行先がなく無視される。`logLevel` は環境変数 `OPENCODE_LOG_LEVEL` に置き換える |
| APIキー失効エラー | プロバイダ側で再発行し、再ログインまたは環境変数を更新する |

## 6. 参考リンク（公式V2ドキュメント）

- 移行ガイド: <https://opencode.ai/v2/docs/migrate-v1>
- 設定: <https://opencode.ai/v2/docs/config>
- 指示ファイル: <https://opencode.ai/v2/docs/instructions>
- ターミナル設定（cli.json）: <https://opencode.ai/v2/docs/cli/config>
- 権限: <https://opencode.ai/v2/docs/permissions>
- エージェント: <https://opencode.ai/v2/docs/agents>

## 付録: この環境での実施記録（2026-10-18）

- `opencode.jsonc`: `instructions` を `./AGENT.md` → `~/dotfiles-opencode/opencode/AGENT.md` 等へ変更
- `opencode.jsonc`: `agent` / `permission` → `agents` / `permissions` のv2ネイティブ形式へ変換（正規化結果の一致を確認）
- `README.md`: シンボリックリンクの節を v2 仕様（`opencode.jsonc` のみsymlink）に更新
- V1残骸の整理: 上記「4. V1残骸の棚卸し」のとおり
- 検証:
  - `opencode reload` 後、実行中セッションに指示が反映されることを確認
  - 中立ディレクトリの新規セッションで AGENT.md の内容が読めることを確認
  - 変換前後で `/api/config` の正規化結果（instructions / default_agent / agents / permissions）が完全一致
