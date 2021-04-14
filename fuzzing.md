# Fuzzing

## AllocateSpace fuzzing

So the AllocateSpace function has interesting functionallity, which if we can abuse, we can potentially get some sort of write primitive. Now the allocateSpace function already has a lot of checks it can use in order to prevent itself from writing outside of the boundaries of the page. We will try to set a breakpoint along the codepath that is executed, if one of those checks fail.

Looking at the binary for this function, we can sse that the `allocateSpace` function has been inlined with `insertCell`, so all of the binary code for that function is in `insertCell`. However we can see what appears to be a function call that is made when a database corruption check is failed, at `0x8fbcd`:

```
    uVar8 = 0x1043e;
LAB_0018fbb8:
    sqlite3_log(0xb,"%s at line %d of [%.10s]","database corruption",uVar8,&DAT_002000ec);
    iVar
```

## parser stack overflow

So one type of error the fuzzer found was `parser stack overflow`. This error is printed in the `yyStackOverflow` function. It appears to happen when we input a large query. To get an idea of what exactly this means, let's take a look via reversing out what this functionallity is.

So, when we look at where `yyStackOverflow` is called from, we see that it is from multiple lcoations. This is one of them, from the `sqlite3RunParser` function.

```
#endif
#if YYSTACKDEPTH>0
        if( yypParser->yytos>=yypParser->yystackEnd ){
          yyStackOverflow(yypParser);
          break;
        }
```

So looking at the check, we see the values being compared are `yypParser->yytos` and `yypParser->yystackEnd`. These are both values from the struct `yypParser`, which is a `yyParser` struct:

```
/* The following structure represents a single element of the
** parser's stack.  Information stored includes:
**
**   +  The state number for the parser at this level of the stack.
**
**   +  The value of the token stored at this level of the stack.
**      (In other words, the "major" token.)
**
**   +  The semantic value stored at this level of the stack.  This is
**      the information used by the action routines in the grammar.
**      It is sometimes called the "minor" token.
**
** After the "shift" half of a SHIFTREDUCE action, the stateno field
** actually contains the reduce action for the second half of the
** SHIFTREDUCE.
*/
struct yyStackEntry {
  YYACTIONTYPE stateno;  /* The state-number, or reduce action in SHIFTREDUCE */
  YYCODETYPE major;      /* The major token value.  This is the code
                         ** number for the token at this stack level */
  YYMINORTYPE minor;     /* The user-supplied minor token value.  This
                         ** is the value of the token  */
};
typedef struct yyStackEntry yyStackEntry;

/* The state of the parser is completely contained in an instance of
** the following structure */
struct yyParser {
  yyStackEntry *yytos;          /* Pointer to top element of the stack */
#ifdef YYTRACKMAXSTACKDEPTH
  int yyhwm;                    /* High-water mark of the stack */
#endif
#ifndef YYNOERRORRECOVERY
  int yyerrcnt;                 /* Shifts left before out of the error */
#endif
  sqlite3ParserARG_SDECL                /* A place to hold %extra_argument */
  sqlite3ParserCTX_SDECL                /* A place to hold %extra_context */
#if YYSTACKDEPTH<=0
  int yystksz;                  /* Current side of the stack */
  yyStackEntry *yystack;        /* The parser's stack */
  yyStackEntry yystk0;          /* First stack entry */
#else
  yyStackEntry yystack[YYSTACKDEPTH];  /* The parser's stack */
  yyStackEntry *yystackEnd;            /* Last entry in the stack */
#endif
};
typedef struct yyParser yyParser;
```

So looking at this struct, it appears to be a stack. The individual entries on the stack are 
`yyStackEntry` structs. Now looking back at the check, we see that both of the values being compared `yytos` and `yystackEnd` are both ptrs. It effectively is checking if the address of the top element of the stack, is past the end of the stack (not sure exactly what they mean by that atm, but I would guess it's the highest spot the want the stack going).

So let's take a look at how a parser data structer is initialized. This happens in the `sqlite3ParserInit` function. We see that effectively the `yystackEnd` value is set equal to `yystack + YYSTACKDEPTH` where `YYSTACKDEPTH` is a constant (set to `100`).

Now, let's take a look at how things are added to the stack. This is done with the `yyGrowStack` function, which allocates new space with either `malloc/realloc`. So looking at the check, I'm pretty sure what is happening is this. As the stack grows, it will incrementally allocate new chunks with `malloc/realloc`. The `yystackEnd` is realistically a "soft" limit. I say this because, the space is always allocated with `malloc/realloc`, as such it's not like a fixed size buffer that we can simply overflow. What it seems like is there is a size limit on how big they want this stack to grow. So effectively, it is just a size check.

```
no such table: rftdyzty
table qirlrcniqi already exists
near "WHEN": syntax error
no such column: tifrvgydif.lwbqcviqba
1 values for 4 columns
UNIQUE constraint failed: pyuiuryoxg.hxhesnlsoz, pyuiuryoxg.ycloqkiizx
parser stack overflow
SELECTs to the left and right of UNION do not have the same number of result columns
NOT NULL constraint failed: nzrjrlwhaa.llpddzimth
RIGHT and FULL OUTER JOINs are not currently supported
near ";": syntax error
near "THEN": syntax error
1 values for 3 columns
```









