# Microsoft Dynamics CRM On-Premises: ETL Pipeline Direct Base Table Queries & ERD

When building an **ETL pipeline** (e.g., Azure Data Factory, SSIS, Informatica, Databricks, or custom Python/SQL pipelines) directly against a Microsoft Dynamics CRM On-Premises SQL Server database, **you should NOT use Filtered Views** (`FilteredOpportunity`).

---

## 1. GUID vs String Representation vs Display Names Explained

When querying CRM Base Tables (`OpportunityBase`, `AccountBase`, `SystemUserBase`, etc.), each lookup field returns **3 distinct types of values**:

1. **Raw GUID (`uniqueidentifier` SQL Data Type):**
   - The native SQL Server 128-bit primary/foreign key column (e.g., `o.CustomerId`, `o.OwnerId`).
   - In T-SQL `SELECT o.CustomerId`, SQL Server returns it as a binary `uniqueidentifier`.
2. **String Representation of the GUID (`NVARCHAR(36)`):**
   - Formatting the GUID as a formatted text string (e.g., `'E82A1B43-2C3B-E111-80C1-00155D010B01'`).
   - Essential if your target destination (JSON, CSV, REST API, or Data Warehouse target string column) requires string-formatted GUIDs.
   - Obtained using `CONVERT(NVARCHAR(36), o.CustomerId) AS CustomerIdString` or `LOWER(CONVERT(NVARCHAR(36), ...))`.
3. **Target Display Name Value (Human-Readable String):**
   - The actual text value stored in the joined lookup target table (e.g., Account Name `'Acme Corp'`, Contact Name `'John Doe'`, User FullName `'Jane Smith'`).
   - Obtained via `LEFT JOIN` (e.g., `COALESCE(cust_acc.Name, cust_con.FullName) AS CustomerName`).

---

## 2. Why Filtered Views are NOT Used for ETL

1. **Performance Overhead:** Filtered Views execute complex underlying subqueries and security functions (`fn_UserSharedAttributeAccess`, security checks per row). In large databases, this causes severe query latency, CPU spikes, and ETL timeouts.
2. **Context Requirement:** Filtered Views require an authenticated Windows / CRM user context. ETL service accounts (e.g. SQL service user / SQL authentication) running bulk queries directly will either return zero rows or fail.
3. **Delta Extraction / CDC:** Filtered views obscure index usage on system columns like `ModifiedOn` or `VersionNumber`, making incremental ETL extractions significantly slower.

---

## 3. Summary of 16 Lookup Attributes & Target Base Tables

| # | Lookup Attribute Name | Direct Foreign Key Column (`uniqueidentifier`) | Entity Type Code Column | Target Base Table(s) | Primary Key Column | Target Display Column (String) |
|---|----------------------|------------------------------------------------|-------------------------|----------------------|--------------------|--------------------------------|
| 1 | **Customer** (Polymorphic) | `CustomerId` | `CustomerIdType` (`1`=Account, `2`=Contact) | `dbo.AccountBase`, `dbo.ContactBase` | `AccountId` / `ContactId` | `Name` / `FullName` |
| 2 | **Owner** (Polymorphic) | `OwnerId` | `OwnerIdType` (`8`=User, `9`=Team) | `dbo.SystemUserBase`, `dbo.TeamBase` | `SystemUserId` / `TeamId` | `FullName` / `Name` |
| 3 | **Created By** | `CreatedBy` | — | `dbo.SystemUserBase` | `SystemUserId` | `FullName` |
| 4 | **Modified By** | `ModifiedBy` | — | `dbo.SystemUserBase` | `SystemUserId` | `FullName` |
| 5 | **Created By (Delegate)** | `CreatedOnBehalfBy` | — | `dbo.SystemUserBase` | `SystemUserId` | `FullName` |
| 6 | **Modified By (Delegate)** | `ModifiedOnBehalfBy` | — | `dbo.SystemUserBase` | `SystemUserId` | `FullName` |
| 7 | **Owning Business Unit** | `OwningBusinessUnit` | — | `dbo.BusinessUnitBase` | `BusinessUnitId` | `Name` |
| 8 | **Owning User** | `OwningUser` | — | `dbo.SystemUserBase` | `SystemUserId` | `FullName` |
| 9 | **Owning Team** | `OwningTeam` | — | `dbo.TeamBase` | `TeamId` | `Name` |
| 10 | **Transaction Currency** | `TransactionCurrencyId` | — | `dbo.TransactionCurrencyBase` | `TransactionCurrencyId` | `ISOCurrencyCode`, `CurrencyName` |
| 11 | **Price List** | `PriceLevelId` | — | `dbo.PriceLevelBase` | `PriceLevelId` | `Name` |
| 12 | **Source Campaign** | `CampaignId` | — | `dbo.CampaignBase` | `CampaignId` | `Name` |
| 13 | **Territory** | `TerritoryId` | — | `dbo.TerritoryBase` | `TerritoryId` | `Name` |
| 14 | **Originating Lead** | `OriginatingLeadId` | — | `dbo.LeadBase` | `LeadId` | `FullName`, `CompanyName` |
| 15 | **SLA** | `SLAId` | — | `dbo.SLABase` | `SLAId` | `Name` |
| 16 | **Parent Account / Contact** | `ParentAccountId` / `ParentContactId` | — | `dbo.AccountBase`, `dbo.ContactBase` | `AccountId` / `ContactId` | `Name` / `FullName` |

