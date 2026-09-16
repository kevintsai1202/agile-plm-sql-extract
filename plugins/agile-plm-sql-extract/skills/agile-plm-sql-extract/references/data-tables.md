# Agile 9.3.x 資料（料號／版本／BOM／AML／附件）的 SQL 讀法

`{schema}` 常見為 `AGILE`；bind 用 `:subclass_id`（Oracle 具名）或 `?`。全部唯讀。

## 1. 料號與「最後發行版」

- `ITEM(id, item_number, description, class, subclass, current_phase, created, last_upd, delete_flag)`；`class` 10000 Parts／9000 Documents。
- `REV(id, item, rev_number, change, release_date, release_type, compliancy)`：`release_date IS NOT NULL` 才是已發行；`change` 指發行它的 `CHANGE.id`；`release_type` 是生命週期節點。
- `CHANGE(id, change_number, release_date, subclass, …)`。

```sql
WITH latest_rev_id AS (          -- 每個 Item 只取最後一個已發行 Rev（保留發行版，忽略更新的未發行草稿）
    SELECT r.item AS item_id,
           MAX(r.id) KEEP (DENSE_RANK LAST ORDER BY r.release_date, r.id) AS rev_id
      FROM {schema}.rev r JOIN {schema}.item i ON i.id = r.item
     WHERE i.subclass = :subclass_id AND r.release_date IS NOT NULL
     GROUP BY r.item)
SELECT i.id, i.item_number, i.description, sub.description AS item_type,
       r.rev_number, r.release_date, phase.description AS lifecycle_phase, c.change_number AS release_change
  FROM latest_rev_id lr
  JOIN {schema}.item i ON i.id = lr.item_id
  JOIN {schema}.rev r  ON r.id = lr.rev_id
  LEFT JOIN {schema}.change c ON c.id = r.change
  LEFT JOIN {schema}.nodetable sub ON sub.id = i.subclass
  LEFT JOIN {schema}.nodetable phase ON phase.id = r.release_type
 ORDER BY i.item_number
```

## 2. BOM 快照（有效列判定）

`BOM(id, item, component, change_in, change_out, quantity, find_num, notes, list01…)`。
有效列＝以母件最後發行 Rev 的 `(release_date, change)` 為截止點：

```sql
WITH parent_rev AS (
    SELECT lr.item_id, r.change AS rev_change, r.release_date
      FROM (SELECT r.item AS item_id, MAX(r.id) KEEP (DENSE_RANK LAST ORDER BY r.release_date, r.id) AS rev_id
              FROM {schema}.rev r JOIN {schema}.item i ON i.id = r.item
             WHERE i.subclass = :subclass_id AND r.release_date IS NOT NULL GROUP BY r.item) lr
      JOIN {schema}.rev r ON r.id = lr.rev_id)
SELECT p.item_number AS parent, c.item_number AS component, b.find_num, b.quantity, b.id AS bom_id
  FROM parent_rev pr
  JOIN {schema}.item p ON p.id = pr.item_id
  JOIN {schema}.bom b ON b.item = p.id
  JOIN {schema}.item c ON c.id = b.component
  LEFT JOIN {schema}.change ci ON ci.id = b.change_in
  LEFT JOIN {schema}.change co ON co.id = b.change_out
 WHERE (b.change_in = 0
        OR (ci.release_date IS NOT NULL
            AND (ci.release_date < pr.release_date OR (ci.release_date = pr.release_date AND ci.id <= pr.rev_change))))
   AND (b.change_out = 0 OR co.release_date IS NULL OR co.release_date > pr.release_date
        OR (co.release_date = pr.release_date AND co.id > pr.rev_change))
 ORDER BY p.item_number, b.find_num
```

- 只用 `change_out = 0` 會把未發行 Redline 新增列混進來、也會漏掉「未發行 Redline 刪除但舊列仍有效」的情況。
- BOM Notes 在 `AGILE_FLEX`（`attid = 1036`、`row_id = BOM.id`、值在 `text`）；`BOM.NOTES` 全庫無值。
- Reference Designator 在 `REFDESIG(bom, label, id)`，`LISTAGG(label, '; ')`。
- 替代料：BOM 自訂 List01＝替代群組、List02＝替代順序（依客戶設定，先看 BOM 表 803 的欄位定義）。
- BOM 頁籤啟用欄位：`TABLEINFO.classid = subclass AND ordering BETWEEN 1 AND 9998`。

## 3. AML（Approved Manufacturer List）

