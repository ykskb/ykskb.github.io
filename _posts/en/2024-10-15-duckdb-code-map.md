---
layout: default
title: "Code Map: DuckDB's Full Scan Query"
lang: en
image:
    path: /assets/images/diagram-duckdb-query.svg
---

# Code Map: DuckDB's Full Scan Query

In this article, I'd like to share the diagram which I took as a note to get an idea of DuckDB's query execution flow.

The diagram is the code map of a query execution in DuckDB version `1.0.0`, focusing on the simplest scenario of full scan.

Please view in your favorite image viewer. It's a light (< 300KB) SVG image, but big in size:
<img src="/assets/images/diagram-duckdb-query.svg">

> Notes:
>
> - Code excerpts are partial most of the time. Sometimes it has `...` to indicate there are skipped lines, but it's not there most of the time.
> 
> - Sometimes function signatures miss return types and/or function arguments. Apologies in advance.
>
> - Different flows are color-coded as noted on left-top of the diagram. 
>
>     - There are flows of query, write, initialization, compression type and sink.
>
>     - The focus of this diagram is query execution, so query flow is roughly complete in the diagram, but all other flows are very partial.


## Overview

It (hopefully) covers an entire flow after statement parsing from:

- Select node binding: returns `BoundQueryNode` which includes details of `SELECT` statement.

- Table binding: returns `BoundTableRef` within which includes `LogicalGet`.

- Logical planning: returns `LogicalOperator`.

- Physical planning: returns `PhysicalOperator`.

- Execution: sets up and runs pipelines, eventually calling `GetData` function of `PhysicalOperator` which is the scan function loaded in table binding stage.

## Scan Function

Scan function is loaded at table-binding time. It goes through the storage abstractions as below:

- `DataTable`

- ( `CollectionScanState` )

- `RowGroupCollection`

- `RowGroup`

- `ColumnData`

- `ColumnSegment`

- ( `BufferManager` )

> Notes:
>
> - There are multiple implementations of buffer manager, but `StandardBufferManager` is traced in this code map.

## Compression Algorithm

Compression algorithm is decided upon write time and stored together with data. Upon data read, it's deserialized and loaded into `ColumnSegment`.

Details:

- Decision of compression type at write time:

    - `void ColumnDataCheckpointer::WriteToDisk` function calls `ColumnDataCheckpointer::DetectBestCompressionMethod` and writes data after compression.

- Compression type detection at read time:

    - `RowGroup::GetColumn` calls `ColumnData::Deserialize` which loads `DataPointer` struct from data. `compression_type` in the struct is the compression type decided at write time.

> Notes:
> 
> - There are multiple types of compression algorithms, but `UncompressedString` and `DictionaryCompression` are traced in this code map.
