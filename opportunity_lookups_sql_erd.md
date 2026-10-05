# Microsoft Dynamics CRM On-Premises: Lookup Joining Simple Example & Full ETL Guide

---

## 1. Concrete Simple Example: Resolving a GUID to a Text Name (e.g., `'Edward'`)

Here is a minimal step-by-step example showing exactly how SQL resolves a GUID stored in a lookup column to its human-readable text string value (e.g., `'Edward'`) from a target table.

### Scenario:
Suppose your `OpportunityBase` table has a lookup field (e.g. `ParentContactId` or a custom contact POC lookup) that holds a GUID value `A1B2C3D4-1111-2222-3333-444455556666`.

#### Table 1: `dbo.OpportunityBase` (Source Table)
| OpportunityId | Name | ParentContactId (Lookup GUID) |
|---|---|---|
| `99999999-0000-0000-0000-000000000001` | Enterprise Software Deal | `A1B2C3D4-1111-2222-3333-444455556666` |

#### Table 2: `dbo.ContactBase` (Target Lookup Table)
| ContactId (Primary Key GUID) | FirstName | LastName | FullName |
|---|---|---|---|
| `A1B2C3D4-1111-2222-3333-444455556666` | **Edward** | **Smith** | **Edward Smith** |

---

### Minimal T-SQL Query:

```sql
SELECT
    o.OpportunityId,
    o.Name AS OpportunityName,
    o.ParentContactId AS ContactGuid,                     -- The GUID stored in Opportunity table
    c.FirstName AS ContactFirstName,                      -- Returns 'Edward'
    c.FullName AS ContactFullName                         -- Returns 'Edward Smith'
FROM dbo.OpportunityBase o WITH (NOLOCK)
LEFT JOIN dbo.ContactBase c WITH (NOLOCK)
    ON o.ParentContactId = c.ContactId;
```

#### Query Result Set:
| OpportunityName | ContactGuid | ContactFirstName | ContactFullName |
|---|---|---|---|
| Enterprise Software Deal | `A1B2C3D4-1111-2222-3333-444455556666` | **Edward** | **Edward Smith** |

---

## 2. GUID Types in SQL Server

When you run the query above:
- **`o.ParentContactId`** returns the raw 128-bit SQL `uniqueidentifier` (`A1B2C3D4-1111-2222-3333-444455556666`).
- **`CONVERT(NVARCHAR(36), o.ParentContactId)`** returns the GUID formatted as a text string `'A1B2C3D4-1111-2222-3333-444455556666'`.
- **`c.FirstName` / `c.FullName`** returns the text string stored in the `ContactBase` table (`'Edward'` / `'Edward Smith'`).

---

## 3. Full Production ETL Query (All 16 Lookups)

