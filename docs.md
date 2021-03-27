## Main Function

So first off, we start with the main function. In the source code, this is located in the `shell.c` file (should be `shell.c.in` if the `shell.c` file has not been generated). In the binary `sqlite`, I found it at the offset `0xf000`.

So first off, there is a lot of parsing of the argc commands. 

After a while, we get to the part where our input from `stdin` is ran. Looking at this, it appears that our input is passed to the `process_input` function to be handled.

In the source code:

```
    if( stdin_is_interactive ){
      char *zHome;
      char *zHistory;
      int nHistory;
      printf(
        "SQLite version %s %.19s\n" /*extra-version-info*/
        "Enter \".help\" for usage hints.\n",
        sqlite3_libversion(), sqlite3_sourceid()
      );
      if( warnInmemoryDb ){
        printf("Connected to a ");
        printBold("transient in-memory database");
        printf(".\nUse \".open FILENAME\" to reopen on a "
               "persistent database.\n");
      }
      zHistory = getenv("SQLITE_HISTORY");
      if( zHistory ){
        zHistory = strdup(zHistory);
      }else if( (zHome = find_home_dir(0))!=0 ){
        nHistory = strlen30(zHome) + 20;
        if( (zHistory = malloc(nHistory))!=0 ){
          sqlite3_snprintf(nHistory, zHistory,"%s/.sqlite_history", zHome);
        }
      }
      if( zHistory ){ shell_read_history(zHistory); }
#if HAVE_READLINE || HAVE_EDITLINE
      rl_attempted_completion_function = readline_completion;
#elif HAVE_LINENOISE
      linenoiseSetCompletionCallback(linenoise_completion);
#endif
      data.in = 0;
      rc = process_input(&data);
      if( zHistory ){
        shell_stifle_history(2000);
        shell_write_history(zHistory);
        free(zHistory);
      }
    }
```

In the binary (around offset `0xfc22`):

```
    if( stdin_is_interactive ){
      char *zHome;
      char *zHistory;
      int nHistory;
      printf(
        "SQLite version %s %.19s\n" /*extra-version-info*/
        "Enter \".help\" for usage hints.\n",
        sqlite3_libversion(), sqlite3_sourceid()
      );
      if( warnInmemoryDb ){
        printf("Connected to a ");
        printBold("transient in-memory database");
        printf(".\nUse \".open FILENAME\" to reopen on a "
               "persistent database.\n");
      }
      zHistory = getenv("SQLITE_HISTORY");
      if( zHistory ){
        zHistory = strdup(zHistory);
      }else if( (zHome = find_home_dir(0))!=0 ){
        nHistory = strlen30(zHome) + 20;
        if( (zHistory = malloc(nHistory))!=0 ){
          sqlite3_snprintf(nHistory, zHistory,"%s/.sqlite_history", zHome);
        }
      }
      if( zHistory ){ shell_read_history(zHistory); }
#if HAVE_READLINE || HAVE_EDITLINE
      rl_attempted_completion_function = readline_completion;
#elif HAVE_LINENOISE
      linenoiseSetCompletionCallback(linenoise_completion);
#endif
      data.in = 0;
      rc = process_input(&data);
      if( zHistory ){
        shell_stifle_history(2000);
        shell_write_history(zHistory);
        free(zHistory);
      }
    }
```

## Process Input

Below is the source code for that function. We notice a few things. First off, this is a general wrapper function to handle different types of input. The first type of input we see is `meta cmds`, which are identified by either a `.` or a `#` as the first character of the command. These are handles by the `do_meta_command` function. The second type of input appears to be actual sql commands, which are ran by the `runOneSqlLine` function. It starts at offset `0x2fc30`.

