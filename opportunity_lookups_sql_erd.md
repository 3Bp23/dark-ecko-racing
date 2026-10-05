# Microsoft Dynamics CRM On-Premises: ETL Pipeline Direct Base Table Queries & ERD

When building an **ETL pipeline** (e.g., Azure Data Factory, SSIS, Informatica, Databricks, or custom Python/SQL pipelines) directly against Microsoft Dynamics CRM On-Premises SQL Server database, **you should NOT use Filtered Views** (`FilteredOpportunity`).

---

## Why Filtered Views are NOT Used for ETL

1. **Performance Overhead:** Filtered Views execute complex underlying subqueries and security functions (`fn_UserSharedAttributeAccess`, security checks per row). In large databases, this causes severe query latency, CPU spikes, and ETL timeouts.
2. **Context Requirement:** Filtered Views require an authenticated Windows / CRM user context. ETL service accounts (e.g. SQL service user / SQL authentication) running bulk queries directly will either return zero rows or fail.
3. **Delta Extraction / CDC:** Filtered views obscure index usage on system columns like `ModifiedOn` or `VersionNumber`, making incremental ETL extractions significantly slower.

---

## ETL Architecture: Direct Base Table Queries

For ETL workloads, you query the **Base Tables** directly using `WITH (NOLOCK)`.

### Key Structural Rules for CRM On-Premises Base Tables:

1. **`OpportunityBase` vs `OpportunityExtensionBase`:**
   - Standard CRM attributes reside in `OpportunityBase`.
   - Custom attributes (`new_...`, `custom_...`) reside in `OpportunityExtensionBase` (joined on `OpportunityId`).
   - *(Note: In CRM 2016 / D365 v8.2+ with Merge Table capability enabled, all attributes may exist in `OpportunityBase`).*
2. **Polymorphic Lookups (`CustomerId` & `OwnerId`):**
   - `CustomerId` links to either `AccountBase` (`CustomerIdType = 1`) or `ContactBase` (`CustomerIdType = 2`).
   - `OwnerId` links to either `SystemUserBase` (`OwnerIdType = 8`) or `TeamBase` (`OwnerIdType = 9`).
3. **Picklist / OptionSet Translation via `StringMapBase`:**
   - OptionSets (Status, State, Sales Stage, Lead Source, etc.) store integer codes in `OpportunityBase`.
   - To get human-readable display labels in ETL, join `StringMapBase` on `ObjectTypeCode = 3000` (Opportunity) and `AttributeName`.

---

## 1. Summary of 16 Lookup Attributes & Target Base Tables

| # | Lookup Attribute Name | Direct Foreign Key Column | Entity Type Code Column | Target Base Table(s) | Primary Key Column | Target Display Column |
|---|----------------------|---------------------------|-------------------------|----------------------|--------------------|-----------------------|
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

## 2. Entity Relationship Diagram (ERD) for Base Tables

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

## 3. Production T-SQL ETL Extraction Query

Use this query in your ETL pipeline (SSIS, Azure Data Factory, Python/pandas SQL alchemy, Spark SQL, etc.) to extract full opportunity records and resolved lookup values directly from the base database:

