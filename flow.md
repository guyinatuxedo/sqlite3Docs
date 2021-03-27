# Code Flow

So this is just meant to briefly document the flow of code through the binary.

```
// Main Function
0.) Main Function:
	
	// Contains loop which runs the shell
	1.) process_input (while loop)

		// meta command execution path
		8.) do_meta_command

		// sql code execution path
		2.) runOneSqlLine
			3.) shell_exec
				?????????????????
				?????????????????
				?????????????????

				4.) sqlite3_prepare_v2

				5.) exec_prepared_stmt
					6.) sqlite3_step
						7.) sqlite3Step
							8.) sqlite3VdbeExec
```

```
0.) Main Function
Is the main function for the program
```

## Main Function

So starting off there is the main funciton

```
gef➤  bt
#0  0x0000555555614de1 in ?? ()
#1  0x000055555561eeb2 in ?? ()
#2  0x0000555555577c41 in ?? ()

#3  0x0000555555579296 in ?? ()

#4  0x00005555555841e4 in ?? ()
#5  0x0000555555563c27 in ?? ()
#6  0x00007ffff7c070b3 in __libc_start_main (main=0x555555563000, argc=0x1, argv=0x7fffffffe0c8, init=<optimized out>, fini=<optimized out>, rtld_fini=<optimized out>, stack_end=0x7fffffffe0b8) at ../csu/libc-start.c:308
#7  0x000055555556487e in ?? ()

```

Select

```
0x5a
0x51
0x05
0x44
```
Insert

```
0x3e:	OP_Init
0x02:	OP_Transaction
0x0b:	OP_Goto
0x61:	OP_OpenRead
0x73:	OP_OpenPseudo
0x78:	OP_SeekHit
0x5b:	OP_Affinity
0x79:	OP_Sequence
0x44:	OP_Halt
```

Delete
```
0x3e:	OP_Init
0x02:	OP_Transaction
0x0b:	OP_Goto
0x89:	OP_IdxRowid
0x44:	OP_Halt
```