```
static int process_input(ShellState *p){
  char *zLine = 0;          /* A single input line */
  char *zSql = 0;           /* Accumulated SQL text */
  int nLine;                /* Length of current line */
  int nSql = 0;             /* Bytes of zSql[] used */
  int nAlloc = 0;           /* Allocated zSql[] space */
  int nSqlPrior = 0;        /* Bytes of zSql[] used by prior line */
  int rc;                   /* Error code */
  int errCnt = 0;           /* Number of errors seen */
  int startline = 0;        /* Line number for start of current input */

  p->lineno = 0;
  while( errCnt==0 || !bail_on_error || (p->in==0 && stdin_is_interactive) ){
    fflush(p->out);
    zLine = one_input_line(p->in, zLine, nSql>0);
    if( zLine==0 ){
      /* End of input */
      if( p->in==0 && stdin_is_interactive ) printf("\n");
      break;
    }
    if( seenInterrupt ){
      if( p->in!=0 ) break;
      seenInterrupt = 0;
    }
    p->lineno++;
    if( nSql==0 && _all_whitespace(zLine) ){
      if( ShellHasFlag(p, SHFLG_Echo) ) printf("%s\n", zLine);
      continue;
    }
    if( zLine && (zLine[0]=='.' || zLine[0]=='#') && nSql==0 ){
      if( ShellHasFlag(p, SHFLG_Echo) ) printf("%s\n", zLine);
      if( zLine[0]=='.' ){
        rc = do_meta_command(zLine, p);
        if( rc==2 ){ /* exit requested */
          break;
        }else if( rc ){
          errCnt++;
        }
      }
      continue;
    }
    if( line_is_command_terminator(zLine) && line_is_complete(zSql, nSql) ){
      memcpy(zLine,";",2);
    }
    nLine = strlen30(zLine);
    if( nSql+nLine+2>=nAlloc ){
      nAlloc = nSql+nLine+100;
      zSql = realloc(zSql, nAlloc);
      if( zSql==0 ) shell_out_of_memory();
    }
    nSqlPrior = nSql;
    if( nSql==0 ){
      int i;
      for(i=0; zLine[i] && IsSpace(zLine[i]); i++){}
      assert( nAlloc>0 && zSql!=0 );
      memcpy(zSql, zLine+i, nLine+1-i);
      startline = p->lineno;
      nSql = nLine-i;
    }else{
      zSql[nSql++] = '\n';
      memcpy(zSql+nSql, zLine, nLine+1);
      nSql += nLine;
    }
    if( nSql && line_contains_semicolon(&zSql[nSqlPrior], nSql-nSqlPrior)
                && sqlite3_complete(zSql) ){
      errCnt += runOneSqlLine(p, zSql, p->in, startline);
      nSql = 0;
      if( p->outCount ){
        output_reset(p);
        p->outCount = 0;
      }else{
        clearTempFile(p);
      }
    }else if( nSql && _all_whitespace(zSql) ){
      if( ShellHasFlag(p, SHFLG_Echo) ) printf("%s\n", zSql);
      nSql = 0;
    }
  }
  if( nSql && !_all_whitespace(zSql) ){
    errCnt += runOneSqlLine(p, zSql, p->in, startline);
  }
  free(zSql);
  free(zLine);
  return errCnt>0;
}
```

## runOneSqlLine

So this is the `runOneSqlLine` function. Looking at it, it appears to just wrap the `shell_exec` function, and has some error handling ontop of it. In the binary, this starts at offset `0x25220`

```
static int runOneSqlLine(ShellState *p, char *zSql, FILE *in, int startline){
  int rc;
  char *zErrMsg = 0;

  open_db(p, 0);
  if( ShellHasFlag(p,SHFLG_Backslash) ) resolve_backslashes(zSql);
  if( p->flgProgress & SHELL_PROGRESS_RESET ) p->nProgress = 0;
  BEGIN_TIMER;
  rc = shell_exec(p, zSql, &zErrMsg);
  END_TIMER;
  if( rc || zErrMsg ){
    char zPrefix[100];
    if( in!=0 || !stdin_is_interactive ){
      sqlite3_snprintf(sizeof(zPrefix), zPrefix,
                       "Error: near line %d:", startline);
    }else{
      sqlite3_snprintf(sizeof(zPrefix), zPrefix, "Error:");
    }
    if( zErrMsg!=0 ){
      utf8_printf(stderr, "%s %s\n", zPrefix, zErrMsg);
      sqlite3_free(zErrMsg);
      zErrMsg = 0;
    }else{
      utf8_printf(stderr, "%s %s\n", zPrefix, sqlite3_errmsg(p->db));
    }
    return 1;
  }else if( ShellHasFlag(p, SHFLG_CountChanges) ){
    raw_printf(p->out, "changes: %3d   total_changes: %d\n",
            sqlite3_changes(p->db), sqlite3_total_changes(p->db));
  }
  return 0;
}
```

## shell_exec

This is the `shell_exec` function, located at offset `0x23a50`. It calls these functions:

```
expertHandleSQL
sqlite3_prepare_v2
sqlite3_finalize
bind_prepared_stmt
exec_prepared_stmt
explain_data_delete
eqp_render
```

