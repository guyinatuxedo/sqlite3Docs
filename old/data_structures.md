## Data Structures

So the db uses various different datastructures to model the database in memory. This section will briefly cover some of those.

#### Database

It is scanned into memroy, with a call to `pread64` within `unixRead`. Interestingly enough, the memory that exists in memory is the same data that exists in the file. This pertains to the data of the records. 


#### Btree

So sqlite3 uses B-Tree datastructures to model tables in memory. If you need a refresher on what a B-Tree is, try either google or https://www.youtube.com/watch?v=C_q5ccN84C8 

A handle to a Btree is called a `Btree` struct. Now the actual memory paging ptrs/info is stored in a `BtShared` struct, which a ptr to it is stored in the `Btree` struct.

So what this structure is, is a handle to a Btree. The Btree is used to model a table, of a database. 

This is defined in `btreeInt.h`, and is allocated with the `sqlite3BtreeCreateTable` function (which wraps the `btreeCreateTable` function).

#### MemPage

This structure is to handle a page of memory, for the database. That means that this datastructure is what specifies where the actual data of the database is stored. This means the records of that tables, for the database.


It is allocated with the `allocateBtreePage` function in `sqlite3.c` file. The structure itself is defined in `btreeInt.h`.
