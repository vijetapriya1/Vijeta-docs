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

## 7. Related fixes

- `GetUserRoleForTheDomain` (`internals/app/datarepo/domain.go`): add `deleted_at IS NULL`, pick by
  priority CONTROLLER > TECH > BIZFIN > KAM instead of `userGroup[0]`.
- `DeleteSubEntitiesUserMapping` (`app/platform/datastoreops/subentity-user-mapping.go`): deletes by the
  claim domain, not the target domain.
- `FKDomainUsersMapping` bulk sync builds KAM access from vendor metadata, not user-access, so it can
  give back access that user-access removed. Either route it through user-access or skip KAMs without
  an active user-access row.