```
/*
** Execute a statement or set of statements.  Print
** any result rows/columns depending on the current mode
** set via the supplied callback.
**
** This is very similar to SQLite's built-in sqlite3_exec()
** function except it takes a slightly different callback
** and callback data argument.
*/
static int shell_exec(
  ShellState *pArg,                         /* Pointer to ShellState */
  const char *zSql,                         /* SQL to be evaluated */
  char **pzErrMsg                           /* Error msg written here */
){
  sqlite3_stmt *pStmt = NULL;     /* Statement to execute. */
  int rc = SQLITE_OK;             /* Return Code */
  int rc2;
  const char *zLeftover;          /* Tail of unprocessed SQL */
  sqlite3 *db = pArg->db;

  if( pzErrMsg ){
    *pzErrMsg = NULL;
  }

#ifndef SQLITE_OMIT_VIRTUALTABLE
  if( pArg->expert.pExpert ){
    rc = expertHandleSQL(pArg, zSql, pzErrMsg);
    return expertFinish(pArg, (rc!=SQLITE_OK), pzErrMsg);
  }
#endif

  while( zSql[0] && (SQLITE_OK == rc) ){
    static const char *zStmtSql;
    rc = sqlite3_prepare_v2(db, zSql, -1, &pStmt, &zLeftover);
    if( SQLITE_OK != rc ){
      if( pzErrMsg ){
        *pzErrMsg = save_err_msg(db);
      }
    }else{
      if( !pStmt ){
        /* this happens for a comment or white-space */
        zSql = zLeftover;
        while( IsSpace(zSql[0]) ) zSql++;
        continue;
      }
      zStmtSql = sqlite3_sql(pStmt);
      if( zStmtSql==0 ) zStmtSql = "";
      while( IsSpace(zStmtSql[0]) ) zStmtSql++;

      /* save off the prepared statment handle and reset row count */
      if( pArg ){
        pArg->pStmt = pStmt;
        pArg->cnt = 0;
      }

      /* echo the sql statement if echo on */
      if( pArg && ShellHasFlag(pArg, SHFLG_Echo) ){
        utf8_printf(pArg->out, "%s\n", zStmtSql ? zStmtSql : zSql);
      }

      /* Show the EXPLAIN QUERY PLAN if .eqp is on */
      if( pArg && pArg->autoEQP && sqlite3_stmt_isexplain(pStmt)==0 ){
        sqlite3_stmt *pExplain;
        char *zEQP;
        int triggerEQP = 0;
        disable_debug_trace_modes();
        sqlite3_db_config(db, SQLITE_DBCONFIG_TRIGGER_EQP, -1, &triggerEQP);
        if( pArg->autoEQP>=AUTOEQP_trigger ){
          sqlite3_db_config(db, SQLITE_DBCONFIG_TRIGGER_EQP, 1, 0);
        }
        zEQP = sqlite3_mprintf("EXPLAIN QUERY PLAN %s", zStmtSql);
        rc = sqlite3_prepare_v2(db, zEQP, -1, &pExplain, 0);
        if( rc==SQLITE_OK ){
          while( sqlite3_step(pExplain)==SQLITE_ROW ){
            const char *zEQPLine = (const char*)sqlite3_column_text(pExplain,3);
            int iEqpId = sqlite3_column_int(pExplain, 0);
            int iParentId = sqlite3_column_int(pExplain, 1);
            if( zEQPLine==0 ) zEQPLine = "";
            if( zEQPLine[0]=='-' ) eqp_render(pArg);
            eqp_append(pArg, iEqpId, iParentId, zEQPLine);
          }
          eqp_render(pArg);
        }
        sqlite3_finalize(pExplain);
        sqlite3_free(zEQP);
        if( pArg->autoEQP>=AUTOEQP_full ){
          /* Also do an EXPLAIN for ".eqp full" mode */
          zEQP = sqlite3_mprintf("EXPLAIN %s", zStmtSql);
          rc = sqlite3_prepare_v2(db, zEQP, -1, &pExplain, 0);
          if( rc==SQLITE_OK ){
            pArg->cMode = MODE_Explain;
            explain_data_prepare(pArg, pExplain);
            exec_prepared_stmt(pArg, pExplain);
            explain_data_delete(pArg);
          }
          sqlite3_finalize(pExplain);
          sqlite3_free(zEQP);
        }
        if( pArg->autoEQP>=AUTOEQP_trigger && triggerEQP==0 ){
          sqlite3_db_config(db, SQLITE_DBCONFIG_TRIGGER_EQP, 0, 0);
          /* Reprepare pStmt before reactiving trace modes */
          sqlite3_finalize(pStmt);
          sqlite3_prepare_v2(db, zSql, -1, &pStmt, 0);
          if( pArg ) pArg->pStmt = pStmt;
        }
        restore_debug_trace_modes();
      }

      if( pArg ){
        pArg->cMode = pArg->mode;
        if( pArg->autoExplain ){
          if( sqlite3_stmt_isexplain(pStmt)==1 ){
            pArg->cMode = MODE_Explain;
          }
          if( sqlite3_stmt_isexplain(pStmt)==2 ){
            pArg->cMode = MODE_EQP;
          }
        }

        /* If the shell is currently in ".explain" mode, gather the extra
        ** data required to add indents to the output.*/
        if( pArg->cMode==MODE_Explain ){
          explain_data_prepare(pArg, pStmt);
        }
      }

      bind_prepared_stmt(pArg, pStmt);
      exec_prepared_stmt(pArg, pStmt);
      explain_data_delete(pArg);
      eqp_render(pArg);

      /* print usage stats if stats on */
      if( pArg && pArg->statsOn ){
        display_stats(db, pArg, 0);
      }

      /* print loop-counters if required */
      if( pArg && pArg->scanstatsOn ){
        display_scanstats(db, pArg);
      }

      /* Finalize the statement just executed. If this fails, save a
      ** copy of the error message. Otherwise, set zSql to point to the
      ** next statement to execute. */
      rc2 = sqlite3_finalize(pStmt);
      if( rc!=SQLITE_NOMEM ) rc = rc2;
      if( rc==SQLITE_OK ){
        zSql = zLeftover;
        while( IsSpace(zSql[0]) ) zSql++;
      }else if( pzErrMsg ){
        *pzErrMsg = save_err_msg(db);
      }

      /* clear saved stmt handle */
      if( pArg ){
        pArg->pStmt = NULL;
      }
    }
  } /* end while */

  return rc;
}
```