`MANU_BY(id, agile_part=ITEM.id, manu_part→MANU_PARTS.id, prefer_status, active, change_in, change_out, created)`；
`MANU_PARTS(id, manu_id→MANUFACTURERS.id, part_number, description, subclass, status, delete_flag)`；
`MANUFACTURERS(id, name, subclass, address…, delete_flag)`。`change_in/out` 快照規則與 BOM 相同；Manufacturer／MPart 沒有 Rev，只看 `delete_flag = 0`。

```sql
SELECT i.item_number, mfr.name AS manufacturer, mp.part_number AS mfr_part, pref.description AS preferred_status, mb.active
  FROM parent_rev pr                                   -- 同 §2 的 parent_rev
  JOIN {schema}.item i ON i.id = pr.item_id
  JOIN {schema}.manu_by mb ON mb.agile_part = i.id
  LEFT JOIN {schema}.manu_parts mp ON mp.id = mb.manu_part
  LEFT JOIN {schema}.manufacturers mfr ON mfr.id = mp.manu_id
  LEFT JOIN {schema}.nodetable pref ON pref.id = mb.prefer_status
  LEFT JOIN {schema}.change ci ON ci.id = mb.change_in
  LEFT JOIN {schema}.change co ON co.id = mb.change_out
 WHERE (mb.change_in = 0 OR (ci.release_date IS NOT NULL AND (ci.release_date < pr.release_date OR (ci.release_date = pr.release_date AND ci.id <= pr.rev_change))))
   AND (mb.change_out = 0 OR co.release_date IS NULL OR co.release_date > pr.release_date OR (co.release_date = pr.release_date AND co.id > pr.rev_change))
```

## 4. Page Two／Page Three 自訂欄位值

- P1 值在 `ITEM`／`REV` 固定欄；P2 在 `PAGE_TWO`、P3 在 `PAGE_THREE`，皆以 `(id = ITEM.id, class = ITEM.class)` 複合鍵連接。
- 哪一欄：看欄位定義 propertyid 10（`PAGE_THREE.LIST31`、`PAGE_TWO.TEXT05`）。沒有固定欄的（自含節點、API 名）在 `AGILE_FLEX(attid, row_id, class, text/number/date…)`，以 attribute id 彙整。
- List 欄存的是 `LISTENTRY.entryid`（-1／0 表示空）；MultiList 是以逗號分隔的 entryid 串；階層清單以 config-metadata §3 的 `CONNECT BY` 路徑反查；動態清單指向 `ITEM.item_number`／`AGILEUSER`／`USER_GROUP`／`MANUFACTURERS` 等。

```sql
SELECT i.item_number, p3.text01, p3.list31,
       le.entryvalue AS list31_value                    -- entryid → 顯示值
  FROM latest_rev_id lr                                 -- 同 §1
  JOIN {schema}.item i ON i.id = lr.item_id
  JOIN {schema}.page_three p3 ON p3.id = i.id AND p3.class = i.class
  LEFT JOIN {schema}.listentry le ON le.entryid = p3.list31 AND le.langid = 0
```

Change（ECO／ECR 等）的 Cover Page／Page Two／Page Three 同樣分別在 `CHANGE`、`PAGE_TWO`、`PAGE_THREE`（`class` 為 change 的 class id）。

## 5. 附件與附件位置（實體檔案在哪裡）

`ATTACHMENT_MAP(parent_id=ITEM.id, parent_class=ITEM.class, parent_id2=REV.change, attach_id, file_id, version, version_id, latest_vsn, created)`
→ `ATTACHMENT(id, attachment_number)`、`FILES(id, filename, file_size, file_type)`、`FILE_INFO(file_id, checksum_value, file_path, ifs_filepath)`。

```sql
SELECT i.item_number, m.attach_id, a.attachment_number, m.version, m.version_id, m.latest_vsn,
       m.file_id, f.filename, f.file_size, fi.checksum_value,
       fi.file_path AS vault_file_path,          -- File Vault（舊式 File Manager）記錄的路徑
       fi.ifs_filepath AS ifs_file_path          -- iFS（9.3 Internal File System）記錄的路徑
  FROM {schema}.item i
  JOIN {schema}.attachment_map m ON m.parent_id = i.id AND m.parent_class = i.class
  LEFT JOIN {schema}.attachment a ON a.id = m.attach_id
  LEFT JOIN {schema}.files f ON f.id = m.file_id
  LEFT JOIN {schema}.file_info fi ON fi.file_id = m.file_id
 WHERE i.subclass = :subclass_id
 ORDER BY i.item_number, m.attach_id, m.version
```

只要最後發行版當下的附件（注意 CTE 不可把 `r.change` 放進 `GROUP BY`，否則一個 Item 會多列）：

