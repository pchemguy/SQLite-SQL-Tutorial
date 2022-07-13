---
layout: default
title: Basic DB Meta
nav_order: 3
parent: SQLite Metadata
permalink: /meta/db-basic
---

Some of the database-related metadata is available via basic SELECT queries. This information includes 

 - [database list][] and [list of tables and views][table-view list] (only available in recent versions);
 - [DDL statements][] for all database objects;
 - several scalar values ([Journal Mode][], [Application ID][], [User Version][], and [Schema Version][], which is not protected but should only be changed by the engine).

~~~sql
-- DATABASES
CREATE VIEW service_list_databases AS
SELECT * FROM pragma_database_list() AS database_list;

-- TABLES AND VIEWS (only available in SQLite 3.37.0 or newer)
CREATE VIEW service_list_tables_views AS
SELECT * FROM pragma_table_list() AS db_tables_views;

-- ========== DDL Statements ==========

-- DATABASE OBJECTS
CREATE VIEW service_list_database_objects AS
SELECT tbl_name AS table_name, type, name AS object_name, sql
FROM main.sqlite_master ORDER BY table_name, object_name;

-- TABLES (excluding system tables with the "sqlite_" name prefix)
CREATE VIEW service_list_tables AS
SELECT tbl_name AS table_name, sql
FROM main.sqlite_master AS db_tables
WHERE type = 'table' AND (name NOT LIKE 'sqlite_%') ORDER BY table_name;

-- INDICES (including system indices with the "sqlite_" name prefix)
CREATE VIEW service_list_indexes AS
SELECT tbl_name AS table_name, name AS index_name, sql
FROM main.sqlite_master AS db_indexes
WHERE type = 'index' ORDER BY table_name;

-- VIEWS
CREATE VIEW service_list_views AS
SELECT name AS view_name, sql
FROM main.sqlite_master AS db_views
WHERE type = 'view' ORDER BY view_name;

-- TRIGGERS
CREATE VIEW service_list_triggers AS
SELECT name AS trigger_name, sql
FROM main.sqlite_master AS db_triggers
WHERE type = 'trigger' ORDER BY trigger_name;

-- ========== Scalars ==========

-- APPLICATION ID (read/write)
CREATE VIEW service_meta_application_id AS
SELECT * FROM pragma_application_id() AS app_id;

-- USER VERSION (read/write)
CREATE VIEW service_meta_user_version AS
SELECT * FROM pragma_user_version() AS user_version;

-- SCHEMA VERSION (should only be changed by the engine)
CREATE VIEW service_meta_schema_version AS
SELECT * FROM pragma_schema_version() AS schema_version;

-- JOURNAL MODE
CREATE VIEW service_meta_journal_mode AS
SELECT * FROM pragma_journal_mode() AS journal_mode;
~~~


<!-- References -->

[Application ID]: https://sqlite.org/pragma.html#pragma_application_id
[User Version]: https://sqlite.org/pragma.html#pragma_user_version
[Schema Version]: https://sqlite.org/pragma.html#pragma_schema_version
[Journal Mode]: https://sqlite.org/pragma.html#pragma_journal_mode
[database list]: https://sqlite.org/pragma.html#pragma_database_list
[table-view list]: https://sqlite.org/pragma.html#pragma_table_list
[DDL statements]: https://sqlite.org/schematab.html
