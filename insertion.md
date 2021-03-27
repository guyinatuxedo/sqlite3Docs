# Insertion

So this document will discuss the process for inserting a cell. So starting off, there is the `OP_Insert` instruction. The instruction will call the `sqlite3BtreeInsert` function. 

To actually construct the cell, we see that the `fillInCell` function.


## Cell Insertion

To actually insert a new cell, the `insertCell` function is called. It is passed the memory page which it will insert into, the cell content, the size if the cell, and a few other things. There are two cases for insertion.

```
0.)	Inserted into Overflow Page (Overflow page is active or size of cell is greater than free bytes)
1.) Standard Case (0 is not ran)
```

For case 1, it will call the `allocateSpace` function to actually allocate the space for the cell. This will write the `idx` value to an integer that is passed by reference. Assuming that space allocation was successful, the cell content is copied to the space allocated (which is signified by the `idx`).

For case 0, the data is written to `temp`. Actual insertion of the cell appears to be dealt with by `sqlite3BtreeInsert`.