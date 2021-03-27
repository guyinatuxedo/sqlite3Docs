# Checks

## 

##  OP_Column

#### Record Size & Header Size Check

So the first check that happens is this:

```
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
```

So this is a check, that the defined size of the record is at least as big as the size of the header for the record. The `pC->szRow` is the size of the record, and `aOffset[0]` is the size of the header. For instance take this record:

```
0c	01	03	1d	01	31	35	39	33	35	37	32	38	64
```

Here `pC->szRow` is `0x0c` (size of the record), and `aOffset[0]` is `0x03` (size of the header).

In the event that the size row is less than the size of the header, a secondary check happens. If the header size is greater than `98307`, or if the header size is greater than the record size, then it marks the database as corrupt.

#### Header/Column Parsing Check

So this check occurs here. This checks multiple things, in regards to parsing the header, and the offsets to different columns. It first checks that it doesn't parse the header, passed the calculated end of the header. The second is that the max parsed column offset is not greater than the recorded size of the record. 

```
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
```

The `zEndHdr` is calculated from adding the size of the header, to the start of the record:

```
88660	      zEndHdr = zData + aOffset[0];
```

The `zHdr` starts off at the first type value of the header, and points to the current byte of the header:

```
88659	      zHdr = zData + pC->iHdrOffset;
```

#### Null Value Check

So this check is to effectively check if we've parsed up to the column that we want to. So think of the columns in terms of having an index. The first colum is the `1`-st, the second is the `2`-nd, the nth is the `n`-th. The `pC->nHdrParsed` value holds the index for the last column parsed. The `p2` value holds the index for the desired value. This checks that `pC->nHdrParsed` is greater than `p2`.

```
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
```