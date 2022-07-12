---
layout: default
title: Modular SQL-SQLite with CTE - Database and Engine Metadata
nav_order: 2
permalink: /meta-cte
---

This project discusses select practical SQL problems, which I come across in the learning process, and the associated solutions. Its primary purpose is the organization of the acquired SQL/[SQLite][] knowledge. A separate project (the [SQLiteC for VBA][SQLiteCAdoReflectVBA] library) serves a similar purpose for the C-API interface of the library and its integration into an information management application. My long-term goal of using SQLite as a backend for managing personal and research information governs the scope of the discussed topics.

### Limitations and Workarounds

Because of the [target uses][SQLite Apps], SQLite lacks several useful features provided by more powerful client/server RDBMS engines, such as stored procedures, support for variables, and dynamic SQL. At the same time, the common tables expressions ([CTEs][]) feature implemented by SQLite provides an important tool for structured/modular programming. CTEs significantly simplify the development and readability of complex queries and provide a convenient debugging tool. [Parameterized queries][] at least partially compensate for missing stored procedures. The set of string manipulation functions provided by SQLite also  misses basic features, some of which can be implemented via its JSON module. Two advanced features -  [recursive CTEs][RCTEs] and [window functions][fWin] - also provide interesting functionality.

### Database Administration Tools

I have tried several GUI database administration (DBA) tools. Most of the time, I use [SQLiteStudio][] and [DB Browser for SQLite][]. Both are feature-rich solid DBA tools. Among other features, SQLiteStudio has a powerful multiple-document interface and better support for extended DDL and multiple database connections. DB Browser for SQLite, on the other hand, provides instant search and multi-tabbed SQL script editor with code highlighting and completion. Occasionally, I use [DBeaver][] (a commercial tool with a community open source edition) and [HeidiSQL][].

<!--
Summary of topics:
* CTE and structural programming
* Database metadata
* Using JSON to implement frontend - backend interface at the SQL level
* String parsing via JSON
* Splitting delimiterless strings
* Path parsing via JSON
* Materialized Paths implementation in SQL (include background, usage patterns, providing performance considerations)
-->


<!-- References -->

[SQLite]: https://sqlite.org
[SQLiteCAdoReflectVBA]: https://pchemguy.github.io/SQLiteC-for-VBA/
[SQLite Apps]: https://sqlite.org/whentouse.html
[CTEs]: https://sqlite.org/lang_with.html
[Parameterized queries]: https://sqlite.org/lang_expr.html#varparam
[RCTEs]: https://sqlite.org/lang_with.html#recursive_common_table_expressions
[fWin]: https://sqlite.org/windowfunctions.html
[SQLiteStudio]: https://sqlitestudio.pl
[DB Browser for SQLite]: https://sqlitebrowser.org
[DBeaver]: https://dbeaver.io
[HeidiSQL]: https://heidisql.com
