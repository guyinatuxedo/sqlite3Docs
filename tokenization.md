# Tokenization

So this is the process of how sqlite will interpret string sql queries.

## Parsing

So the function which actually handles parsing will be the `sqlite3RunParser` (at offset `0xdba30`) function located in the `tokenize.c` file. This function will use the `sqlite3GetToken` function to tokenize parts of the inputed sql. A few things, the max sql len is defined by `mxSqlLen`.

One thing to note. There is a `Parse` struct that is passed around to the various parsing functionallity. It can be found in `sqlite3.c`.

## Tokenization

A lot of the tokenization will take place in `sqlite3GetToken` in the `tokenize.c` file. In the binary, this is at offset `0x45ed0`. This function operates off of a switch statement. We can see that the value being evaluated for the switch statement is the first character of the sql query:

```
int sqlite3GetToken(const unsigned char *z, int *tokenType){
  int i, c;
  switch( aiClass[*z] ){  /* Switch on the character-class of the first byte
                          ** of the token. See the comment on the CC_ defines
```

Now for the exact cases. We can see that the cases are defined as enums, such as `CC_DOLLAR` or `CC_KYWD0`. Looking here, we see the case for keywords such as `SELECT`. It calculates the length of the keyword, and then uses the `keywordCode` (at binary offset `0x45be0`) function in order to identify which keyword it is.

```
    case CC_KYWD0: {
      for(i=1; aiClass[z[i]]<=CC_KYWD; i++){}
      if( IdChar(z[i]) ){
        /* This token started out using characters that can appear in keywords,
        ** but z[i] is a character not allowed within keywords, so this must
        ** be an identifier instead */
        i++;
        break;
      }
      *tokenType = TK_ID;
      return keywordCode((char*)z, i, tokenType);
    }
```

Now how the tokenization will work, is it will parse a sql statement in parts. Take for instance this sql statement:

```
SELECT * FROM x;

Will be parsed in the order:
SELECT
*
FROM
xbt
```
