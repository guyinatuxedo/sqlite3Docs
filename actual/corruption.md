# Database File Corruption


## OP_Column

The `OP_Column` opcode is responsible for extracting memory from a record, thus extracting memory from a region we control.

Looking at this code for this opcode, it can be split up into three basic sections:

```
0.)		Adjusting the cursor
1.)		Getting the column type/offset
2.)		Extracting the Data
```

To calculate the offset of the particular column, this loop happens. Effectively what is happening is this. It iterates through the header of the record, looking at the type byte. It then uses the `sqlite3VdbeOneByteSerialTypeLen/sqlite3VdbeSerialTypeLen` functions to extract the size of the column. With that information, it can determine the offset to the column (which the offsets are stored in `aOffset`).

```
      do{
        if( (pC->aType[i] = t = zHdr[0])<0x80 ){
          zHdr++;
          offset64 += sqlite3VdbeOneByteSerialTypeLen(t);
        }else{
          zHdr += sqlite3GetVarint32(zHdr, &t);
          pC->aType[i] = t;
          offset64 += sqlite3VdbeSerialTypeLen(t);
        }
        aOffset[++i] = (u32)(offset64 & 0xffffffff);
      }while( (u32)i<=p2 && zHdr<zEndHdr );
```


So next, we have the spot that actually extract the data from the cell. First it checks the type of the column (stored as the `u32` `t` variable). If it is less than `12` (meaning it is a numerical value) it uses the `sqlite3VdbeSerialGet` function. If not, then it is a string, which it will use `memcpy` for.

```
  if( pC->szRow>=aOffset[p2+1] ){
    /* This is the common case where the desired content fits on the original
    ** page - where the content is not on an overflow page */
    zData = pC->aRow + aOffset[p2];
    if( t<12 ){
      sqlite3VdbeSerialGet(zData, t, pDest);
    }else{
      /* If the column value is a string, we need a persistent value, not
      ** a MEM_Ephem value.  This branch is a fast short-cut that is equivalent
      ** to calling sqlite3VdbeSerialGet() and sqlite3VdbeDeephemeralize().
      */
      static const u16 aFlag[] = { MEM_Blob, MEM_Str|MEM_Term };
      pDest->n = len = (t-12)/2;
      pDest->enc = encoding;
      if( pDest->szMalloc < len+2 ){
        pDest->flags = MEM_Null;
        if( sqlite3VdbeMemGrow(pDest, len+2, 0) ) goto no_mem;
      }else{
        pDest->z = pDest->zMalloc;
      }
      memcpy(pDest->z, zData, len);
      pDest->z[len] = 0;
      pDest->z[len+1] = 0;
      pDest->flags = aFlag[t&1];
    }
  }
```

## Page Space Allocation

So allocation of space on a page happens in the `allocateSpace` function. There are three spots from which memory on a page can be allocated. They are these three, and will be attempted in this order:

```
0.)		From the freelist
1.)		From defragmentation
2.)		From the unallocated space
```

One more thing. There are two things commonly referred to here, which are the `gap` and the `top`. The `top` refers to the first byte of the cell content area. The gap refers to the first byte directly after the cell ptrs area (so the first byte of the cell content area).  

#### FreeList Allocation

So this is the first memory allocation attempted. This will be attempted if the `offsetFree` value of the page header is not null, and `gap+2<=top` is true (gap is more than 1 bytes above the top). It will attempt to find a free spot that is the correct size with the `pageFindSlot` function. It will search for a free page with the `pageFindSlot` function.

#### Page Defragmentation

This will be attempted if the `gap+2+nByte>top` (new allocated space will go past the top). Defragmentation is the process of realligning the cells of the page, as to merge fragments together into usable blocks of freed memory.

#### Unallocated Space

So this will be the last resort. This just allocates the space from the unallocated region.













sqlite> .open natural
sqlite> create table x (y varchar(100), z int);
insert into x (y, z) values ("15935728", 100);
insert into x (y, z) values ("15935728", 200);
insert into x (y, z) values ("75395128", 300);
insert into x (y, z) values ("15935728", 400);
insert into x (y, z) values ("15935728", 500);


86755
88526

818
2589



`sqlite3VdbeSerialGet`

`pie b *0xb55b0`



0x5555556b7c00