## exec_prepared_stmt

So this function is a part of a function chain that will actually execute sql statements. Important thing I see here rn is that it calls `sqlite3_step`, which is in `vdbeapi.c`. 
```
static void exec_prepared_stmt(
  ShellState *pArg,                                /* Pointer to ShellState */
  sqlite3_stmt *pStmt                              /* Statment to run */
){
  int rc;

  if( pArg->cMode==MODE_Column
   || pArg->cMode==MODE_Table
   || pArg->cMode==MODE_Box
   || pArg->cMode==MODE_Markdown
  ){
    exec_prepared_stmt_columnar(pArg, pStmt);
    return;
  }

  /* perform the first step.  this will tell us if we
  ** have a result set or not and how wide it is.
  */
  rc = sqlite3_step(pStmt);
  /* if we have a result set... */
  if( SQLITE_ROW == rc ){
    /* allocate space for col name ptr, value ptr, and type */
    int nCol = sqlite3_column_count(pStmt);
    void *pData = sqlite3_malloc64(3*nCol*sizeof(const char*) + 1);
    if( !pData ){
      rc = SQLITE_NOMEM;
    }else{
      char **azCols = (char **)pData;      /* Names of result columns */
      char **azVals = &azCols[nCol];       /* Results */
      int *aiTypes = (int *)&azVals[nCol]; /* Result types */
      int i, x;
      assert(sizeof(int) <= sizeof(char *));
      /* save off ptrs to column names */
      for(i=0; i<nCol; i++){
        azCols[i] = (char *)sqlite3_column_name(pStmt, i);
      }
      do{
        /* extract the data and data types */
        for(i=0; i<nCol; i++){
          aiTypes[i] = x = sqlite3_column_type(pStmt, i);
          if( x==SQLITE_BLOB && pArg && pArg->cMode==MODE_Insert ){
            azVals[i] = "";
          }else{
            azVals[i] = (char*)sqlite3_column_text(pStmt, i);
          }
          if( !azVals[i] && (aiTypes[i]!=SQLITE_NULL) ){
            rc = SQLITE_NOMEM;
            break; /* from for */
          }
        } /* end for */

        /* if data and types extracted successfully... */
        if( SQLITE_ROW == rc ){
          /* call the supplied callback with the result row data */
          if( shell_callback(pArg, nCol, azVals, azCols, aiTypes) ){
            rc = SQLITE_ABORT;
          }else{
            rc = sqlite3_step(pStmt);
          }
        }
      } while( SQLITE_ROW == rc );
      sqlite3_free(pData);
      if( pArg->cMode==MODE_Json ){
        fputs("]\n", pArg->out);
      }
    }
  }
}
```

