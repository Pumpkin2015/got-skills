---
name: got-common-pitfalls
description: >
  Common pitfalls and patterns for GOT ERP secondary development.
  Covers entity copy/insert (castToMap, init, doInsert), transaction
  management, framework quirks, and field validation.
  Use when insert() fails with ttsrollback, copying records, or
  hitting framework-level validation errors.
  Trigger phrases: "GOT坑", "insert失败", "ttsrollback", "复制记录",
  "doInsert", "castToMap", "EnableTree", "生产组织树".
---

# GOT Common Pitfalls

Practical solutions for recurring issues in GOT ERP secondary development.

## 1. Entity Copy: init() THEN cast()

When copying an entity programmatically, order matters.
init() resets ALL fields to defaults. If you cast() first then init(),
all copied values are silently lost.

```java
// WRONG - init() wipes everything cast() just copied:
newMst.cast(source.castToMap());
newMst.init();  // All fields become null/default

// CORRECT - init() first, then cast() overwrites defaults:
Map<String, Object> data = source.castToMap();
data.remove("recId");               // lowerCamel!
data.put("masterScheduleId", "");   // Clear business PK
newMst.init();
newMst.cast(data);
newMst.setMasterScheduleGroupId(source.getMasterScheduleGroupId());
newMst.initGroup();
```

## 2. castToMap() Uses lowerCamel Keys

castToMap() returns keys in lowerCamel case. PascalCase silently fails.

| Wrong (no effect) | Correct |
|---|---|
| MasterScheduleId | masterScheduleId |
| RecId | recId |
| ProductionOrganizationId | productionOrganizationId |
| YearId | yearId |

## 3. doInsert() vs insert()

insert() triggers full chain: validate -> generateId -> SQL INSERT
-> post-insert hooks (e.g. data-sync HTTP). Switch to doInsert()
to bypass hook failures (e.g. 127.0.0.1:10108 connection refused).

doInsert() skips generateXxxId()/setInventoryRatio()/validation.
Must replicate essential logic manually:

```java
newMst.setInventoryRatio(BigDecimal.ZERO);
newMst.setSurplusInQty(newMst.getQty());
if (StringHelper.isBlank(newMst.getMasterScheduleId())) {
    MasterScheduleGroup grp = newMst.group(MasterScheduleGroup.class);
    if (grp != null && StringHelper.isNotBlank(grp.getMasterScheduleSeqId())) {
        newMst.setMasterScheduleId(GongqiRecordHelper.fetchNextNumSeq(grp.getMasterScheduleSeqId(), newMst));
    }
}
newMst.doInsert();
```

## 4. ProductionOrganizationId When EnableTree=1

MasterScheduleParameters.EnableTree=1 triggers validation requiring
ProductionOrganizationId non-empty on insert. Set default if blank:

```java
if (StringHelper.isBlank(entity.getProductionOrganizationId())) {
    entity.setProductionOrganizationId("W1");
}
```

## 5. Transaction Management

- Framework auto-ttsrollback on insert() failure. Do not call it again.
- Programmatic insert() needs explicit ttsbegin/ttscommit.
- ttsabort() does not exist, only ttsrollback().

```java
GongqiSession.ttsbegin();
try {
    entity.doInsert();
    GongqiSession.ttscommit();
} catch (ERPException e) {
    // Framework already rolled back
    result.alert("Failed: " + e.getMessage());
} catch (Exception e) {
    try { GongqiSession.ttsrollback(); } catch (Exception e2) {}
    result.alert("Failed: " + e.getMessage());
}
```

## 6. SQL Table Names: $ vs _

| Context | Separator | Example |
|---|---|---|
| Native SQL | $ | gongqi_df_master$MasterScheduleTable |
| HQL | _ | gongqi_df_master_MasterScheduleTable |

Using _ in native SQL returns empty results silently.

## Quick Diagnosis

When insert() fails with ttsrollback:

1. Show full exception class + cause chain
2. Duplicate key? -> castToMap key case wrong, ID not cleared
3. Field required? -> EnableTree/ProductionOrganizationId, NotNull
4. HTTP refused? -> Framework sync hook -> use doInsert()
5. Transaction? -> Missing ttsbegin or double ttsrollback