## Insertion

```
/* Opcode: Insert P1 P2 P3 P4 P5
** Synopsis: intkey=r[P3] data=r[P2]
**
** Write an entry into the table of cursor P1.  A new entry is
** created if it doesn't already exist or the data for an existing
** entry is overwritten.  The data is the value MEM_Blob stored in register
** number P2. The key is stored in register P3. The key must
** be a MEM_Int.
**
** If the OPFLAG_NCHANGE flag of P5 is set, then the row change count is
** incremented (otherwise not).  If the OPFLAG_LASTROWID flag of P5 is set,
** then rowid is stored for subsequent return by the
** sqlite3_last_insert_rowid() function (otherwise it is unmodified).
**
** If the OPFLAG_USESEEKRESULT flag of P5 is set, the implementation might
** run faster by avoiding an unnecessary seek on cursor P1.  However,
** the OPFLAG_USESEEKRESULT flag must only be set if there have been no prior
** seeks on the cursor or if the most recent seek used a key equal to P3.
**
** If the OPFLAG_ISUPDATE flag is set, then this opcode is part of an
** UPDATE operation.  Otherwise (if the flag is clear) then this opcode
** is part of an INSERT operation.  The difference is only important to
** the update hook.
**
** Parameter P4 may point to a Table structure, or may be NULL. If it is 
** not NULL, then the update-hook (sqlite3.xUpdateCallback) is invoked 
** following a successful insert.
**
** (WARNING/TODO: If P1 is a pseudo-cursor and P2 is dynamically
** allocated, then ownership of P2 is transferred to the pseudo-cursor
** and register P2 becomes ephemeral.  If the cursor is changed, the
** value of register P2 will then change.  Make sure this does not
** cause any problems.)
**
** This instruction only works on tables.  The equivalent instruction
** for indices is OP_IdxInsert.
*/
case OP_Insert: {
  Mem *pData;       /* MEM cell holding data for the record to be inserted */
  Mem *pKey;        /* MEM cell holding key  for the record */
  VdbeCursor *pC;   /* Cursor to table into which insert is written */
  int seekResult;   /* Result of prior seek or 0 if no USESEEKRESULT flag */
  const char *zDb;  /* database name - used by the update hook */
  Table *pTab;      /* Table structure - used by update and pre-update hooks */
  BtreePayload x;   /* Payload to be inserted */

  pData = &aMem[pOp->p2];
  assert( pOp->p1>=0 && pOp->p1<p->nCursor );
  assert( memIsValid(pData) );
  pC = p->apCsr[pOp->p1];
  assert( pC!=0 );
  assert( pC->eCurType==CURTYPE_BTREE );
  assert( pC->deferredMoveto==0 );
  assert( pC->uc.pCursor!=0 );
  assert( (pOp->p5 & OPFLAG_ISNOOP) || pC->isTable );
  assert( pOp->p4type==P4_TABLE || pOp->p4type>=P4_STATIC );
  REGISTER_TRACE(pOp->p2, pData);
  sqlite3VdbeIncrWriteCounter(p, pC);

  pKey = &aMem[pOp->p3];
  assert( pKey->flags & MEM_Int );
  assert( memIsValid(pKey) );
  REGISTER_TRACE(pOp->p3, pKey);
  x.nKey = pKey->u.i;

  if( pOp->p4type==P4_TABLE && HAS_UPDATE_HOOK(db) ){
    assert( pC->iDb>=0 );
    zDb = db->aDb[pC->iDb].zDbSName;
    pTab = pOp->p4.pTab;
    assert( (pOp->p5 & OPFLAG_ISNOOP) || HasRowid(pTab) );
  }else{
    pTab = 0;
    zDb = 0;  /* Not needed.  Silence a compiler warning. */
  }

#ifdef SQLITE_ENABLE_PREUPDATE_HOOK
  /* Invoke the pre-update hook, if any */
  if( pTab ){
    if( db->xPreUpdateCallback && !(pOp->p5 & OPFLAG_ISUPDATE) ){
      sqlite3VdbePreUpdateHook(p, pC, SQLITE_INSERT, zDb, pTab, x.nKey,pOp->p2);
    }
    if( db->xUpdateCallback==0 || pTab->aCol==0 ){
      /* Prevent post-update hook from running in cases when it should not */
      pTab = 0;
    }
  }
  if( pOp->p5 & OPFLAG_ISNOOP ) break;
#endif

  if( pOp->p5 & OPFLAG_NCHANGE ) p->nChange++;
  if( pOp->p5 & OPFLAG_LASTROWID ) db->lastRowid = x.nKey;
  assert( (pData->flags & (MEM_Blob|MEM_Str))!=0 || pData->n==0 );
  x.pData = pData->z;
  x.nData = pData->n;
  seekResult = ((pOp->p5 & OPFLAG_USESEEKRESULT) ? pC->seekResult : 0);
  if( pData->flags & MEM_Zero ){
    x.nZero = pData->u.nZero;
  }else{
    x.nZero = 0;
  }
  x.pKey = 0;
  rc = sqlite3BtreeInsert(pC->uc.pCursor, &x,
      (pOp->p5 & (OPFLAG_APPEND|OPFLAG_SAVEPOSITION|OPFLAG_PREFORMAT)), 
      seekResult
  );
  pC->deferredMoveto = 0;
  pC->cacheStatus = CACHE_STALE;

  /* Invoke the update-hook if required. */
  if( rc ) goto abort_due_to_error;
  if( pTab ){
    assert( db->xUpdateCallback!=0 );
    assert( pTab->aCol!=0 );
    db->xUpdateCallback(db->pUpdateArg,
           (pOp->p5 & OPFLAG_ISUPDATE) ? SQLITE_UPDATE : SQLITE_INSERT,
           zDb, pTab->zName, x.nKey);
  }
  break;
}
```

