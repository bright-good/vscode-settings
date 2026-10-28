# テスト用ファイル

`rgj` / `rgjf` の動作確認用です。`test/sample` に移動してから実行します。

```bash
cd test/sample
rgj foo
```

## ファイル

| ファイル | エンコード / 改行 | 用途 |
|---|---|---|
| `SjisPlain.java` | Shift_JIS / CRLF | Shift_JIS の日本語がヒットするか |
| `Utf8Plain.java` | UTF-8 / LF | 大文字小文字（`FOO`）、単語単位（`fooCount`） |
| `Alpha/Zeta.java` | UTF-8 / LF | 大文字で始まるフォルダの並び順 |
| `web/Config.xml` | Shift_JIS / CRLF | `-i` / `-x` の確認、XML の構文色 |
| `web/util/Helper.java` | UTF-8 / CRLF | 深いフォルダ、CRLF の UTF-8 |
| `sjis/SjisSample.java` | Shift_JIS / CRLF | 前後の行（離れた2か所の一致） |
| `sjis/config.xml` | Shift_JIS / CRLF | 日本語を含む XML |
| `utf8/Utf8Sample.java` | UTF-8 / LF | `SjisSample.java` と同じ内容の UTF-8 版 |

`test/.gitattributes` で改行の自動変換を止めています。Git が CRLF / LF を書き換えないようにするためです。

## 期待する結果

Linux の bash と rg 14.1.0 で確認した件数です。

| コマンド | 結果 | 確認できること |
|---|---|---|
| `rgj 検索テスト` | 6 件 - 3 ファイル | Shift_JIS と UTF-8 の両方で日本語がヒットする |
| `rgj foo` | 14 件 - 8 ファイル | ASCII だけの一致でもファイルが重複しない |
| `rgj -c Foo` | 2 件 - 2 ファイル | 大文字小文字の区別 |
| `rgj -w foo` | 10 件 - 7 ファイル | 単語単位（`fooCount` と `foobar` が外れる） |
| `rgj -r 'fo+bar'` | 3 件 - 3 ファイル | 正規表現 |
| `rgj -i '*.xml' foo` | 2 件 - 2 ファイル | 含めるファイル |
| `rgj -x '*.xml' 検索テスト` | 4 件 - 2 ファイル | 除外するファイル |
| `rgj bar 1` | 6 件 - 4 ファイル | 前後の行。`sjis/SjisSample.java` で離れた一致の間に `--` が入る |

`rgjf` で同じコマンドを実行すると、同じ件数の `.code-search` ファイルが開きます。あわせて次を確認します。

- 行をクリックするとソースへ飛ぶ
- `sjis/config.xml` などの結果行に XML の構文色が付く
- 前後の行が薄く表示される
- 実行するたびに同じ順序で並ぶ
