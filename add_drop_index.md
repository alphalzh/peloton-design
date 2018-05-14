# Add/Drop Index (Concurrently)

## Overview
Originally, add/drop index in peloton doesn't work properly. (will refer to add index as create index in the following context)
For drop index, the implementation is basically correct except for some bugs.
For create index, the implementaation doesn't use locks or consider concurrent modifications to the index.
Our project implemented lock-based/lock-free version of create index. In the future, we also can support create index in parallel.
## Scope
+ parser: add support for parsing the clause 'CREATE INDEX CONCURRENTLY'.
+ optimizer: changed rules for create index, when concurrently is specified.
+ planner: add support for create index concurrently.
+ executor: modified seq_scan_executor for supporting scanning mode change when populating index; modified populate_index_executor for bug fixes and supporting concurrent add index; modified create/drop/insert/delete/update executors for supporting locked version of create index.
+ codegen: modified inserter/deleter/updater for supporting locked version of create index.
+ txn_context: add table that contains lock info and related unlocking logic, for supporting unlocking transaction level locks.
+ txn_manager: add table that contains all concurrent transactions, for supporting create index concurrently.
+ index: add table that contains all insertions in the index. Will be cleaned once populate index is finished.
## Glossary 
lock_manager(new)
+ Centralized: get a static copy of it just like you get the instance of catalog
+ Per-table lock: lock the tables based on their oid
+ Two modes: you can use it as a scope lock, which will unlocks when the scope ends. You can also lock the table until current transaction ends.
+ Currently supports only read_write lock, but can be extended.

## Architectural Design
+ input: a sql with 'CREATE INDEX' or 'CREATE INDEX CONCURRENTLY'
+ output: table with correctly created and populated index
+ For create index, it will first acquire exclusive lock for the table; then it will call create executor, which will create index in catalog, create the index object itself and add it to the table; It will then call sequential scan executor and scan the whole table in 'read committed' isolation level; the scan results will be added into the index. After all that, the lock is released.
+ For create index concurrently, it's basically the same as create index, but without acquiring locks and a different logic.
## Design Rationale
+ Support create index with locks: the lock manager was designed for this. Shared lock are acquired for the duration of transaction, so we have transaction-wide locks; exclusive locks are acquired for the duration of create index operation only, so we have per-scope lock. We only need to lock the tables as far as create index is concerned, so we implemented per table lock. 
+ Support create index concurrently: need to address 'write to table before create index in another txn', so we used concurrent transaction set in txn_manager to let the create index txn wait until all previous txns commit; need to address 'another txn write while seq_scan could result in double write in index', so we added per-index insertion set and performed checking before inserting entries in index.
## Testing Plan
+ unit test for populate_index_executor (already existed)
+ unit test for lock_manager (added)
+ enhanced test cases for concurrent transactions with create index statement (added)
+ junit test that launch parallel transactions, testing concurrent operations with or without enforced ordering of events.
+ (future) performance test
## Trade-offs and Potential Problems
+ Trade offs: every insert/delete/update will acquire a shared lock. This should just trivially impact performance, since when there is no contention for exclusive locks, a shared lock is cheap.
+ Potential Problems: The current lock design doesn't enforce acquire order of locks in transactions, which may cause dead locks. However, our current implementation will not have this problem. Deadlock will only occurs when the lock manager is not used properly.
## Future Work
+ Populate index is currently an executor and marked as deprecated. Should add it as one of the codegen modules
+ Support create index in parallel, after parallel sequential scan is sorted out
+ More on indexing: add indexes to tables, just like Cicada
+ Insert/Delete/Update actions in transaction -> goes through codegen modules, like codegen::Inserter, codegen::Deleter, etc; Insert/Delete/Update action not in transaction -> goes through executors (marked as deprecated)
+ Weird behavior that Peloton kept switching between executor and their corresponding codegen version. When it uses executor, bugs caused segmentation fault when accessing bw_tree index in certain ways. This bug originated from cmu-db/peloton, and as we were told that those executors are going to be deprecated, we didn’t take effort in fixing them.


