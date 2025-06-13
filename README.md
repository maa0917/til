# 2025-06-13: 関数型プログラミング

## 関数型プログラミングとは

- データを「別のデータに変換する」ことを中心に考えるプログラミングスタイル
- 処理の単位は「関数」
- 副作用のない「純粋関数」が基本
- 同じ入力に対して常に同じ出力（参照透過性）を返す関数が理想
- 状態を持たない設計、不変性が重要

## 手続き型 vs 関数型

| 観点   | 手続き型            | 関数型                   |
|------|-----------------|-----------------------|
| 基本思想 | 手順・命令を記述        | 入力 -> 出力を関数で記述        |
| 状態管理 | 変数を更新（可変）       | 不変（変数を再代入しない）         |
| 処理方法 | ループ（for, while） | 再帰・高階関数（map, fold など） |
| 副作用  | あり（状態変更、IOなど）   | なしが理想（純粋関数）           |

## 関数の比較

sum 関数の例を手続き型と関数型で比較

### 手続き型(JavaScript)

```javascript
function sum(arr) {
    let total = 0;
    for (let i = 0; i < arr.length; i++) {
        total += arr[i];
    }
    return total;
}
```

- 状態を変えながら合計を計算する

### 関数型(Scala)

```scala
def sum(xs: List[Int]): Int = xs match{
    case Nil => 0
    case x :: xs => x + sum(xs)
}
```

- 再帰を使い、状態を持たずに合計を返す

## データの「変換」とは

- 関数型では、元のデータを変更せずに新しいデータを生成することを「変換」と捉える
- 手続き型でも変換はするが、変数の中身を書き換える形が多い（破壊的変更）

## 副作用（side effect）とは

- 関数外部の状態に影響を与える処理
    - 例： `print()`、ファイル書き込み、DB更新、グローバル変数の変更
- 関数型では副作用を避け、関数は「値の変換だけ」に集中するのが理想

## 参照透過とは

- ある式を、その評価結果（値）に置き換えてもプログラムの意味が変わらないこと
- 同じ入力 → 同じ出力を返すことが前提
- 副作用があると参照透過性が成立しない

## 感想

- 関数型の考え方は「何をするか」に集中していて、処理の安全性やテストの使やすさに強みがある
- 並列処理や大規模システムの場合には特に役立ちそう

# 2025-06-13: Claude と MCPサーバを連携させてツールを実行するクライアントを構築した

作成したリポジトリは[こちら](https://github.com/maa0917/mcp-client-python)

## 使ったもの

- [MCP](https://modelcontextprotocol.io/quickstart/client)
- 以前作成した[weather-server-python](https://github.com/maa0917/weather-server-python)
- Claude 3.5 Sonnet API

## 処理の全体の流れ

1. ユーザーが質問
   例: What are the weather alerts in California?
2. Claude API にリクエストを送信
   質問と一緒に、MCPサーバが提供しているツール（`get_forecast`, `get_alerts` など）の情報を渡す
3. Claude がツール仕様を提案(tool_use)
   例: `get_forecast` を使って、緯度経度を指定して取得すべきと判断
4. クライアントがMCPサーバにツール実行要求
   `self.session.call_tool()` を使って `get_forecast` を実行
5. MCPサーバがレスポンスを返す
   例: 天気予報のデータが返ってくる
6. Claude にツール結果を渡して最終返答を生成
   `tool_result` を渡し、Claude が自然な文章でユーザーへの返答を生成

# 2025-06-09: Eloquent の join は SoftDeletes が効かない

## 概要

Laravel の Eloquent では、 SoftDeletes を利用している場合、 `deleted_at` が null 出ないレコードは自動で除外される。
しかし、join句を使った場合、このグローバルスコープが効かず、論理削除済みのレコードも取得されてしまう。

## 体験談

- prescriptions テーブルから論理削除を考慮してレコードを取得したかった
- `join()` で `details` テーブルを結合した
- `details` テーブルに同じ `prescription_id` の論理削除前・後レコードが両方残っていた
- 結果、1件のはずが重複して2件取得されてしまった

## 原因

- Eloquentのリレーションや `with()` を使った場合は `deleted_at IS NULL` の条件が自動で付与される
- `join()` はあくまでSQLの `join` で、グローバルスコープ（論理削除を除外するwhere句）が効かない
- そのため、論理削除済みレコードもjoin結果に含まれる

## 教訓

- Eloquentリレーションや `with()` を使える場合はできるだけそちらを使う
    - 自動でSoftDeletesの除外が効く
- やむを得ず `join()` を使う場合は、明示的に `whereNull('deleted_at')` を付与すること
    - 例: `->whereNull('details.deleted_at')`

```php
// Eloquentのみ
$prescriptions = Prescription::with('prescriptionDetail')
    ->whereIn('id', $ids)
    ->get();

// joinを使う場合
$prescriptions = Prescription::join('details', 'prescriptions.id', '=', 'details.prescription_id')
    ->whereNull('details.deleted_at')
    ->whereIn('prescriptions.id', $ids)
    ->get();
```

# 2025-06-05: Claude から天気APIを呼び出す MCP サーバーを作ってみた

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