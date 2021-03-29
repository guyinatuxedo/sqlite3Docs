# Index

So an index is a seperate datastructure that stored metadata about a table. It is used to store information which can speed up queries:

```
sqlite> .open x
sqlite> CREATE TABLE x (y int, z varchar);
sqlite> CREATE INDEX index_x ON x(y);

.	.	.

sqlite> SELECT * FROM x;
5|55555
6|66666
7|77777
```

Now to see how some of the functionallity will change. We will run this query:

```
sqlite> SELECT * FROM x WHERE y > 5;
```

Which will translate to these opcodes:

```
0x3e: OP_Init
0x02: OP_Transaction
0x0b: OP_Goto
0x61: OP_OpenRead
0x61: OP_OpenRead
0x45: OP_Integer
0x19: OP_SeekGT
0x88: OP_DeferredSeek
0x5a: OP_Column
0x5a: OP_Column
0x51: OP_ResultRow
0x05: OP_Next
0x88: OP_DeferredSeek
0x5a: OP_Column
0x5a: OP_Column
0x51: OP_ResultRow
0x05: OP_Next
0x44: OP_Halt
```

So when we look at the opcodes, we see there are two `OP_OpenRead` opcodes. This is because it has to open up both the table, and the index associated with the table. Proceeding that it uses the `OP_Integer` opcode in order to store the integer we are comparing against in the `P2` regitser. Proceeding that, the `OP_SeekGT` opcode is used. This is used to position the cursor on the index, to the least value that is greater than the specified value, and configure the cursor to move in ascending order. That way in order to get all of the values, it will just have to move that cursor forward by one every time. Proceeding that, it will use the `OP_DeferredSeek` opcode to get the corresponding record in the actual table for the index row. Then it will follow the normal `OP_Column/OP_ResultRow` extraction method, before using `OP_Next` to move to the next record in the index.


```
[ Legend: Modified register | Code | Heap | Stack | String ]
────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x3e              
$rbx   : 0x00005555556a0a20  →  0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
$rcx   : 0x0000555555660d6c  →  0xfffacf8dfffacc53
$rdx   : 0x3e              
$rsp   : 0x00007fffffffb5a0  →  0x0000000000000001
$rbp   : 0x3e              
$rsi   : 0x00005555556a0570  →  0x000000000000003e (">"?)
$rdi   : 0x00005555556a0a20  →  0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
$rip   : 0x00005555556096f0  →  <sqlite3VdbeExec+304> movsxd rax, DWORD PTR [rcx+rax*4]
$r8    : 0x0000555555665300  →  0x00de00bf00dc00bf
$r9    : 0x0000555555664540  →  0x003b003b00000000
$r10   : 0x00005555555e8100  →  <sqlite3BtreePrevious+0> endbr64 
$r11   : 0x00005555556a0a20  →  0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
$r12   : 0x0               
$r13   : 0x00005555556a0570  →  0x000000000000003e (">"?)
$r14   : 0x00005555556a0570  →  0x000000000000003e (">"?)
$r15   : 0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
$eflags: [zero CARRY parity adjust SIGN trap INTERRUPT direction OVERFLOW resume virtualx86 identification]
$cs: 0x0033 $ss: 0x002b $ds: 0x0000 $es: 0x0000 $fs: 0x0000 $gs: 0x0000 
────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x00007fffffffb5a0│+0x0000: 0x0000000000000001	 ← $rsp
0x00007fffffffb5a8│+0x0008: 0x3967ad1600000000
0x00007fffffffb5b0│+0x0010: 0x00005555556a0a20  →  0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
0x00007fffffffb5b8│+0x0018: 0x00005555556a0898  →  0x0000000000000000
0x00007fffffffb5c0│+0x0020: 0x00005555556a0570  →  0x000000000000003e (">"?)
0x00007fffffffb5c8│+0x0028: 0xffffffffffffffff
0x00007fffffffb5d0│+0x0030: 0x00005555556a2750  →  "SELECT*FROM"main".sqlite_master ORDER BY rowid"
0x00007fffffffb5d8│+0x0038: 0x00007fffffffb8d0  →  0x00005555556a0a20  →  0x0000555555693fb0  →  0x000055555568e300  →  0x0000007800000003
──────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
   0x5555556096df <sqlite3VdbeExec+287> ja     0x5555556099a0 <sqlite3VdbeExec+992>
   0x5555556096e5 <sqlite3VdbeExec+293> lea    rcx, [rip+0x57680]        # 0x555555660d6c
   0x5555556096ec <sqlite3VdbeExec+300> movzx  eax, bpl
 → 0x5555556096f0 <sqlite3VdbeExec+304> movsxd rax, DWORD PTR [rcx+rax*4]
   0x5555556096f4 <sqlite3VdbeExec+308> add    rax, rcx
   0x5555556096f7 <sqlite3VdbeExec+311> notrack jmp rax
   0x5555556096fa <sqlite3VdbeExec+314> nop    WORD PTR [rax+rax*1+0x0]
   0x555555609700 <sqlite3VdbeExec+320> mov    DWORD PTR [rsp+0x10], 0x0
   0x555555609708 <sqlite3VdbeExec+328> mov    r13, QWORD PTR [rsp+0x20]
───────────────────────────────────────────────────────────────────────── source:sqlite3.c+86774 ────
   86769	 #endif
   86770	 #if defined(SQLITE_DEBUG) || defined(VDBE_PROFILE)
   86771	     pOrigOp = pOp;
   86772	 #endif
   86773	 
 → 86774	     switch( pOp->opcode ){
   86775	 
   86776	 /*****************************************************************************
   86777	 ** What follows is a massive switch statement where each case implements a
   86778	 ** separate instruction in the virtual machine.  If we follow the usual
   86779	 ** indentation conventions, each case should be indented by 6 spaces.  But
──────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "sqlite3", stopped 0x5555556096f0 in sqlite3VdbeExec (), reason: SINGLE STEP
────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x5555556096f0 → sqlite3VdbeExec(p=0x5555556a0a20)
[#1] 0x555555613f20 → sqlite3Step(p=0x5555556a0a20)
[#2] 0x555555613f20 → sqlite3_step(pStmt=0x5555556a0a20)
[#3] 0x55555561479d → sqlite3_exec(db=0x555555693fb0, zSql=<optimized out>, xCallback=0x55555562ceb0 <sqlite3InitCallback>, pArg=0x7fffffffb950, pzErrMsg=0x0)
[#4] 0x555555615063 → sqlite3InitOne(db=0x555555693fb0, iDb=0x0, pzErrMsg=0x7fffffffc868, mFlags=0x0)
[#5] 0x5555556152cc → sqlite3Init(db=0x555555693fb0, pzErrMsg=0x7fffffffc868)
[#6] 0x55555561530f → sqlite3ReadSchema(pParse=0x7fffffffc860)
[#7] 0x555555615998 → sqlite3LocateTable(pParse=0x7fffffffc860, flags=0x0, zName=0x5555556a2b50 "x", zDbase=0x0)
[#8] 0x55555561716e → sqlite3SrcListLookup(pParse=0x7fffffffc860, pSrc=0x5555556a2bd0)
[#9] 0x55555561cd09 → sqlite3Insert(pParse=0x7fffffffc860, pTabList=0x5555556a2bd0, pSelect=0x0, pColumn=0x5555556a2ad0, onError=0xb, pUpsert=0x0)
─────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  p $rcx
$1 = 0x555555660d6c
gef➤  p $rax

