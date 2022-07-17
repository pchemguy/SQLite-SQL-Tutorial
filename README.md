---
layout: default
title: Introduction
nav_order: 1
permalink: /
---

This project discusses select practical SQL problems and the associated solutions. I primarily use this project to organize my acquired SQL/[SQLite][] knowledge. A separate project (the [SQLiteC for VBA][SQLiteCAdoReflectVBA] library) serves a similar purpose for the C-API interface of the library and its integration into an information management application. My interest in SQLite is motivated by the need for local personal and research information management. I am particularly interested in use cases necessitating short database response times (as measured by the application) on the order of 10-20 ms and up to 50-100 ms for relatively simple and more complicated queries, respectively. Realistically, only an embedded database engine might be able to meet these specs, hence my interest in SQLite and the scope of the discussed topics. While I attempt to make the material accessible, this tutorial may not be suitable for first-timers.

### Structured developement strategies and separation of concerns

Because of the [target uses][SQLite Apps], SQLite lacks several useful features provided by more powerful client/server RDBMS engines, such as stored procedures, support for variables, and dynamic SQL. The absence of these features limits the ability to develop structured/modularized SQL code and separate the internal database organization and application business logic. Another possible concern is associated with SQLite's limited string manipulation capabilities.

At the same time, SQLite provides several tools which help compensate for the mentioned deficiencies. These tools include common table expressions ([CTEs][]), [recursive CTEs][RCTEs], the [JSON][] library, [window functions][fWin], and [parameterized queries][].

CTEs provide a powerful means for structuring complex queries into simple reusable code blocks, simplifying the development process and readability of the code. CTEs also make it possible to conveniently interrogate any intermediate code block without the need to make changes to the CTEs section. Intermediate queries may also be replaced with mocks constructed from immediate values, making it possible to verify the functionality of different code sections and develop them independently. The JSON library extends string manipulation capabilities, while parameterized queries help encapsulate SQL code. Single row queries constructed from immediate values placed within the CTEs clause may act as surrogate variables. Later, these values can be replaced with named parameters, yielding parameterized queries. The combination of CTEs, JSON, and parameterized queries permits an even higher degree of SQL code encapsulation with a JSON-based structured SQL interface and certain input flexibility.


<!-- References -->

[SQLite]: https://sqlite.org
[SQLiteCAdoReflectVBA]: https://pchemguy.github.io/SQLiteC-for-VBA/
[SQLite Apps]: https://sqlite.org/whentouse.html
[CTEs]: https://sqlite.org/lang_with.html
[RCTEs]: https://sqlite.org/lang_with.html#recursive_common_table_expressions
[JSON]: https://sqlite.org/json1.html
[Parameterized queries]: https://sqlite.org/lang_expr.html#varparam
[fWin]: https://sqlite.org/windowfunctions.html
