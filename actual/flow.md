# Code Flow

So this documents describes the control flow execution of sqlite3 for executing a sql query:

```
Scan in Sql query
String Processing / vdbe code generation
vdbe code execution / callback function
```

## Query Scan In

So at first the query is scanned into memory. This happens with `fgets` in the `one_input_line` function called from `process_input` (which is called from main). There is basically an infinite loop which happens within `process_input` that will continually scan in and process commands.

Now there are two seperate types of commands. There are meta commands, which specify sql settings. These are specified via having the first character be a `.`. These commands are handles with the `do_meta_command` function. All queries that do not begin with a `.` are assumed to be a sql command, and handles with the `runOneSqlLine` function.

## String Processing

So this next part is when 

## vdbe ops execution






