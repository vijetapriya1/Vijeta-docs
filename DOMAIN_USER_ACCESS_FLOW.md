# Domain User Access Flow (Flipkart)

How `user-access`, `bu-kam-vendor-retailer mapping` and `vendor listing` reach the platform
domain users and sub-entities, what is missing for updates and deletes, and how a person with
both CONTROLLER and KAM access should be handled.

## 1. Current flow

```mermaid
flowchart TD
    subgraph UA["user-access (user-access.go)"]
        UAC[CreateUserAccess] --> KAMLOOKUP{UserGroup == KAM?}
        KAMLOOKUP -- yes --> KMREAD1[read KAM mapping<br/>take email + name]
        KAMLOOKUP -- no --> RESOLVE
        KMREAD1 --> RESOLVE
        UAU[UpdateUserAccess] --> RESOLVE
        RESOLVE[resolveUserAccessDomains] --> SAVEUA[(fk_user_access_metadata)]
    end

    subgraph KM["kam mapping (bu-kam-retailer-mapping.go)"]
        KMC[CreateBuKamVendorRetailerMapping] --> SAVEKM[(kam mapping table)]
        KMU[UpdateBuKamVendorRetailerMapping] --> SAVEKM
        SAVEKM -.->|no platform call| NOTHING((nothing))
    end

    subgraph VL["vendor listing (vendors.go)"]
        VLC[CreateVendorsList] --> SAVEVL[(vendors table)]
        VLU[UpdateVendorsList] --> SAVEVL
        SAVEVL --> AVU[addVendorUsersToDomain<br/>runs VendorBuMapping first]
    end

    subgraph SYNC["bulk sync (setting.go)"]
        SDU[SyncDomainAndUsers] --> FKM[FKDomainUsersMapping<br/>KAM from ListVendorMetadata.KAMEmail,<br/>not from user-access]
    end

    SAVEKM -.->|read only when user-access is saved| RESOLVE
    SAVEUA --> CFK
    AVU --> CFK
    FKM --> CFK

    CFK[CreateFKDomainUsers] --> EXISTS{user already<br/>in domain?}
    EXISTS -- no --> ADD[platform AddUsers<br/>role + sub-entities]
    EXISTS -- yes --> USE[UpdateSubEntities<br/>INSERT ... ON CONFLICT DO NOTHING<br/>role not changed,<br/>empty list skipped]
```

## 2. How `resolveUserAccessDomains` builds the access

```mermaid
flowchart TD
    START[user-access row] --> GRP{UserGroup}
    GRP -- CONTROLLER / TECH --> ALLR[every retailer domain<br/>no sub-entities]
    GRP -- BIZFIN --> ALLB[every retailer domain<br/>sub-entities = vendor x BU of its BU]
    GRP -- KAM --> KMROWS[KAM mapping rows<br/>where Kam == email]
    KMROWS --> KDOM[domains = RetailerCode of those rows]
    KMROWS --> KSUB[sub-entities = BuVendorMapping pairs<br/>vendorCode -> vendorId, BU -> buId,<br/>looked up in fk_vendor_bu_mapping]
    GRP -- VENDOR / INVALID --> REJ[rejected]
    ALLR --> DUM[DomainUserMapping per retailer]
    ALLB --> DUM
    KDOM --> DUM
    KSUB --> DUM
    DUM --> CFK[CreateFKDomainUsers]
```

## 3. How access is enforced at request time

```mermaid
flowchart LR
    REQ[request] --> MW[AuthnStoSMiddleware]
    MW --> SE[GetSubEntityIdsForTheUser<br/>rows where user_id = email<br/>and domain = current domain]
    SE --> SECHK{any rows?}
    SECHK -- no --> ALL[sees every vendor]
    SECHK -- yes --> ONLY[sees only those vendors]
    MW --> ROLE[GetUserRoleForTheDomain<br/>SELECT user_group WHERE email = ?<br/>no deleted_at filter, takes userGroup 0]
```

## 4. Gaps on update / delete

```mermaid
flowchart TD
    CH[change] --> T{what changed}
    T -- user-access BU / group / domains narrowed --> G1[old domains and sub-entities stay<br/>CreateFKDomainUsers only adds]
    T -- KAM mapping vendor/BU pair added --> G2[nothing until user-access is re-saved]
    T -- KAM mapping vendor/BU pair removed --> G3[sub-entity stays, even after re-save]
    T -- KAM mapping RetailerCode changed --> G4[old domain access stays]
    T -- KAM mapping Kam email changed --> G5[old email keeps access<br/>its user-access update fails:<br/>no KAM mapping found]
    T -- vendor login id removed --> G6[user stays in domain]
    T -- vendor code / retailer changed --> G7[old default user / old domain stays]
    T -- new vendor --> G8[KAMs mapped to its code<br/>never get its sub-entity]
    T -- user-access soft deleted --> G9[platform access untouched<br/>role query still sees the row]
```

### Why platform `RemoveUsers` cannot be reused

