---
layout: default
title: Mock Database
nav_order: 2
parent: Materialized Paths
permalink: /mat-paths/mock-database
---

Materialized paths (MPs) model is one of several approaches to storing a tree structure in a relational database (for details, please see [References][TreesAndRDBMS]). This project focuses on developing SQL/SQLite scripts for performing various MPs operations using this [mock SQLite database][].

### Schema

An MPs database must have at least three tables:

  - *categories* - the main table for storing the categories tree;
  - *items* (the *files* table in the mock db) - table for storing categorized items;
  - *items_categories* (the *files_categories* table in the mock db) - many-to-many table for storing category assignments.

The database may contain any number of "items" tables together with associated *items_categories* tables. Note that the included in the repository database has the "dev" status. For example, the *categories* table includes columns, so some of them will be unnecessary in a production database.

#### Categories

~~~sql
CREATE TABLE categories (
    id            INTEGER PRIMARY KEY NOT NULL,
    ascii_id      TEXT UNIQUE COLLATE BINARY AS (
                      char(
                          (abs(id) >> 8 * 7) & 255,
                          (abs(id) >> 8 * 6) & 255,
                          (abs(id) >> 8 * 5) & 255,
                          (abs(id) >> 8 * 4) & 255,
                          (abs(id) >> 8 * 3) & 255,
                          (abs(id) >> 8 * 2) & 255,
                          (abs(id) >> 8 * 1) & 255,
                          (abs(id) >> 8 * 0) & 255
                      )
                  ) STORED
                  CHECK (length(ascii_id) = 8),
    name          TEXT NOT NULL COLLATE NOCASE
                  CHECK (NOT instr(name, '/')        AND 
                         NOT instr(name, char(0x5C)) AND 
                         NOT instr(name, ',')        AND 
                         NOT instr(name, '"') ),
    prefix        TEXT NOT NULL COLLATE NOCASE
                  CHECK (iif(length(prefix) > 0, substr(prefix, 1, 1) <> '/' AND substr(prefix, - 1, 1) = '/', 1)),
    path          TEXT NOT NULL UNIQUE COLLATE NOCASE AS (prefix || name || '/') STORED,
    CONSTRAINT uq_categories_prefix_name UNIQUE(name, prefix),
    CONSTRAINT fk_categories_categories_prefix_path FOREIGN KEY (prefix)
      REFERENCES categories (path) ON DELETE NO ACTION ON UPDATE NO ACTION
);

CREATE INDEX idx_categories_prefix ON categories (prefix);
~~~

This table contains three independent and two stored generated columns. The former (all required) include

  - *name* - category name,
  - *prefix* - category prefix,
  - *id* - category id.

The category name should preferably include alphanumeric characters only. Other characters may cause problems; slashes, comma, and double quote are prohibited (see the CHECK constraint). Names starting with the tilde character are reserved for internal use.

There are several [approaches][TreesAndRDBMS] to encoding the *prefix* value based on an FS-like path using node names or IDs. With fixed length IDs, the path separator is unnecessary. For example, 8-character ASCII IDs and delimiterless VARCHAR(255) (where available) may encode a hierarchy with up to 32 levels. The current implementation uses plain FS-like paths with the forward slash as the path separator. All prefixes are absolute and include trailing but not leading slash to simplify code. Top-level nodes have the blank string as their prefix.

The *ascii_id* column is a random eight-character long string, which may contain alphanumeric ASCII characters, dash, and underscore (the last two characters bring the alphabet size to 2^6 characters). This column  encodes a 64-bit number generated from the *id* column. In practice, however, the process is the [opposite](../patterns/ascii-id). The ID-based prefix encoding may use this column in the future.

The *path* column (generated from *prefix* and *name*) is the reference target of the foreign key from the *files_categories* table. This deliberate deviation from the more natural *id* or *ascii_id* targets simplifies several operations. 

By definition, the employed MPs model fulfills [design **Rule #1**](./design-rules#Rules). The generated *path* column implements **Rule #3**, and the unique constraint enforces **Rule #5**. By design, each category can have only a single parent reference via the *prefix* column, which defines its parent's *path*, enforcing **Rule #2**. **Rule #4** is semantic and not subject to database controls. *name*/*prefix*/*path* collations enforce **Rule #6**. Many-to-many item-category relation tables implement **Rule #7**, and the separation of tables  for categories and items ensures **Rule #8**. The CHECK constraint on the *name* column ensures the absence of the most troublesome characters in the values.

The last constraint is a self-referential foreign key *prefix* &rarr; *path* to the parent category. For now, both UPDATE and DELETE cascades are disabled. These cascades are only usable for basic scenarios and should not be relied upon in general.

Note that only the prefix column needs an index, with other columns having automatic indices.

#### Item tables

~~~sql
CREATE TABLE files (
    ascii_id TEXT PRIMARY KEY NOT NULL COLLATE BINARY,
    name     TEXT NOT NULL COLLATE NOCASE,
    path     TEXT NOT NULL COLLATE NOCASE
);
~~~

The demo *files* table has a minimalistic structure. The *path* column is not related to the *path* column of the *categories* table, even though mock data comes from the same dataset. The many-to-many *files_categories* table establishes the actual relationship.

~~~sql
CREATE TABLE files_categories (
    cat_id  TEXT COLLATE NOCASE,
    file_id TEXT COLLATE BINARY,
    CONSTRAINT fk_files_categories_categories FOREIGN KEY (cat_id)
      REFERENCES categories (path) ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT fk_files_categories_files FOREIGN KEY (file_id)
      REFERENCES files (ascii_id) ON DELETE CASCADE,
    CONSTRAINT uq_files_categories_cat_id_file_id UNIQUE (cat_id, file_id)
      ON CONFLICT REPLACE
);

CREATE INDEX idx_files_categories_file_id ON files_categories (file_id);
~~~

As discussed later, the delete category operation moves the target subtree to the *~Trash* category instead of deleting the records from the table. For this reason, the cascaded deletion on the foreign key to the *categories* table usually is a no-op. For a similar reason, the unique constraint is used instead of the primary key on the table columns. This constraint scheme permits setting constrained columns to null, facilitating the "move to Trash" operation mentioned above (see the relevant section for further details). The conflict resolution clause on the unique index simplifies the category move operation.


<!-- References -->

[TreesAndRDBMS]: ./design-rules#TreesAndRDBMS
[mock SQLite database]: https://github.com/pchemguy/SQLite-SQL-Tutorial/raw/gh-pages/Assets/database/Materialized%20Paths.db
