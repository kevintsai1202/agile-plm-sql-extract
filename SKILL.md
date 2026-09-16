---
name: agile-plm-sql-extract
description: Use when reading Oracle Agile PLM 9.3.x data or configuration directly from its Oracle schema with SQL (JDBC, python-oracledb, .NET/PowerShell, Node) instead of the Agile SDK — subclasses, Page Two/Three fields, extra (flex) attributes, lists (LISTNAME/LISTENTRY), workflows, items, revisions, BOM, BOM notes, reference designators (REFDESIG), AML, attachments and their vault file paths. Triggers include 直接查 Agile DB、Agile schema、NODETABLE、PROPERTYTABLE、AGILE_FLEX、擴充欄位、BOM Notes、插件位置、位號、RefDes、附件位置、最後發行版 BOM、Agile 匯出、agile9 唯讀查詢.
---

# Agile PLM 9.3.x：以 SQL 直讀設定與資料

## Overview

Agile 的設定（類別、欄位、清單、流程）不是存在「CLASS_FIELD」這類直覺的表，而是一棵
`NODETABLE` 節點樹＋`PROPERTYTABLE` 鍵值對；資料（料號、版本、BOM、AML）則靠
`CHANGE_IN/CHANGE_OUT` 與發行版 `REV` 還原快照。**憑印象猜表名一定錯**——本技能的每張表、
每個 propertyid、每個 objtype 都是在 Agile 9.3.6 正式庫用唯讀探針＋SDK 交叉驗證過的。

核心原則：**只 SELECT、只用 bind 參數、schema 前綴由程式驗證後代入、IN 清單每批 ≤ 1000。**

## When to Use

- 要把 Agile 的子類別／欄位／清單／流程「設定」搬到別的系統（匯入 metadata）
- 要匯出料號、最後發行版 BOM、AML、附件清單等「資料」做遷移或對帳
- SDK 連不上（t3 bootstrap 需 JDK 8）、沒有 Admin EJB 權限，或量大到 SDK 太慢
- 不適用：要寫回 Agile（一律走 SDK／PX，見 `agile-plm-dev`）；要即時業務規則（生命週期、簽核）——DB 只反映儲存結果

## Quick Reference：先分清楚你要的是設定還是資料

| 要什麼 | 主表 | 關鍵 | 細節 |
| --- | --- | --- | --- |
| 類別／子類別 | `NODETABLE` | class objtype 5（10000 Parts、9000 Documents、6000 Change Orders、7000 Change Requests）；subclass objtype 13 掛在 `<class>+4`（objtype 14）下；啟用＝`PROPERTYTABLE` propertyid 40='1' | [references/config-metadata.md](references/config-metadata.md) §1 |
| 欄位定義 | `NODETABLE`(objtype 1)＋`PROPERTYTABLE` | 欄位掛在 class/subclass 的 `Attributes`(objtype 4) 節點下；可見＝propertyid **9**；型別＝`COALESCE(自身 14, base 14)`；清單＝propertyid 15 的 **SELECTION**；名稱＝覆寫節點自身 `DESCRIPTION` | §2 |
| 頁籤／順序 | `TABLEINFO` | `(classid, tabid, att, ordering)`；`ordering BETWEEN 1 AND 9998` 才是畫面上啟用 | §2 |
| 清單與選項 | `LISTNAME`／`LISTENTRY` | `LISTENTRY.parentid`=list id；**`ACTIVE=0` 是使用中**；`parent_entry` 指上層 entryid；值要 `TRIM` | §3 |
| 流程 | `NODETABLE` 106/32/107/108/109/112＋`ADMINMSATT`＋`ADMINCRITERIA` | 簽核者 attid 218/219；下一關 attid 206 在 objtype 112；條件 attid 412→`ADMINCRITERIA` | §4 |
| 料號／版本 | `ITEM`／`REV` | 最後發行版＝`REV.release_date IS NOT NULL` 取 `MAX(id) KEEP (DENSE_RANK LAST ORDER BY release_date, id)` | [references/data-tables.md](references/data-tables.md) §1 |
| BOM／AML | `BOM`／`MANU_BY`→`MANU_PARTS`→`MANUFACTURERS` | 有效列＝`CHANGE_IN` 已於截止點發行 且 `CHANGE_OUT` 尚未發行；**不能只用 `CHANGE_OUT=0`** | §2、§3 |
| P2/P3 自訂欄位值 | `PAGE_TWO`／`PAGE_THREE` | 以 `(ID, CLASS)` 複合鍵連 ITEM；欄位落點看 propertyid 10（`PAGE_THREE.LIST31`）；清單值是 entryid 要反查 | §4 |
| 擴充欄位（base 池用完後另建） | `NODETABLE` objtype 1 `INHERIT=0`＋`AGILE_FLEX` | 定義在自身（型別 14／清單 15／API 名 10）；值在 `AGILE_FLEX(attid=node id, id/class=物件)`：`text`／`number1`（-1 為空）、多選為逗號 entryid | [references/flex-fields-and-bom-notes.md](references/flex-fields-and-bom-notes.md) §1–3 |
| BOM Notes | `AGILE_FLEX` | `attid=1036, row_id=BOM.id`，值在 `text`（`BOM.NOTES` 全庫無值） | 同上 §4 |
| 插件位置（Reference Designator／RefDes） | `REFDESIG(id, bom, label)` | `bom=BOM.id`（BOM 列 id）、一列一個位號、`LISTAGG(label,'; ') ORDER BY id`；跟著有效 BOM 列的快照走 | 同上 §4 |
| 附件與實體檔位置 | `ATTACHMENT_MAP`→`ATTACHMENT`／`FILES`／`FILE_INFO` | `PARENT_ID2`=`REV.CHANGE`（不是 REV.ID）；版本用 `VERSION_ID` 不用 `LATEST_VSN`；位置只有 `FILE_INFO.file_path`／`ifs_filepath` 相對路徑，vault 根目錄不在 DB，FILE_ID 推導路徑須實體核對 | §5 |

