# BG4C5170 Batch Analysis: Term Deposit Maturity Report

This document outlines the data flow, entity relationships, and processing logic for the batch program `BG4C5170`, which generates the listing of term deposits maturing in the following month.

## 1. System Data Flow Diagram
This flowchart illustrates the high-level data movement, from input files through the synchronization process and enrichment to the final report output.

```mermaid
flowchart TD
    %% Define Styles
    classDef file fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef db fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef process fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef subroutine fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;

    %% Input Files
    I1[("I1DQ0720<br>Account Master (Sorted)")]:::file
    I2[("I2DQ0330<br>Deposits Master (Sorted)")]:::file

    %% Main Process
    subgraph PROCESS ["BG4C5170 Main Process"]
        MERGE{"Merge Logic<br>(Sync by CAC/Account)"}:::process
        VALIDATE["Validate Maturity<br>& Calculate Dates"]:::process
        ENRICH["Enrich Data"]:::process
        CALC["Calculate Interest<br>& Totals"]:::process
    end

    %% Database Interactions
    subgraph DB ["DB2 Tables"]
        BGDT041[("BGDT041<br>Account Master")]:::db
        BGDT033[("BGDT033<br>Term Deposits")]:::db
        BGDT013[("BGDT013<br>System Dates")]:::db
    end

    %% Subroutines
    subgraph CALLS ["Subroutines"]
        BG8C5630[["BG8C5630<br>Get Customer Name"]]:::subroutine
        PE9C1500[["PE9C1500<br>Get Address"]]:::subroutine
        TC9C1200[["TC9C1200<br>Date Calc"]]:::subroutine
        TC9C1710[["TC9C1710<br>Acct Conversion"]]:::subroutine
    end

    %% Output
    REPORT[("BGLS5170<br>Maturity Report")]:::file

    %% Connections
    I1 --> MERGE
    I2 --> MERGE
    MERGE -->|Match Found| VALIDATE
    VALIDATE -->|Qualified| ENRICH
    
    ENRICH --> BGDT041
    ENRICH --> BGDT033
    ENRICH --> CALLS

    BG8C5630 -.->|Customer Info| ENRICH
    PE9C1500 -.->|Address| ENRICH
    TC9C1710 -.->|NSC Format| ENRICH
    
    ENRICH --> CALC
    CALC --> REPORT

    %% Link System Dates
    PGM_START(Start) --> BGDT013
    BGDT013 --> VALIDATE
```

## 2. Entity Relationship & Joins Diagram (ERD)
This diagram details the specific technical relationships between the input files (acting as driving tables) and the DB2 tables/Subroutines accessed during the batch execution. Connections represent the specific "joins" or lookups performed.

```mermaid
erDiagram
    %% DRIVING FILES (Merge Join)
    FILE_DEPOSITS_I2DQ0330 ||--|| FILE_ACCOUNTS_I1DQ0720 : "I. Implicit Merge Join (Key: CAC)"
    
    %% DB2 LOOKUPS (Explicit Joins)
    FILE_DEPOSITS_I2DQ0330 ||--|| DB_BGDT033_TERM_DEPOSITS : "II. Explicit SQL Join (Keys: ENT, CEN, ACC, SEQ)"
    FILE_ACCOUNTS_I1DQ0720 ||--|| DB_BGDT041_ACCOUNT_MASTER : "III. Explicit SQL Join (Keys: ENT, CEN, ACC)"
    
    %% SUBROUTINE LOOKUPS (Functional Joins)
    FILE_DEPOSITS_I2DQ0330 ||--|| SUB_BG8C5630_CUSTOMER : "IV. Call for Holder Name"
    FILE_DEPOSITS_I2DQ0330 ||--|| SUB_PE9C1500_ADDRESS : "V. Call for Address"
    FILE_DEPOSITS_I2DQ0330 ||--|| SUB_TC9C1200_CALENDAR : "VI. Call for Working Days"
    
    FILE_DEPOSITS_I2DQ0330 {
        string SOURCE "Flat File: I2DQ0330"
        string V033_CAC "Join Key (Contract)"
        string V033_ENT "Entity"
        string V033_CEN_REG "Branch"
        string V033_ACC "Account Num"
        string V033_SEQUENCE "Seq Num"
        date V033_DAT_NEXTMAT "Maturity Date"
        decimal V033_BALANCE "Balance"
        decimal V033_IRT "Interest Rate"
    }

    FILE_ACCOUNTS_I1DQ0720 {
        string SOURCE "Flat File: I1DQ0720"
        string F074_CAC "Join Key (Contract)"
        string F074_ENT "Entity"
        string F074_CEN_REG "Branch"
        string F074_ACC "Account Num"
        string F074_COD_SUBPRODUCT "Product"
    }

    DB_BGDT041_ACCOUNT_MASTER {
        string SOURCE "DB2 Table"
        string T041_ENT "PK"
        string T041_CEN_REG "PK"
        string T041_ACC "PK"
        string T041_MNGSCT "Retrieved: Sector"
    }

    DB_BGDT033_TERM_DEPOSITS {
        string SOURCE "DB2 Table"
        string T033_ENT "PK"
        string T033_CEN_REG "PK"
        string T033_ACC "PK"
        string T033_SEQUENCE "PK"
        string T033_TCR "Retrieved: Tax Rate"
    }

    SUB_BG8C5630_CUSTOMER {
        string LOGIC "Retrieve Customer Name"
        string INPUT "Ent, Branch, Acc"
        string OUTPUT "Holder Name"
    }
```

