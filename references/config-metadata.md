# Agile 9.3.x 設定（metadata）的 SQL 讀法

所有 SQL 以 `{schema}` 代表 schema 前綴（常見 `AGILE`），由程式驗證後代入；`?` 為 bind 位置。
實測環境：Agile 9.3.6，SDK 交叉驗證通過（2026-09-15）。

## 1. 節點樹（NODETABLE）

```text
5002 Agile Classes (objtype 6)
└─ <class> objtype 5           10000 Parts / 9000 Documents / 6000 Change Orders / 7000 Change Requests
   ├─ <class>+5 Attributes (objtype 4)      class 層欄位覆寫節點容器（10005／9005／6001／7005）
   │    └─ 覆寫節點 objtype 1, INHERIT = base 欄位 id
   └─ <class>+4 User-Defined Subclasses (objtype 14)
        └─ <subclass> objtype 13            啟用：PROPERTYTABLE propertyid 40 = '1'
             ├─ Attributes (objtype 4)
             │    ├─ 覆寫節點 objtype 1, INHERIT = base 欄位 id（Page Three）
             │    └─ 自含節點 objtype 1, INHERIT = 0（自帶型別／清單／entity source）
             └─ Tabs (objtype 10) → tab (objtype 9)
800 Agile Base Attribute
└─ base 表 objtype 2：801 Title Block、808 Cover Page、810 Page Two、1501 Page Three、803 BOM、805 Where Used
     └─ base 欄位 objtype 1：型別(14)、UI 元件(1)、儲存欄位(10) 定義在這層
```

### 1.1 啟用的子類別（含可見欄位數）

```sql
SELECT sub.id AS subclass_id,
       NVL(p30.value, sub.description) AS subclass_name,   -- 子類別名：p30 優先、退回 DESCRIPTION
       cls.id AS class_id, cls.description AS class_name,
       sub.last_upd
  FROM {schema}.nodetable sub
  JOIN {schema}.nodetable udcs ON udcs.id = sub.parentid AND udcs.objtype = 14
  JOIN {schema}.nodetable cls  ON cls.id = udcs.parentid AND cls.objtype = 5
  JOIN {schema}.propertytable en ON en.parentid = sub.id AND en.propertyid = 40 AND en.value = '1'
  LEFT JOIN {schema}.propertytable p30 ON p30.parentid = sub.id AND p30.propertyid = 30
 WHERE sub.objtype = 13 AND cls.id IN (?, ?, ?, ?)
 ORDER BY cls.id, 2
```

### 1.2 由 subclass id 反推 class id（2.1 需要兩個 id）

```sql
SELECT cls.id AS class_id, cls.description AS class_name
  FROM {schema}.nodetable sub
  JOIN {schema}.nodetable udcs ON udcs.id = sub.parentid AND udcs.objtype = 14
  JOIN {schema}.nodetable cls  ON cls.id = udcs.parentid AND cls.objtype = 5
 WHERE sub.id = ? AND sub.objtype = 13
```

## 2. 欄位定義（PROPERTYTABLE 對照）

`PROPERTYTABLE(parentid=節點 id, propertyid, value, selection)`。propertyid 名稱表：`LISTENTRY parentid=181`。

| 用途 | propertyid | 取法 |
| --- | --- | --- |
| 顯示名稱 | — | 覆寫節點自身 `NODETABLE.DESCRIPTION`（**不是** propertyid 30，那是舊值） |
| 可見 | **9** | `TRIM(value)='1'`（451：0 No／1 Yes） |
| 必填 | 8 | '1'／'0'／NULL |
| 型別 | 14 | 在 base 欄位上；自含節點在自己身上 → `COALESCE(自身, base)`；代碼表 11743 |
| UI 元件 | 1 | base 上；141：2 SingleLineTextBox／3 MultiLine／4 SingleSelect／5 MultiSelect（僅參考） |
| 清單 | 15 | **`selection` = LISTNAME.ID**（value 固定 '(List)'） |
| 預設值 | 5 | List 型是 entryid；日期 `$NOW` |
| 最大長度 | 6 | |
| 儲存欄位 | 10 | `REV.LIST11`、`PAGE_TWO.LIST11`、`PAGE_THREE.TEXT01`、自含節點為 API 名（客製前綴＋欄位名，如 `<PREFIX>_List26`，值在 AGILE_FLEX） |
| 值變更受控 | 923 | |
| API 名稱 | 925 | |

型別代碼（11743）：2 Text、3 MultiText、4 List、5 MultiList、8 Date、9 Heading、10 Numeric、12 Money、19 Large Text；6/7/11/13～18 為 EditListBox／DialogBox／Image／UOM／Link 等。

