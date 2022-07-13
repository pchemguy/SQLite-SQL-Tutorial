---
layout: default
title: Engine
nav_order: 2
parent: SQLite Metadata
permalink: /meta/engine
---

The following code creates views showing engine-related metadata, including [SQLite version][], [Source ID][], [function list][], [collation list][], [compile options][], [module list][], and [pragma list][]. The included SELECT statements retrieve this information directly if executed from a console or a GUI DBA tool.

~~~sql
CREATE VIEW service_meta_sqlite_version AS
SELECT sqlite_version() AS version;

CREATE VIEW service_meta_sqlite_version_ex AS
SELECT sqlite_version() AS version, sqlite_source_id() AS source_id

CREATE VIEW service_list_functions AS
SELECT * FROM pragma_function_list() AS functions ORDER BY name;

CREATE VIEW service_list_collations AS
SELECT * FROM pragma_collation_list() AS collations;

CREATE VIEW service_list_compile_options AS
SELECT * FROM pragma_compile_options() AS compile_options;

CREATE VIEW service_list_modules AS
SELECT * FROM pragma_module_list() AS modules ORDER BY name;

CREATE VIEW service_list_pragmas AS
SELECT * FROM pragma_pragma_list() AS pargmas ORDER BY name;
~~~


<!-- References -->

[SQLite version]: https://sqlite.org/lang_corefunc.html#sqlite_version
[Source ID]: https://www.sqlite.org/lang_corefunc.html#sqlite_source_id
[function list]: https://sqlite.org/pragma.html#pragma_function_list
[collation list]: https://sqlite.org/pragma.html#pragma_collation_list
[compile options]: https://sqlite.org/pragma.html#pragma_compile_options
[module list]: https://sqlite.org/pragma.html#pragma_module_list
[pragma list]: https://sqlite.org/pragma.html#pragma_pragma_list