---

## 4. Entity Relationship Diagram (ERD) for Base Tables

```mermaid
erDiagram
    OpportunityBase {
        uniqueidentifier OpportunityId PK
        string Name
        uniqueidentifier CustomerId FK
        int CustomerIdType
        uniqueidentifier OwnerId FK
        int OwnerIdType
        uniqueidentifier CreatedBy FK
        uniqueidentifier ModifiedBy FK
        uniqueidentifier CreatedOnBehalfBy FK
        uniqueidentifier ModifiedOnBehalfBy FK
        uniqueidentifier OwningBusinessUnit FK
        uniqueidentifier OwningUser FK
        uniqueidentifier OwningTeam FK
        uniqueidentifier TransactionCurrencyId FK
        uniqueidentifier PriceLevelId FK
        uniqueidentifier CampaignId FK
        uniqueidentifier TerritoryId FK
        uniqueidentifier OriginatingLeadId FK
        uniqueidentifier SLAId FK
        uniqueidentifier ParentAccountId FK
        uniqueidentifier ParentContactId FK
        int StateCode
        int StatusCode
        datetime ModifiedOn
        bigint VersionNumber
    }

    OpportunityExtensionBase {
        uniqueidentifier OpportunityId PK_FK
    }

    AccountBase {
        uniqueidentifier AccountId PK
        string Name
    }

    ContactBase {
        uniqueidentifier ContactId PK
        string FullName
    }

    SystemUserBase {
        uniqueidentifier SystemUserId PK
        string FullName
        string DomainName
    }

    TeamBase {
        uniqueidentifier TeamId PK
        string Name
    }

    BusinessUnitBase {
        uniqueidentifier BusinessUnitId PK
        string Name
    }

    TransactionCurrencyBase {
        uniqueidentifier TransactionCurrencyId PK
        string CurrencyName
        string ISOCurrencyCode
    }

    PriceLevelBase {
        uniqueidentifier PriceLevelId PK
        string Name
    }

    CampaignBase {
        uniqueidentifier CampaignId PK
        string Name
    }

    TerritoryBase {
        uniqueidentifier TerritoryId PK
        string Name
    }

    LeadBase {
        uniqueidentifier LeadId PK
        string FullName
        string CompanyName
    }

    SLABase {
        uniqueidentifier SLAId PK
        string Name
    }

    StringMapBase {
        int ObjectTypeCode
        string AttributeName
        int AttributeValue
        string Value
        int LangId
    }

    OpportunityBase ||--o| OpportunityExtensionBase : "1:1 Extension"
    OpportunityBase }|--o| AccountBase : "CustomerId (type=1) / ParentAccountId"
    OpportunityBase }|--o| ContactBase : "CustomerId (type=2) / ParentContactId"
    OpportunityBase }|--o| SystemUserBase : "OwnerId (type=8) / CreatedBy / ModifiedBy / CreatedOnBehalfBy / ModifiedOnBehalfBy / OwningUser"
    OpportunityBase }|--o| TeamBase : "OwnerId (type=9) / OwningTeam"
    OpportunityBase }|--o| BusinessUnitBase : "OwningBusinessUnit"
    OpportunityBase }|--o| TransactionCurrencyBase : "TransactionCurrencyId"
    OpportunityBase }|--o| PriceLevelBase : "PriceLevelId"
    OpportunityBase }|--o| CampaignBase : "CampaignId"
    OpportunityBase }|--o| TerritoryBase : "TerritoryId"
    OpportunityBase }|--o| LeadBase : "OriginatingLeadId"
    OpportunityBase }|--o| SLABase : "SLAId"
    OpportunityBase }|--o| StringMapBase : "StateCode / StatusCode OptionSets (ObjectTypeCode=3000)"
```

