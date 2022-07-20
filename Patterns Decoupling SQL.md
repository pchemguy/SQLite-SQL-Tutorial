---
layout: default
title: Decoupling SQL
nav_order: 1
parent: Design Patterns
permalink: /patterns/decoupling-sql
---

### Parameterized queries as surrogate stored procedures - decoupling the SQL code

Where supported, stored procedures permit encapsulation of SQL code details, reducing the coupling between the application and the storage backend. Stored procedures are similar to typical library routines:  the application only needs to know about the **SQL interface**, which includes the calling signature (names of the stored procedure and its parameters) and the return format (returned column names or ordering). SQLite does not support stored procedures, but parameterized queries provide similar functionality.

In this case, the application is responsible for managing the SQL code. A common approach involves interspersing the SQL code within the application source code as necessary and embedding the SQL queries as text within the application binaries. A better strategy is to combine SQL snippets into a source code library represented by textual modules containing SQL code only. The application should load this library at startup and parse it (SQL comments in the source library can hold the necessary metadata along with inline documentation).

Another possible approach to managing the SQL queries is storing them in a database table, such as *queries(name, sql, doc, ...)*. A dedicated application routine would query this table, returning an SQL snippet based on its name. This routine is the only potential exception for inlining basic SQL, responsible for accessing the *queries* table.

As long as the interfaces of individual SQL snippets are unchanged, both approaches decouple the SQL library from the application. Also, similar to stored procedures, parameterized queries can present black boxes to the application. The application code can use a conventional query label/name to identify the appropriate snippet (in either file or database) and send it verbatim to the database. This usage pattern is supported via a similar SQL interface, involving the query label, query parameters, and returned columns. Such an SQL library may be reused, for example, on different platforms or, where relevant, with multiple API bindings.