```mermaid
flowchart TD
    FK[flipkart service<br/>operator domain claim] --> RU[platform RemoveUsers / removeUser]
    RU --> CHK{request.DomainId ==<br/>claim domain?}
    CHK -- no, target is a retailer child domain --> FAIL[rejected:<br/>ErrInvalidDomainRequested]
    CHK -- yes --> DEL[DeleteSubEntitiesUserMapping<br/>WHERE domain = claim domain<br/>deletes from the wrong domain]
```

## 5. CONTROLLER + KAM on the same person (today)

Both rows exist because `UniqueId = BU + email + domain` and BU defaults to the user group.

```mermaid
flowchart TD
    P[person has CONTROLLER row<br/>and KAM row] --> A[KAM row synced]
    A --> B[KAM sub-entity rows inserted<br/>in the KAM's retailer domains]
    B --> C[in those domains the controller<br/>sees only the KAM's vendors]
    P --> R[role = userGroup 0<br/>random: KAM or CONTROLLER]
    C --> D{KAM row soft deleted / updated}
    D --> E[KAM sub-entity rows stay<br/>controller still restricted]
    D --> F[role query still returns<br/>the deleted KAM row]
```

## 6. Proposed flow: sync per person

```mermaid
flowchart TD
    subgraph TRIGGERS[triggers]
        T1[user-access create / update / delete] --> E1[that email]
        T2[KAM mapping create / update / delete] --> E2[mapping Kam email<br/>+ old email if Kam changed]
        T3[vendor create / update] --> E3[vendor login ids + default user<br/>+ KAMs whose BuVendorMapping has the vendor code]
    end
    E1 --> SYNC
    E2 --> SYNC
    E3 --> SYNC

    SYNC[syncUserDomainAccess email] --> LOAD[load all active user-access rows<br/>deleted_at IS NULL]
    LOAD --> NONE{any rows?}
    NONE -- no --> RMALL[RemoveFKDomainUsers<br/>from every flipkart domain]
    NONE -- yes --> CTRL{any CONTROLLER / TECH row?}
    CTRL -- yes --> WANTALL[desired = every retailer domain<br/>no sub-entities<br/>DOMAIN_OWNERS if asked<br/>KAM sub-entities ignored]
    CTRL -- no --> WANTU[desired = union of KAM + BIZFIN<br/>from resolveUserAccessDomains]
    WANTALL --> DIFF
    WANTU --> DIFF

    DIFF[diff desired vs current<br/>domain.Users + sub-entity rows] --> D1[missing domain:<br/>CreateFKDomainUsers]
    DIFF --> D2[kept domain:<br/>delete rows for email + domain,<br/>insert desired sub-entities,<br/>update role]
    DIFF --> D3[dropped domain:<br/>RemoveFKDomainUsers]

    D3 --> RFK[RemoveFKDomainUsers via fkrepo direct DB:<br/>GetUserById / PutUser,<br/>GetDomainsByIds + new PutDomain,<br/>new delete sub-entity rows by email + domain,<br/>UpdateDomainUsersInCache]
```

### Example: KAM row of a CONTROLLER soft deleted

```mermaid
sequenceDiagram
    participant API as user-access delete
    participant S as syncUserDomainAccess
    participant DB as fk_user_access_metadata
    participant P as platform domains / sub-entity mapping

    API->>DB: set deleted_at on KAM row
    API->>S: sync(email)
    S->>DB: active rows for email
    DB-->>S: CONTROLLER row only
    S->>S: desired = all domains, no sub-entities
    S->>P: delete sub-entity rows for email in KAM domains
    S->>P: keep user in every domain, role from CONTROLLER row
    Note over P: controller sees every vendor again
```

## 7. Per user group (simple version)

Two words used below:
- **Domain** = one retailer (FKI, ANR, ...). Being "in a domain" means the user can open that retailer.
- **Sub-entity** = one vendor x BU pair. If a user has **no** sub-entities in a domain, they see
  **every** vendor there. If they have some, they see **only those**.

### 7.1 CONTROLLER

**What they should get:** every retailer, every vendor. Can be domain owner.

```mermaid
flowchart TD
    C1[add CONTROLLER in user-access] --> C2[added to every retailer]
    C2 --> C3[no sub-entities<br/>= sees every vendor]
    C4[new retailer created later] --> C5[controller is NOT added to it<br/>bulk sync only adds KAM, BIZFIN, vendors]
    C6[update / delete CONTROLLER row] --> C7[nothing removed on the platform]
    C8[same person also gets a KAM row] --> C9[KAM vendors are added in KAM retailers<br/>controller now sees ONLY those vendors there]
```

**Problems today**
- Deleting the row does not remove access.
- A new retailer does not get the existing controllers.
- If they also have a KAM row, the KAM vendors limit them (see 5).
- If they were already in a domain, the "domain owner" role is never applied.

**What to do:** run the per-person sync (section 6). If any CONTROLLER row is active, the result is
"every retailer, no sub-entities", whatever other rows say. Also run it for all controllers when a
new retailer domain is created.

### 7.2 KAM

**What they should get:** only the retailers and vendor x BU pairs written in the KAM mapping
for their email.

