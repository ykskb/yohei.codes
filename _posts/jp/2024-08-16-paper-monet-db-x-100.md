---
layout: default
title: "論文：MonetDB/X100: Hyper-Pipelining Query Execution"
lang: jp
---

# 論文：MonetDB/X100: Hyper-Pipelining Query Execution

この記事は、[「MonetDB/X100: Hyper-Pipelining Query Execution」](http://cidrdb.org/cidr2005/papers/P19.pdf)という論文からのメモです。この論文は、2005年に[The Conference on Innovative Data Systems Research (CIDR)](https://www.cidrdb.org/)会議で発表されました。

> ### シリーズ：DuckDBが採択した論文
>
> このシリーズは、[DuckDBが採用しいた論文](https://duckdb.org/why_duckdb.html#standing-on-the-shoulders-of-giants)を読んだ際のメモをまとめたものです。
>
> - ベクターベースのクエリエンジン: 本記事
> - 高速でSerializableなMVCC（近日公開）
> - Join順序の最適化 （近日公開）
> - サブクエリ展開（近日公開）

## 主な内容

この論文は、現代的なCPUをより効果的に活用するために、DBMSにおけるベクター化されたクエリ実行モデルを提案し、従来のVolcanoイテレータモデルと比較しています。

* 性能比較はTPC-Hベンチマークを使用して行われており、大規模なスキャン、結合、集計、ネストされたクエリを含む22の複雑なSQLクエリを評価しています。

* 論文には直接記載されていませんが、MonetDBはカラム指向のデータベースであり、比較されている従来のVolcanoモデルは行指向のデータベースであると推測されます。

## クエリ解釈

* 従来のモデル: Volcanoイテレータモデル = タプルごとに処理

    各タプルはフィルタ、Join結合、集計などのさまざまな演算子を何度も通過する必要があり、解釈オーバーヘッドが発生し、CPUの並列処理の機会が限られてしまう。

* 提案されたモデル: ベクター化された処理

    フィルタ、Join結合、集計などの操作をデータ全体のベクトルに一度に適用します。

## CPUキャッシュ

* 従来のモデル: メモリにタプル全体を読み込む必要があるため、データがアクセスされるとき、未使用のデータもキャッシュラインを占有し、キャッシュミスが頻繁に発生します。

* 提案されたモデル: 関連するデータのみをメモリに読み込む

    * 空間的局所性（Spacial locality）: 単一の列のデータがメモリに連続して格納されているため、キャッシュヒットが増え、連続したメモリアクセスが可能になります。

    * 時間的局所性（Temporal locality）: データがキャッシュに読み込まれると、次のチャンクが読み込まれる前にそのデータが繰り返し使用されます。

例:

* テーブル: `sale_id INT, date DATE, product_id INT, quantity INT, price FLOAT`

* クエリ: `SELECT SUM (quantity * price) AS total_revenue FROM sales;`

* キャッシュライン:

    * 行指向データベース: キャッシュラインに未使用の列が含まれる

        `[ sale_id | date | product_id | quantity | price | sale_id... ]`

    * カラム指向データベース: 使用されるデータのみがキャッシュされる

        `[ quantity | quantity | quantity | quantity... ]`

## CPUパイプラインの実行

* 従来のモデル: タプルごとに処理 & 行指向

    * 前述のように、頻繁なキャッシュミスのため、データが読み込まれる際にCPUがストールします。

    * 各タプルごとに命令を繰り返しフェッチする必要があります (SIMDが使用されない)。

    * 各タプルのIFやWHEREなどの条件により、ブランチミス予測が発生します。

    * データ/制御ハザード = 命令の依存関係。命令を進める前に前の命令が完了するのを待つ必要があります。

* 提案されたモデル: ベクター化された処理

    * キャッシュミスが少ない = CPUのストールが少ない。

    * ベクター化されたデータはSIMDとよく一致し、データ処理を大幅に最適化できます。

    * 命令間の依存関係が少ないか、まったくないため、CPUパイプラインが命令を効果的に計画し、実行することができます。