## sqlite3_step



```
int sqlite3_step(sqlite3_stmt *pStmt){
  int rc = SQLITE_OK;      /* Result from sqlite3Step() */
  Vdbe *v = (Vdbe*)pStmt;  /* the prepared statement */
  int cnt = 0;             /* Counter to prevent infinite loop of reprepares */
  sqlite3 *db;             /* The database connection */

  if( vdbeSafetyNotNull(v) ){
    return SQLITE_MISUSE_BKPT;
  }
  db = v->db;
  sqlite3_mutex_enter(db->mutex);
  v->doingRerun = 0;
  while( (rc = sqlite3Step(v))==SQLITE_SCHEMA
         && cnt++ < SQLITE_MAX_SCHEMA_RETRY ){
    int savedPc = v->pc;
    rc = sqlite3Reprepare(v);
    if( rc!=SQLITE_OK ){
      /* This case occurs after failing to recompile an sql statement. 
      ** The error message from the SQL compiler has already been loaded 
      ** into the database handle. This block copies the error message 
      ** from the database handle into the statement and sets the statement
      ** program counter to 0 to ensure that when the statement is 
      ** finalized or reset the parser error message is available via
      ** sqlite3_errmsg() and sqlite3_errcode().
      */
      const char *zErr = (const char *)sqlite3_value_text(db->pErr); 
      sqlite3DbFree(db, v->zErrMsg);
      if( !db->mallocFailed ){
        v->zErrMsg = sqlite3DbStrDup(db, zErr);
        v->rc = rc = sqlite3ApiExit(db, rc);
      } else {
        v->zErrMsg = 0;
        v->rc = rc = SQLITE_NOMEM_BKPT;
      }
      break;
    }
    sqlite3_reset(pStmt);
    if( savedPc>=0 ) v->doingRerun = 1;
    assert( v->expired==0 );
  }
  sqlite3_mutex_leave(db->mutex);
  return rc;
}
```

## sqlite3Step

So this is another step in the process for executing sql statements (located in `vdbeapi.c`). Important thing I'm seeing here now, is it executes `sqlite3VdbeExec`. This is located at offset `0xcabf0`.

