# agile-plm-sql-extract

Claude Code skill：以 SQL 直讀 Oracle Agile PLM 9.3.x 的設定（類別／欄位／擴充欄位／清單／流程）與資料（料號／版本／BOM／BOM Notes／插件位置 RefDes／AML／附件位置），並提供 JDBC、python-oracledb、.NET、Node 的查詢樣板。所有表名與代碼皆於 Agile 9.3.6 正式庫唯讀實測並與 SDK 交叉驗證。

## 安裝

```powershell
# 方式一：junction 連結到個人 skills 目錄（改此 repo 即生效）
cmd /c mklink /J "$HOME\.claude\skills\agile-plm-sql-extract" "D:\GitHub\agile-plm-sql-extract"
# 方式二：直接複製
Copy-Item -Recurse D:\GitHub\agile-plm-sql-extract "$HOME\.claude\skills\"
```

## 內容

- `SKILL.md`：何時用、設定 vs 資料快速對照、開工前檢查、常見錯法
- `references/config-metadata.md`：NODETABLE／PROPERTYTABLE／TABLEINFO／LISTNAME／LISTENTRY／流程
- `references/data-tables.md`：ITEM／REV／BOM／AML／P2-P3／附件與實體位置
- `references/flex-fields-and-bom-notes.md`：擴充欄位（AGILE_FLEX）、BOM Notes、插件位置（REFDESIG）
- `references/language-snippets.md`：各語言連線與 bind／分批／逾時樣板
