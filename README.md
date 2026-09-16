# agile-plm-sql-extract

Claude Code skill：以 SQL 直讀 Oracle Agile PLM 9.3.x 的設定（類別／欄位／擴充欄位／清單／流程）與資料（料號／版本／BOM／BOM Notes／插件位置 RefDes／AML／附件位置），並提供 JDBC、python-oracledb、.NET、Node 的查詢樣板。所有表名與代碼皆於 Agile 9.3.6 測試環境唯讀實測並與 SDK 交叉驗證；文件只寫死 Agile 內建共用 id，環境專屬 id 一律依名稱反查。

## 安裝（Claude Code plugin，官方方式）

在 Claude Code 內執行：

```text
/plugin marketplace add kevintsai1202/agile-plm-sql-extract
/plugin install agile-plm-sql-extract@kevintsai-skills
```

或在終端機：

```powershell
claude plugin marketplace add kevintsai1202/agile-plm-sql-extract
claude plugin install agile-plm-sql-extract@kevintsai-skills
```

安裝後重啟 Claude Code，技能會在提到 Agile schema、NODETABLE、AGILE_FLEX、BOM Notes、位號、附件位置等關鍵字時自動觸發，也可用 `/agile-plm-sql-extract:agile-plm-sql-extract` 手動呼叫。

更新：`/plugin marketplace update kevintsai-skills` 後重新安裝。

## 其他安裝方式

```powershell
# 社群工具 skills CLI（非官方）
npx skills add kevintsai1202/agile-plm-sql-extract

# 手動複製到個人 skills 目錄
git clone https://github.com/kevintsai1202/agile-plm-sql-extract.git
Copy-Item -Recurse agile-plm-sql-extract/plugins/agile-plm-sql-extract/skills/agile-plm-sql-extract "$HOME\.claude\skills\"
```

## 內容

技能本體在 `plugins/agile-plm-sql-extract/skills/agile-plm-sql-extract/`：

- `SKILL.md`：何時用、設定 vs 資料快速對照、開工前檢查、常見錯法
- `references/config-metadata.md`：NODETABLE／PROPERTYTABLE／TABLEINFO／LISTNAME／LISTENTRY／流程
- `references/data-tables.md`：ITEM／REV／BOM／AML／P2-P3／附件與實體位置
- `references/flex-fields-and-bom-notes.md`：擴充欄位（AGILE_FLEX）、BOM Notes、插件位置（REFDESIG）
- `references/language-snippets.md`：各語言連線與 bind／分批／逾時樣板