```
static int sqlite3Step(Vdbe *p){
  sqlite3 *db;
  int rc;

  assert(p);
  if( p->iVdbeMagic!=VDBE_MAGIC_RUN ){
    /* We used to require that sqlite3_reset() be called before retrying
    ** sqlite3_step() after any error or after SQLITE_DONE.  But beginning
    ** with version 3.7.0, we changed this so that sqlite3_reset() would
    ** be called automatically instead of throwing the SQLITE_MISUSE error.
    ** This "automatic-reset" change is not technically an incompatibility, 
    ** since any application that receives an SQLITE_MISUSE is broken by
    ** definition.
    **
    ** Nevertheless, some published applications that were originally written
    ** for version 3.6.23 or earlier do in fact depend on SQLITE_MISUSE 
    ** returns, and those were broken by the automatic-reset change.  As a
    ** a work-around, the SQLITE_OMIT_AUTORESET compile-time restores the
    ** legacy behavior of returning SQLITE_MISUSE for cases where the 
    ** previous sqlite3_step() returned something other than a SQLITE_LOCKED
    ** or SQLITE_BUSY error.
    */
#ifdef SQLITE_OMIT_AUTORESET
    if( (rc = p->rc&0xff)==SQLITE_BUSY || rc==SQLITE_LOCKED ){
      sqlite3_reset((sqlite3_stmt*)p);
    }else{
      return SQLITE_MISUSE_BKPT;
    }
#else
    sqlite3_reset((sqlite3_stmt*)p);
#endif
  }

  /* Check that malloc() has not failed. If it has, return early. */
  db = p->db;
  if( db->mallocFailed ){
    p->rc = SQLITE_NOMEM;
    return SQLITE_NOMEM_BKPT;
  }

  if( p->pc<0 && p->expired ){
    p->rc = SQLITE_SCHEMA;
    rc = SQLITE_ERROR;
    if( (p->prepFlags & SQLITE_PREPARE_SAVESQL)!=0 ){
      /* If this statement was prepared using saved SQL and an 
      ** error has occurred, then return the error code in p->rc to the
      ** caller. Set the error code in the database handle to the same value.
      */ 
      rc = sqlite3VdbeTransferError(p);
    }
    goto end_of_step;
  }
  if( p->pc<0 ){
    /* If there are no other statements currently running, then
    ** reset the interrupt flag.  This prevents a call to sqlite3_interrupt
    ** from interrupting a statement that has not yet started.
    */
    if( db->nVdbeActive==0 ){
      AtomicStore(&db->u1.isInterrupted, 0);
    }

    assert( db->nVdbeWrite>0 || db->autoCommit==0 
        || (db->nDeferredCons==0 && db->nDeferredImmCons==0)
    );

#ifndef SQLITE_OMIT_TRACE
    if( (db->mTrace & (SQLITE_TRACE_PROFILE|SQLITE_TRACE_XPROFILE))!=0
        && !db->init.busy && p->zSql ){
      sqlite3OsCurrentTimeInt64(db->pVfs, &p->startTime);
    }else{
      assert( p->startTime==0 );
    }
#endif

    db->nVdbeActive++;
    if( p->readOnly==0 ) db->nVdbeWrite++;
    if( p->bIsReader ) db->nVdbeRead++;
    p->pc = 0;
  }
#ifdef SQLITE_DEBUG
  p->rcApp = SQLITE_OK;
#endif
#ifndef SQLITE_OMIT_EXPLAIN
  if( p->explain ){
    rc = sqlite3VdbeList(p);
  }else
#endif /* SQLITE_OMIT_EXPLAIN */
  {
    db->nVdbeExec++;
    rc = sqlite3VdbeExec(p);
    db->nVdbeExec--;
  }

  if( rc!=SQLITE_ROW ){
#ifndef SQLITE_OMIT_TRACE
    /* If the statement completed successfully, invoke the profile callback */
    checkProfileCallback(db, p);
#endif

    if( rc==SQLITE_DONE && db->autoCommit ){
      assert( p->rc==SQLITE_OK );
      p->rc = doWalCallbacks(db);
      if( p->rc!=SQLITE_OK ){
        rc = SQLITE_ERROR;
      }
    }else if( rc!=SQLITE_DONE && (p->prepFlags & SQLITE_PREPARE_SAVESQL)!=0 ){
      /* If this statement was prepared using saved SQL and an 
      ** error has occurred, then return the error code in p->rc to the
      ** caller. Set the error code in the database handle to the same value.
      */ 
      rc = sqlite3VdbeTransferError(p);
    }
  }

  db->errCode = rc;
  if( SQLITE_NOMEM==sqlite3ApiExit(p->db, p->rc) ){
    p->rc = SQLITE_NOMEM_BKPT;
    if( (p->prepFlags & SQLITE_PREPARE_SAVESQL)!=0 ) rc = p->rc;
  }
end_of_step:
  /* There are only a limited number of result codes allowed from the
  ** statements prepared using the legacy sqlite3_prepare() interface */
  assert( (p->prepFlags & SQLITE_PREPARE_SAVESQL)!=0
       || rc==SQLITE_ROW  || rc==SQLITE_DONE   || rc==SQLITE_ERROR 
       || (rc&0xff)==SQLITE_BUSY || rc==SQLITE_MISUSE
  );
  return (rc&db->errMask);
}
```

## sqlite3VdbeExec

So this is the function which actually handles executing sql statements. It is located in the `vdbe.c` file. This is at offset `0xc0cb0` in the binary. Lookin at this function, we can tell several things. First off, this operates off a virtual machine. What happens, is there are opcodes that specify certain operations. There is a switch statement which takes an opcode, and operates the corresponding operation. We can see that here:

```
    switch( pOp->opcode ){
```

and then the cases:

```
case OP_Goto: {             /* jump */
```

Now a few things. The list of the opcodes are defined under `bld/opcodes.h`. For example:

```
#define OP_Savepoint       0
#define OP_AutoCommit      1
#define OP_Transaction     2
```

## sqlite3RunParser

So this function is what parses sql code. It is at offset `0xdba30` in the binary. In the source code, it is in the `tokenize.c` file.
