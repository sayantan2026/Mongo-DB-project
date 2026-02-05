# COBOL Join Analysis & Data Relationship Report

## 1. Executive Summary
This document analyzes the data relationships and join logic within the identified COBOL codebase. Unlike modern SQL-heavy applications that use explicit `JOIN` syntax, this system primarily utilizes **Implicit Application-Logic Joins**. The programs fetch data from a master table, validate keys, and then use those keys to open cursors or select from related tables.

## 2. Identified Join Patterns

| Join Type | Description | Example Relationships |
| :--- | :--- | :--- |
| **Master-Detail** | Retrieving detailed child records based on a validated master record. | `BGDT041` (Account) $\to$ `BGDT077` (Pre-notifs) |
| **Product Extension** | Extending a base entity with product-specific attributes. | `BGDT041` (Account) $\to$ `BGDT033` (Term Deposits) |
| **Fallback / Alternate** | Checking a secondary table if the primary lookup fails. | `BGDT041` (Active) $\to$ `BGDT003` (Pending) |
| **Override / Exception** | Checking a specific exception table to override general rules. | `TCDV0200` (Global Holidays) $\to$ `TCDV0220` (Exceptions) |
| **Conditional** | Loading additional data only if specific conditions are met. | `BGDT041` $\to$ `BGDT100` (Statement Titles) |

---

## 3. Detailed Analysis & Diagrams

### 3.1. Withdrawal Prenotifications (Master-Detail)
**Source Code:** `BG2C0950.cbl`
**Tables:** `BGDT041` (Master Accounts), `BGDT077` (Pre-notifications)

**Logic:**
The program first validates that the account exists in `BGDT041`. If valid, it opens a cursor on `BGDT077` using the Account Number, Entity, and Branch as keys to retrieve all withdrawal pre-notifications.

#### Code Logic
```cobol
       212-VALIDATE-ACCOUNT.
           EXEC SQL
               SELECT T041_ENT, ...
                 FROM BGDT041
                WHERE T041_ENT = :T041-ENT ...
           END-EXEC.
```

#### Sequence Diagram
```mermaid
sequenceDiagram
    participant PGM as BG2C0950
    participant T41 as BGDT041 (Accounts)
    participant T77 as BGDT077 (Pre-Notifs)

    PGM->>T41: SELECT keys FROM BGDT041 WHERE ENT=x AND ACC=y
    T41-->>PGM: Return Details (SQLCODE 0)
    
    alt Account Exists
        PGM->>T77: OPEN CURSOR BGDC0770 WHERE ENT=x AND ACC=y
        loop Fetch
            PGM->>T77: FETCH NEXT ROW
            T77-->>PGM: Return Status, Amount, Date
        end
        PGM->>T77: CLOSE CURSOR
    else Account Not Found
        PGM->>PGM: Signal Error "Account Not Found"
    end
```

#### Entity Relationship
```mermaid
erDiagram
    BGDT041 ||--o{ BGDT077 : "Has Pre-notifications"
    BGDT041 {
        string T041_ENT PK
        string T041_CEN_REG PK
        string T041_ACC PK
    }
    BGDT077 {
        string T077_ENT PK,FK
        string T077_CEN_REG PK,FK
        string T077_ACC PK,FK
        decimal T077_AMT
        date T077_PRENTF
    }
```

---

### 3.2. Term Deposit Interest (Product Extension)
**Source Code:** `BG4C5170.cbl`
**Tables:** `BGDT041` (Master Accounts), `BGDT033` (Term Deposits)

**Logic:**
The system processes an account. It reads `BGDT041` to get the "Management Sector" (`MNGSCT`). Then, it reads `BGDT033` (Term Deposit details) using the same account keys plus a `SEQUENCE` number to get interest rates (`IRT`) and tax rates (`TCR`).

