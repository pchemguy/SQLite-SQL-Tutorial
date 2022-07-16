---
layout: default
title: Introduction
nav_order: 1
permalink: /
---

This project discusses select practical SQL problems, which I come across in the learning process, and the associated solutions. I primarily use this project to organize the acquired SQL/[SQLite][] knowledge. A separate project (the [SQLiteC for VBA][SQLiteCAdoReflectVBA] library) serves a similar purpose for the C-API interface of the library and its integration into an information management application. My interest in SQLite is motivated by the need for local personal and research information management. I am particularly interested in use cases necessitating short database response times (as measured by the application) on the order of 10-20 ms and up to 50-100 ms for relatively simple and more complicated queries, respectively. Realistically, only an embedded database engine might be able to meet these specs, hence my interest in SQLite and the scope of the discussed topics. While I attempt to make the material accessible, this tutorial may not be suitable for first-timers.

### SQLite Limitations and Workarounds

Because of the [target uses][SQLite Apps], SQLite lacks several useful features provided by more powerful client/server RDBMS engines, such as stored procedures, support for variables, and dynamic SQL. At the same time, the "common tables expressions" ([CTEs][]) feature implemented by SQLite provides a powerful tool for structured/modular programming. CTEs significantly simplify the development and readability of complex queries and provide a convenient debugging tool. [Parameterized queries][] at least partially compensate for missing stored procedures. The set of SQLite's string manipulation functions also misses basic features, though its JSON module provides some alternative options. Two advanced features -  [recursive CTEs][RCTEs] and [window functions][fWin] - also provide helpful functionality.

### Developer Tools

I have tried several GUI database administration (DBA) tools. Most of the time, I use [SQLiteStudio][] and [DB Browser for SQLite][]. Both are feature-rich solid DBA tools. Among other features, SQLiteStudio has a powerful multiple-document interface and better support for extended DDL and multiple database connections. DB Browser for SQLite, on the other hand, provides instant search and multi-tabbed SQL script editor with code highlighting and completion. Occasionally, I use [DBeaver][] (a commercial tool with a community open source edition) and [HeidiSQL][].

I use [StackEdit.io][] as the primary Markdown editor, [Tables Generator][] to prepare Markdown-formatted tables, and [yEd][] for diagrams.

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
[StackEdit.io]: https://stackedit.io
[Tables Generator]: https://tablesgenerator.com
[yEd]: https://yworks.com/products/yed