### 2.1 一個子類別的全部可見欄位（class 層 P1/P2 ∪ subclass 層 P3 ∪ 自含）

```sql
SELECT NVL(NULLIF(ov.inherit, 0), ov.id) AS agile_attr_id,     -- 繼承者=base id、自含=自身 id（TABLEINFO.att 也用這個）
       ov.description AS title,
       CASE WHEN NVL(ov.inherit,0)=0 OR b.parentid=1501 THEN 'P3'
            WHEN b.parentid=810 THEN 'P2' ELSE 'P1' END AS page_code,
       COALESCE(TRIM(t14o.value), TRIM(t14b.value)) AS type_code,
       CASE WHEN TRIM(p8.value)='1' THEN 1 ELSE 0 END AS required,
       p15.selection AS list_id, ln.name AS list_name, ln.readonly AS list_readonly, ln.cascade_ind,
       p5.value AS default_value, p6.value AS max_length, p10.value AS entity_source,
       NVL(ti.ordering, 0) AS ordering
  FROM {schema}.nodetable attrs
  JOIN {schema}.nodetable ov ON ov.parentid = attrs.id AND ov.objtype = 1
  JOIN {schema}.propertytable vis ON vis.parentid = ov.id AND vis.propertyid = 9 AND TRIM(vis.value) = '1'
  LEFT JOIN {schema}.nodetable b ON b.id = ov.inherit
  LEFT JOIN {schema}.propertytable t14b ON t14b.parentid = b.id  AND t14b.propertyid = 14
  LEFT JOIN {schema}.propertytable t14o ON t14o.parentid = ov.id AND t14o.propertyid = 14
  LEFT JOIN {schema}.propertytable p8  ON p8.parentid  = ov.id AND p8.propertyid  = 8
  LEFT JOIN {schema}.propertytable p15 ON p15.parentid = ov.id AND p15.propertyid = 15
  LEFT JOIN {schema}.listname ln ON ln.id = p15.selection
  LEFT JOIN {schema}.propertytable p5  ON p5.parentid  = ov.id AND p5.propertyid  = 5
  LEFT JOIN {schema}.propertytable p6  ON p6.parentid  = ov.id AND p6.propertyid  = 6
  LEFT JOIN {schema}.propertytable p10 ON p10.parentid = COALESCE(b.id, ov.id) AND p10.propertyid = 10
  LEFT JOIN {schema}.tableinfo ti ON ti.att = NVL(NULLIF(ov.inherit,0), ov.id) AND ti.classid = attrs.parentid
 WHERE attrs.objtype = 4
   AND attrs.parentid IN (?, ?)                                -- (class id, subclass id)
   AND (NVL(ov.inherit,0) = 0 OR b.parentid IN (801, 808, 810, 1501))
 ORDER BY CASE WHEN NVL(ov.inherit,0)=0 OR b.parentid=1501 THEN 3 WHEN b.parentid=810 THEN 2 ELSE 1 END,
          NVL(ti.ordering,0), 1
```

注意：同一 base 欄位若 class 與 subclass 兩層都覆寫，會出現兩列；以 subclass 層為準去重。
若目標只是「畫面上有排進去的欄位」，改以 `TABLEINFO.ordering BETWEEN 1 AND 9998` 篩。

### 2.2 頁籤（TABLEINFO）

`TABLEINFO(classid, tabid, tableid, att, ordering)`：`classid` 是 class 或 subclass id；tab 名在 `NODETABLE(id=tabid, objtype 9)`；
`att` 是 base 欄位 id（自含節點則為自身 id）。`ordering` 未維護時為 0。

## 3. 清單（LISTNAME／LISTENTRY）

- `LISTNAME(id, name, description, readonly, cascade_ind, parent_list, apiname, last_upd)`：`readonly=1` 系統內建、`cascade_ind=1` 階層根、`parent_list` 指上層。
- `LISTENTRY(parentid=list id, entryid, entryvalue, active, langid, parent_entry, apiname)`：**`active=0` 使用中**；沒有順序欄；`parent_entry` 指上層清單的 entryid；`langid=0` 為主語系。

