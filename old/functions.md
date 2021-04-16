# Functions

So this document is regarding functions, which are user defined bits of code that are executed. This document will explore them.

So there are two VDBE opcodes that we have that will execute this functionallity, which are `OP_PureFunc` and `OP_Function`. The `sqlite3VdbeAddFunctionCall` is the only function call where either of these opcodes is generated/added to the vdbe opcodes. This function appears to be potentially added with the following tokens:

```
the token TK_FUNCTION from sqlite3ExprCodeTarget
callStatGet function
analyzeOneTable (2 times)
codeAttach function
```