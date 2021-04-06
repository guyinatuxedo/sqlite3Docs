# vdbe

So the operations of sql database are handled by a virtual machine. This document will describe that functionallity. The function that is actually responsible for executing the vdbe code is `sqlite3VdbeExec`.

## OPcode Enums

So there are various type of opcode iterations, such as `OP_INSERT`/`OP_Column`/`OP_INTEGER`. These are defined in `sqlite3.c`:

```
#define OP_ColumnsUsed   118
#define OP_SeekScan      119 /* synopsis: Scan-ahead up to P1 rows         */
#define OP_SeekHit       120 /* synopsis: set P2<=seekHit<=P3              */
#define OP_Sequence      121 /* synopsis: r[P2]=cursor[P1].ctr++           */
#define OP_NewRowid      122 /* synopsis: r[P2]=rowid                      */
#define OP_Insert        123 /* synopsis: intkey=r[P3] data=r[P2]          */
#define OP_RowCell       124
#define OP_Delete        125
```

## OP

So the vdbe code consists of ops, which can be viewed as instructions. An opcode is a structure that is referred to as either `Op` or `VdbeOp`. The main componets of the op inlcude the opcode, and the parameters. The parameters form the arguments to the opcode.