#### Sequence Diagram
```mermaid
sequenceDiagram
    participant PGM as BG4C5170
    participant T41 as BGDT041 (Master)
    participant T33 as BGDT033 (Term Deposit)

    Note over PGM: Calculate Interest Task
    PGM->>T41: SELECT MNGSCT FROM BGDT041
    PGM->>T33: SELECT TCR, IRT FROM BGDT033 WHERE ENT=x AND SEQ=n
    T33-->>PGM: Return Rates
    PGM->>PGM: Formula: Balance * Rate * Days
```

#### Entity Relationship
```mermaid
erDiagram
    BGDT041 ||--|{ BGDT033 : "Extends as Term Deposit"
    BGDT033 {
        string T033_ACC PK,FK
        decimal T033_SEQUENCE PK
        decimal T033_IRT "Interest Rate"
        decimal T033_TCR "Tax Rate"
    }
```

---

### 3.3. Account Setup Validation (Fallback Logic)
**Source Code:** `BG9C7100.cbl`
**Tables:** `BGDT041` (Active Accounts), `BGDT003` (Opening Requests)

**Logic:**
When validating an account input, the system checks `BGDT041`. If the account is found, it proceeds. If `SQLCODE` indicates "Not Found", it immediately checks `BGDT003` to see if it is a pending opening request.

#### Sequence Diagram
```mermaid
sequenceDiagram
    participant PGM as BG9C7100
    participant T41 as BGDT041
    participant T03 as BGDT003

    PGM->>T41: SELECT Status FROM BGDT041
    alt Found
        T41-->>PGM: Return Status (Active)
    else Not Found
        PGM->>T03: SELECT Status FROM BGDT003
        alt Found (Pending)
            T03-->>PGM: Return Details
        else Not Found
            PGM->>PGM: Error "Invalid Account"
        end
    end
```

---

### 3.4. Statement Setup (Conditional Join)
**Source Code:** `BG9C7100.cbl`
**Tables:** `BGDT041`, `BGDT100` (Statement Titles)

**Logic:**
`BGDT100` stores custom titles for account statements. This join is conditional; it is only attempted after the account is validated in `BGDT041`.

#### Entity Relationship
```mermaid
erDiagram
    BGDT041 ||--o| BGDT100 : "Custom Statement Titles"
    BGDT100 {
        string T100_ACC PK,FK
        string T100_TXT_LDGTIT1
        string T100_TXT_LDGTIT2
    }
```

---

### 3.5. Holiday Calculation (Override/Exception)
**Source Code:** `TC9C1200.cbl`
**Tables:** `TCDV0200` (General Holidays), `TCDV0220` (Branch Exceptions)

**Logic:**
To determine if a specific date is a working day:
1.  Check `TCDV0200` to see if the date is a holiday for the `COD_REGION` / `COD_CITY`.
2.  Check `TCDV0220` to see if the specific `COD_BRANCH` has an exception (`FLG_HDYEXP`) for that date (e.g., a local branch holiday that isn't regional, or being open on a regional holiday).

#### Sequence Diagram
```mermaid
sequenceDiagram
    participant PGM as TC9C1200
    participant T200 as TCDV0200 (General)
    participant T220 as TCDV0220 (Exceptions)

    PGM->>T200: SELECT TYP_HOLIDAY WHERE Date=Input
    T200-->>PGM: Return Holiday Type (if any)
    
    PGM->>T220: SELECT FLG_HDYEXP WHERE Date=Input AND Branch=MyBranch
    T220-->>PGM: Return Exception Flag
    
    PGM->>PGM: Calculate Effective Holiday Status
```

#### Entity Relationship
```mermaid
erDiagram
    TCDV0200 ||--o{ TCDV0220 : "Overridden by"
    TCDV0200 {
        date DAT_HOLIDAY PK
        string COD_REGION
        string COD_CITY
    }
    TCDV0220 {
        date DAT_HOLIDAY PK
        string COD_BRANCH PK
        string FLG_HDYEXP "Exception Flag"
    }
```
