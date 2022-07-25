---
layout: default
title: Decoupling SQL
nav_order: 1
parent: Design Patterns
permalink: /patterns/decoupling-sql
---

### Parameterized queries as surrogate stored procedures - decoupling the SQL code

Where supported, stored procedures permit encapsulation of SQL code details, reducing the coupling between the application and the storage backend. Stored procedures are similar to typical library routines:  the application only needs to know the **SQL interface**, which includes the calling signature (names of the stored procedure and its parameters) and the return format (returned column names/ordering).

SQLite does not support stored procedures, and the application is responsible for managing the SQL code. A common approach involves interspersing the SQL code within the application source code as necessary and embedding the SQL queries as text within the application binaries. This approach mixes two very different development targets (which might be the responsibility of separate developers) within the same source code modules. As a result, small pieces of SQL code may end up in multiple source files, complicating SQL code navigation and making it difficult to follow the DRY principle (avoiding SQL code duplication).

A better strategy is to combine SQL snippets into a source code library represented by textual modules containing SQL code only. The application should load this library at startup and parse it (SQL comments in the source library can hold the necessary metadata along with inline documentation). Alternatively, the application may store SQL snippets in a database table, such as *queries(name, sql, doc, ...)*. A dedicated application routine would query this table, returning an SQL snippet based on its name. This routine is the only potential exception for inlining basic SQL, responsible for accessing the *queries* table.

With either storage approach, such an SQL library puts all SQL code in compact SQL-only form, facilitating the development process. It is essential, however, that both the calling application and SQL rely on parameterized queries rather than dynamic SQL generation by the application. Parameterized queries, like stored procedures, can present black boxes to the application with a similar SQL interface, which involves the query label, query parameters, and returned columns. As long as these SQL interfaces are stable, the application code can use a conventional query label/name to identify the appropriate snippet (in either file or database library) and send it verbatim to the database. Such an SQL library may be reused, for example, on different platforms or, where relevant, with multiple API bindings.
