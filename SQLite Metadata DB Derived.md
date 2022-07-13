---
layout: default
title: Derived DB Meta and Modular SQL with CTE
nav_order: 4
parent: SQLite Metadata
permalink: /meta/db-derived-cte
---

Extended parsed and structured metadata is available for table columns ([table_xinfo][]), foreign keys ([foreign_key_list][]), and indices ([index_list][] and [index_xinfo][]) via respective pragma functions. However, none of this PRAGMAs provide database-wide information: *index_xinfo* returns information for a specific index, while the three other PRAGMAs return information for a particular table.

<a name="TableColumnsCTE"></a>  
<div align="center"><img src="https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/yEd/Table%20Columns.svg" alt="Table Columns and CTE" /></div>
<p align="center"><b>Fig. 1. Structure of "Table Columns" Query</b></p>  





<!-- References -->

[foreign_key_list]: https://sqlite.org/pragma.html#pragma_foreign_key_list
[index_list]: https://sqlite.org/pragma.html#pragma_index_list
[index_xinfo]: https://sqlite.org/pragma.html#pragma_index_xinfo
[table_xinfo]: https://sqlite.org/pragma.html#pragma_table_xinfo
