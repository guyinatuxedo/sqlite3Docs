# Sqlite

## Installation

```
$	sudo apt-get install sqlite3
```

## Location

The binary is stored at the following location:

```
/usr/bin/sqlite3
```

## Reversing

So reversing the binary, the `"main"` function appears to start at the offset `0xf000`.

There is a check that happens at `0xf468`, that checks if there were additional arguments, and if there are, it processes them.

Input is scanned into a function, which is stored at `0x63920`. This acts as a wrapper to `readline`, which I have seen the call specifically happen at the offset `0x2fd65'`. Now we are going to follow the path of this input.

We see that there is a check to ensure the input was scanned in, and if it was, there is a call to `add_history` at `0x2fd79`.

`;` end @ `0x30051`

#### Tokenization

the function which handles tokenization, and subsequent processing, is at offset `0xdba30`. The function which actually handles tokenizing the input string is at `0x45ed0`. The first argument is the sql query itself, the second appears to be a struct pointer:

```
ulong tokenize_string(byte *input_sql,undefined4 *param_2)

{
  byte *pbVar1;
  byte bVar2;
  int iVar3;
  ulong uVar4;
  uint uVar5;
  undefined4 uVar6;
  ulong uVar7;
  ulong uVar8;
  uint uVar9;
  byte *pbVar10;
  int iVar11;
  long lVar12;
  byte first_char;
  
  first_char = *input_sql;
  switch((&char_values)[first_char]) {
  case 0:
```

Now what happens is a bit interesting. It grabs the first character of the input sql query, and uses it as an array index to `char_values`. `char_values` is a buffer in the bss at offset `0x1304e0`. This is the values it holds:

```
gef➤  x/32g $rdx
0x5555556844e0:	0x1b1b1b1b1b1b1b1c	0x1b1b07071b07071b
0x5555556844f0:	0x1b1b1b1b1b1b1b1b	0x1b1b1b1b1b1b1b1b
0x555555684500:	0x818160405080f07	0x101a0b1714151211
0x555555684510:	0x303030303030303	0x60d0e0c13050303
0x555555684520:	0x101010101010105	0x101010101010101
0x555555684530:	0x101010101010101	0x11b1b1b09010100
0x555555684540:	0x101010101010108	0x101010101010101
0x555555684550:	0x101010101010101	0x1b191b0a1b010100
0x555555684560:	0x202020202020202	0x202020202020202
0x555555684570:	0x202020202020202	0x202020202020202
0x555555684580:	0x202020202020202	0x202020202020202
0x555555684590:	0x202020202020202	0x202020202020202
0x5555556845a0:	0x202020202020202	0x202020202020202
0x5555556845b0:	0x202020202020202	0x202020202020202
0x5555556845c0:	0x202020202020202	0x202020202020202
0x5555556845d0:	0x202020202020202	0x202020202020202
```

All ascii values will fall within the range for `0x01`. So all sql statements will end up here:

```
  case 1:
    i = 2;
    if ((byte)(&char_values)[input_sql[1]] < 2) {
      do {
        c = input_sql + i;
        i_cpy = i & 0xffffffff;
        i = i + 1;
      } while ((byte)(&char_values)[*c] < 2);
      if (((&DAT_00233be0)[*c] & 0x46) == 0) {
        *param_2 = 0x3b;
        parse_sql_statement(input_sql,i_cpy,param_2);
        return i_cpy;
      }
      uVar4 = (int)i_cpy + 1;
      i = (ulong)uVar4;
      lVar9 = (long)(int)uVar4;
    }
    else {
      if (((&DAT_00233be0)[input_sql[1]] & 0x46) == 0) {
        *param_2 = 0x3b;
        return 1;
      }
      lVar9 = 2;
      i = 2;
    }
    goto LAB_00145ff3;
```

```
    pbVar8 = *ppbVar9;
    if (*pbVar8 == 0x2e) {
      iVar4 = parse_cmd(pbVar8,&local_15b8);
      if (iVar4 != 0) goto joined_r0x0011072f;
    }
    else {
      if (local_15b8 == 0) {
        FUN_00124dc0(&local_15b8,0);
      }
      iVar4 = FUN_00123a50(&local_15b8,pbVar8,&local_15c0);
      if (local_15c0 != 0) {
        __fprintf_chk(stderr,1,"Error: %s\n");
joined_r0x001106e7:
        if (iVar4 == 0) {
LAB_0010fd2a:
          iVar4 = 1;
        }
        goto LAB_0010fa3f;
      }
      if (iVar4 != 0) {
        __fprintf_chk(stderr,1,"Error: unable to process SQL: %s\n",pbVar8);
        goto LAB_0010fa3f;
      }
    }
```

`0x12ff4c`

`0x13d970`

`0x13da0f`

## Special parsing

Characters:
```
Group 0:
"\t", "\n", "\r", "\x0e", " "

Group 1:

Group 2:
'"', "'", "`"

Group 3:
"-"

Group 4:
"/"

Group 5:
";"

Group 6:
"["
```