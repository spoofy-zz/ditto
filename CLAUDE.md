# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DITTO is a full-screen VSAM KSDS dataset browser for KICKS (CICS-compatible TP monitor) running under TSO on MVS 3.8J. Written in COBOL 68/74 (OS/VS COBOL, IKFCBL00) with an assembler helper for VSAM dynamic allocation.

## Architecture

The application is pseudo-conversational (standard CICS pattern):
- **DITTO.cbl** - Main COBOL program handling screen I/O via BMS maps (`EXEC CICS SEND MAP / RECEIVE MAP`) and VSAM browsing via `EXEC CICS STARTBR / READNEXT / READPREV / ENDBR`. State is preserved in a COMMAREA across transactions (dataset name, current key, VSAM attributes).
- **DITTOM.bms** - BMS map definition (DFHMSD/DFHMDI/DFHMDF macros). Assembled to produce both a physical map (screen layout) and a symbolic map (COBOL copybook). The symbolic map is inlined in DITTO.cbl for portability.
- **DITTOIO.asm** - Assembler helper called via `CALL 'DITTOIO'`. Handles SVC 99 dynamic allocation (allocate any dataset to DD BROWFIL at runtime) and VSAM SHOWCB to retrieve LRECL, key position (RKP), and key length from the catalog.

## Language Constraints (COBOL 68/74)

- No END-IF, END-PERFORM, END-EVALUATE, END-READ — sentences terminated by periods
- No inline PERFORM — use `PERFORM paragraph THRU exit-paragraph`
- No EVALUATE — use IF/GO TO dispatch chains
- No reference modification (`WS-FIELD(1:5)`) — use REDEFINES or subscripts
- EXEC CICS commands are preprocessed by KICKS preprocessor (KIKCOBP) before compilation

## Build Process

All building happens on MVS 3.8J TK5 via JCL (`jcl/BUILD.jcl`). The steps are:
1. Assemble BMS map (`IFOX00` Assembler F with KICKS maclib) — produces physical map + symbolic COBOL copybook
2. Preprocess COBOL (`KIKCOBP`) — translates `EXEC CICS` to CALL statements
3. Compile COBOL (`IKFCBL00`)
4. Assemble DITTOIO helper (`IFOX00` with SYS1.MACLIB + AMODGEN)
5. Link-edit all objects into KICKS load library (`IEWL`)

Dataset names configured for:
- KICKS: `KICKS.KICKSSYS.V1R5M0.*`
- User: `RVEZ001.*`

## KICKS Installation

After building, register in KICKS tables:
- **PCT**: transaction `DTTO` → program `DITTO`
- **PPT**: program `DITTO`, language `COBOL`
- **FCT**: file `BROWFIL`, type `KSDS`, DD name `BROWFIL`

## Screen Layout (3270 Model 2, 24x80)

```
Row 1:     Title bar
Row 2:     DATASET ===> [44-char input field]
Row 3:     KEY     ===> [44-char positioning field]
Row 4:     Ruler line
Rows 5-20: Record data (16 lines x 68 chars = 1088 bytes visible)
Row 21:    Ruler line
Row 22:    LRECL=nnnnn  KEYPOS=nnnnn  KEYLEN=nnn  REC# nnnnnnnn
Row 23:    PF key legend
Row 24:    Message line
```

## Key Navigation

- **ENTER** with dataset name: dynamically allocate and open dataset, display first record
- **ENTER** with key value: position to key (GTEQ) and display matching record
- **ENTER** alone (file open): read next record
- **PF8**: next record
- **PF7**: previous record
- **PF3**: exit program

## File Structure

```
src/cobol/DITTO.cbl    Main COBOL program
src/bms/DITTOM.bms     BMS map definition
src/asm/DITTOIO.asm    Assembler helper (SVC 99 + SHOWCB)
jcl/BUILD.jcl          Build JCL for MVS 3.8J
```
