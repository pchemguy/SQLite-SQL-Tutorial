---
layout: default
title: Design Rules
nav_order: 1
parent: Materialized Paths
permalink: /mat-paths/design-rules
---

This project explores SQLite capacity and features for managing hierarchical systems of categories. While the classical relational model does not mesh well with hierarchical data, there are several [approaches](#TreesAndRDBMS) to marrying these concepts. Category systems facilitate information [management](#ClassRefs) by associating each category with a specific characteristic/feature that distinguishes its members from nonmembers. The design of these category systems shall meet the following specification:

 1. Categories form a tree/forest structure.
 2. Each category has at most one parent. 
 3. A category is identified by its *path* (vis-a-vis an absolute normalized file system path).
 4. The category *path* encodes a simple classification feature associated with that category.
 5. The category *path* is unique (no namesake siblings).
 6. Category names are case-preserving and case-insensitive.
 7. Any categorized item may be assigned multiple categories.
 8. No category-item name collision.

It is instructive to compare and contrast systems described by these rules with file systems (FS), probably the most familiar hierarchical structures for the organization of file system objects (FSO), particularly files. Like directories organize files, categories facilitate the organization of categorized items. A large portion of the above rules governs naming conventions, some of which are similar to those of FSs, while others may appear counterintuitive. Fundamental differences between the two systems justify the latter choices. For once, while categories and categorized objects belong to separate object spaces (database tables), directories and files share the same namespace. Also, categories are closer to selection filters than surrogate containers, so category assignment is akin to tagging rather than placing objects in folder-like bins/containers and does not cause any object relocation, physical or virtual. With these differences in mind, let us provide some justification for the selected rules.

#### Rule #1

Classification schemes often employ tree-based FS-like structures. Typically, the hierarchy establishes general-to-specific vis-a-vis root-to-leaf mapping. The forest extension with multiple top-level/root nodes is beneficial for combining multiple classification modalities. While a "super" root node may convert a forest into a tree, the additional meaningless hierarchy level does not provide apparent benefits.

#### Rule #2

This rule is already implied by Rule #1 and is more restrictive than modern FSs, which support multiparent structures. While the multiparent feature has certain benefits, it enables loops and potentially unbound paths, complicating basic FS operations. In the case of category systems, I cannot think of a sound justification for having these complications.

#### Rules #3

Each category is assigned a name (vis-a-vis FSO name) and prefix (vis-a-vis absolute normalized FS path of the parent directory). Widespread classification systems illustrate this FS-like organization with several common variations in path encoding schemes. Natural classifications without any particular order, such as biological taxonomy, employ textual names for categories. Other systems use numerical names combined with a dictionary assigning semantic meaning to individual codes or a combination of a hierarchical numeric code and descriptive name. Conceptually, all these schemes are equivalent. Using descriptive names as human-facing identifiers is probably the most natural approach for a generic classification system. Additionally, category records may include numeric or alphanumeric identifiers for internal use.

#### Rules #4

This rule is essentially common sense. Many broadly used classification systems freely [available](#ClassRefs) on the Internet vividly illustrate it. 

#### Rule #5

Each category is associated with a distinct classification feature (encoded in the category's path, see Rule #4) used to identify a subset of objects sharing the same feature. Allowing sibling namesakes among categories is, therefore, meaningless and degrades, if not breaks, the utility of the classification system (cf. with FS paths, which identify FSOs).

#### Rule #6

There are FS with both case-sensitive and case-insensitive FSO names. For example, Windows FSO names are case-preserving and case-insensitive. IMHO, human-facing object identifiers should always be case-insensitive and, if possible, case-preserving, hence this rule.

#### Rules #7

The ability to assign multiple categories to items is a de-facto standard. This feature does not create loop-related complications for two reasons. On the one hand, categories are always parents to categorized items. More importantly, categories and items belong to separate object spaces (database tables). For these reasons, a category path and the item name combination is meaningless (no FS path analogy here).

#### Rules #8

Technically, category and item names do not share the same namespace; semantically, both are item attributes (this is very different from FS counterparts). Therefore, category and item names cannot collide.

The similar item-item interaction is beyond the scope of category system rules. Generally, categories act as tags and should not affect the item-item interaction. Typically, items should not define any common namespace. While, occasionally, having multiple identically named items within the same category may be suboptimal, they should not cause any interference in the functioning of the category system as an information management tool.


<a name="References"></a>
### References


<a name="TreesAndRDBMS"></a>
#### Trees and RDBMS

1. [Joe Celko's Trees and Hierarchies in SQL for Smarties, 2nd edition][Celko's Trees]
2. [SQL Design Patterns: The Expert Guide to SQL Programming][Tropashko]
3. [Nested Sets and Materialized Path SQL Trees][NS-MP]
4. [Storing trees in RDBMS][Kolesnikova]
5. [Django-Treebeard tree library for Django Web Framework][django-treebeard]
6. [PostgreSQL tree module ltree][PostgreSQL ltree]


<a name="ClassRefs"></a>
#### Information organization

 1. [Hierarchy][]
 2. [Taxonomy][]
 3. [Classification][]
 4. [Categorization][]
 5. [Tree structure][]
 6. [Meronomy][]
 7. [Patent classification][]
 8. [Decimal classification][]
 9. [Data classification][]
 10. [Knowledge organization][]
 11. [Personal information management][]


<!-- References -->

[Patent classification]: https://en.wikipedia.org/wiki/Patent_classification
[Classification]: https://en.wikipedia.org/wiki/Classification
[Decimal classification]: https://en.wikipedia.org/wiki/Decimal_classification
[Taxonomy]: https://en.wikipedia.org/wiki/Taxonomy
[Hierarchy]: https://en.wikipedia.org/wiki/Hierarchy
[Tree structure]: https://en.wikipedia.org/wiki/Tree_structure
[Knowledge organization]: https://en.wikipedia.org/wiki/Knowledge_organization
[Personal information management]: https://en.wikipedia.org/wiki/Personal_information_management
[Data classification]: https://en.wikipedia.org/wiki/Data_classification
[Categorization]: https://en.wikipedia.org/wiki/Categorization
[Meronomy]: https://en.wikipedia.org/wiki/Meronomy

[Celko's Trees]: https://sciencedirect.com/book/9780123877338
[Tropashko]: https://vadimtropashko.wordpress.com/%22sql-design-patterns%22-book/about
[NS-MP]: http://rampant-books.com/art_vadim_nested_sets_sql_trees.htm
[django-treebeard]: https://django-treebeard.readthedocs.io
[PostgreSQL ltree]: https://www.postgresql.org/docs/current/ltree.html
[Kolesnikova]: https://bitworks.software/en/2017-10-20-storing-trees-in-rdbms.html
