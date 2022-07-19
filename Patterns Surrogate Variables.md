---
layout: default
title: Surrogate Variables and JSON SQL Interface
nav_order: 3
parent: Design Patterns
permalink: /patterns/variables-json-interface
---

Let us rewrite the previous [query](/patterns/split-dsv#DSV-Query) using common tables expressions (CTEs):

~~~sql
WITH
    folders AS (
        SELECT "terms"."key" AS term_id, "terms"."value" AS term
        FROM json_each('["' || replace(trim('C:/Winows/System32/drivers/etc/', '/'), '/', '", "') || '"]') AS terms
    )
SELECT * FROM folders;
~~~
