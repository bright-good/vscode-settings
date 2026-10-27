# vscode-settings

Windows の VS Code で、Shift_JIS と UTF-8 が混在した Java ソースを全文検索するための設定です。

VS Code の検索（Ctrl+Shift+F）は、Shift_JIS のファイル内の日本語がヒットしません。
そこで VS Code 内蔵の ripgrep（rg）を Git Bash から呼び、両方のエンコードで検索します。
結果は VS Code の検索結果と同じ形式（`.code-search`）で開くので、行をクリックするとソースへ飛べます。

## 前提

- Windows 11
- VS Code（ユーザーインストール版）
- Git for Windows（`C:\Program Files\Git`）
- PowerShell のスクリプト実行が禁止されていても動く（PowerShell プロファイルを使わない）
- 追加のファイルやインストールは不要。`settings.json` だけで完結する

## 使い方

1. `settings.json` の中身を `%APPDATA%\Code\User\settings.json` に反映する
2. `<USER>` などの置き換え箇所を自分の環境に合わせる（ファイル冒頭のコメント参照）
3. ターミナルを開き直す（既定のターミナルが `Git Bash (rg)` になる）
4. 検索するフォルダで実行する

```bash
rgj "検索語"                         # ターミナルに表示
rgjf "検索語"                        # 結果を .code-search で開く
rgjf "検索語" 2                      # 前後2行も表示
rgjf -w -c -i "*.java" -x "test/**" "p_cnvcf" 2
```

| オプション | 意味 |
|---|---|
| `-w` | 単語単位 |
| `-c` | 大文字小文字を区別する（既定は区別しない） |
| `-r` | 正規表現（既定は文字どおり） |
| `-i "glob"` | 含めるファイル（複数回指定可） |
| `-x "glob"` | 除外するファイル（複数回指定可） |

## キーバインドを設定すると楽

毎回ターミナルで打つより、キー1つで呼べるようにしておくと楽です。
`keybindings.json`（Ctrl+Shift+P →「基本設定: キーボード ショートカットを開く (JSON)」）に追加します。

```jsonc
[
    // rgjf 版（結果をファイルで開く）
    // ターミナルに rgjf "選択した文字列" を入れて止める（Enter は押さない）
    // オプションや前後の行数を足してから Enter で実行する
    // 何も選択していないときは rgjf "" が入る
    {
        "key": "ctrl+alt+f",
        "command": "runCommands",
        "args": {
            "commands": [
                "workbench.action.terminal.focus",
                {
                    "command": "workbench.action.terminal.sendSequence",
                    "args": { "text": "rgjf \"${selectedText}\"" }
                }
            ]
        },
        "when": "editorTextFocus"
    },

    // rgj 版（結果をターミナルに表示する）
    // 選択している文字列で rgj を実行する
    {
        "key": "ctrl+alt+g",
        "command": "workbench.action.terminal.sendSequence",
        "args": { "text": "rgj \"${selectedText}\"\u000D" },
        "when": "editorTextFocus && editorHasSelection"
    },
    // 何も選択していないときは、ターミナルに rgj "" を入れて検索語の入力待ちにする
    {
        "key": "ctrl+alt+g",
        "command": "runCommands",
        "args": {
            "commands": [
                "workbench.action.terminal.focus",
                {
                    "command": "workbench.action.terminal.sendSequence",
                    "args": { "text": "rgj \"\"\u001b[D" }
                }
            ]
        },
        "when": "editorTextFocus && !editorHasSelection"
    }
]
```

- `rgjf` 版は結果をファイルで開く。オプションを足すことが多いので、実行せずに止める
- `rgj` 版はターミナルに出すだけ。軽く確認したいときに使うので、選択していればすぐ実行する
- `\u000D` は Enter、`\u001b[D` はカーソルを1文字左へ動かす（`""` の間に入る）
- キーは例です。他と重なっていないかは、キーボード ショートカット画面（Ctrl+K Ctrl+S）の「キーの記録」で確認する
- 選択した文字列に `"` が含まれると、正しく渡りません

## VS Code を更新したとき

内蔵 rg のパスに含まれるフォルダ名（コミットハッシュ）が変わります。
PowerShell で次を実行し、`terminal.integrated.env.windows` の `PATH` を書き換えてください。

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Programs\Microsoft VS Code" -Recurse -Filter rg.exe -ErrorAction SilentlyContinue | Where-Object { $_.FullName -like '*\@vscode\ripgrep*' } | Select-Object FullName
```

Copilot 拡張も `rg.exe` を持っているため、`@vscode\ripgrep` を含むものを選びます。

## 仕組み

- `Git Bash (rg)` プロファイルの `args` で関数 `rgj` / `rgjf` を定義し、`export -f` してから対話シェルを起動している
- JSON の文字列は改行できないため、関数は1行で書いている。読むときは Alt+Z で折り返す
- `rgjf` は rg の出力を `.code-search` の形式に変換している
  - ファイル名の行: 末尾に `:`
  - 一致した行: `␣␣行番号:␣内容`
  - 前後の行: `␣␣行番号␣内容`
  - 行頭の空白は半角でないと構文色が付かない。そのため sed では `\x20` で書いている
- `rgjf` は UTF-8 と Shift_JIS の両方で検索し、同じファイルが両方に出たら1つにまとめる
  - UTF-8 で読んだ結果に文字化けの印（U+FFFD）が無ければ UTF-8 の結果を使う。あれば Shift_JIS の結果を使う
  - 処理は awk で行っている。2つの検索結果の境目に `@@SJIS@@` という行を挟んで区別している
- `rgjf` の結果はファイルのパス順に並べる
  - rg は並行して検索するため、そのままだと実行ごとに順番が変わる
  - awk で1ファイル1行にまとめて `sort -f` で並べ、元の形に戻している
- 形式は VS Code 本体の `extensions/search-result` を読んで合わせた

## 既知の制限

- 一致箇所の黄色ハイライトは付かない。Ctrl+F で検索語を入れると代わりになる
- `rgj` は Shift_JIS と UTF-8 の2回検索するため、ASCII だけで一致するファイルは結果に2回出る（`rgjf` は重複を除いている）
- 並び順は VS Code 標準の検索と完全には一致しない可能性がある（標準の比較方法は未確認）
- 結果ファイル `rg-日時.code-search` はカレントディレクトリに溜まる。不要分は手で消す
- 結果ファイルを開いたあと、操作によっては VS Code が組み込み検索を再実行して上書きすることがある（引き金は未特定）