```sql
SELECT
    -- Opportunity Fields
    o.OpportunityId,
    o.Name AS OpportunityName,
    o.EstimatedValue,

    -- 1. Contact POC / Parent Contact Lookup
    o.ParentContactId AS ContactGuid,
    c.FirstName AS ContactFirstName,                         -- 'Edward'
    c.FullName AS ContactFullName,                           -- 'Edward Smith'

    -- 2. Customer Lookup (Polymorphic: Account or Contact)
    o.CustomerId,
    o.CustomerIdType,
    COALESCE(cust_acc.Name, cust_con.FullName) AS CustomerName,

    -- 3. Owner Lookup (Polymorphic: SystemUser or Team)
    o.OwnerId,
    o.OwnerIdType,
    COALESCE(owner_usr.FullName, owner_team.Name) AS OwnerName,

    -- 4. Created By
    cb_usr.FullName AS CreatedByName,

    -- 5. Modified By
    mb_usr.FullName AS ModifiedByName,

    -- 6. Created By Delegate
    cob_usr.FullName AS CreatedOnBehalfByName,

    -- 7. Modified By Delegate
    mob_usr.FullName AS ModifiedOnBehalfByName,

    -- 8. Business Unit
    bu.Name AS OwningBusinessUnitName,

    -- 9. Owning User
    ou_usr.FullName AS OwningUserName,

    -- 10. Owning Team
    ot_team.Name AS OwningTeamName,

    -- 11. Transaction Currency
    curr.ISOCurrencyCode AS CurrencyCode,

    -- 12. Price List
    pl.Name AS PriceListName,

    -- 13. Campaign
    camp.Name AS CampaignName,

    -- 14. Territory
    ter.Name AS TerritoryName,

    -- 15. Originating Lead
    lead.FullName AS OriginatingLeadName,

    -- 16. SLA
    sla.Name AS SLAName

FROM dbo.OpportunityBase o WITH (NOLOCK)
LEFT JOIN dbo.ContactBase c WITH (NOLOCK) ON o.ParentContactId = c.ContactId
LEFT JOIN dbo.AccountBase cust_acc WITH (NOLOCK) ON o.CustomerId = cust_acc.AccountId AND o.CustomerIdType = 1
LEFT JOIN dbo.ContactBase cust_con WITH (NOLOCK) ON o.CustomerId = cust_con.ContactId AND o.CustomerIdType = 2
LEFT JOIN dbo.SystemUserBase owner_usr WITH (NOLOCK) ON o.OwnerId = owner_usr.SystemUserId AND o.OwnerIdType = 8
LEFT JOIN dbo.TeamBase owner_team WITH (NOLOCK) ON o.OwnerId = owner_team.TeamId AND o.OwnerIdType = 9
LEFT JOIN dbo.SystemUserBase cb_usr WITH (NOLOCK) ON o.CreatedBy = cb_usr.SystemUserId
LEFT JOIN dbo.SystemUserBase mb_usr WITH (NOLOCK) ON o.ModifiedBy = mb_usr.SystemUserId
LEFT JOIN dbo.SystemUserBase cob_usr WITH (NOLOCK) ON o.CreatedOnBehalfBy = cob_usr.SystemUserId
LEFT JOIN dbo.SystemUserBase mob_usr WITH (NOLOCK) ON o.ModifiedOnBehalfBy = mob_usr.SystemUserId
LEFT JOIN dbo.BusinessUnitBase bu WITH (NOLOCK) ON o.OwningBusinessUnit = bu.BusinessUnitId
LEFT JOIN dbo.SystemUserBase ou_usr WITH (NOLOCK) ON o.OwningUser = ou_usr.SystemUserId
LEFT JOIN dbo.TeamBase ot_team WITH (NOLOCK) ON o.OwningTeam = ot_team.TeamId
LEFT JOIN dbo.TransactionCurrencyBase curr WITH (NOLOCK) ON o.TransactionCurrencyId = curr.TransactionCurrencyId
LEFT JOIN dbo.PriceLevelBase pl WITH (NOLOCK) ON o.PriceLevelId = pl.PriceLevelId
LEFT JOIN dbo.CampaignBase camp WITH (NOLOCK) ON o.CampaignId = camp.CampaignId
LEFT JOIN dbo.TerritoryBase ter WITH (NOLOCK) ON o.TerritoryId = ter.TerritoryId
LEFT JOIN dbo.LeadBase lead WITH (NOLOCK) ON o.OriginatingLeadId = lead.LeadId
LEFT JOIN dbo.SLABase sla WITH (NOLOCK) ON o.SLAId = sla.SLAId;
```

---

## 4. Querying Dynamics CRM Metadata via SQL (Discovering Target Entities & Lookups Dynamically)

To programmatically inspect the CRM schema and discover what target entity and table any lookup attribute points to, Microsoft Dynamics CRM On-Premises provides system **Metadata Views** and **MetadataSchema tables** in the database.

---

### Method A: Querying Metadata System Views (`EntityView`, `AttributeView`, `RelationshipView`)

This query returns every lookup column for a given entity (e.g. `'opportunity'`), showing:
- The lookup attribute name in CRM (e.g. `parentcontactid`)
- The SQL physical column name (e.g. `ParentContactId`)
- The target entity name (e.g. `contact`)
- The target physical SQL table name (e.g. `ContactBase`)
- The target primary key column (e.g. `ContactId`)
- The target primary display column (e.g. `fullname`)

