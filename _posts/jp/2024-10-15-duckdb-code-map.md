---
layout: default
title: "コードマップ: DuckDBのフルスキャンクエリ"
lang: jp
image:
    path: /assets/images/diagram-duckdb-query.svg
---

# コードマップ: DuckDBのフルスキャンクエリ

自分がDuckDBの全体像を何となく理解するためにコードを読みつつメモしたコードマップの記事です。

DuckDBのバージョン`1.0.0`時点でのコードを、一番シンプルであろうフルスキャンのクエリ実行にフォーカスしてトレースしたものです。

大きな画像が見やすいビューワーで開くのをお勧めします。300KBもない軽いSVGですが画像サイズはかなり大きいので。
<img src="/assets/images/diagram-duckdb-query.svg">

> 注釈:
>
> - 時々は`...`といった読み飛ばした表示があるのですが、記されているコードはほぼ全て部分的に引用されており、関数のコード全部が入っていることはほぼ無いです。
> 
> - 関数のシグネチャや引数が箇所によっては書き漏れている場合があります。
>
> - 図の左上に示されているように、複数に分類されたフロー（流れ）が色分けされています。
>
>     - クエリ、書き込み、スキャン初期化、圧縮タイプ、シンクのフローがあります。
>
>     - このメモの焦点はクエリ実行にあるため、クエリのフローほぼ示されていますが、他のフローはすべて非常に部分的です。


## 全体像

クエリステートメントが解析された後、以下の様な流れで実行されます：

- Select nodeのバインディング：`SELECT`ステートメントの詳細を含む`BoundQueryNode`を返す。

- Tableのバインディング：`LogicalGet`を含む`BoundTableRef`を返す。

- 論理プラニング：`LogicalOperator`を返す。

- 物理プラニング：`PhysicalOperator`を返す。

- 実行：パイプラインを設定して実行し、最終的にはテーブルバインディング段階でロードされたスキャン関数である`PhysicalOperator`の`GetData`関数を呼び出す。

## スキャン関数

データをスキャンする式はテーブルをバインドするときにロードされ、以下の抽象化されたストレージ構造を辿ってデータが読み込まれます。

- `DataTable`

- ( `CollectionScanState` )

- `RowGroupCollection`

- `RowGroup`

- `ColumnData`

- `ColumnSegment`

- ( `BufferManager` )

> メモ:
>
> - バッファマネージャーの実装は複数ありますが、このコードマップでは`StandardBufferManager`を例としてトレースしています。

## データ圧縮アルゴリズム

データの圧縮アルゴリズムは書き込み時に決定し、データと一緒に保存されます。読み込み時には`ColumnSegment`単位でメタデータにロードされます。

詳細:

- 書き込み時の圧縮アルゴリズム決定:

    - `void ColumnDataCheckpointer::WriteToDisk`の式が`ColumnDataCheckpointer::DetectBestCompressionMethod`を呼んだ後にデータを圧縮して保存する。

- 読み込み時のアルゴリズムのロード:

    - `RowGroup::GetColumn`の式が`ColumnData::Deserialize`を呼ぶことで`DataPointer`のstructに読み込まれる。 このstructの`compression_type`が書き込み時に決定されたアルゴリズムのタイプとなる。

> メモ:
> 
> - 圧縮アルゴリズムは複数存在しますが、このコードマップでは`UncompressedString`と`DictionaryCompression`を例としてトレースしています。