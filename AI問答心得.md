# sqlean 的 eval() 函數產生多個獨立的標準 SQL 欄位(Grid)
SQLite 的 sqlean 擴充功能中的 eval() 函數無法直接產生多個獨立的標準 SQL 欄位（Grid），它會將查詢結果組合成單一字串。
## eval() 函數的運作方式

* 單一返回值：eval('select 1, 2, 3') 會回傳用空格或自訂分隔符號串接起來的單一字串（例如 "1 2 3"）。
* 純量函數限制：eval() 是一個純量函數（Scalar Function），在 SQL 語法結構上只能輸出一個欄位，無法像表格值函數（TVF）或動態樞紐分析一樣自動展開成多個獨立欄位。 [1] 

## 替代建議

* 如果需要動態產生多欄或表格，需改用其他方式在應用層處理，或利用 SQLite 的 JSON 擴充功能先將資料組合成 JSON 再進行解析。

如果你需要將動態查詢結果展開為多個欄位，請問你想了解如何在應用程式端（如 Python/Node.js）解析 eval() 的字串，還是需要使用 JSON 函數來處理結構化資料？

# 若 搭配 abiliojr/sqlite-lua 擴充套件 來執行 語句
搭配 [abiliojr/sqlite-lua](https://github.com/abiliojr/sqlite-lua) 擴充套件，依然無法直接在單一 SELECT 語句中產生動態的標準 SQL 獨立欄位（Grid）。 [1, 2] 
原因在於該套件的底層限制，以及 SQLite 原生缺乏對動態虛擬資料表（Table-Valued Functions, TVF）的高階原生支援。
## 為什麼 sqlite-lua 無法達成？

   1. 僅支援純量與聚合函數：根據 abiliojr/sqlite-lua 的官方說明，它透過 createlua() 註冊的自訂函數只支援純量（Scalar）與聚合（Aggregate）兩種模式。純量函數在 SQL 語法結構中，不論內部邏輯多複雜，最終對每一列只能返回單一值（通常是字串、數字或 NULL）。 [1, 2] 
   2. 缺乏資料表值函數（TVF）支援：若要讓一個函數像執行 SELECT * 一樣直接展開成多個欄位，擴充套件必須實作 SQLite 的 xBestIndex、xFilter、xColumn 等虛擬資料表（Virtual Table）API。而 sqlite-lua 並未封裝這套複雜的 C 語言虛擬資料表介面。

------------------------------
## 解套方案：Lua 搭配 SQLite 的變通寫法
雖然無法「一語句直接變出新欄位」，但由於 Lua 擁有強大的動態字串處理能力，你可以透過以下兩種技巧在 SQLite 內部繞過限制：
## 方案一：在 Lua 中動態組裝並執行多欄位 SQL（最實用）
你無法讓 SELECT eval_lua(...) 變成多欄位，但你可以讓 Lua 函數在背後幫你拼接出標準的 CREATE VIEW 或 CREATE TABLE 語句，然後直接在資料庫中執行它。
```sql
-- 1. 註冊一個可以動態建立 View 的 Lua 函數SELECT createlua('build_my_grid', '
    -- 假設這段 Lua 程式碼會動態讀取某個配置，並組裝 SQL
    local cols = {"col1", "col2", "col3"}
    local sql = "CREATE VIEW temp.my_grid AS SELECT "
    
    for i, col in ipairs(cols) do
        sql = sql .. i .. " AS " .. col .. (i < #cols and ", " or "")
    end
    
    sql = sql .. " FROM (SELECT 1);"
    
    -- 透過某種方式（如觸發另一個內部機制）讓此 SQL 被執行
    -- 或者單純讓此函數回傳這段字串
    return sql
');
-- 2. 取得這段動態 SQL 後，在你的應用程式端或透過預備語句執行它-- 執行後你就可以直接查詢它：SELECT * FROM temp.my_grid;
```
## 方案二：利用 Lua 返回 JSON，再由 SQLite 展開（不需動態 DDL）
這是最推薦的標準作法。讓 Lua 負責把動態的多個欄位打包成一個 JSON 物件字串，然後利用 SQLite 內建的 JSON 擴充功能將其拆解成獨立欄位。
```sql
-- 1. 用 Lua 定義一個回傳 JSON 的純量函數SELECT createlua('lua_to_json', '
    -- 模擬動態產生多個欄位的資料
    return "{\"name\": \"Alice\", \"age\": 25, \"score\": 95}"
');
-- 2. 在 SQL 中搭配 json_each 或 json_tree 展開為獨立欄位（Grid）SELECT 
    json_extract(json_str, '$.name') AS name,
    json_extract(json_str, '$.age') AS age,
    json_extract(json_str, '$.score') AS scoreFROM (
    SELECT lua_to_json() AS json_str
);
```
## 總結
abiliojr/sqlite-lua 的定位是讓你能在 SQL 中使用 Lua 的邏輯運算或字串處理。若要產生多欄位的網格資料，最完美的組合是「Lua 負責生成 JSON 結構」+「SQLite JSON 函數負責展開欄位」。

# 那假如 搭配 0x09/sqlite-statement-vtab 呢
如果搭配 [0x09/sqlite-statement-vtab](https://github.com/0x09/sqlite-statement-vtab) 擴充套件，答案是：可以，而且這正是最完美的解決方案！ [1] 
sqlite-statement-vtab 本身就是為了解決 SQLite 缺乏動態資料表值函數（Table-Valued Function）而設計的。它允許你將一段帶有參數的 SQL 語句包裝成一個虛擬資料表（Virtual Table），並在查詢時動態帶入參數，直接輸出標準的 多欄位獨立結果（Grid）。 [2] 
在這種架構下，你甚至不需要 sqlean 的 eval() 函數了，因為 statement-vtab 已經完美取代了動態執行的需求。

---
## 如何運作？（實際範例）
statement-vtab 允許你在定義虛擬資料表時使用 SQLite 的參數綁定語法（如 ? 或 :name）。這些參數會自動變成虛擬資料表的隱藏輸入欄位。
### 步驟 1：建立一個參數化的虛擬資料表（Grid 範本）
你可以先宣告一個固定的多欄位結構，其中的查詢條件或計算邏輯是動態的： [2] 
```sql
-- 建立一個名為 dynamic_grid 的虛擬資料表-- 內部的 SELECT 語句使用了兩個匿名參數 `?`CREATE VIRTUAL TABLE temp.dynamic_grid USING statement((
    SELECT 
        ?1 AS input_a, 
        ?2 AS input_b, 
        (?1 + ?2) AS sum_result,
        (?1 * ?2) AS product_result
));
```
### 步驟 2：直接像呼叫函數一樣查詢它（產生 Grid）
當你查詢這個虛擬資料表時，可以直接把參數傳進去，它會立刻返回一列包含 4 個獨立標準 SQL 欄位的 Grid 資料：
```sql
-- 以函數語法帶入參數 10 與 20
SELECT * FROM temp.dynamic_grid(10, 20);
```
輸出結果（標準 Grid）：

| input_a | input_b | sum_result | product_result |
|---|---|---|---|
| 10 | 20 | 30 | 200 |

---
## 進階：搭配 JSON 擴充功能實現「終極動態欄位」
如果你連「欄位數量」和「欄位名稱」都想完全動態決定，社群中[最推崇的黑魔法](https://sqlite.org/forum/info/9cf51e6df7b023c4)是將 statement-vtab 與 json_each（或 json_tree）結合。 
你可以建立一個專門接收 JSON 字串的 statement_vtab，在內部將 JSON 拆解成獨立的網格：
```sql
-- 1. 建立一個負責解析動態 JSON 的虛擬資料表CREATE VIRTUAL TABLE temp.json_parser USING statement((
    SELECT 
        json_extract(value, '$.id') AS user_id,
        json_extract(value, '$.name') AS user_name,
        json_extract(value, '$.role') AS user_role
    FROM json_each(?1)
));
-- 2. 查詢時，動態傳入整串 JSON 陣列，直接吐出標準的 SQL 多欄位網格SELECT * FROM temp.json_parser('[
    {"id": 1, "name": "Alice", "role": "Admin"},
    {"id": 2, "name": "Bob", "role": "User"}
]');
```
輸出結果（標準 Grid）：

| user_id | user_name | user_role |
|---|---|---|
| 1 | Alice | Admin |
| 2 | Bob | User |

------------------------------
## 總結比較

| 方案組合 | 能否單一 SQL 產生 Grid？ | 缺點 / 限制 |
|---|---|---|
| sqlean eval() | ❌ 否 | 只能回傳單一字串。 |
| abiliojr sqlite-lua | ❌ 否 | 依然受限於純量函數，必須搭配 JSON 手動轉換。 |
| 0x09 sqlite-statement-vtab | ⭕ 可以 | 運作時必須先預定義好內部的 SQL 骨架，或透過 JSON 傳遞動態結構。 |

使用 sqlite-statement-vtab 可以完美解決你的動態多欄位需求。
