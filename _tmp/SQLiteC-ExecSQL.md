---
layout: default
title: ExecSQL
nav_order: 7
parent: SQLiteC
permalink: /sqlitec/execsql
---

**Database cursor**
SQLite provides only the simplest _forward-only_ cursor via the [step API][]. The SQLite step API is the primary method for executing a prepared statement and moving the database cursor forward to the next available record (the shortcut *exec* API is a wrapper about the *step* API). The *ExecuteStepAPI()* method wraps the step API and is meant for internal use by the class. SQLiteCStatement does not provide an *exec* API wrapper, which is provided by SQLiteCConnection (*ExecuteNonQueryPlain()*).

**Rows loop**
SQLite does not provide the number of rows returned by a SELECT query. Instead, to determine rowset size, the application must specifically issue a counting query. Otherwise, the C API caller must run a while loop, moving the database cursor one row forward at a time via the *step* API, retrieve the row values, and repeat the process until the database indicates that no more rows are available. (The legacy interface *get_table* returns a 2D array with all values cast as strings.) This workflow is suboptimal for retrieval of larger rowsets, and it may be necessary to apply some compensation techniques to avoid possible performance bottlenecks.

*GetPagedRowSet()* loops through rows in the result rowset and retrieves the data. The receiving VBA data structure requires special handling because the number of rows is unknown. One way to handle the data would be to save rows as elements in a runtime resizable array. This approach would necessitate preallocating slots for a set maximum number of rows or using ReDim Preserve. Following the algorithm of doubling the array size when the current space is out may yield a reasonable compromise between the space used and the overhead of dynamic resizing. This approach would also result in a simpler code than the currently implemented one.

Nevertheless, the presently used primary data structure is similar to that used by ADODB with three nested levels of dynamically allocated 1D arrays. At the top is the *Pages* array of size *PageCount*, the second layer is the *Page* array of size *PageSize*, and the last layer, *RowValues*, contains individual result rows as arrays of row values. *GetPagedRowSet()* preallocates the Pages array and allocates a new Page in the outer 'Page' loop when the inner 'Row' loop fills all slots in the current Page via the *GetRow()* function. The total capacity of the Pages array (*PageCount* times *PageSize*) defines the maximum number of rows retrieved. If the number of available rows is smaller than the capacity of the Pages array, a partially filled structure is returned with *FilledPagesCount* indicating the number of filled pages and *RowCount* indicating the total number of rows retrieved from the database.

**Row values loop**
Retrieving a single row is a somewhat involved process still. SQLite does not provide an interface to retrieve the entire row. Individual column values must be retrieved, For-looping through the columns. *GetRow()* and *GetColumnValueAPI()* implement this workflow. *GetColumnValueAPI()* selects appropriate value retrieval interface, using the value type API by default. Before retrieving the data, *GetPagedRowSet()* retrieves any available [metadata](./meta), including the declared type, which is used instead of value type, if the *UseDeclaredTypes* flag is True.

The *GetScalar()* method wraps *GetColumnValueAPI()* directly to execute scalar queries.

**Data shaping**
Two additional methods, *GetRowSet2D()* and *GetFabRecordset()* wrap *GetPagedRowSet()* method and provide two alternative data shapes. As the names suggest, *GetRowSet2D()* returns a plain row-wise 2D array, and *GetFabRecordset()* returns an ADODB.Recordset used for ILiteADO interface implementation and also suitable for Excel QueryTables.


<!-- References -->

[step API]: https://www.sqlite.org/c3ref/step.html
