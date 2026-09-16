# 擴充欄位（自含節點／AGILE_FLEX）與 BOM Notes

`{schema}` 常見為 `AGILE`。全部唯讀。實測環境 Agile 9.3.6（DOC 子類別最多 103 個擴充欄位，SDK 逐筆對照一致）。

## 1. 擴充欄位是什麼

Agile 每個頁面有固定的 base 欄位池（Page Three：25 Text／25 List／3 MultiList／…，實體欄位就是
`PAGE_THREE.TEXT01…LIST25…`）。子類別把池用完後，管理員仍可「新增屬性」——這些欄位：

- 在 `NODETABLE` 是 objtype 1、**`INHERIT = 0`** 的自含節點，掛在子類別的 `Attributes`(objtype 4) 下；
- 型別（propertyid 14）、清單（15 的 `SELECTION`）、可見（9）、必填（8）全部在**自己身上**，沒有 base 可退；
- propertyid 10（entity source）不是 `PAGE_THREE.xxx`，而是 API 名（客製前綴＋欄位名，如 `<PREFIX>_List26`）；
- `TABLEINFO.att` 用**自身 id** 登錄在該頁；
- **值不在 PAGE_TWO／PAGE_THREE，而在 `AGILE_FLEX`**，以 `attid = 自身 node id` 存放。

規模參考：測試環境（9.3.6）532 個（372 可見），全在 Page Three，DOC／CR／CO 子類別才有，PART 沒有。

## 2. 擴充欄位的定義

```sql
-- 指定子類別的擴充欄位（含型別、清單、API 名）
SELECT ov.id AS attr_id, ov.description AS title,
       TRIM(t14.value) AS type_code,          -- 11743：2 Text／3 MultiText／4 List／5 MultiList／8 Date／10 Numeric
       p15.selection AS list_id, ln.name AS list_name,
       p10.value AS api_name,
       CASE WHEN TRIM(vis.value)='1' THEN 1 ELSE 0 END AS visible,
       NVL(ti.ordering, 0) AS ordering
  FROM {schema}.nodetable attrs
  JOIN {schema}.nodetable ov ON ov.parentid = attrs.id AND ov.objtype = 1 AND NVL(ov.inherit, 0) = 0
  LEFT JOIN {schema}.propertytable t14 ON t14.parentid = ov.id AND t14.propertyid = 14
  LEFT JOIN {schema}.propertytable p15 ON p15.parentid = ov.id AND p15.propertyid = 15
  LEFT JOIN {schema}.listname ln ON ln.id = p15.selection
  LEFT JOIN {schema}.propertytable p10 ON p10.parentid = ov.id AND p10.propertyid = 10
  LEFT JOIN {schema}.propertytable vis ON vis.parentid = ov.id AND vis.propertyid = 9
  LEFT JOIN {schema}.tableinfo ti ON ti.att = ov.id AND ti.classid = attrs.parentid
 WHERE attrs.objtype = 4 AND attrs.parentid = ?           -- subclass id
 ORDER BY NVL(ti.ordering, 0), ov.id
```

讀「全部可見欄位」時必須把這批與繼承欄位合併（config-metadata §2.1 已含 `NVL(ov.inherit,0)=0` 分支）；
型別一律 `COALESCE(自身 14, base 14)`，否則擴充欄位型別全是 NULL。

## 3. 擴充欄位的值（AGILE_FLEX）

`AGILE_FLEX(id, class, attid, row_id, text, number1, …)`：

| 欄 | 意義 |
| --- | --- |
| `id`、`class` | 所屬物件（`ITEM.id`／`ITEM.class`，Change 則是 `CHANGE.id`／class）；以 `(id, class)` 連接 |
| `attid` | 擴充欄位的 node id（或 BOM Notes 的 1036） |
| `row_id` | 表格列 id（BOM Notes 用 `BOM.id`）；物件層欄位為 NULL／0 |
| `text` | Text／MultiText 值；MultiList 為逗號分隔的 entryid 串 |
| `number1` | List（單選）的 `LISTENTRY.entryid`；動態清單為目標物件 id（Item／User／Group／Site）；`-1` 表示空 |

```sql
-- 擴充欄位值：一列一欄位，含清單反查與多選展開
WITH latest_rev_id AS (
    SELECT r.item AS item_id, MAX(r.id) KEEP (DENSE_RANK LAST ORDER BY r.release_date, r.id) AS rev_id
      FROM {schema}.rev r JOIN {schema}.item i ON i.id = r.item
     WHERE i.subclass = :subclass_id AND r.release_date IS NOT NULL GROUP BY r.item)
SELECT i.item_number, af.attid, attr.description AS field_title, TRIM(t14.value) AS type_code,
       CASE TRIM(t14.value)
            WHEN '4' THEN COALESCE(le.entryvalue,                                  -- 靜態清單
                                   (SELECT x.item_number FROM {schema}.item x WHERE x.id = NULLIF(af.number1, -1)),
                                   (SELECT u.loginid FROM {schema}.agileuser u WHERE u.id = NULLIF(af.number1, -1)),
                                   (SELECT g.name FROM {schema}.user_group g WHERE g.id = NULLIF(af.number1, -1)))
            WHEN '5' THEN (SELECT LISTAGG(TRIM(e.entryvalue), '; ') WITHIN GROUP (ORDER BY e.entryid)   -- 多選：csv entryid
                             FROM {schema}.listentry e
                            WHERE ',' || af.text || ',' LIKE '%,' || e.entryid || ',%')
            ELSE af.text END AS display_value
  FROM latest_rev_id lr
  JOIN {schema}.item i ON i.id = lr.item_id
  JOIN {schema}.agile_flex af ON af.id = i.id AND af.class = i.class AND NVL(af.row_id, 0) = 0
  JOIN {schema}.nodetable attr ON attr.id = af.attid AND attr.objtype = 1
  LEFT JOIN {schema}.propertytable t14 ON t14.parentid = attr.id AND t14.propertyid = 14
  LEFT JOIN {schema}.listentry le ON le.entryid = NULLIF(af.number1, -1) AND le.langid = 0
 WHERE af.attid IN (?, ?, ?)          -- 第 2 節查出的 attr_id，≤1000 一批
 ORDER BY i.item_number, af.attid
```