```mermaid
flowchart TD
    K1[add KAM in user-access] --> K2[read KAM mapping rows for that email]
    K2 --> K3[retailers from the rows]
    K2 --> K4[vendor x BU pairs from the rows]
    K3 --> K5[added to those retailers with those vendors]
    K4 --> K5
    K6[change KAM mapping] --> K7[nothing happens on the platform]
    K8[re-save KAM user-access] --> K9[new things are added<br/>removed things stay]
    K10[new vendor in the KAM's mapping] --> K11[KAM does not get it<br/>until user-access is re-saved]
```

**Problems today**
- Changing the mapping does nothing until someone re-saves the user-access row.
- Even after re-saving, anything removed (vendor, BU, retailer) stays.
- If the mapping's email changes, the old email keeps access, and re-saving its user-access
  row fails with "no KAM mapping found".
- Bulk sync gives KAM access from vendor metadata, not from user-access. So it can give back
  access you removed.

**What to do:** after every KAM mapping save, run the per-person sync for that KAM email (both
emails if it changed). The sync removes what is no longer in the mapping.

### 7.3 VENDOR

Vendors are **not** in user-access. User-access rejects the VENDOR group. Vendor users come from
the vendor list: `<vendorCode>@vapusdata.com` plus every login ID.

```mermaid
flowchart TD
    V1[add vendor in vendor list] --> V2[create vendor x BU sub-entities]
    V2 --> V3[add default user + login ids<br/>to the vendor's retailer only]
    V3 --> V4[sub-entities = this vendor's BUs]
    V5[remove a login id] --> V6[that user stays in the domain]
    V7[change vendor code] --> V8[old default user stays]
    V9[change retailer] --> V10[users stay in the old retailer]
    V11[user with no user-access row] --> V12[treated as VENDOR on login]
    V13[vendor has no sub-entity yet] --> V14[users added anyway<br/>= see every vendor in that retailer]
```

**Problems today**
- Nothing is ever removed: old login IDs, the old default user, and the old retailer all stay.
- A vendor without sub-entities is added with no sub-entities, so its users see **all** vendors in
  that retailer. This is a security risk.

**What to do:** on vendor update, compare old and new login IDs, vendor code and retailer, and
remove the old users (`RemoveFKDomainUsers`). Do not add vendor users until their sub-entities
exist. After a vendor change, also run the KAM sync for KAMs whose mapping lists this vendor code.

### 7.4 BIZFIN

**What they should get:** every retailer, but only the vendors in their own BU.

```mermaid
flowchart TD
    B1[add BIZFIN in user-access] --> B2[added to every retailer]
    B2 --> B3[sub-entities = all vendors of its BU]
    B4[change BU] --> B5[new BU vendors added<br/>old BU vendors stay]
    B6[new vendor in that BU] --> B7[BIZFIN does not get it<br/>until bulk sync or re-save]
    B8[delete BIZFIN row] --> B9[access stays<br/>bulk sync even re-adds it:<br/>GetBizFinUsers has no deleted_at filter]
```

**What to do:** the same per-person sync. Also add `deleted_at IS NULL` to `GetBizFinUsers`. After a
new vendor x BU sub-entity is created, run the sync for the BIZFIN users of that BU.

### 7.5 TECH

**What they should get:** every retailer, every vendor, **read only**. Never a domain owner.

```mermaid
flowchart TD
    T1[add TECH in user-access] --> T2[added to every retailer as DOMAIN_USERS]
    T2 --> T3[no sub-entities = sees every vendor]
    T3 --> T4[writes blocked by denyTechUser<br/>only if login resolves to TECH]
    T5[same person also has another row] --> T6[login group is random<br/>may not resolve to TECH]
```

**Problems today**
- Same as CONTROLLER: nothing is removed, and a new retailer does not get them.
- Read-only works only if login picks the TECH row.

**What to do:** the same per-person sync. Fix the login query so the group is picked by priority,
not at random.

### 7.6 One person, many groups

| Rows the person has | What they should get |
|---|---|
| CONTROLLER + anything | every retailer, every vendor (CONTROLLER wins) |
| TECH + KAM / BIZFIN | every retailer, every vendor, read only (TECH wins) |
| BIZFIN + KAM | every retailer: own-BU vendors **plus** KAM vendors in the KAM retailers |
| KAM only | KAM retailers, KAM vendors only |
| one row deleted | re-run the sync on the rows left, e.g. KAM deleted from CONTROLLER + KAM gives full access back |
| all rows deleted | removed from every retailer |

Rule: **access is worked out from all active rows of one person together, never from one row alone.**

## 8. Related fixes

- `GetUserRoleForTheDomain` (`internals/app/datarepo/domain.go`): add `deleted_at IS NULL`, pick by
  priority CONTROLLER > TECH > BIZFIN > KAM instead of `userGroup[0]`.
- `DeleteSubEntitiesUserMapping` (`app/platform/datastoreops/subentity-user-mapping.go`): deletes by the
  claim domain, not the target domain.
- `FKDomainUsersMapping` bulk sync builds KAM access from vendor metadata, not user-access, so it can
  give back access that user-access removed. Either route it through user-access or skip KAMs without
  an active user-access row.
