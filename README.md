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
この dotfiles の **3ファイルすべて** をシンボリックリンクで配置する必要がある。

| ファイル | 役割 |
|----------|------|
| `opencode.jsonc` | 本体設定（モデル・エージェント・パーミッション） |
| `AGENT.md` | カスタム指示（opencode.jsonc の `instructions` から参照） |
| `instructions.md` | カスタム指示（同上） |

**注意**: `opencode.jsonc` の `instructions` は設定ファイル設置ディレクトリ
（`~/.config/opencode/`）基準の相対パスで解決される。
`AGENT.md` / `instructions.md` の symlink を忘れると**エラーにならず黙って読み込まれない**ため、
指示が反映されているかは別途確認すること。

#### ① 既存ファイルの状態確認
- 通常ファイルなら次へ
- 既に symlink なら `->` が表示される

```bash
ls -l ~/.config/opencode/opencode.jsonc ~/.config/opencode/AGENT.md ~/.config/opencode/instructions.md
```

#### ② 既存ファイルを退避（安全）

```bash
mv ~/.config/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc.bak
```
または削除
```bash
rm ~/.config/opencode/opencode.jsonc
```

#### ③ シンボリックリンク作成（3ファイル分）

```bash
ln -s ~/dotfiles-opencode/opencode/opencode.jsonc ~/.config/opencode/opencode.jsonc
ln -s ~/dotfiles-opencode/opencode/AGENT.md ~/.config/opencode/AGENT.md
ln -s ~/dotfiles-opencode/opencode/instructions.md ~/.config/opencode/instructions.md
```

#### ④ シンボリックリンク確認（最重要）

```bash
ls -l ~/.config/opencode/
```
期待される出力（3本とも `->` で dotfiles 側を指していること）:
```
AGENT.md -> /Users/ユーザー名/dotfiles-opencode/opencode/AGENT.md
instructions.md -> /Users/ユーザー名/dotfiles-opencode/opencode/instructions.md
opencode.jsonc -> /Users/ユーザー名/dotfiles-opencode/opencode/opencode.jsonc
```

#### ⑤ 実体ファイルが読めているか確認

```bash
cat ~/.config/opencode/opencode.jsonc
cat ~/.config/opencode/AGENT.md
cat ~/.config/opencode/instructions.md
```
