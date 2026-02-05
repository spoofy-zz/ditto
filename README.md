# DITTO - VSAM Dataset Browser for KICKS/MVS 3.8J

A full-screen 3270 VSAM KSDS dataset browser running under KICKS (CICS-compatible transaction processor) on MVS 3.8J Turnkey 5.

## Features

- Browse any VSAM KSDS dataset by entering its name at runtime
- Dynamic dataset allocation via SVC 99 (no pre-allocation required)
- Automatic retrieval of VSAM attributes (LRECL, key position, key length)
- Forward/backward record navigation (PF8/PF7)
- Position to a specific key value
- 16-line record display with column ruler
- Info line showing LRECL, key position, key length, and record number

## Architecture

```
DITTO (COBOL 68/74)          DITTOIO (Assembler)
+------------------+         +------------------+
| Pseudo-conv CICS |         | SVC 99 DYNALLOC  |
| BMS map I/O      |---CALL--| VSAM ACB/RPL I/O |
| Screen logic     |         | OPEN/POINT/GET   |
| COMMAREA state   |         | SHOWCB attributes|
+------------------+         +------------------+
        |
   DITTOM (BMS)
+------------------+
| 3270 map layout  |
| 24x80 model 2    |
+------------------+
```

VSAM file I/O is handled entirely in the assembler helper (DITTOIO), bypassing KICKS file control. This avoids the FCT DISABLED issue that occurs when DD names are dynamically allocated after KICKS startup.

## Components

| File | Type | Description |
|------|------|-------------|
| `src/cobol/DITTO` | COBOL + JCL | Main program with build JCL wrapper |
| `src/asm/DITTOIO` | Assembler | VSAM I/O and dynamic allocation helper |
| `src/bms/DITTOM` | BMS map | 3270 screen layout definition |

## Prerequisites

- MVS 3.8J Turnkey 5 (TK5)
- KICKS V1R5M0 installed
- IFOX00 assembler (Assembler F)
- IKFCBL00 compiler (OS/VS COBOL)
- PDS: `RVEZ001.KICKS.DITTO` (FB/80) with members DITTO, DITTOM, DITTOIO

## Build

The `src/cobol/DITTO` file is a self-contained build job. It includes the JCL, inline COBOL source, and linker control statements.

**Build steps performed by the job:**

1. Assemble BMS map (`DITTOM`) into physical map object
2. Assemble DITTOIO helper into object module
3. KICKS preprocessor translates EXEC CICS to CALL statements
4. OS/VS COBOL compilation
5. Link-edit: COBOL object + DITTOIO + BMS map + KIKCOBGL

**Output:** `KICKS.KICKSSYS.V1R5M0.KIKRPL(DITTO)`

### Steps

1. Upload sources to `RVEZ001.KICKS.DITTO`:
   - Member `DITTOM` from `src/bms/DITTOM`
   - Member `DITTOIO` from `src/asm/DITTOIO`
2. Submit `src/cobol/DITTO` as a job
3. Verify all steps complete with RC=0

## KICKS Installation

After a successful build, add these definitions to KICKS:

| Table | Entry | Description |
|-------|-------|-------------|
| PCT | `DTTO,DITTO` | Transaction code to program |
| PPT | `DITTO,COBOL` | Program definition |

No FCT entry is needed. VSAM I/O is handled directly by the assembler helper.

## Usage

1. Start KICKS under TSO
2. Enter transaction `DTTO`
3. Type a VSAM KSDS dataset name in the `DATASET` field and press Enter
4. Browse records:
   - **PF8** - Next record
   - **PF7** - Previous record
   - **Enter** - Next record (when file is open)
   - **KEY field** - Type a key value and press Enter to position
   - **PF3** - Exit

## Screen Layout

```
DITTO   VSAM DATASET BROWSER                                          V1.0
DATASET ===> SYS1.UCAT.TSO
KEY     ===>
----+----1----+----2----+----3----+----4----+----5----+----6----+---
(record data line 1)
(record data line 2)
...
(record data line 16)
----+----1----+----2----+----3----+----4----+----5----+----6----+---
LRECL=  100  KEYPOS=    0  KEYLEN= 44  REC#        1
PF3=EXIT  PF7=BACK  PF8=FWD  ENTER=OPEN/NEXT
FIRST RECORD DISPLAYED
```

## Technical Notes

- **COBOL dialect:** OS/VS COBOL (1968/1974 standard) - no END-IF, no inline PERFORM, no EVALUATE, no reference modification
- **Pseudo-conversational:** State preserved via COMMAREA across EXEC CICS RETURN TRANSID
- **VSAM I/O strategy:** Each read operation opens the ACB, positions via POINT (GTEQ), reads via GET, and closes. This is necessary because KICKS cannot keep files open across pseudo-conversational iterations.
- **Dynamic allocation:** SVC 99 allocates the dataset to DD BROWFIL at runtime with DISP=SHR
- **Max record size:** 4096 bytes displayed as 16 lines of 68 characters
