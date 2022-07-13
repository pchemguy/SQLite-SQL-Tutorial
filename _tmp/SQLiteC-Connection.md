---
layout: default
title: Connection
nav_order: 3
parent: SQLiteC
permalink: /sqlitec/connection
---

Functionally, the SQLiteCConnection class roughly follows the ADODB.Connection class. It is usually instantiated via the SQLiteC.CreateConnection method and is responsible for several function groups. The primary responsibility of SQLiteCConnection is to open a database connection and keep a connection handle. SQLiteCConnection is involved in a circular reference. This project presently uses the [CleanUp cascade][ObjectStore] approach to resolve the loops, so the clean-up code is placed in the CleanUp() sub and not Class_Terminate().

#### SQLite API wrappers

* Database connection  
  *OpenDb()* and *CloseDb()* methods responsible for connection handling.  
* Error information  
  *ErrInfo()*, *GetErrc()*, *PrintErr()*, and *ErrInfoRetrieve()* methods retrieve and provide access to error information.  
* Connection information  
  *AttachedDbPathName()* and *AccessMode()* provide file pathname and access mode for a specified attached database;  
  *ChangesCount()* returns the number of changes for the last modifying transaction or the total count performed via a particular connection.  
* Basic query  
  *ExecuteNonQueryPlain()* wraps the SQLite *exec* API and executes SQL queries not returning data. The SQLite engine provides diverse APIs necessary to run a query and retrieve the result, including a simplified "shortcut" *exec*. This interface does not support parameterized statements and returns data via a callback function. At the same time, it handles multi-statement queries automatically, whereas the primary query interface stops processing the query command at the first semicolon. For simplicity, this method does not use a callback function and cannot return any data, so it is only suitable for controlling and modifying statements.

#### Statement functionality

SQLiteCConnection acts as an abstract factory for the SQLiteCStatement class, providing three related members, including the *CreateStatement()* factory, the *StmtDb()* accessor, and the *Finalize()* destructor. Additionally, *ExecADO()* is a shortcut returning an ILiteADO/SQLiteCStatement instance.

#### Transaction control

This group of methods includes one SQLite API wrapper, *TxnState()*, which queries the current transaction state, and a set of *ExecuteNonQueryPlain()* wrappers controlling the transaction state, including *Begin()*, *Commit()*, *SavePoint()*, *ReleasePoint()*, *Rollback()*. Additionally, *DbIsLocked()* attempts to determine whether the database is locked by trying to start an immediate transaction.

#### High-level database manipulation

*Attach()*, *Detach()*, *Vacuum()*, and *JournalModeSet()* provide corresponding functionality.


<!-- References -->

[ObjectStore]: https://pchemguy.github.io/ObjectStore/
