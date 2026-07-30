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