```
/*
** Insert a new record into the BTree.  The content of the new record
** is described by the pX object.  The pCur cursor is used only to
** define what table the record should be inserted into, and is left
** pointing at a random location.
**
** For a table btree (used for rowid tables), only the pX.nKey value of
** the key is used. The pX.pKey value must be NULL.  The pX.nKey is the
** rowid or INTEGER PRIMARY KEY of the row.  The pX.nData,pData,nZero fields
** hold the content of the row.
**
** For an index btree (used for indexes and WITHOUT ROWID tables), the
** key is an arbitrary byte sequence stored in pX.pKey,nKey.  The 
** pX.pData,nData,nZero fields must be zero.
**
** If the seekResult parameter is non-zero, then a successful call to
** MovetoUnpacked() to seek cursor pCur to (pKey,nKey) has already
** been performed.  In other words, if seekResult!=0 then the cursor
** is currently pointing to a cell that will be adjacent to the cell
** to be inserted.  If seekResult<0 then pCur points to a cell that is
** smaller then (pKey,nKey).  If seekResult>0 then pCur points to a cell
** that is larger than (pKey,nKey).
**
** If seekResult==0, that means pCur is pointing at some unknown location.
** In that case, this routine must seek the cursor to the correct insertion
** point for (pKey,nKey) before doing the insertion.  For index btrees,
** if pX->nMem is non-zero, then pX->aMem contains pointers to the unpacked
** key values and pX->aMem can be used instead of pX->pKey to avoid having
** to decode the key.
*/
int sqlite3BtreeInsert(
  BtCursor *pCur,                /* Insert data into the table of this cursor */
  const BtreePayload *pX,        /* Content of the row to be inserted */
  int flags,                     /* True if this is likely an append */
  int seekResult                 /* Result of prior MovetoUnpacked() call */
){
  int rc;
  int loc = seekResult;          /* -1: before desired location  +1: after */
  int szNew = 0;
  int idx;
  MemPage *pPage;
  Btree *p = pCur->pBtree;
  BtShared *pBt = p->pBt;
  unsigned char *oldCell;
  unsigned char *newCell = 0;

  assert( (flags & (BTREE_SAVEPOSITION|BTREE_APPEND|BTREE_PREFORMAT))==flags );
  assert( (flags & BTREE_PREFORMAT)==0 || seekResult || pCur->pKeyInfo==0 );

  if( pCur->eState==CURSOR_FAULT ){
    assert( pCur->skipNext!=SQLITE_OK );
    return pCur->skipNext;
  }

  assert( cursorOwnsBtShared(pCur) );
  assert( (pCur->curFlags & BTCF_WriteFlag)!=0
              && pBt->inTransaction==TRANS_WRITE
              && (pBt->btsFlags & BTS_READ_ONLY)==0 );
  assert( hasSharedCacheTableLock(p, pCur->pgnoRoot, pCur->pKeyInfo!=0, 2) );

  /* Assert that the caller has been consistent. If this cursor was opened
  ** expecting an index b-tree, then the caller should be inserting blob
  ** keys with no associated data. If the cursor was opened expecting an
  ** intkey table, the caller should be inserting integer keys with a
  ** blob of associated data.  */
  assert( (flags & BTREE_PREFORMAT) || (pX->pKey==0)==(pCur->pKeyInfo==0) );

  /* Save the positions of any other cursors open on this table.
  **
  ** In some cases, the call to btreeMoveto() below is a no-op. For
  ** example, when inserting data into a table with auto-generated integer
  ** keys, the VDBE layer invokes sqlite3BtreeLast() to figure out the 
  ** integer key to use. It then calls this function to actually insert the 
  ** data into the intkey B-Tree. In this case btreeMoveto() recognizes
  ** that the cursor is already where it needs to be and returns without
  ** doing any work. To avoid thwarting these optimizations, it is important
  ** not to clear the cursor here.
  */
  if( pCur->curFlags & BTCF_Multiple ){
    rc = saveAllCursors(pBt, pCur->pgnoRoot, pCur);
    if( rc ) return rc;
  }

  if( pCur->pKeyInfo==0 ){
    assert( pX->pKey==0 );
    /* If this is an insert into a table b-tree, invalidate any incrblob 
    ** cursors open on the row being replaced */
    invalidateIncrblobCursors(p, pCur->pgnoRoot, pX->nKey, 0);

    /* If BTREE_SAVEPOSITION is set, the cursor must already be pointing 
    ** to a row with the same key as the new entry being inserted.
    */
#ifdef SQLITE_DEBUG
    if( flags & BTREE_SAVEPOSITION ){
      assert( pCur->curFlags & BTCF_ValidNKey );
      assert( pX->nKey==pCur->info.nKey );
      assert( loc==0 );
    }
#endif

    /* On the other hand, BTREE_SAVEPOSITION==0 does not imply
    ** that the cursor is not pointing to a row to be overwritten.
    ** So do a complete check.
    */
    if( (pCur->curFlags&BTCF_ValidNKey)!=0 && pX->nKey==pCur->info.nKey ){
      /* The cursor is pointing to the entry that is to be
      ** overwritten */
      assert( pX->nData>=0 && pX->nZero>=0 );
      if( pCur->info.nSize!=0
       && pCur->info.nPayload==(u32)pX->nData+pX->nZero
      ){
        /* New entry is the same size as the old.  Do an overwrite */
        return btreeOverwriteCell(pCur, pX);
      }
      assert( loc==0 );
    }else if( loc==0 ){
      /* The cursor is *not* pointing to the cell to be overwritten, nor
      ** to an adjacent cell.  Move the cursor so that it is pointing either
      ** to the cell to be overwritten or an adjacent cell.
      */
      rc = sqlite3BtreeMovetoUnpacked(pCur, 0, pX->nKey, flags!=0, &loc);
      if( rc ) return rc;
    }
  }else{
    /* This is an index or a WITHOUT ROWID table */

    /* If BTREE_SAVEPOSITION is set, the cursor must already be pointing 
    ** to a row with the same key as the new entry being inserted.
    */
    assert( (flags & BTREE_SAVEPOSITION)==0 || loc==0 );

    /* If the cursor is not already pointing either to the cell to be
    ** overwritten, or if a new cell is being inserted, if the cursor is
    ** not pointing to an immediately adjacent cell, then move the cursor
    ** so that it does.
    */
    if( loc==0 && (flags & BTREE_SAVEPOSITION)==0 ){
      if( pX->nMem ){
        UnpackedRecord r;
        r.pKeyInfo = pCur->pKeyInfo;
        r.aMem = pX->aMem;
        r.nField = pX->nMem;
        r.default_rc = 0;
        r.errCode = 0;
        r.r1 = 0;
        r.r2 = 0;
        r.eqSeen = 0;
        rc = sqlite3BtreeMovetoUnpacked(pCur, &r, 0, flags!=0, &loc);
      }else{
        rc = btreeMoveto(pCur, pX->pKey, pX->nKey, flags!=0, &loc);
      }
      if( rc ) return rc;
    }

    /* If the cursor is currently pointing to an entry to be overwritten
    ** and the new content is the same as as the old, then use the
    ** overwrite optimization.
    */
    if( loc==0 ){
      getCellInfo(pCur);
      if( pCur->info.nKey==pX->nKey ){
        BtreePayload x2;
        x2.pData = pX->pKey;
        x2.nData = pX->nKey;
        x2.nZero = 0;
        return btreeOverwriteCell(pCur, &x2);
      }
    }

  }
  assert( pCur->eState==CURSOR_VALID 
       || (pCur->eState==CURSOR_INVALID && loc)
       || CORRUPT_DB );

  pPage = pCur->pPage;
  assert( pPage->intKey || pX->nKey>=0 || (flags & BTREE_PREFORMAT) );
  assert( pPage->leaf || !pPage->intKey );
  if( pPage->nFree<0 ){
    if( pCur->eState>CURSOR_INVALID ){
      rc = SQLITE_CORRUPT_BKPT;
    }else{
      rc = btreeComputeFreeSpace(pPage);
    }
    if( rc ) return rc;
  }

  TRACE(("INSERT: table=%d nkey=%lld ndata=%d page=%d %s\n",
          pCur->pgnoRoot, pX->nKey, pX->nData, pPage->pgno,
          loc==0 ? "overwrite" : "new entry"));
  assert( pPage->isInit );
  newCell = pBt->pTmpSpace;
  assert( newCell!=0 );
  if( flags & BTREE_PREFORMAT ){
    rc = SQLITE_OK;
    szNew = pBt->nPreformatSize;
    if( szNew<4 ) szNew = 4;
    if( ISAUTOVACUUM && szNew>pPage->maxLocal ){
      CellInfo info;
      pPage->xParseCell(pPage, newCell, &info);
      if( info.nPayload!=info.nLocal ){
        Pgno ovfl = get4byte(&newCell[szNew-4]);
        ptrmapPut(pBt, ovfl, PTRMAP_OVERFLOW1, pPage->pgno, &rc);
      }
    }
  }else{
    rc = fillInCell(pPage, newCell, pX, &szNew);
  }
  if( rc ) goto end_insert;
  assert( szNew==pPage->xCellSize(pPage, newCell) );
  assert( szNew <= MX_CELL_SIZE(pBt) );
  idx = pCur->ix;
  if( loc==0 ){
    CellInfo info;
    assert( idx<pPage->nCell );
    rc = sqlite3PagerWrite(pPage->pDbPage);
    if( rc ){
      goto end_insert;
    }
    oldCell = findCell(pPage, idx);
    if( !pPage->leaf ){
      memcpy(newCell, oldCell, 4);
    }
    rc = clearCell(pPage, oldCell, &info);
    testcase( pCur->curFlags & BTCF_ValidOvfl );
    invalidateOverflowCache(pCur);
    if( info.nSize==szNew && info.nLocal==info.nPayload 
     && (!ISAUTOVACUUM || szNew<pPage->minLocal)
    ){
      /* Overwrite the old cell with the new if they are the same size.
      ** We could also try to do this if the old cell is smaller, then add
      ** the leftover space to the free list.  But experiments show that
      ** doing that is no faster then skipping this optimization and just
      ** calling dropCell() and insertCell(). 
      **
      ** This optimization cannot be used on an autovacuum database if the
      ** new entry uses overflow pages, as the insertCell() call below is
      ** necessary to add the PTRMAP_OVERFLOW1 pointer-map entry.  */
      assert( rc==SQLITE_OK ); /* clearCell never fails when nLocal==nPayload */
      if( oldCell < pPage->aData+pPage->hdrOffset+10 ){
        return SQLITE_CORRUPT_BKPT;
      }
      if( oldCell+szNew > pPage->aDataEnd ){
        return SQLITE_CORRUPT_BKPT;
      }
      memcpy(oldCell, newCell, szNew);
      return SQLITE_OK;
    }
    dropCell(pPage, idx, info.nSize, &rc);
    if( rc ) goto end_insert;
  }else if( loc<0 && pPage->nCell>0 ){
    assert( pPage->leaf );
    idx = ++pCur->ix;
    pCur->curFlags &= ~BTCF_ValidNKey;
  }else{
    assert( pPage->leaf );
  }
  insertCell(pPage, idx, newCell, szNew, 0, 0, &rc);
  assert( pPage->nOverflow==0 || rc==SQLITE_OK );
  assert( rc!=SQLITE_OK || pPage->nCell>0 || pPage->nOverflow>0 );

  /* If no error has occurred and pPage has an overflow cell, call balance() 
  ** to redistribute the cells within the tree. Since balance() may move
  ** the cursor, zero the BtCursor.info.nSize and BTCF_ValidNKey
  ** variables.
  **
  ** Previous versions of SQLite called moveToRoot() to move the cursor
  ** back to the root page as balance() used to invalidate the contents
  ** of BtCursor.apPage[] and BtCursor.aiIdx[]. Instead of doing that,
  ** set the cursor state to "invalid". This makes common insert operations
  ** slightly faster.
  **
  ** There is a subtle but important optimization here too. When inserting
  ** multiple records into an intkey b-tree using a single cursor (as can
  ** happen while processing an "INSERT INTO ... SELECT" statement), it
  ** is advantageous to leave the cursor pointing to the last entry in
  ** the b-tree if possible. If the cursor is left pointing to the last
  ** entry in the table, and the next row inserted has an integer key
  ** larger than the largest existing key, it is possible to insert the
  ** row without seeking the cursor. This can be a big performance boost.
  */
  pCur->info.nSize = 0;
  if( pPage->nOverflow ){
    assert( rc==SQLITE_OK );
    pCur->curFlags &= ~(BTCF_ValidNKey);
    rc = balance(pCur);

    /* Must make sure nOverflow is reset to zero even if the balance()
    ** fails. Internal data structure corruption will result otherwise. 
    ** Also, set the cursor state to invalid. This stops saveCursorPosition()
    ** from trying to save the current position of the cursor.  */
    pCur->pPage->nOverflow = 0;
    pCur->eState = CURSOR_INVALID;
    if( (flags & BTREE_SAVEPOSITION) && rc==SQLITE_OK ){
      btreeReleaseAllCursorPages(pCur);
      if( pCur->pKeyInfo ){
        assert( pCur->pKey==0 );
        pCur->pKey = sqlite3Malloc( pX->nKey );
        if( pCur->pKey==0 ){
          rc = SQLITE_NOMEM;
        }else{
          memcpy(pCur->pKey, pX->pKey, pX->nKey);
        }
      }
      pCur->eState = CURSOR_REQUIRESEEK;
      pCur->nKey = pX->nKey;
    }
  }
  assert( pCur->iPage<0 || pCur->pPage->nOverflow==0 );

end_insert:
  return rc;
}
```

