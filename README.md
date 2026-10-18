# dotfiles-opencode
dotfileのopencode設定

## セットアップ手順

`secrets_sh` は手動で作成してください（Git管理対象外）。

読み込み
```
source ~/dotfiles-opencode/common_sh
```

### シンボリックリンクについて（重要）

opencode は `~/.config/opencode/` にある設定ファイルを読み込むため、
`opencode.jsonc` をシンボリックリンクで配置する。

| ファイル | 役割 |
|----------|------|
| `opencode.jsonc` | 本体設定（モデル・エージェント・パーミッション） |
| `AGENT.md` | カスタム指示（opencode.jsonc の `instructions` から参照） |
| `instructions.md` | カスタム指示（同上） |

**注意（v2以降）**: `instructions` は
`~/dotfiles-opencode/opencode/AGENT.md` のように**ホーム基準のパス**で指定している。
v1時代は設定ディレクトリ（`~/.config/opencode/`）基準の相対パスで解決されていたが、
v2 の相対パスは**セッションのワークスペース基準**で解決されるため symlink では読まれない。
よって `AGENT.md` / `instructions.md` の symlink は**不要**（作成しないこと）。

**注意**: 指示ファイルが読まれなくても**エラーにならず黙って無視される**。
変更後は実際に読み込まれているか確認すること（下記⑤参照）。

#### ① 既存ファイルの状態確認
- 通常ファイルなら次へ
- 既に symlink なら `->` が表示される

```bash
ls -l ~/.config/opencode/opencode.jsonc
```

#### ② 既存ファイルを退避（安全）

```bash
mv ~/.config/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc.bak
```
または削除
```bash
rm ~/.config/opencode/opencode.jsonc
```

#### ③ シンボリックリンク作成

```bash
ln -s ~/dotfiles-opencode/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc
```

#### ④ シンボリックリンク確認（最重要）

```bash
ls -l ~/.config/opencode/opencode.jsonc
```
期待される出力（`->` で dotfiles 側を指していること）:
```
opencode.jsonc -> ~/dotfiles-opencode/opencode/opencode.jsonc
```

#### ⑤ 設定・指示ファイルが読めているか確認

```bash
cat ~/.config/opencode/opencode.jsonc
```
反映は `opencode reload` または新規セッションで行われる。
指示ファイル（AGENT.md など）が実際に読み込まれているかは、opencode のセッションで
「AGENT.md の『## 行動』セクションの内容を列挙して」のように質問して確認する
（読み込まれていなければ「なし」と返る）。
