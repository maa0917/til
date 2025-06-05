# Claude から天気APIを呼び出す MCP サーバーを作ってみた

[MCP](https://modelcontextprotocol.io/introduction) を使って天気予報を返すサーバーを Python 実装してみた。

参考にしたのは[公式クイックスタート](https://modelcontextprotocol.io/quickstart/server)ほぼそのまま

作成したリポジトリは[こちら](https://github.com/maa0917/weather-server-python)

## やったこと

- MCP 対応のPython サーバーを構築
- [National Weather Service (NWS)](https://www.weather.gov/documentation/services-web-api) の API を叩いて
    - 指定された週の気象警告を返す `get_alerts`
    - 緯度経度に基づいた天気予報を返す `get_forecast`
- Claude for Desktop に組み込み、実際に「サクラメントの天気は？」「NYの警報ある？」などと話しかけてレスポンスを確認

## MCPツールの構成

### `get_alerts(state: str)`

2文字の週コードを指定しアクティブな気象警告を一覧で返す

### `get_forecast(latitude: float, longitude: float)`

緯度経度を指定、予報の1〜2日分（最大5件）を整形して返す

Claude はユーザーの自然分から緯度経度を自動で推論して呼び出してくれる。

## ポイント

- MCP は `@mcp.tool()` を使って、Python関数をそのまま Claude から呼び出せるツールにできる
- `mcp.run(transport="stdio")` で Claude と標準入出力を通じた通信が可能
- Claude は args や docstring からツールの使い方や型を自動で認識する（プロンプトなしで補完される）
- Claude は地名→緯度経度のジオコーディングも自前でやってくれる

# MCP(Model context protocol)とは

MCP(Model Context Protocol)は、大規模言語モデル（LLM）に外部からコンテキスト情報を提供するためのプロトコル。
MCPは以下の３つの構成要素からなる

- MCPホスト: AIモデルを含むアプリケーション本体
- MCPクライアント: ホスト内に存在し、MCPサーバと１対１で通信するモジュール
- MCPサーバ: リソース（文書など）、ツール、プロンプトなどの情報を提供し、リクエストに応じてJSON-RPC形式で返却する

モデルは、クライアントを通じて必要な文脈情報を受け取り、ユーザーへの応答を生成する。

# Laravel でCSV出力時に文字コード変換（UTF-8 -> Shift_JIS）

## 概要

レセプト関連のシステムでCSVファイルを出力し、Zipでまとめてダウンロードさせる処理を実装した際、UTF-8でのままだとレセ電でエラーになる問題。

## 背景・発生した問題

- 「JIS
  X0201/X0208の範囲内（＝Shift_JIS）」のみ許可されている[オンライン又は光ディスク等による請求に係る標準仕様（調剤用）](https://www.ssk.or.jp/seikyushiharai/iryokikan/download/index.files/iryokikan_in_08.pdf)
- UTF-8で出力したCSVを取り込もうとすると、JIS規格外の文字や文字コード自体でエラーになる

## 対応方法

fputcsvでCSVに書き出す前に、各セルの文字列を[mb_convert_encoding](https://www.php.net/manual/en/function.mb-convert-encoding.php)
でShift_JIS（SJIS-win）に変換することで対応

```php
$fp = fopen('php://temp', 'r+');
foreach ($rows as $row) {
    // 各値をSJISに変換してfputcsv
    $row_sjis = array_map(fn($v) => mb_convert_encoding($v, 'SJIS', 'UTF-8'), $row);
    fputcsv($fp, $row_sjis);
}
rewind($fp);
$csvString = stream_get_contents($fp);
fclose($fp);

$zip->addFromString("{$subDirPath}/RECEIPTY.CYO", $csvString);
```

SJIS、JIS外文字は「?」などに自動で化けるため、規格違反を防げる。

CSVをZip化・Base64エンコードする部分はそのままでOK。

必要なら「?」等に化けた場合の警告やログ出力も検討。

# 応用情報技術者試験 解説：エンディアンとは？（平成23年特別 問11）

## 問題文

> 主記憶の1000番地から、表のように4バイトの整数データが格納されている。
> これを32ビットのレジスタにロードするとき、プロセッサのエンディアンとレジスタにロードされる数値の組合せとして、正しいものはどれか。

| バイトアドレス | データ |
|---------|-----|
| 1000    | 00  |
| 1001    | 01  |
| 1002    | 02  |
| 1003    | 03  |

選択肢：

|   | リトルエンディアン | ビッグエンディアン |
|---|-----------|-----------|
| ア | 00010203  | 02030001  |
| イ | 00010203  | 03020100  |
| ウ | 02030001  | 00010203  |
| エ | 03020100  | 00010203  |

## 答え

**正解：エ**

## 解説

### 「32ビットのレジスタにロードする」とは？

- **レジスタ**はパソコンの「超高速なメモ帳」みたいなもの。
- **32ビット**は「0か1が32個分」＝「4バイト」
- 「ロード」とは、メモリからデータを**読み込む**という意味。
- つまり、「4バイト分のデータをCPUのメモ帳にまとめて読み込む」こと

### エンディアンとは？

「データをどの順番で並べるか？」のルール。

- ビッグエンディアン：大きい桁（上位）から保存する
- リトルエンディアン：小さい桁（下位）から保存する

### 今回のデータ

バイトアドレスとデータは

```
1000番地 → 00
1001番地 → 01
1002番地 → 02
1003番地 → 03
```

この順番にメモリに並んでいるということ

### 各エンディアンでの読み取り方法

#### ビッグエンディアン

ビッグエンディアンは「そのままの順番」で読む

```
[1000]→00, [1001]→01, [1002]→02, [1003]→03
結果：00010203
```

#### リトルエンディアン

リトルエンディアンは「逆の順番」で読む

```
[1003]→03, [1002]→02, [1001]→01, [1000]→00
結果：03020100
```

## なぜ2つのエンディアンがあるのか？

CPUを作った人たちの考え方が違うから。

- **ビッグエンディアン**：人間にとって読みやすい。表示重視。
- **リトルエンディアン**：CPUが計算しやすい。下の桁からすぐ使える。

どちらが正しいということではなく、目的や設計によって使い分けている

## まとめ

- **エンディアン**とは、バイトの並び順のルール
- **ビッグエンディアン**：そのままの順番
- **リトルエンディアン**：逆の順番

# cva の型チェックとエラー解消

## 背景

- `cva` 関数は、スタイル（クラス名）を条件に応じて切り替えるための関数

## 問題点

- 設定オブジェクトの型が適切に推論されず、 `Config<unknown>` として扱われる

## 解決策

- `cva` に対してジェネリック型引数として `tyoeof variants` を明示的に渡すことで、設定オブジェクトの正確な型情報を伝え、型チェックを厳密に行うようにした
- 具体例:

```ts
const buttonVariants = cva<typeof variants>(
    "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
    {
        variants,
        defaultVariants: {
            variant: "default",
            size: "default",
        },
    }
);
```

## 学び

- ジェネリック型引数を明示することで、`cva` に渡す設定オブジェクトの型安全性が向上し、意図しない値やキーのミスをコンパイル時に防ぐことができる