---
layout: default
title: SQLite Metadata
nav_order: 2
permalink: /sqlite-meta
---

### SQLite Metadata

Metadata information is essential for both developer and administrator. Various metadata chunks may provide information about the database engine (available features) or the schema of a specific database.

The database-related metadata makes it possible to reverse engineer and verify its schema. Structured database metadata is available via the [PRAGMA][] statement and associated [PRAGMA functions][] and
includes information about all database objects and the most important properties. This metadata also permits certain schema-related quality checks. The SQLite engine derives structured metadata from the Data Definition/Description Language (DLL) statements and stores it in the system schema tables (*sqlite_master* and *sqlite_temp_master* for the *temp* database). An application can query schema tables like any other regular table to access complete schema information; however, the application would need to parse the DDL statements to extract desired information.

Because of the small footprint, applications often include a copy of the SQLite engine, ensuring that a recent library with all necessary features is used (the systemwide library copy is usually older and feature-limited). And because of the modular architecture, different SQLite library copies on the same computer usually provide different feature sets. For example, the default build of the [SQLiteODBC][] driver embeds an outdated (as of this writing) SQLite copy, providing a reduced feature set (alternative build scripts for Windows MSYS and MSBuild toolchains are [available][SQLiteODBC PChemGuy]). Consequently, a developer relying, for example, on the default SQLiteODBC driver for the target application and on a GUI DBA tool to facilitate the development process may face odd problems. For these reasons, it is prudent, if not essential, to retrieve engine-related metadata both via the DBA tool and through the instrumented application directly. When the developer's application provides the ability to use an external copy of SQLite, it is wise to instrument the end-user version of the application and verify during the application launch that the supplied SQLite library provides all necessary features.

### Engine Metadata

The following code creates views showing engine-related metadata, and the included SELECT statements retrieve this information directly.

~~~sql
-- URL: https://sqlite.org/lang_corefunc.html#sqlite_version
CREATE VIEW service_meta_sqlite_version AS
SELECT sqlite_version() AS version;

-- URL: https://www.sqlite.org/lang_corefunc.html#sqlite_source_id
CREATE VIEW service_meta_sqlite_version_ex AS
SELECT sqlite_version() AS version, sqlite_source_id() AS source_id

-- URL: https://sqlite.org/pragma.html#pragma_function_list
CREATE VIEW service_list_functions AS
SELECT * FROM pragma_function_list() AS functions ORDER BY name;

-- URL: https://sqlite.org/pragma.html#pragma_collation_list
CREATE VIEW service_list_collations AS
SELECT * FROM pragma_collation_list() AS collations;

-- URL: https://sqlite.org/pragma.html#pragma_compile_options
CREATE VIEW service_list_compile_options AS
SELECT * FROM pragma_compile_options() AS compile_options;

-- URL: https://sqlite.org/pragma.html#pragma_module_list
CREATE VIEW service_list_modules AS
SELECT * FROM pragma_module_list() AS modules ORDER BY name;

-- URL: https://sqlite.org/pragma.html#pragma_pragma_list
CREATE VIEW service_list_pragmas AS
SELECT * FROM pragma_pragma_list() AS pargmas ORDER BY name;
~~~


<!-- References -->

[SQLiteODBC]: http://ch-werner.de/sqliteodbc
[SQLiteODBC PChemGuy]: https://pchemguy.github.io/SQLite-ICU-MinGW/odbc
[PRAGMA]: https://sqlite.org/pragma.html
[PRAGMA functions]: https://sqlite.org/pragma.html#pragfunc
