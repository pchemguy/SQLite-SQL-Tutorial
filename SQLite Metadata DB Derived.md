---
layout: default
title: Derived DB Metadata and Modular SQL with CTE
nav_order: 4
parent: SQLite Metadata
permalink: /meta/db-derived-cte
---

Extended parsed and structured metadata is available for table columns ([table_xinfo][]), foreign keys ([foreign_key_list][]), and indices ([index_list][] and [index_xinfo][]) via respective pragma functions. However, none of these PRAGMAs provide database-wide information. *index_xinfo* returns index-specific metadata. The three other PRAGMAs return a list of columns (all columns or columns participating in foreign keys or indices, respectively) for a particular table. Retrieval of database-wide metadata tables requires a combination of base queries.

### Table Columns

Consider the *pragma_table_xinfo* table-valued function, which takes either the target table name as a string (not an identifier!) or a column identifier. In the latter case, each column value should contain a table name, and this option permits the construction of a query collecting database-wide column information. The first query retrieves the list of database tables (*tables*), and the second query (*columns*) uses *pragma_table_xinfo* with the retrieved table list in its JOIN clause to produce a database-wide column set. A combination of the two via either the subquery construct or common table expressions (CTEs) results in a relatively simple query, but it is instructive to follow the CTEs route. [Fig. 1](#TableColumnsCTE) shows a schematic structure of the CTEs query (left panel), which includes two subqueries discussed above. The right panel shows the corresponding CTEs code blocks.

<a name="TableColumnsCTE"></a>
<div align="center"><img src="https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/yEd/Table%20Columns.svg" alt="Table Columns and CTE" /></div>
<p align="center"><b>Fig. 1. Structure of the "Table Columns" Query</b></p>  

The combined query below consists of the two CTEs blocks and the following main query (the placement of the *columns* query inside the WITH clause is intentional, and the first CREATE VIEW line can be disregarded).

<iframe id="TableColumnsHTML" width="100%" height="325" frameBorder="0"
        src="https://pchemguy.github.io/SQLite-SQL-Tutorial/Table%20Columns.html">
</iframe>

#### Sample output

| table_name     | cid | col_name    | type    | notnull | dflt_value | pk |
|----------------|-----|-------------|---------|---------|------------|----|
| classes        | 0   | class       | TEXT    | 1       |            | 1  |
| classes        | 1   | super_class | TEXT    | 1       |            | 0  |
| classes        | 2   | class_name  | TEXT    | 0       |            | 0  |
| classes        | 3   | wikipedia   | INTEGER | 0       |            | 0  |
| classes_groups | 0   | class       | TEXT    | 1       |            | 1  |
| classes_groups | 1   | group_name  | TEXT    | 1       |            | 2  |
| classes_groups | 2   | super_group | TEXT    | 1       |            | 0  |


### Debugging CTE queries

Well-structured CTEs queries provide several advantages for designing complex queries, particularly in SQLite. On the one hand, individual WITH members are convenient building blocks, as illustrated later. On the other hand, CTEs enable several debugging options. The main query after the WITH clause can interrogate any WITH clause member. For instance, in the example above, the last line `SELECT * FROM tables` verifies the output of the *tables* query. Another option is replacing WITH members with immediate mocks having the same signature. The *table* query above has the output signature *table(table_name, sql)*, and the following mock replacement enables independent development of the two components.

~~~sql
    tables(table_name, sql) AS (
        VALUES ('sqlite_master', ''),
               ('sqlite_schema', '')
    ),
~~~

Additionally, given the output signature of the *tables* query and its semantics, it can now be reused as a code snippet.

### Foreign Keys

Both *pragma_foreign_key_list* and *pragma_index_list* return tables containing a row for each unique (column, index/FK) combination. The Foreign Keys query starts with a copy of the *table* block. The *columns* block, however, needs adjustments to take into account the signature and semantics of the returned table (*fkey_columns*). Furthermore, for composite indices/FKs, one row is returned for each participating column. Often, it is more convenient to have one row per index/FK, meaning that an additional WITH block is necessary to collapse rows corresponding to the same composite index/FK, as shown in [Fig. 2](#ForeignKeys).

<a name="ForeignKeys"></a>
<div align="center"><img src="https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/yEd/Foreign Keys.svg" alt="Foreign Keys" /></div>
<p align="center"><b>Fig. 2. Structure of the "Foreign Keys" Query</b></p>  

Note that the *src* (source) and *dst* (destination) prefixes refer to the *child* and *parent* sides of the foreign key relation, respectively. Also, the *fkey_columns* table must be ordered by composite component position (*seq/fk_seq*) within each foreign key (identified by the pair *src_table, fk_id*) before grouping in the next step. The standard approach to handling grouped column names is concatenation via the *group_concat* function to yield a CSV-like value. An alternative approach, shown in the *foreign_keys* block of the schematic, is to use the *json_group_array* function. The final query may look like this:

<iframe id="ForeignKeysHTML" width="100%" height="500" frameBorder="0"
        src="https://pchemguy.github.io/SQLite-SQL-Tutorial/Foreign Keys.html">
</iframe>

#### Sample output

| src_table | src_cols           | dst_table  | dst_cols           | on_update | on_delete | fk_id |
|-----------|--------------------|------------|--------------------|-----------|-----------|-------|
| classes   | ["super_class"]    | c_sclasses | ["super_class"]    | CASCADE   | NO ACTION | 0     |
| cgroups   | ["class"]          | classes    | ["class"]          | CASCADE   | NO ACTION | 1     |
| cgroups   | ["gname","sgroup"] | groups     | ["gname","sgroup"] | CASCADE   | NO ACTION | 0     |
| groups    | ["sgroup"]         | g_sgroups  | ["sgroup"]         | CASCADE   | NO ACTION | 0     |


### Indices

While the only structured source of foreign key metadata is *pragma_foreign_key_list*, index metadata comes from three sources:

 - dedicated entries in the *sqlite_master* table (database-wide source);
 - *pragma_index_list* function (table-wide source; it must be joined with the *tables* table (on *table_name*) to yield a database-wide source *index_list*);
 - *pragma_index_xinfo* function (per index column information; it must be joined with *index_list* on *index_name* to produce the *index_columns* table).

After composite component columns in the *index_columns* table are collapsed, yielding the *noddl_indices* table, the latter is joined with *sqlite_master* (on *index_name*), appending the *sql* column and producing the final *indices* table. (A possibly more efficient approach to the last join is using the LEFT JOIN with a subset of *index* rows from *sqlite_master* having non-null *sql* field.)
 
<iframe id="IndicesHTML" width="100%" height="625" frameBorder="0"
        src="https://pchemguy.github.io/SQLite-SQL-Tutorial/Indices.html">
</iframe>

#### Sample output

| table_name     | index_name  | col_names          | unique | origin | partial | sql           |
|----------------|-------------|--------------------|--------|--------|---------|---------------|
| groups         | idx_groups  | ["sgroup"]         | 0      | c      | 0       | CREATE IND... |
| classes_groups | idx_cgroups | ["gname","sgroup"] | 0      | c      | 0       | CREATE IND... |


### Indexing child columns of foreign keys

Although SQLite does not require indices on the child column(s) of foreign key relationships (as opposed to some other RDBMS), having them is a good practice. Therefore, it would be convenient to have a tool for verifying the presence of such indices. A straightforward approach is to take the lists of foreign keys and indices and match table names and indexed columns against foreign key's child columns. This approach does not account for possible explicit collations and ordering (both table and index definitions may include these attributes). A combination of these attributes may invalidate an otherwise suitable index. However, SQLite does not provide structured information on these attributes used in table definitions, making it necessary to parse DDL statements, which is a difficult task for pure SQL.

Criteria determining the suitability of a particular index differ slightly for simple and composite indices. For a single-column index, its column must be the same as the child column of a foreign key relationship (FKC column). For a composite index, its column vector must include the FKC column vector as its prefix (which also works for simple indices). For example, an index on (c.super_class, c.name, c.subclass) is suitable for foreign keys on (c.super_class), (c.super_class, c.name), and (c.super_class, c.name, c.subclass). A query returning index information for FKC columns can be constructed using the CTEs sections from the *indices* and *foreign key* queries above. One approach uses these queries as view definitions, and their join produces the final result. Alternatively, the new query includes individual CTEs sections directly and the join query at the end, as schematically illustrated in [Fig. 3](#FKeyChildIndices) and the code snippet below. 

<a name="FKeyChildIndices"></a>
<div align="center"><img src="https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/yEd/FKeys Child Indices.svg" alt="FKeys Child Indices" /></div>
<p align="center"><b>Fig. 3. Structure of the "FKeys Child Indices" Query</b></p><br />

<iframe id="FKeyChildIndicesHTML" width="100%" height="1050" frameBorder="0"
        src="https://pchemguy.github.io/SQLite-SQL-Tutorial/FKeys Child Indices.html">
</iframe>

Note that the *tables* block is a part of both *foreign_keys* and *indices*; the new query includes a single copy only before the rest of the copied code. An additional code block (*foreign_key_child_indices*) at the end joins *foreign_keys* and *indices* using *src_cols* and *col_names*, respectively. The algorithm defining these columns makes it possible to use a simple string-based prefix matching for the join operation. There is one significant detail, however. If the *group_concat* function was used for *src_cols/col_names* construction, prefix matching could be applied directly. With *json_group_array* used, the *foreign_key_child_indices* code must remove the closing bracket on the *src_cols* term from the prefix matching pattern.


<!-- References -->

[foreign_key_list]: https://sqlite.org/pragma.html#pragma_foreign_key_list
[index_list]: https://sqlite.org/pragma.html#pragma_index_list
[index_xinfo]: https://sqlite.org/pragma.html#pragma_index_xinfo
[table_xinfo]: https://sqlite.org/pragma.html#pragma_table_xinfo
