# 各語言連線與查詢樣板

共通規則：唯讀帳號、只 SELECT、schema 前綴以白名單驗證後代入（不接受使用者字串）、id 清單展開成 bind 佔位符且每批 ≤ 1000、
設查詢逾時（60 秒）、值一律 `TRIM`。連線字串：`host:1521/SERVICE` 或 thin `@host:1521:SID`。

## Java（JDBC，ojdbc8）

```java
/** 依 subclass 讀可見欄位；SQL 放資源檔，{schema} 與 {ids} 由程式展開 */
public List<FieldRow> readFields(Connection conn, String schema, long classId, long subclassId) throws SQLException {
    String sql = loadSql("class-fields.sql").replace("{schema}", normalizeSchema(schema)); // 白名單：^[A-Z][A-Z0-9_]{0,29}$
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setQueryTimeout(60);
        ps.setLong(1, classId); ps.setLong(2, subclassId);
        try (ResultSet rs = ps.executeQuery()) {
            List<FieldRow> out = new ArrayList<>();
            while (rs.next()) out.add(mapRow(rs));       // rs.getString("title") 等，trimToNull
            return out;
        }
    }
}
/** IN 清單分批：Oracle 單一 IN 上限 1000（ORA-01795） */
static <T> List<List<T>> chunk(List<T> ids, int size) { ... }
static String bindPlaceholders(int n) { return String.join(",", Collections.nCopies(n, "?")); }
```
連線：`DriverManager.getConnection("jdbc:oracle:thin:@//host:1521/agile9", user, pw)`；`conn.setReadOnly(true)`。

## Python（python-oracledb，thin mode 免 Instant Client）

```python
import oracledb

SQL = """SELECT le.parentid AS list_id, le.entryid, TRIM(le.entryvalue) AS value
           FROM {schema}.listentry le
          WHERE le.parentid IN ({ids}) AND NVL(le.active,0) = 0 AND NVL(le.langid,0) = 0
          ORDER BY le.parentid, le.entryid"""

def list_entries(dsn, user, password, schema, list_ids):
    """讀取多個清單的使用中選項；ids 分批 ≤1000，具名 bind"""
    assert schema.isidentifier() and schema.upper() == schema
    rows = []
    with oracledb.connect(user=user, password=password, dsn=dsn) as conn:  # dsn="host:1521/agile9"
        conn.call_timeout = 60_000
        with conn.cursor() as cur:
            cur.arraysize = 1000
            for i in range(0, len(list_ids), 1000):
                batch = list_ids[i:i + 1000]
                binds = ",".join(f":id{j}" for j in range(len(batch)))
                cur.execute(SQL.format(schema=schema, ids=binds), {f"id{j}": v for j, v in enumerate(batch)})
                cols = [c[0].lower() for c in cur.description]
                rows += [dict(zip(cols, r)) for r in cur]
    return rows
```
大量匯出：`cursor.arraysize = 5000`、逐批 `fetchmany()` 寫 CSV；避免 `fetchall()` 吃記憶體。

## PowerShell／.NET（Oracle.ManagedDataAccess）

```powershell
Add-Type -Path 'C:\oracle\Oracle.ManagedDataAccess.dll'
$conn = New-Object Oracle.ManagedDataAccess.Client.OracleConnection("User Id=ro;Password=***;Data Source=host:1521/agile9")
$conn.Open()
$cmd = $conn.CreateCommand()
$cmd.CommandTimeout = 60
$cmd.CommandText = "SELECT sub.id, NVL(p30.value, sub.description) AS name FROM AGILE.nodetable sub
  JOIN AGILE.nodetable u ON u.id = sub.parentid AND u.objtype = 14
  JOIN AGILE.propertytable en ON en.parentid = sub.id AND en.propertyid = 40 AND en.value = '1'
  LEFT JOIN AGILE.propertytable p30 ON p30.parentid = sub.id AND p30.propertyid = 30
 WHERE sub.objtype = 13 AND u.parentid = :classId"
$cmd.BindByName = $true
[void]$cmd.Parameters.Add("classId", 7000)
$rd = $cmd.ExecuteReader(); while ($rd.Read()) { [pscustomobject]@{ Id = $rd["ID"]; Name = $rd["NAME"] } }
$conn.Close()
```
`BindByName = $true` 必設，否則 ODP.NET 依順序綁定。

## Node.js（node-oracledb）

```js
const oracledb = require("oracledb");
oracledb.outFormat = oracledb.OUT_FORMAT_OBJECT;
async function itemsOfSubclass(subclassId) {
  const conn = await oracledb.getConnection({ user, password, connectString: "host:1521/agile9" });
  try {
    const rs = await conn.execute(
      `SELECT i.item_number, i.description FROM AGILE.item i WHERE i.subclass = :id ORDER BY i.item_number`,
      { id: subclassId }, { fetchArraySize: 1000 });
    return rs.rows;
  } finally { await conn.close(); }
}
```
串流大表用 `conn.queryStream(sql, binds)`。

## SQL*Plus／SQL Developer（一次性驗證）

```sql
-- 不要寫死任何環境專屬 id；先用 Agile 內建的共用 id（class 10000/9000/6000/7000、objtype 13/14）依名稱查出
VARIABLE subclass_id NUMBER;
BEGIN
  SELECT sub.id INTO :subclass_id
    FROM agile.nodetable sub
    JOIN agile.nodetable u ON u.id = sub.parentid AND u.objtype = 14
   WHERE sub.objtype = 13 AND u.parentid = 10000            -- 10000 Parts（共用 id）
     AND sub.description = '<子類別顯示名稱>';
END;
/
-- 之後的查詢直接用 :subclass_id
```

## 驗證對照的做法（跨通道）

1. 同一物件用 SDK（`getTableDescriptors()`→`getAttributes()`，JDK 8）與 SQL 各取一份，比 id 集合與名稱。
2. 比對列數同時比非空值數：`COUNT(*)` vs `COUNT(col)`。
3. 不認識的代碼先查 Agile 自身的列舉表（`LISTENTRY parentid = 181／11743／101／141／451`），不要推論。