```sql
WITH latest_rev AS (
    SELECT lr.item_id, r.change AS rev_change
      FROM (SELECT r.item AS item_id, MAX(r.id) KEEP (DENSE_RANK LAST ORDER BY r.release_date, r.id) AS rev_id
              FROM {schema}.rev r JOIN {schema}.item i ON i.id = r.item
             WHERE i.subclass = :subclass_id AND r.release_date IS NOT NULL GROUP BY r.item) lr
      JOIN {schema}.rev r ON r.id = lr.rev_id)
SELECT i.item_number, m.attach_id, m.version_id, f.filename, fi.file_path, fi.ifs_filepath
  FROM latest_rev lr
  JOIN {schema}.item i ON i.id = lr.item_id
  JOIN {schema}.attachment_map m ON m.parent_id = i.id AND m.parent_class = i.class AND m.parent_id2 = lr.rev_change
  LEFT JOIN {schema}.files f ON f.id = m.file_id
  LEFT JOIN {schema}.file_info fi ON fi.file_id = m.file_id
```

附件位置要特別處理的點：

1. **哪一版**：`ATTACHMENT_MAP.parent_id2` 對的是 `REV.change`（不是 `REV.id`）；只要最後發行版的附件就用該 Rev 的 `change` 過濾，版本取 `version_id`，**不要** `latest_vsn`（會帶到發行之後的新版檔案）。
2. **DB 只記邏輯位置**：`FILE_INFO.file_path`／`ifs_filepath` 是 File Manager 相對路徑，實體檔在 File Manager 主機的 vault／iFS 根目錄之下，DB 裡沒有完整 UNC；根目錄要向 Agile 管理員取（File Manager 設定）。兩欄可能只有一欄有值（依該環境用 File Vault 還是 iFS），先跑 `all_tab_columns` 列出 `ATTACHMENT_MAP`／`FILE_INFO` 含 PATH／VAULT／FOLDER／LOCATION 的欄位，不預設欄位名。
3. **路徑欄空白時**：Agile 常見以 `FILE_ID` 推導實體路徑（ID 左補零到 11 碼、切成三層子目錄、檔名 `agile<FILE_ID>.<ext>`，副檔名取自 `FILES.filename`）。**這條規則各客戶 vault 不一定相同，必須抽 100 筆對實體目錄核對後才能用**；用 `MIN/MAX(file_info.id)` 看值域邊界、`FILES.filename` 的副檔名分布補齊 ext。
4. **同一實體檔多處引用**：同一 `file_id` 會掛在多個物件／版本上，搬檔以 `file_id` 去重，對帳以 `attachment_map` 列數計。
5. **存在性與完整性**：匯出後對每個 `file_id` 做實體檔存在掃描並比對 `FILE_INFO.checksum_value`；不存在或 checksum 不符的列標記失敗，不要靜默略過。
6. **Change 的附件**：`parent_class` 為 change 的 class id、`parent_id = CHANGE.id`，其餘相同。

盤點用：`SELECT COUNT(*), ROUND(SUM(file_size)/1024/1024/1024,2) AS size_gb FROM {schema}.file_info f JOIN {schema}.files x ON x.id = f.file_id`；
副檔名 Top N：`LOWER(SUBSTR(filename, INSTR(filename,'.',-1)+1))` 分組。

## 6. 使用者與群組

`AGILEUSER(id, loginid, first_name, last_name, email, status)`、`USER_GROUP(id, name)`、成員關係 `USER_GROUP_MAP`（依環境確認）。流程簽核者 `ADMINMSATT.value` 對得上 `AGILEUSER.id` 或 `USER_GROUP.id`。

## 7. 盤點 SQL（跑之前先看規模）

```sql
SELECT nt_class.description AS class_name, nt_sub.description AS subclass_name, COUNT(*) AS item_count
  FROM {schema}.item i
  JOIN {schema}.nodetable nt_sub ON nt_sub.id = i.subclass
  JOIN {schema}.nodetable nt_mid ON nt_mid.id = nt_sub.parentid
  JOIN {schema}.nodetable nt_class ON nt_class.id = nt_mid.parentid
 GROUP BY nt_class.description, nt_sub.description ORDER BY 3 DESC
```

實測規模（測試環境 9.3.6）：ITEM 21.6 萬、已發行 Part 9.9 萬／Document 10.9 萬；`ITEM`×`REV` 的最後發行版 COUNT 約 5 秒。大量匯出用 `/*+ MATERIALIZE */` CTE、在備援庫或離峰跑。