```

/*
** Insert a new cell on pPage at cell index "i".  pCell points to the
** content of the cell.
**
** If the cell content will fit on the page, then put it there.  If it
** will not fit, then make a copy of the cell content into pTemp if
** pTemp is not null.  Regardless of pTemp, allocate a new entry
** in pPage->apOvfl[] and make it point to the cell content (either
** in pTemp or the original pCell) and also record its index. 
** Allocating a new entry in pPage->aCell[] implies that 
** pPage->nOverflow is incremented.
**
** *pRC must be SQLITE_OK when this routine is called.
*/
static void insertCell(
  MemPage *pPage,   /* Page into which we are copying */
  int i,            /* New cell becomes the i-th cell of the page */
  u8 *pCell,        /* Content of the new cell */
  int sz,           /* Bytes of content in pCell */
  u8 *pTemp,        /* Temp storage space for pCell, if needed */
  Pgno iChild,      /* If non-zero, replace first 4 bytes with this value */
  int *pRC          /* Read and write return code from here */
){
  int idx = 0;      /* Where to write new cell content in data[] */
  int j;            /* Loop counter */
  u8 *data;         /* The content of the whole page */
  u8 *pIns;         /* The point in pPage->aCellIdx[] where no cell inserted */

  assert( *pRC==SQLITE_OK );
  assert( i>=0 && i<=pPage->nCell+pPage->nOverflow );
  assert( MX_CELL(pPage->pBt)<=10921 );
  assert( pPage->nCell<=MX_CELL(pPage->pBt) || CORRUPT_DB );
  assert( pPage->nOverflow<=ArraySize(pPage->apOvfl) );
  assert( ArraySize(pPage->apOvfl)==ArraySize(pPage->aiOvfl) );
  assert( sqlite3_mutex_held(pPage->pBt->mutex) );
  assert( sz==pPage->xCellSize(pPage, pCell) || CORRUPT_DB );
  assert( pPage->nFree>=0 );
  if( pPage->nOverflow || sz+2>pPage->nFree ){
    if( pTemp ){
      memcpy(pTemp, pCell, sz);
      pCell = pTemp;
    }
    if( iChild ){
      put4byte(pCell, iChild);
    }
    j = pPage->nOverflow++;
    /* Comparison against ArraySize-1 since we hold back one extra slot
    ** as a contingency.  In other words, never need more than 3 overflow
    ** slots but 4 are allocated, just to be safe. */
    assert( j < ArraySize(pPage->apOvfl)-1 );
    pPage->apOvfl[j] = pCell;
    pPage->aiOvfl[j] = (u16)i;

    /* When multiple overflows occur, they are always sequential and in
    ** sorted order.  This invariants arise because multiple overflows can
    ** only occur when inserting divider cells into the parent page during
    ** balancing, and the dividers are adjacent and sorted.
    */
    assert( j==0 || pPage->aiOvfl[j-1]<(u16)i ); /* Overflows in sorted order */
    assert( j==0 || i==pPage->aiOvfl[j-1]+1 );   /* Overflows are sequential */
  }else{
    int rc = sqlite3PagerWrite(pPage->pDbPage);
    if( rc!=SQLITE_OK ){
      *pRC = rc;
      return;
    }
    assert( sqlite3PagerIswriteable(pPage->pDbPage) );
    data = pPage->aData;
    assert( &data[pPage->cellOffset]==pPage->aCellIdx );
    rc = allocateSpace(pPage, sz, &idx);
    if( rc ){ *pRC = rc; return; }
    /* The allocateSpace() routine guarantees the following properties
    ** if it returns successfully */
    assert( idx >= 0 );
    assert( idx >= pPage->cellOffset+2*pPage->nCell+2 || CORRUPT_DB );
    assert( idx+sz <= (int)pPage->pBt->usableSize );
    pPage->nFree -= (u16)(2 + sz);
    if( iChild ){
      /* In a corrupt database where an entry in the cell index section of
      ** a btree page has a value of 3 or less, the pCell value might point
      ** as many as 4 bytes in front of the start of the aData buffer for
      ** the source page.  Make sure this does not cause problems by not
      ** reading the first 4 bytes */
      memcpy(&data[idx+4], pCell+4, sz-4);
      put4byte(&data[idx], iChild);
    }else{
      memcpy(&data[idx], pCell, sz);
    }
    pIns = pPage->aCellIdx + i*2;
    memmove(pIns+2, pIns, 2*(pPage->nCell - i));
    put2byte(pIns, idx);
    pPage->nCell++;
    /* increment the cell count */
    if( (++data[pPage->hdrOffset+4])==0 ) data[pPage->hdrOffset+3]++;
    assert( get2byte(&data[pPage->hdrOffset+3])==pPage->nCell || CORRUPT_DB );
#ifndef SQLITE_OMIT_AUTOVACUUM
    if( pPage->pBt->autoVacuum ){
      /* The cell may contain a pointer to an overflow page. If so, write
      ** the entry for the overflow page into the pointer map.
      */
      ptrmapPutOvflPtr(pPage, pPage, pCell, pRC);
    }
#endif
  }
}

```