## 3. Processing Sequence Diagram
This diagram shows the step-by-step logical execution for a single matching record.

```mermaid
sequenceDiagram
    participant BATCH as BG4C5170 (Batch)
    participant FILE1 as I2DQ0330 (Deposits)
    participant FILE2 as I1DQ0720 (Accounts)
    participant DB as DB2 (BGDT041/BGDT033)
    participant SUB as Subroutines

    Note over BATCH: Initialization
    BATCH->>SUB: Call BG9C5710 (Get System Dates)
    
    loop For Each Deposit Record
        BATCH->>FILE1: Read Deposit Record
        BATCH->>FILE2: Read Account Record (Sync by CAC)
        
        opt If CAC Match Found
            BATCH->>BATCH: Validate Maturity Date (Next Month?)
            
            alt Is Maturing?
                BATCH->>SUB: Call TC9C1200 (Calc Working Days)
                BATCH->>SUB: Call BG8C5630 (Get Holder Name)
                BATCH->>SUB: Call PE9C1500 (Get Address)
                
                BATCH->>DB: SELECT MNGSCT FROM BGDT041
                BATCH->>DB: SELECT TAX_RATE FROM BGDT033
                
                BATCH->>BATCH: Calculate Interest & Net Financial Position
                
                BATCH->>BATCH: Write to BGLS5170 Report
            end
        end
    end
```

## 4. Detailed Join Analysis

The program utilizes a combination of file-based merge joins and explicit SQL database joins.

### A. Implicit File Join (Merge Join)
The core processing loop relies on a synchronized read of two sorted input files.
*   **Source 1:** `DEPOSITS-FILE` (`I2DQ0330`) - Driven by paragraph `13000-READ-DEPOSIT`.
*   **Source 2:** `ACC-IFCTDA-FILE` (`I1DQ0720`) - Driven by paragraph `14000-SELECT-RECORD`.
*   **Join Condition:** `I2DQ0330.CAC = I1DQ0720.CAC` (Where CAC is the Contract/Account Code).
*   **Logic:** The system iterates through the Deposit file and acts as a driver, advancing the Account file pointer until the CAC keys match.

### B. Explicit DB2 Table Joins
When a file record is processed, specific data elements are retrieved from DB2 using explicit `SELECT` statements (Nested Loop Join pattern).

| Target Table | Logic Paragraph | Join Keys (WHERE Clause) | Purpose |
| :--- | :--- | :--- | :--- |
| **BGDT041** (Account Master) | `21150-ACCESS-MNGSCT` | `ENT`, `CEN-REG`, `ACC` | Retrieves the Management Sector (`MNGSCT`) for the account. |
| **BGDT033** (Term Deposits) | `21160-SELECT-TAX` | `ENT`, `CEN-REG`, `ACC`, `SEQUENCE` | Retrieves the Tax Rate (`TCR`) specific to the deposit sequence. |

### C. Functional / Subroutine Joins
External subroutines are called to "join" to other functional areas (Customer, Address, etc.) without direct table access in this program.

| Subroutine | Logical Entity | Join/Input Keys | Returns |
| :--- | :--- | :--- | :--- |
| **BG8C5630** | Customer/Person | `ENT`, `CEN`, `ACC` | Account Holder Name (resolved for Individual vs Legal Entity). |
| **PE9C1500** | Address | `ENT`, `CEN`, `PROD`, `ACC` | Physical Address for the account. |
| **TC9C1710** | System Keys | `ENT`, `CEN`, `ACC` | NSC (National Sort Code) mapping. |
| **TC9C1200** | Calendar | `Date`, `Branch` | Working day calculations (Holiday checks). |
| **BG9C5710** | System Config | `Entity` | System closing dates (via `BGDT013`). |