```
88506	/* Opcode: Column P1 P2 P3 P4 P5
88507	** Synopsis: r[P3]=PX
88508	**
88509	** Interpret the data that cursor P1 points to as a structure built using
88510	** the MakeRecord instruction.  (See the MakeRecord opcode for additional
88511	** information about the format of the data.)  Extract the P2-th column
88512	** from this record.  If there are less that (P2+1)
88513	** values in the record, extract a NULL.
88514	**
88515	** The value extracted is stored in register P3.
88516	**
88517	** If the record contains fewer than P2 fields, then extract a NULL.  Or,
88518	** if the P4 argument is a P4_MEM use the value of the P4 argument as
88519	** the result.
88520	**
88521	** If the OPFLAG_LENGTHARG and OPFLAG_TYPEOFARG bits are set on P5 then
88522	** the result is guaranteed to only be used as the argument of a length()
88523	** or typeof() function, respectively.  The loading of large blobs can be
88524	** skipped for length() and all content loading can be skipped for typeof().
88525	*/
88526	case OP_Column: {
88527	  u32 p2;            /* column number to retrieve */
88528	  VdbeCursor *pC;    /* The VDBE cursor */
88529	  BtCursor *pCrsr;   /* The BTree cursor */
88530	  u32 *aOffset;      /* aOffset[i] is offset to start of data for i-th column */
88531	  int len;           /* The length of the serialized data for the column */
88532	  int i;             /* Loop counter */
88533	  Mem *pDest;        /* Where to write the extracted value */
88534	  Mem sMem;          /* For storing the record being decoded */
88535	  const u8 *zData;   /* Part of the record being decoded */
88536	  const u8 *zHdr;    /* Next unparsed byte of the header */
88537	  const u8 *zEndHdr; /* Pointer to first byte after the header */
88538	  u64 offset64;      /* 64-bit offset */
88539	  u32 t;             /* A type code from the record header */
88540	  Mem *pReg;         /* PseudoTable input register */
88541	
88542	  assert( pOp->p1>=0 && pOp->p1<p->nCursor );
88543	  pC = p->apCsr[pOp->p1];
88544	  assert( pC!=0 );
88545	  p2 = (u32)pOp->p2;
88546	
88547	  /* If the cursor cache is stale (meaning it is not currently point at
88548	  ** the correct row) then bring it up-to-date by doing the necessary
88549	  ** B-Tree seek. */
88550	  rc = sqlite3VdbeCursorMoveto(&pC, &p2);
88551	  if( rc ) goto abort_due_to_error;
88552	
88553	  assert( pOp->p3>0 && pOp->p3<=(p->nMem+1 - p->nCursor) );
88554	  pDest = &aMem[pOp->p3];
88555	  memAboutToChange(p, pDest);
88556	  assert( pC!=0 );
88557	  assert( p2<(u32)pC->nField );
88558	  aOffset = pC->aOffset;
88559	  assert( pC->eCurType!=CURTYPE_VTAB );
88560	  assert( pC->eCurType!=CURTYPE_PSEUDO || pC->nullRow );
88561	  assert( pC->eCurType!=CURTYPE_SORTER );
88562	
88563	  if( pC->cacheStatus!=p->cacheCtr ){                /*OPTIMIZATION-IF-FALSE*/
88564	    if( pC->nullRow ){
88565	      if( pC->eCurType==CURTYPE_PSEUDO ){
88566	        /* For the special case of as pseudo-cursor, the seekResult field
88567	        ** identifies the register that holds the record */
88568	        assert( pC->seekResult>0 );
88569	        pReg = &aMem[pC->seekResult];
88570	        assert( pReg->flags & MEM_Blob );
88571	        assert( memIsValid(pReg) );
88572	        pC->payloadSize = pC->szRow = pReg->n;
88573	        pC->aRow = (u8*)pReg->z;
88574	      }else{
88575	        sqlite3VdbeMemSetNull(pDest);
88576	        goto op_column_out;
88577	      }
88578	    }else{
88579	      pCrsr = pC->uc.pCursor;
88580	      assert( pC->eCurType==CURTYPE_BTREE );
88581	      assert( pCrsr );
88582	      assert( sqlite3BtreeCursorIsValid(pCrsr) );
88583	      pC->payloadSize = sqlite3BtreePayloadSize(pCrsr);
88584	      pC->aRow = sqlite3BtreePayloadFetch(pCrsr, &pC->szRow);
88585	      assert( pC->szRow<=pC->payloadSize );
88586	      assert( pC->szRow<=65536 );  /* Maximum page size is 64KiB */
88587	      if( pC->payloadSize > (u32)db->aLimit[SQLITE_LIMIT_LENGTH] ){
88588	        goto too_big;
88589	      }
88590	    }
88591	    pC->cacheStatus = p->cacheCtr;
88592	    pC->iHdrOffset = getVarint32(pC->aRow, aOffset[0]);
88593	    pC->nHdrParsed = 0;
88594	
88595	
88596	    if( pC->szRow<aOffset[0] ){      /*OPTIMIZATION-IF-FALSE*/
88597	      /* pC->aRow does not have to hold the entire row, but it does at least
88598	      ** need to cover the header of the record.  If pC->aRow does not contain
88599	      ** the complete header, then set it to zero, forcing the header to be
88600	      ** dynamically allocated. */
88601	      pC->aRow = 0;
88602	      pC->szRow = 0;
88603	
88604	      /* Make sure a corrupt database has not given us an oversize header.
88605	      ** Do this now to avoid an oversize memory allocation.
88606	      **
88607	      ** Type entries can be between 1 and 5 bytes each.  But 4 and 5 byte
88608	      ** types use so much data space that there can only be 4096 and 32 of
88609	      ** them, respectively.  So the maximum header length results from a
88610	      ** 3-byte type for each of the maximum of 32768 columns plus three
88611	      ** extra bytes for the header length itself.  32768*3 + 3 = 98307.
88612	      */
88613	      if( aOffset[0] > 98307 || aOffset[0] > pC->payloadSize ){
88614	        goto op_column_corrupt;
88615	      }
88616	    }else{
88617	      /* This is an optimization.  By skipping over the first few tests
88618	      ** (ex: pC->nHdrParsed<=p2) in the next section, we achieve a
88619	      ** measurable performance gain.
88620	      **
88621	      ** This branch is taken even if aOffset[0]==0.  Such a record is never
88622	      ** generated by SQLite, and could be considered corruption, but we
88623	      ** accept it for historical reasons.  When aOffset[0]==0, the code this
88624	      ** branch jumps to reads past the end of the record, but never more
88625	      ** than a few bytes.  Even if the record occurs at the end of the page
88626	      ** content area, the "page header" comes after the page content and so
88627	      ** this overread is harmless.  Similar overreads can occur for a corrupt
88628	      ** database file.
88629	      */
88630	      zData = pC->aRow;
88631	      assert( pC->nHdrParsed<=p2 );         /* Conditional skipped */
88632	      testcase( aOffset[0]==0 );
88633	      goto op_column_read_header;
88634	    }
88635	  }
88636	
88637	  /* Make sure at least the first p2+1 entries of the header have been
88638	  ** parsed and valid information is in aOffset[] and pC->aType[].
88639	  */
88640	  if( pC->nHdrParsed<=p2 ){
88641	    /* If there is more header available for parsing in the record, try
88642	    ** to extract additional fields up through the p2+1-th field
88643	    */
88644	    if( pC->iHdrOffset<aOffset[0] ){
88645	      /* Make sure zData points to enough of the record to cover the header. */
88646	      if( pC->aRow==0 ){
88647	        memset(&sMem, 0, sizeof(sMem));
88648	        rc = sqlite3VdbeMemFromBtreeZeroOffset(pC->uc.pCursor,aOffset[0],&sMem);
88649	        if( rc!=SQLITE_OK ) goto abort_due_to_error;
88650	        zData = (u8*)sMem.z;
88651	      }else{
88652	        zData = pC->aRow;
88653	      }
88654	
88655	      /* Fill in pC->aType[i] and aOffset[i] values through the p2-th field. */
88656	    op_column_read_header:
88657	      i = pC->nHdrParsed;
88658	      offset64 = aOffset[i];
88659	      zHdr = zData + pC->iHdrOffset;
88660	      zEndHdr = zData + aOffset[0];
88661	      testcase( zHdr>=zEndHdr );
88662	      do{
88663	        if( (pC->aType[i] = t = zHdr[0])<0x80 ){
88664	          zHdr++;
88665	          offset64 += sqlite3VdbeOneByteSerialTypeLen(t);
88666	        }else{
88667	          zHdr += sqlite3GetVarint32(zHdr, &t);
88668	          pC->aType[i] = t;
88669	          offset64 += sqlite3VdbeSerialTypeLen(t);
88670	        }
88671	        aOffset[++i] = (u32)(offset64 & 0xffffffff);
88672	      }while( (u32)i<=p2 && zHdr<zEndHdr );
88673	
88674	      /* The record is corrupt if any of the following are true:
88675	      ** (1) the bytes of the header extend past the declared header size
88676	      ** (2) the entire header was used but not all data was used
88677	      ** (3) the end of the data extends beyond the end of the record.
88678	      */
88679	      if( (zHdr>=zEndHdr && (zHdr>zEndHdr || offset64!=pC->payloadSize))
88680	       || (offset64 > pC->payloadSize)
88681	      ){
88682	        if( aOffset[0]==0 ){
88683	          i = 0;
88684	          zHdr = zEndHdr;
88685	        }else{
88686	          if( pC->aRow==0 ) sqlite3VdbeMemRelease(&sMem);
88687	          goto op_column_corrupt;
88688	        }
88689	      }
88690	
88691	      pC->nHdrParsed = i;
88692	      pC->iHdrOffset = (u32)(zHdr - zData);
88693	      if( pC->aRow==0 ) sqlite3VdbeMemRelease(&sMem);
88694	    }else{
88695	      t = 0;
88696	    }
88697	
88698	    /* If after trying to extract new entries from the header, nHdrParsed is
88699	    ** still not up to p2, that means that the record has fewer than p2
88700	    ** columns.  So the result will be either the default value or a NULL.
88701	    */
88702	    if( pC->nHdrParsed<=p2 ){
88703	      if( pOp->p4type==P4_MEM ){
88704	        sqlite3VdbeMemShallowCopy(pDest, pOp->p4.pMem, MEM_Static);
88705	      }else{
88706	        sqlite3VdbeMemSetNull(pDest);
88707	      }
88708	      goto op_column_out;
88709	    }
88710	  }else{
88711	    t = pC->aType[p2];
88712	  }
88713	
88714	  /* Extract the content for the p2+1-th column.  Control can only
88715	  ** reach this point if aOffset[p2], aOffset[p2+1], and pC->aType[p2] are
88716	  ** all valid.
88717	  */
88718	  assert( p2<pC->nHdrParsed );
88719	  assert( rc==SQLITE_OK );
88720	  assert( sqlite3VdbeCheckMemInvariants(pDest) );
88721	  if( VdbeMemDynamic(pDest) ){
88722	    sqlite3VdbeMemSetNull(pDest);
88723	  }
88724	  assert( t==pC->aType[p2] );
88725	  if( pC->szRow>=aOffset[p2+1] ){
88726	    /* This is the common case where the desired content fits on the original
88727	    ** page - where the content is not on an overflow page */
88728	    zData = pC->aRow + aOffset[p2];
88729	    if( t<12 ){
88730	      sqlite3VdbeSerialGet(zData, t, pDest);
88731	    }else{
88732	      /* If the column value is a string, we need a persistent value, not
88733	      ** a MEM_Ephem value.  This branch is a fast short-cut that is equivalent
88734	      ** to calling sqlite3VdbeSerialGet() and sqlite3VdbeDeephemeralize().
88735	      */
88736	      static const u16 aFlag[] = { MEM_Blob, MEM_Str|MEM_Term };
88737	      pDest->n = len = (t-12)/2;
88738	      pDest->enc = encoding;
88739	      if( pDest->szMalloc < len+2 ){
88740	        pDest->flags = MEM_Null;
88741	        if( sqlite3VdbeMemGrow(pDest, len+2, 0) ) goto no_mem;
88742	      }else{
88743	        pDest->z = pDest->zMalloc;
88744	      }
88745	      memcpy(pDest->z, zData, len);
88746	      pDest->z[len] = 0;
88747	      pDest->z[len+1] = 0;
88748	      pDest->flags = aFlag[t&1];
88749	    }
88750	  }else{
88751	    pDest->enc = encoding;
88752	    /* This branch happens only when content is on overflow pages */
88753	    if( ((pOp->p5 & (OPFLAG_LENGTHARG|OPFLAG_TYPEOFARG))!=0
88754	          && ((t>=12 && (t&1)==0) || (pOp->p5 & OPFLAG_TYPEOFARG)!=0))
88755	     || (len = sqlite3VdbeSerialTypeLen(t))==0
88756	    ){
88757	      /* Content is irrelevant for
88758	      **    1. the typeof() function,
88759	      **    2. the length(X) function if X is a blob, and
88760	      **    3. if the content length is zero.
88761	      ** So we might as well use bogus content rather than reading
88762	      ** content from disk.
88763	      **
88764	      ** Although sqlite3VdbeSerialGet() may read at most 8 bytes from the
88765	      ** buffer passed to it, debugging function VdbeMemPrettyPrint() may
88766	      ** read more.  Use the global constant sqlite3CtypeMap[] as the array,
88767	      ** as that array is 256 bytes long (plenty for VdbeMemPrettyPrint())
88768	      ** and it begins with a bunch of zeros.
88769	      */
88770	      sqlite3VdbeSerialGet((u8*)sqlite3CtypeMap, t, pDest);
88771	    }else{
88772	      rc = sqlite3VdbeMemFromBtree(pC->uc.pCursor, aOffset[p2], len, pDest);
88773	      if( rc!=SQLITE_OK ) goto abort_due_to_error;
88774	      sqlite3VdbeSerialGet((const u8*)pDest->z, t, pDest);
88775	      pDest->flags &= ~MEM_Ephem;
88776	    }
88777	  }
88778	
88779	op_column_out:
88780	  UPDATE_MAX_BLOBSIZE(pDest);
88781	  REGISTER_TRACE(pOp->p3, pDest);
88782	  break;
88783	
88784	op_column_corrupt:
88785	  if( aOp[0].p3>0 ){
88786	    pOp = &aOp[aOp[0].p3-1];
88787	    break;
88788	  }else{
88789	    rc = SQLITE_CORRUPT_BKPT;
88790	    goto abort_due_to_error;
88791	  }
88792	}
88793	
```