```sql
SELECT
    -- Opportunity Primary Identifiers
    o.OpportunityId,
    o.Name AS OpportunityName,
    o.Description,
    o.EstimatedValue,
    o.EstimatedCloseDate,
    o.ActualValue,
    o.ActualCloseDate,
    o.CloseProbability,

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
    o.CustomerId,
    o.CustomerIdType,
    CASE
        WHEN o.CustomerIdType = 1 THEN 'Account'
        WHEN o.CustomerIdType = 2 THEN 'Contact'
        ELSE NULL
    END AS CustomerType,
    COALESCE(cust_acc.Name, cust_con.FullName) AS CustomerName,
    cust_acc.AccountNumber AS CustomerAccountNumber,
    cust_con.EMailAddress1 AS CustomerContactEmail,

    -- 2. Owner Lookup (Polymorphic: SystemUser or Team)
    o.OwnerId,
    o.OwnerIdType,
    CASE
        WHEN o.OwnerIdType = 8 THEN 'SystemUser'
        WHEN o.OwnerIdType = 9 THEN 'Team'
        ELSE NULL
    END AS OwnerType,
    COALESCE(owner_usr.FullName, owner_team.Name) AS OwnerName,
    owner_usr.DomainName AS OwnerDomainName,

    -- 3. Created By
    o.CreatedBy,
    cb_usr.FullName AS CreatedByName,

    -- 4. Modified By
    o.ModifiedBy,
    mb_usr.FullName AS ModifiedByName,

    -- 5. Created By (Delegate)
    o.CreatedOnBehalfBy,
    cob_usr.FullName AS CreatedOnBehalfByName,

    -- 6. Modified By (Delegate)
    o.ModifiedOnBehalfBy,
    mob_usr.FullName AS ModifiedOnBehalfByName,

    -- 7. Owning Business Unit
    o.OwningBusinessUnit,
    bu.Name AS OwningBusinessUnitName,

    -- 8. Owning User
    o.OwningUser,
    ou_usr.FullName AS OwningUserName,

    -- 9. Owning Team
    o.OwningTeam,
    ot_team.Name AS OwningTeamName,

    -- 10. Transaction Currency
    o.TransactionCurrencyId,
    curr.ISOCurrencyCode AS CurrencyCode,
    curr.CurrencyName,
    curr.CurrencySymbol,
    o.ExchangeRate,

    -- 11. Price List
    o.PriceLevelId,
    pl.Name AS PriceListName,

    -- 12. Source Campaign
    o.CampaignId,
    camp.Name AS CampaignName,
    camp.CodeName AS CampaignCode,

    -- 13. Territory
    o.TerritoryId,
    ter.Name AS TerritoryName,

    -- 14. Originating Lead
    o.OriginatingLeadId,
    lead.FullName AS OriginatingLeadName,
    lead.CompanyName AS OriginatingLeadCompanyName,

    -- 15. SLA
    o.SLAId,
    sla.Name AS SLAName,

    -- 16. Parent Account & Parent Contact
    o.ParentAccountId,
    parent_acc.Name AS ParentAccountName,
    o.ParentContactId,
    parent_con.FullName AS ParentContactName

FROM dbo.OpportunityBase o WITH (NOLOCK)

-- Extension Base table for custom fields (if present in your environment)
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

## 4. Incremental / Delta ETL Query Strategy

For high-volume ETL pipelines, extract incremental changes using `ModifiedOn` or `VersionNumber` watermark tracking:

```sql
-- Delta ETL Query Example (e.g., extracting records updated since last execution)
SELECT
    o.OpportunityId,
    o.Name,
    o.ModifiedOn,
    o.VersionNumber,
    -- (Include required lookups from main query above)
    COALESCE(cust_acc.Name, cust_con.FullName) AS CustomerName,
    COALESCE(owner_usr.FullName, owner_team.Name) AS OwnerName
FROM dbo.OpportunityBase o WITH (NOLOCK)
LEFT JOIN dbo.AccountBase cust_acc WITH (NOLOCK)
    ON o.CustomerId = cust_acc.AccountId AND o.CustomerIdType = 1
LEFT JOIN dbo.ContactBase cust_con WITH (NOLOCK)
    ON o.CustomerId = cust_con.ContactId AND o.CustomerIdType = 2
LEFT JOIN dbo.SystemUserBase owner_usr WITH (NOLOCK)
    ON o.OwnerId = owner_usr.SystemUserId AND o.OwnerIdType = 8
LEFT JOIN dbo.TeamBase owner_team WITH (NOLOCK)
    ON o.OwnerId = owner_team.TeamId AND o.OwnerIdType = 9
WHERE o.ModifiedOn >= @LastETLWatermarkDateTime
   OR o.VersionNumber > @LastETLWatermarkVersionNumber;
```

---

## 5. Summary Checklist for CRM Base Table ETLs

1. **Always use `WITH (NOLOCK)`** on all base table queries to prevent database blocking in production.
2. **Handle Polymorphism explicitly** (`CustomerIdType` and `OwnerIdType` joins).
3. **Join `StringMapBase`** for OptionSet / Picklist labels using `ObjectTypeCode = 3000` (Opportunity) and your organization's `LangId` (e.g. `1033` for English).
4. **Use `VersionNumber` or `ModifiedOn`** for incremental watermark delta extractions.