程式語言範例（JDBC／python-oracledb／.NET／Node，含 bind、分批 IN、逾時）：[references/language-snippets.md](references/language-snippets.md)。

## 開工前三個檢查

1. **表存在嗎**：`SELECT table_name FROM all_tables WHERE owner='AGILE' AND table_name IN ('NODETABLE','PROPERTYTABLE','TABLEINFO','LISTNAME','LISTENTRY','ITEM','REV','BOM','MANU_BY','AGILE_FLEX')`；owner 依環境（常見 `AGILE`）。
2. **代碼先查列舉表再用**：propertyid 名稱在 `LISTENTRY parentid=181`、欄位型別代碼 11743、objtype 101、AttType 141、Yes/No 451。看到不認識的數字先 `SELECT entryid, entryvalue FROM listentry WHERE parentid=181`。
3. **只寫死共用 id，環境專屬 id 一律依名稱查出**：可寫死的只有 Agile 內建 id（class 10000/9000/6000/7000、base 表 800/801/808/810/1501/803、objtype、propertyid、列舉表 181/11743/101/141/451、動態清單 361/13048/8222/8223/8224/8225、BOM Notes attid 1036）。子類別 id、清單 id、擴充欄位 node id、流程 id 每個環境都不同，程式與文件都不得出現實際值，要以 `description`／`LISTNAME.name`／API 名在執行期反查（見 language-snippets 的 SQL*Plus 範例）。
4. **列數一致不等於值一致**：驗證對照時同時比 `COUNT(*)` 與 `COUNT(col)`（曾因 propertyid 30 列數對得上但 86 筆 VALUE 為 NULL 而誤用）。

## Common Mistakes

| 錯法 | 正解 |
| --- | --- |
| 用 propertyid 40 判斷欄位可見 | 40 是物件（subclass／workflow）的 Enabled；欄位可見是 **9** |
| 用 propertyid 30 當欄位名稱 | 30 是改名前舊值；欄位名取覆寫節點的 `NODETABLE.DESCRIPTION`。子類別／流程名則是 `NVL(p30.value, description)` |
| 從 propertyid 15 的 `VALUE` 找清單 id | VALUE 固定 '(List)'；清單 id 在 **`SELECTION`** 欄 |
| 只走「覆寫節點→INHERIT→base」 | 還有 `INHERIT=0` 的自含欄位（Page Three 用完 base 池後另建），型別／清單在自己身上 |
| `LISTENTRY.ACTIVE=1` 當使用中 | 0＝使用中、1＝停用（測試環境 30266 vs 699） |
| 把 List 欄位的 entryid 當值輸出 | 反查 `LISTENTRY.entryvalue`；階層清單用 `CONNECT BY PRIOR entryid = parent_entry` 串成路徑 |
| BOM 只用 `CHANGE_OUT = 0` | 未發行 Redline 會混進來；要以母件最後發行 REV 的 `change`/`release_date` 為截止點 |
| `BOM.NOTES` 取備註 | 全庫無值；備註在 `AGILE_FLEX`（ATTID 1036、ROW_ID=BOM.ID） |
| 到 BOM 表找插件位置欄 | 位號在獨立表 `REFDESIG`，`bom = BOM.id`，一列一個位號；要先套 BOM 有效列快照再連 |
| 擴充欄位的值去 `PAGE_THREE` 找 | `INHERIT=0` 的擴充欄位值在 `AGILE_FLEX(attid=node id)`：單選在 `number1`（-1 為空）、多選為 `text` 逗號 entryid |
| 用 `LATEST_VSN` 抓附件 | 最後發行版要用 `parent_id2 = REV.change` ＋ `version_id`；`latest_vsn` 會帶到發行後的新版 |
| 把 `FILE_INFO.file_path` 當完整路徑 | 只是 File Manager 相對路徑；vault／iFS 根目錄不在 DB，用 FILE_ID 推導的規則必須實體核對 |
| 把 id 清單字串拼進 SQL | 一律展開成 `?`／`:n` bind，每批 ≤ 1000（ORA-01795） |
| 動態清單當靜態 | Users 361、User Groups 13048、Items 8222、Changes 8223、Manufacturers 8224/8225 不在 LISTENTRY；SELECTION 等於 class id（10000/9000/6000/7000）＝子類別下拉 |

## Real-World Impact

- 三個子類別（61／170／26 個可見欄位）DB 直讀與 SDK `getTableAttributes` 逐筆一致、0 名稱差異。
- 流程 97／關卡 770／簽核者 attid 218/219 與 SDK 全量比對吻合；`ADMINCRITERIA` 結構化條件 JDBC 甚至比 SDK 完整（RELOP 對照表見 config-metadata §4）。
- 反面教材：沒有這份對照時，模型會直接發明 `CLASS_FIELD`／`DATATYPE`／`LIST_ENTRY` 等不存在的表。