```sql
-- 使用中的選項
SELECT le.parentid AS list_id, le.entryid, TRIM(le.entryvalue) AS value, NVL(le.parent_entry,0) AS parent_entry
  FROM {schema}.listentry le
 WHERE le.parentid IN (?, ?, ?) AND NVL(le.active,0) = 0 AND NVL(le.langid,0) = 0
 ORDER BY le.parentid, le.entryid;

-- 階層清單攤成「根|…|葉」路徑（entryid → 顯示值，供資料匯出反查）
SELECT le.entryid, le.parentid AS list_id,
       REPLACE(SUBSTR(SYS_CONNECT_BY_PATH(le.entryvalue, '<#>'), 4), '<#>', '|') AS display_value, LEVEL AS depth
  FROM {schema}.listentry le JOIN {schema}.listname ln ON ln.id = le.parentid
 WHERE le.langid = 0
 START WITH NVL(le.parent_entry,0) = 0
 CONNECT BY NOCYCLE PRIOR le.entryid = le.parent_entry AND PRIOR le.parentid = ln.parent_list AND PRIOR le.langid = le.langid
```

動態清單（不在 LISTENTRY）：Users 361（→`AGILEUSER`）、User Groups 13048（→`USER_GROUP`）、Items 8222（→`ITEM`）、
Changes 8223（→`CHANGE`）、Manufacturers 8224／Mfr Parts 8225；`selection` 等於 class id 時是「子類別下拉」。

## 4. 流程（Workflow）

```text
objtype 105 Agile Workflows 根 → 106 Workflow → 32 Status List → 107 Status
                                                    ├─ 112 Status Properties  ── ADMINMSATT attid 206 = 下一關 status id
                                                    └─ 108 Criteria-Specific Properties → 109 ExitCriteria
                                                          ├─ ADMINMSATT attid 218 Approvers／219 Observers（value = AGILEUSER.id 或 USER_GROUP.id；查無＝$APPROVER 等符號）
                                                          ├─ ADMINMSATT attid 202 = 必填欄位 node id
                                                          └─ ADMINMSATT attid 412 → objtype 111 criteria 節點 → ADMINCRITERIA(parentid)
```

- 流程／關卡名稱：`NVL(propertyid 30 value, NODETABLE.description)`（客戶自建流程 p30 為 NULL）。
- 流程：40 啟用、53 適用 class、925 API 名；關卡型別 propertyid 200（0 Pending／1 Submitted／2 Review／3 Released／4 Complete／5 Cancel／6 Hold，以名稱交叉表校驗）。
- `ADMINCRITERIA(parentid, id, attid, relop, value, joinop, lead, trail)`：**`ORDER BY id` 不可省**；`attid` 正數為 `NODETABLE` objtype 1 欄位、負數為系統偽屬性（-3 `$CHECKOUTUSER`、-6 `$CURRENTREV`、-61 `$LATESTREV`）；`value` 是 VARCHAR2，清單值以 `TO_CHAR(nodetable.id)` 反查。
- RELOP：1 contains、2 equal to、3 not equal to、4 starts with、9 is null、10 is not null、11 does not start with、14 is released、15 is introductory、23 in（使用者／文字）、24 not in、25 contains any、26 contains all words、31 in（清單，值為節點 id）。

```sql
-- 關卡簽核者
SELECT wf.id AS wf_id, NVL(wn.value, wf.description) AS wf_name,
       st.id AS status_id, NVL(sn.value, st.description) AS status_name,
       msatt.attid, msatt.value, au.loginid, ug.name AS group_name
  FROM {schema}.nodetable wf
  JOIN {schema}.nodetable sl  ON sl.parentid = wf.id AND sl.objtype = 32
  JOIN {schema}.nodetable st  ON st.parentid = sl.id AND st.objtype = 107
  LEFT JOIN {schema}.nodetable csp ON csp.parentid = st.id AND csp.objtype = 108
  LEFT JOIN {schema}.nodetable ec  ON ec.parentid = csp.id AND ec.objtype = 109
  LEFT JOIN {schema}.adminmsatt msatt ON msatt.parentid = ec.id AND msatt.attid IN (218, 219)
  LEFT JOIN {schema}.agileuser au ON au.id = msatt.value
  LEFT JOIN {schema}.user_group ug ON ug.id = msatt.value
  LEFT JOIN {schema}.propertytable wn ON wn.parentid = wf.id AND wn.propertyid = 30
  LEFT JOIN {schema}.propertytable sn ON sn.parentid = st.id AND sn.propertyid = 30
 WHERE wf.objtype = 106 AND wf.id IN (?)
 ORDER BY wf.id, st.id, ec.id, msatt.attid
```

下一關（attid 206）與必填欄位（attid 202）各自獨立查詢，不要與 218/219 併在同一支（會相乘成笛卡兒積）。

## 5. 過期檢查

發布前重讀 `MAX(last_upd)`：子類別子樹＝`nodetable.id IN (subclass) OR parentid IN (subclass) OR parentid IN (其 Attributes 節點)`；
清單＝`LISTNAME.last_upd`。查不到時間戳要 fail closed。
