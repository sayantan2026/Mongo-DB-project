# COBOL Code Analysis & Data Structure

## 1. Data Structure Analysis
The system appears to be a Banking/Inventory application (likely Alnova framework) structured around **Entities**, **Branches (Centers)**, and **Accounts**.

*   **Table Prefixes**:
    *   `BGDT...` (Business Global Data Table): Functional data like Accounts, Deposits, Requests.
    *   `TCDV...` / `TCDT...` (Technical Common Data Table/View): Reference data like Branches, Holidays, System Parameters.

## 2. Identified Joins (Implicit & Explicit)
Most "joins" in this COBOL code are **implicit**, handled by the application logic using common keys across tables rather than single SQL `JOIN` statements. The primary keys linking these tables are:
*   **Entity** (`COD_ENTITY`, `Txxx_ENT`)
*   **Branch/Center** (`COD_BRANCH`, `CEN_REG`, `Txxx_CEN_REG`)
*   **Account Number** (`Txxx_ACC`)

## 3. Entity Relationship Diagram (ERD)
The diagram below illustrates the relationships.
*   **`TCDV0500` (Branch)** is the master table for organizational units.
*   **`BGDT041` (Account)** is the central master table for customer products, linked to the Branch.
*   **`BGDT033` (Term Deposit)**, **`BGDT077` (Pre-notification)**, and **`BGDT003` (Open Request)** are child tables of the Account.

```mermaid
erDiagram
    TCDV0500 ||--|{ BGDT041 : "Manages Accounts (PK: ENT, BRANCH)"
    TCDV0500 ||--|{ TCDV0220 : "Has Holiday Exceptions"
    TCDV0500 ||--|{ TCDT071 : "Manages Contracts"

    BGDT041 ||--|{ BGDT033 : "Has Term Deposits (PK: ACC, SEQ)"
    BGDT041 ||--|{ BGDT003 : "Opening Requests"
    BGDT041 ||--|{ BGDT077 : "Withdrawal Pre-notifs"

    TCDV0500 {
        string COD_ENTITY PK "Bank Entity"
        string COD_BRANCH PK "Branch Code"
        string DES_BRANCH "Branch Name"
        string COD_CITY
        string COD_REGION
    }

    BGDT041 {
        string T041_ENT PK "FK to Entity"
        string T041_CEN_REG PK "FK to Branch"
        string T041_ACC PK "Account Number"
        string T041_COD_PRODUCT "Product Code"
        string T041_FLG_BLOCKED "Block Status"
    }

    BGDT033 {
        string T033_ENT PK
        string T033_CEN_REG PK
        string T033_ACC PK
        decimal T033_SEQUENCE PK "Deposit Sequence"
        string T033_COD_SPROD "Sub-Product"
        decimal T033_PER_INCAPP "Interest Rate"
    }

    BGDT003 {
        string T003_ENT PK
        string T003_CEN_REG PK
        string T003_ACC PK
        string T003_PURPOSE "Opening Purpose"
        string T003_ORIFUNDS "Origin of Funds"
    }

    BGDT077 {
        string T077_ENT PK
        string T077_CEN_REG PK
        string T077_ACC PK
        decimal T077_SEQUENCE PK
        decimal T077_AMT "Amount"
        date T077_DAT_MAT "Maturity Date"
    }

    TCDV0200 {
        date DAT_HOLIDAY PK
        string TYP_HOLIDAY PK
        string COD_REGION PK
        string COD_CITY PK
        string COD_ENTITY PK
        string DES_HOLIDAY
    }

    TCDV0220 {
        date DAT_HOLIDAY PK
        decimal COD_ENTITY PK
        decimal COD_BRANCH PK
        string FLG_HDYEXP "Holiday Exception Flag"
    }

    TCDT071 {
        string COD_ENTITY PK
        string NSC
        string COD_BRANCH PK
        string NUM_CONTRACT
    }
```

## 4. Table Dictionary
| Table Name | Description | Key Relationships |
| :--- | :--- | :--- |
| **TCDV0500** | **Branch / Center Table** | Master table for Branches. |
| **BGDT041** | **Master Table of Accounts** | Links to `TCDV0500` via Entity/Branch. Parent of all product tables. |
| **BGDT033** | **Term Deposits** | Child of `BGDT041`. Contains specific Term Deposit details (Interest, Renewal). |
| **BGDT003** | **Opening Request** | Stores requests to open new accounts/products. |
| **BGDT077** | **Pre-notifications** | Tracks withdrawal pre-notifications for accounts. |
| **BGDT016** | **Commissions** | Parameters for calculation of commissions. |
| **TCDV0200** | **Holidays** | General calendar of holidays by City/Region. |
| **TCDV0220** | **Holiday Exceptions** | Branch-specific overrides to the general holiday calendar. |