大量匯出時把每個 attid 轉成一欄：`MAX(CASE WHEN af.attid = <id> THEN … END) AS "<title>"` 再 `GROUP BY af.id`。
階層清單的 display_value 用 config-metadata §3 的 `CONNECT BY` 路徑表取代 `le.entryvalue`。

判斷一個欄位的值該去哪裡讀：看定義的 propertyid 10（entity source）。`PAGE_THREE.LIST31` → 讀 `PAGE_THREE.LIST31`；
非 `REV.`／`PAGE_TWO.`／`PAGE_THREE.` 前綴的 API 名 → 讀 `AGILE_FLEX`。

## 4. BOM Notes 與插件位置（Reference Designator／RefDes／位號）

- `BOM.NOTES` 欄位存在但**全庫無值**（實測 9.3.6）；BOM Notes 實際在 `AGILE_FLEX`：`attid = 1036`、`row_id = BOM.id`、值在 `text`。
- 插件位置（Reference Designator）**不在 BOM 表的任何欄位**，在獨立表 `REFDESIG(id, bom, label)`：`bom` = `BOM.id`（BOM 列 id，不是料號 id）、`label` = 一個位號（如 `R12`）、**一列一個位號**；同一 BOM 列有幾個插件位置就幾列。輸出時 `LISTAGG(label, '; ') WITHIN GROUP (ORDER BY id)` 串成一欄；`id` 順序即 Agile 輸入順序。
- 因為 RefDes 跟著 BOM 列走，快照規則與 BOM 相同：先算出母件最後發行版的有效 BOM 列（`CHANGE_IN/CHANGE_OUT` 截止點，data-tables §2），再以 `REFDESIG.bom` 連上；Redline 未發行的 BOM 列，其位號也不算。
- 對帳：`COUNT(*) FROM refdesig` 應等於匯出串接前的位號總數（測試環境 30,135 列、9 個母件子類別有位號、455 列有 Notes、347 列兩者皆有）；位號數與 `BOM.quantity` 不一致是常態，不要拿它互相校驗。
- Agile 網頁顯示的 `R1-R5` 區間寫法在 DB 裡是逐一展開的列（`R1`、`R2`…），匯入他系統若要壓縮成區間，在程式端做。
- BOM 頁籤自訂欄位（List01…Text…）在 `BOM` 表本身（如替代群組／順序），定義在 base 表 803 BOM；哪些有啟用看 `TABLEINFO.classid = subclass AND ordering BETWEEN 1 AND 9998`。

```sql
-- 最後發行版 BOM 列 + Notes + 位號（active_bom 的有效列判定見 data-tables §2）
WITH active_bom AS ( ...data-tables §2 的有效 BOM 列，含 b.id、b.item、b.component... )
SELECT p.item_number AS parent, c.item_number AS component, b.find_num, b.quantity,
       COALESCE(nt.notes, b.notes) AS bom_notes,          -- 先 AGILE_FLEX，BOM.NOTES 只作備援
       rd.ref_des
  FROM active_bom b
  JOIN {schema}.item p ON p.id = b.item
  JOIN {schema}.item c ON c.id = b.component
  LEFT JOIN (SELECT f.row_id, MAX(f.text) AS notes
               FROM {schema}.agile_flex f WHERE f.attid = 1036 GROUP BY f.row_id) nt ON nt.row_id = b.id
  LEFT JOIN (SELECT r.bom, LISTAGG(r.label, '; ') WITHIN GROUP (ORDER BY r.id) AS ref_des
               FROM {schema}.refdesig r GROUP BY r.bom) rd ON rd.bom = b.id
 ORDER BY p.item_number, b.find_num
```

盤點哪些子類別的 BOM 真的有 Notes／位號（決定要不要匯出這兩欄）：對 `active_bom` 分別 `LEFT JOIN` 上面兩個子查詢，
`GROUP BY parent.subclass` 數 `rows_with_notes`／`rows_with_refdes`。

## 5. 常見錯誤

| 錯法 | 正解 |
| --- | --- |
| 擴充欄位型別取 base（`b.id`）→ NULL | 自含節點 `INHERIT=0` 無 base，型別／清單在自身 |
| 到 `PAGE_THREE` 找擴充欄位的值 | 值在 `AGILE_FLEX(attid=node id, id/class=物件)` |
| `AGILE_FLEX.number1 = -1` 當成 entryid | `-1` 是空值，`NULLIF(number1, -1)` |
| MultiList 只取第一個 id | `text` 是逗號分隔多個 entryid，要展開反查 |
| 讀 `BOM.NOTES` | 讀 `AGILE_FLEX attid 1036 row_id = BOM.id`，`BOM.NOTES` 只作 COALESCE 備援 |
| 位號從 BOM 表某欄找 | 在 `REFDESIG.bom = BOM.id`，多列聚合 |