---

## 5. Production T-SQL ETL Extraction Query

This query explicitly extracts:
1. **Raw GUIDs** (`uniqueidentifier`)
2. **String-formatted GUIDs** (`NVARCHAR(36)`)
3. **Target Display Name values** (`NVARCHAR` text labels)

```sql
SELECT
    -- Opportunity Primary Identifiers
    o.OpportunityId,                                                     -- Raw GUID (uniqueidentifier)
    CONVERT(NVARCHAR(36), o.OpportunityId) AS OpportunityIdString,       -- String GUID (NVARCHAR(36))
    o.Name AS OpportunityName,
    o.Description,
    o.EstimatedValue,
    o.EstimatedCloseDate,

    -- System Fields for Incremental / Delta ETL Extraction
    o.CreatedOn,
    o.ModifiedOn,
    o.VersionNumber,

    -- OptionSet / Picklist Translations (via StringMapBase joins)
    o.StateCode,
    sm_state.Value AS StateLabel,
    o.StatusCode,
    sm_status.Value AS StatusLabel,

    -- 1. Customer Lookup (Polymorphic: Account or Contact)
    o.CustomerId,                                                        -- Raw GUID
    CONVERT(NVARCHAR(36), o.CustomerId) AS CustomerIdString,             -- String GUID
    o.CustomerIdType,
    CASE
        WHEN o.CustomerIdType = 1 THEN 'Account'
        WHEN o.CustomerIdType = 2 THEN 'Contact'
        ELSE NULL
    END AS CustomerType,
    COALESCE(cust_acc.Name, cust_con.FullName) AS CustomerName,          -- Display Name (Human Readable String)

    -- 2. Owner Lookup (Polymorphic: SystemUser or Team)
    o.OwnerId,                                                           -- Raw GUID
    CONVERT(NVARCHAR(36), o.OwnerId) AS OwnerIdString,                   -- String GUID
    o.OwnerIdType,
    CASE
        WHEN o.OwnerIdType = 8 THEN 'SystemUser'
        WHEN o.OwnerIdType = 9 THEN 'Team'
        ELSE NULL
    END AS OwnerType,
    COALESCE(owner_usr.FullName, owner_team.Name) AS OwnerName,          -- Display Name

    -- 3. Created By
    o.CreatedBy,                                                         -- Raw GUID
    CONVERT(NVARCHAR(36), o.CreatedBy) AS CreatedByGuidString,           -- String GUID
    cb_usr.FullName AS CreatedByName,                                    -- Display Name

    -- 4. Modified By
    o.ModifiedBy,
    CONVERT(NVARCHAR(36), o.ModifiedBy) AS ModifiedByGuidString,
    mb_usr.FullName AS ModifiedByName,

    -- 5. Created By (Delegate)
    o.CreatedOnBehalfBy,
    CONVERT(NVARCHAR(36), o.CreatedOnBehalfBy) AS CreatedOnBehalfByGuidString,
    cob_usr.FullName AS CreatedOnBehalfByName,

    -- 6. Modified By (Delegate)
    o.ModifiedOnBehalfBy,
    CONVERT(NVARCHAR(36), o.ModifiedOnBehalfBy) AS ModifiedOnBehalfByGuidString,
    mob_usr.FullName AS ModifiedOnBehalfByName,

    -- 7. Owning Business Unit
    o.OwningBusinessUnit,
    CONVERT(NVARCHAR(36), o.OwningBusinessUnit) AS OwningBusinessUnitGuidString,
    bu.Name AS OwningBusinessUnitName,

    -- 8. Owning User
    o.OwningUser,
    CONVERT(NVARCHAR(36), o.OwningUser) AS OwningUserGuidString,
    ou_usr.FullName AS OwningUserName,

    -- 9. Owning Team
    o.OwningTeam,
    CONVERT(NVARCHAR(36), o.OwningTeam) AS OwningTeamGuidString,
    ot_team.Name AS OwningTeamName,

    -- 10. Transaction Currency
    o.TransactionCurrencyId,
    CONVERT(NVARCHAR(36), o.TransactionCurrencyId) AS TransactionCurrencyGuidString,
    curr.ISOCurrencyCode AS CurrencyCode,
    curr.CurrencyName,

    -- 11. Price List
    o.PriceLevelId,
    CONVERT(NVARCHAR(36), o.PriceLevelId) AS PriceLevelGuidString,
    pl.Name AS PriceListName,

    -- 12. Source Campaign
    o.CampaignId,
    CONVERT(NVARCHAR(36), o.CampaignId) AS CampaignGuidString,
    camp.Name AS CampaignName,

    -- 13. Territory
    o.TerritoryId,
    CONVERT(NVARCHAR(36), o.TerritoryId) AS TerritoryGuidString,
    ter.Name AS TerritoryName,

    -- 14. Originating Lead
    o.OriginatingLeadId,
    CONVERT(NVARCHAR(36), o.OriginatingLeadId) AS OriginatingLeadGuidString,
    lead.FullName AS OriginatingLeadName,

    -- 15. SLA
    o.SLAId,
    CONVERT(NVARCHAR(36), o.SLAId) AS SLAGuidString,
    sla.Name AS SLAName,

    -- 16. Parent Account & Parent Contact
    o.ParentAccountId,
    CONVERT(NVARCHAR(36), o.ParentAccountId) AS ParentAccountIdGuidString,
    parent_acc.Name AS ParentAccountName,
    o.ParentContactId,
    CONVERT(NVARCHAR(36), o.ParentContactId) AS ParentContactIdGuidString,
    parent_con.FullName AS ParentContactName

FROM dbo.OpportunityBase o WITH (NOLOCK)

-- Extension Base table for custom fields (if present)
LEFT JOIN dbo.OpportunityExtensionBase ext WITH (NOLOCK)
    ON o.OpportunityId = ext.OpportunityId

-- 1. Customer Joins
LEFT JOIN dbo.AccountBase cust_acc WITH (NOLOCK)
    ON o.CustomerId = cust_acc.AccountId AND o.CustomerIdType = 1
LEFT JOIN dbo.ContactBase cust_con WITH (NOLOCK)
    ON o.CustomerId = cust_con.ContactId AND o.CustomerIdType = 2

-- 2. Owner Joins
LEFT JOIN dbo.SystemUserBase owner_usr WITH (NOLOCK)
    ON o.OwnerId = owner_usr.SystemUserId AND o.OwnerIdType = 8
LEFT JOIN dbo.TeamBase owner_team WITH (NOLOCK)
    ON o.OwnerId = owner_team.TeamId AND o.OwnerIdType = 9

-- 3. CreatedBy
LEFT JOIN dbo.SystemUserBase cb_usr WITH (NOLOCK)
    ON o.CreatedBy = cb_usr.SystemUserId

-- 4. ModifiedBy
LEFT JOIN dbo.SystemUserBase mb_usr WITH (NOLOCK)
    ON o.ModifiedBy = mb_usr.SystemUserId

-- 5. CreatedOnBehalfBy
LEFT JOIN dbo.SystemUserBase cob_usr WITH (NOLOCK)
    ON o.CreatedOnBehalfBy = cob_usr.SystemUserId

-- 6. ModifiedOnBehalfBy
LEFT JOIN dbo.SystemUserBase mob_usr WITH (NOLOCK)
    ON o.ModifiedOnBehalfBy = mob_usr.SystemUserId

-- 7. OwningBusinessUnit
LEFT JOIN dbo.BusinessUnitBase bu WITH (NOLOCK)
    ON o.OwningBusinessUnit = bu.BusinessUnitId

-- 8. OwningUser
LEFT JOIN dbo.SystemUserBase ou_usr WITH (NOLOCK)
    ON o.OwningUser = ou_usr.SystemUserId

-- 9. OwningTeam
LEFT JOIN dbo.TeamBase ot_team WITH (NOLOCK)
    ON o.OwningTeam = ot_team.TeamId

-- 10. TransactionCurrency
LEFT JOIN dbo.TransactionCurrencyBase curr WITH (NOLOCK)
    ON o.TransactionCurrencyId = curr.TransactionCurrencyId

-- 11. PriceLevel
LEFT JOIN dbo.PriceLevelBase pl WITH (NOLOCK)
    ON o.PriceLevelId = pl.PriceLevelId

-- 12. Campaign
LEFT JOIN dbo.CampaignBase camp WITH (NOLOCK)
    ON o.CampaignId = camp.CampaignId

-- 13. Territory
LEFT JOIN dbo.TerritoryBase ter WITH (NOLOCK)
    ON o.TerritoryId = ter.TerritoryId

-- 14. OriginatingLead
LEFT JOIN dbo.LeadBase lead WITH (NOLOCK)
    ON o.OriginatingLeadId = lead.LeadId

-- 15. SLA
LEFT JOIN dbo.SLABase sla WITH (NOLOCK)
    ON o.SLAId = sla.SLAId

-- 16. Parent Account & Contact
LEFT JOIN dbo.AccountBase parent_acc WITH (NOLOCK)
    ON o.ParentAccountId = parent_acc.AccountId
LEFT JOIN dbo.ContactBase parent_con WITH (NOLOCK)
    ON o.ParentContactId = parent_con.ContactId

-- StringMap Joins for OptionSet Translation (ObjectTypeCode 3000 = Opportunity, LangId 1033 = English)
LEFT JOIN dbo.StringMapBase sm_state WITH (NOLOCK)
    ON sm_state.ObjectTypeCode = 3000
   AND sm_state.AttributeName = 'statecode'
   AND sm_state.AttributeValue = o.StateCode
   AND sm_state.LangId = 1033

LEFT JOIN dbo.StringMapBase sm_status WITH (NOLOCK)
    ON sm_status.ObjectTypeCode = 3000
   AND sm_status.AttributeName = 'statuscode'
   AND sm_status.AttributeValue = o.StatusCode
   AND sm_status.LangId = 1033;
```

---

## 6. Key ETL Recommendations

1. **If storing in a Relational Data Warehouse (SQL Server, Postgres, Snowflake):**
   - Keep the raw `uniqueidentifier` GUID columns for foreign key integrity in your target schema.
2. **If exporting to JSON, CSV, Parquet, or NoSQL Document Stores:**
   - Use `CONVERT(NVARCHAR(36), ...)` or `LOWER(CONVERT(NVARCHAR(36), ...))` to store standard string-formatted GUIDs (e.g. `'e82a1b43-2c3b-e111-80c1-00155d010b01'`).
3. **For Analytics & BI Reporting:**
   - Include the joined display names (`CustomerName`, `OwnerName`, `PriceListName`, etc.) alongside the GUIDs so downstream report developers don't have to re-join the dimension tables.