```sql
SELECT
    e_source.Name               AS SourceEntityLogicalName,
    e_source.PhysicalName       AS SourceEntityTable,
    a_source.Name               AS LookupAttributeLogicalName,
    a_source.PhysicalName       AS LookupColumnName,
    e_target.Name               AS TargetEntityLogicalName,
    e_target.PhysicalName       AS TargetEntityTable,
    a_target_pk.PhysicalName    AS TargetPrimaryKeyColumn,
    e_target.PrimaryNameAttribute AS TargetDisplayColumn
FROM dbo.RelationshipView r WITH (NOLOCK)
INNER JOIN dbo.EntityView e_source WITH (NOLOCK)
    ON r.ReferencingEntityId = e_source.EntityId
INNER JOIN dbo.AttributeView a_source WITH (NOLOCK)
    ON r.ReferencingAttributeId = a_source.AttributeId
INNER JOIN dbo.EntityView e_target WITH (NOLOCK)
    ON r.ReferencedEntityId = e_target.EntityId
INNER JOIN dbo.AttributeView a_target_pk WITH (NOLOCK)
    ON r.ReferencedAttributeId = a_target_pk.AttributeId
WHERE e_source.Name = 'opportunity' -- Change to any entity logical name (e.g. 'account', 'contact', 'lead')
ORDER BY a_source.PhysicalName;
```

---

### Method B: Querying `MetadataSchema` Tables

Alternatively, you can query CRM's internal `MetadataSchema` tables directly:

```sql
SELECT
    e_source.LogicalName           AS SourceEntity,
    a_source.PhysicalName          AS LookupColumn,
    e_target.LogicalName           AS TargetEntity,
    e_target.BaseTableName         AS TargetBaseTable,
    e_target.PrimaryIdAttribute    AS TargetPrimaryKeyColumn,
    e_target.PrimaryNameAttribute  AS TargetPrimaryDisplayColumn
FROM MetadataSchema.EntityRelationship er WITH (NOLOCK)
INNER JOIN MetadataSchema.Entity e_source WITH (NOLOCK)
    ON er.ReferencingEntityId = e_source.EntityId
INNER JOIN MetadataSchema.Attribute a_source WITH (NOLOCK)
    ON er.ReferencingAttributeId = a_source.AttributeId
INNER JOIN MetadataSchema.Entity e_target WITH (NOLOCK)
    ON er.ReferencedEntityId = e_target.EntityId
WHERE e_source.LogicalName = 'opportunity'
ORDER BY a_source.PhysicalName;
```

---

### Method C: Inspecting Polymorphic Lookup Target Entities (`CustomerId`, `OwnerId`)

Polymorphic lookups point to multiple entity types. You can inspect the valid target entities and their `ObjectTypeCode` / `EntityTypeCode` mappings using this query:

```sql
SELECT
    e_source.Name            AS SourceEntity,
    a_source.PhysicalName    AS LookupColumnName,
    e_target.Name            AS TargetEntity,
    e_target.ObjectTypeCode  AS TargetObjectTypeCode,
    e_target.PhysicalName    AS TargetTable
FROM dbo.RelationshipView r WITH (NOLOCK)
INNER JOIN dbo.EntityView e_source WITH (NOLOCK)
    ON r.ReferencingEntityId = e_source.EntityId
INNER JOIN dbo.AttributeView a_source WITH (NOLOCK)
    ON r.ReferencingAttributeId = a_source.AttributeId
INNER JOIN dbo.EntityView e_target WITH (NOLOCK)
    ON r.ReferencedEntityId = e_target.EntityId
WHERE e_source.Name = 'opportunity'
  AND a_source.PhysicalName IN ('CustomerId', 'OwnerId')
ORDER BY a_source.PhysicalName, e_target.Name;
```

#### Expected Polymorphic Metadata Output:
| SourceEntity | LookupColumnName | TargetEntity | TargetObjectTypeCode | TargetTable |
|---|---|---|---|---|
| opportunity | CustomerId | account | 1 | AccountBase |
| opportunity | CustomerId | contact | 2 | ContactBase |
| opportunity | OwnerId | systemuser | 8 | SystemUserBase |
| opportunity | OwnerId | team | 9 | TeamBase |
