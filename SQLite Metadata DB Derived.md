---
layout: default
title: Derived DB Meta and Modular SQL with CTE
nav_order: 4
parent: SQLite Metadata
permalink: /meta/db-derived-cte
---

Extended parsed and structured metadata is available for table columns ([table_xinfo][]), foreign keys ([foreign_key_list][]), and indices ([index_list][] and [index_xinfo][]) via respective pragma functions. However, none of this PRAGMAs provide database-wide information: *index_xinfo* returns information for a specific index, while the three other PRAGMAs return information for a particular table. To produce database-wide metadata tables, it is necessary to combine base queries.

For example, consider the *pragma_table_xinfo* table-valued function, which takes either target table name as a string (not an identifier!) or a column identifier. In the latter case, the each column value should contain a table name. Therefore, to construct a table containing all columns for all tables, the first subquery should retrieve the list of tables and pass it to *pragma_table_xinfo* in the second subquery via the JOIN clause. While this query is relatively simple, it is instructive to construct it using common table expressions (CTEs). [Fig. 1](#TableColumnsCTE) shows on the left a schematic structure of the CTEs query, which includes two subqueries discussed above. The right panel of the figure shows the corresponding subqueries. The *tables* subquery retrieves the list of tables from the *sqlite_master* table, and the result is used by the *columns* subquery.

<a name="TableColumnsCTE"></a>  
<div align="center"><img src="https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/yEd/Table%20Columns.svg" alt="Table Columns and CTE" /></div>
<p align="center"><b>Fig. 1. Structure of "Table Columns" Query</b></p>  


<iframe id="TableColumnsHTML" width="100%" height="325"
        src="https://pchemguy.github.io/SQLite-SQL-Tutorial/Table%20Columns.html">
</iframe>



<!-- References -->

[foreign_key_list]: https://sqlite.org/pragma.html#pragma_foreign_key_list
[index_list]: https://sqlite.org/pragma.html#pragma_index_list
[index_xinfo]: https://sqlite.org/pragma.html#pragma_index_xinfo
[table_xinfo]: https://sqlite.org/pragma.html#pragma_table_xinfo
