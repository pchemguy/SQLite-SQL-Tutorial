---
layout: default
title: Engine
nav_order: 2
parent: SQLite Metadata
permalink: /meta/engine
---

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
