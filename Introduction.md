---
layout: default
title: Introduction
nav_order: 1
permalink: /
---

This project discusses select practical SQL problems and the associated solutions. I primarily use this project to organize my acquired SQL/[SQLite][] knowledge. A separate project (the [SQLiteC for VBA][SQLiteCAdoReflectVBA] library) serves a similar purpose for the C-API interface of the library and its integration into an information management application. My interest in SQLite is motivated by the need for local personal and research information management. Special consideration is given to storing tree structures used for information classification. I am particularly interested in use cases necessitating short database response times (as measured by the application) on the order of 10-20 ms and up to 50-100 ms for relatively simple and more complicated queries, respectively. Realistically, only an embedded database engine might be able to meet these specs, hence my interest in SQLite and the scope of the discussed topics. While I attempt to make the material accessible, this tutorial may not be suitable for first-timers.

### Project application scope and motivation

[SQL][] is a [domain-specific][] [declarative][] programming language; by design, it either lacks completely or provides only rudimentary support for many essential features available in [general-purpose][] languages. Support for other features may only be available through vendor-specific extensions. Limited support for procedural and structured code, variables, and flow control constructs impedes the development of loosely [coupled][coupling] SQL code amendable to [refactoring][] and code reuse. The scope of these limitations is vendor-specific, and SQLite, for a good reason, is more affected than popular client/server database engines. On the other hand, the extent to which these limitations may hinder the development of database-oriented applications depends on specific circumstances.

For example, developers often delegate SQL generation to [middleware][] components, minimizing the need to write SQL directly. Middleware decouples applications from SQL and often provides unified multi-vendor APIs. This added abstraction layer, however, also imposes additional limitations, making SQL limitations largely irrelevant. While the increased level of abstraction facilitates the development process, it also adds a varying run-time performance penalty (which may not be an issue).

At the other extreme is the development strategy not relying on SQL abstracting components, which exposes all features of a particular SQL dialect and its limitations to the developer. Whether these limitations become an issue depends on how much extra work the database performs on top of base data retrieval and persistence operations. Delegating higher-level model-related data manipulations to the database may provide several benefits:

  - Efficient data manipulation operations are at the core of database engines. If the desired result involves a substantial amount of such operations, often the database may do the job more efficiently.
  - Minimizing data transfers between the database and the application is almost always beneficial. Such operations are inherently slow (to a lesser extent for embedded in-process engines, such as SQLite), and the characteristic time of data transfer operations can be several magnitudes higher than local processing times.
  - Implementing model-related logic in the SQL code may resemble the object-oriented paradigm and reduce coupling between the application and its backend. Such code chunks can be used by various frontends on different platforms without any modification so long as the backend remains the same.

SQLite supports several advanced features facilitating the development of structured, reusable, and loosely coupled SQL code, including parameterized queries, common table expressions (CTEs), recursive CTEs, and the JSON library. The [Metadata and Reflection][] section provides basic and more advanced SQL queries for retrieving database- and engine-related metadata, illustrating the use of simple CTEs for structuring SQL code. The [Design Patterns][] section discusses all these mentioned features in detail. It also provides possible approaches to compensating for the missing support for variables and limited support for string operations, illustrating the utility of these features. Finally, the [Materialized Paths][] applies discussed ideas to a sample implementation of a hierarchical category system in SQL. This section explores the possibility of developing an OOP-like SQL source code library implementing all typical operations for such a system in SQL.

### Structured developement strategies and separation of concerns

Because of the [target uses][SQLite Apps], SQLite lacks several useful features provided by more powerful client/server RDBMS engines, such as stored procedures, support for variables, and dynamic SQL. The absence of these features limits the ability to develop structured/modularized SQL code and separate the internal database organization and application business logic. Another possible concern is associated with SQLite's limited string manipulation capabilities.

At the same time, SQLite provides several tools which help compensate for the mentioned deficiencies. These tools include common table expressions ([CTEs][]), [recursive CTEs][RCTEs], the [JSON][] library, [window functions][fWin], and [parameterized queries][].

CTEs provide a powerful means for structuring complex queries into simple reusable code blocks, simplifying the development process and readability of the code. CTEs also make it possible to conveniently interrogate any intermediate code block without the need to make changes to the CTEs section. Intermediate queries may also be replaced with mocks constructed from immediate values, making it possible to verify the functionality of different code sections and develop them independently. The JSON library extends string manipulation capabilities, while parameterized queries help encapsulate SQL code. Single row queries constructed from immediate values placed within the CTEs clause may act as surrogate variables. Later, these values can be replaced with named parameters, yielding parameterized queries. The combination of CTEs, JSON, and parameterized queries permits an even higher degree of SQL code encapsulation with a JSON-based structured SQL interface and certain input flexibility.

### Developer tools

I have tried several GUI database administration (DBA) tools. Most of the time, I use [SQLiteStudio][] and [DB Browser for SQLite][]. Both are feature-rich solid DBA tools. Among other features, SQLiteStudio has a powerful multiple-document interface and better support for extended DDL and multiple database connections. DB Browser for SQLite, on the other hand, provides instant search and multi-tabbed SQL script editor with code highlighting and completion. Occasionally, I use [DBeaver][] (a commercial tool with a community open source edition) and [HeidiSQL][]. I use [StackEdit.io][] as the primary Markdown editor, [Tables Generator][] to prepare Markdown-formatted tables, and [yEd][] for diagrams.

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

[SQL]: https://en.wikipedia.org/wiki/SQL
[domain-specific]: https://en.wikipedia.org/wiki/Domain-specific_language
[declarative]: https://en.wikipedia.org/wiki/Declarative_programming
[general-purpose]: https://en.wikipedia.org/wiki/General-purpose_language
[coupling]: https://en.wikipedia.org/wiki/Coupling_(computer_programming)
[refactoring]: https://en.wikipedia.org/wiki/Code_refactoring
[middleware]: https://en.wikipedia.org/wiki/Middleware

[Metadata and Reflection]: meta
[Design Patterns]: patterns
[Materialized Paths]: mat-paths

[SQLite Apps]: https://sqlite.org/whentouse.html
[CTEs]: https://sqlite.org/lang_with.html
[RCTEs]: https://sqlite.org/lang_with.html#recursive_common_table_expressions
[JSON]: https://sqlite.org/json1.html
[Parameterized queries]: https://sqlite.org/lang_expr.html#varparam
[fWin]: https://sqlite.org/windowfunctions.html
[SQLiteStudio]: https://sqlitestudio.pl
[DB Browser for SQLite]: https://sqlitebrowser.org
[DBeaver]: https://dbeaver.io
[HeidiSQL]: https://heidisql.com
[StackEdit.io]: https://stackedit.io
[Tables Generator]: https://tablesgenerator.com
[yEd]: https://yworks.com/products/yed
