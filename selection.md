# Selection

So this document will describe the functionallity surrounding actually reading data from a database. Data is read one column at a time, for each row. This is done with the `OP_Column` opcode, and most of the actual functionallity for extracting the value occurs in this opcode (not handled by a called function).

Now when it extracts column values, it will calculate the offset to that column. It will do so using the record header. It will parse through the bytes of the header, summating the offset until it reaches the desired column.

There are several checks that happen with this. First initially it will check the total size of the header, that it is not greater than the size of the row, greater than the payload size, and greater than `0x18003` (checks happen in a sequence).

There are also checks to see that when it was parsing the header, it did not overextend past the header size. It also checks that if the entire header was parsed, but the data offset is not at the end. Lastly it also checks that the data does not extend past the end of the record.

Also another thing to note. If the data for a record extends off the page, it will assume that the data is in an overflow page, and execute additional functionallity to get it.

Then for actually extracting the value. If it is an integer, it will extract it using `sqlite3VdbeSerialGet`. If it is a string, it will extract it using a `memcpy` call.
