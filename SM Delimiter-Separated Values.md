---
layout: default
title: Delimiter-Separated Values
nav_order: 1
parent: String Manipulation
permalink: /strings/split-dsv
---

Note that the scripts in this section rely on the SQLite JSON library. In the latest version of SQLite, the JSON library is part of the core and is included in the build by default. Until recently, however, JSON has been developed as a loadable extension, which was not part of the default builds. Verify that the JSON functionality is available before using/adapting code from this tutorial (for example, use these [queries](/meta/engine)).

### Splitting delimiter-separated value strings

SQLite does not have a dedicated SQL function for splitting delimiter-separated values (DSV) strings, such as a file system path or a CSV string. Still, the *json_each* table-valued function provides functionality suitable for performing this operation. This function splits a string containing a well-formatted JSON object or array and returns a table having one first-level member per row. Before splitting, preprocessing code must convert the format from DSV to JSON array. The conversion process involves replacing the DSV separator with a quoted comma via the *replace* function and adding opening/closing quotes and brackets to complete the JSON array. The following query splits a path-like string into individual path elements:

~~~sql
SELECT terms.id AS term_id, terms.value AS term
FROM json_each('["' || replace('usr/share/man', '/', '", "') || '"]') AS terms;
~~~

**Output**

| term_id | term  |
|---------|-------|
| 1       | usr   |
| 2       | share |
| 3       | man   |

This query will fail if the string contains quotation marks or backslashes (assuming these characters are not JSON escaped). Another potential issue involves leading and terminal separators. However, developing a universal string-splitting SQL script is impractical. Instead, the script functionality should match reasonable assumptions about the input format for a specific use case.

For instance, let's improve the script to

 - split path-like variables (both Linux and Windows styles),
 - handle leading/terminal value separator,
 - handle doubled value separators (such as ";;"),
 - normalize all paths by stripping leading/terminal path separators,
 - convert any backslashes to slashes.

<a name="DSV-Query"></a>
~~~sql
WITH
    params(sep, path_sep) AS (VALUES (':', '\/')),
    string_data(string_id, string) AS (
        VALUES
            (1, '\usr\share\man\::bin:etc/mc:'),
            (2, '/dev/stderr/:/dev/stdout/')
    ),
    clean_strings AS (
        SELECT strs.string_id,
               replace(replace(trim(strs.string, p.sep), p.sep || p.sep, p.sep), '\', '\\') AS string
        FROM string_data AS strs, params AS p
    ),
    json_strings AS (
        SELECT strs.string_id,
               '["' || replace(strs.string, p.sep, '", "') || '"]' AS string
        FROM clean_strings AS strs, params AS p
    ),
    raw_terms AS (
        SELECT string_id, terms.id AS term_id, terms.value AS term
        FROM json_strings AS strs, json_each(strs.string) AS terms
    ),
    terms AS (
        SELECT string_id, term_id, replace(trim(term, p.path_sep), '\', '/') AS term
		FROM raw_terms, params AS p
	)
SELECT * FROM terms;
~~~

Going from top to bottom:

- *params* acts as surrogate variables. *params* has one row, and it defines value and path separators. If this flexibility is excessive or unnecessary, adjust or remove this block. With the given definition, this query handles the Linux-style path variable, and the *params.sep* must be adjusted for Windows-style paths.
- *string_data* defines the input data to be processed.
- *clean_strings* performs input preprocessing. This code may clean leading/terminal/duplicate value separators and escape backslashes.
- *json_strings* formats the preprocessed output of *clean_strings* as JSON arrays.
- *raw_terms* splits the input (*json_each* unescapes backslashes automatically).
- *term* performs postprocessing (in this case, it replaces backslashes with slashes).
