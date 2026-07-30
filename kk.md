這完全指出了 SQLite 延伸模組的底層設計限制！不論是 hoelzro 還是 abiliojr 的版本，它們所註冊的 Lua 函數都只存活在「當前的記憶體連線（Session）」中。一旦資料庫連線中斷（.exit、關閉程式或重啟電腦），記憶體被釋放，這些自訂函數就徹底消失了。

SQLite 本身並沒有像 MySQL 或 PostgreSQL 那樣可以把函數程式碼永久寫入磁碟的「預存程序（Stored Procedure）」功能。

為了達到「一開機就能直接用，不需要每次手動重新註冊」的永久儲存效果，在實際生產環境中，業界一律採用以下三大終極自動化解決方案：

## 方案一：資料庫層級的「永久自動觸發器（Trigger）」心法
這是唯一不需要修改您應用程式程式碼、完全封裝在 .db 檔案內部的解法。
雖然 SQLite 不能直接儲存 Lua 函數，但它可以永久儲存 TRIGGER（觸發器）。
我們可以在資料庫內建立一個永久的觸發器，讓它在資料寫入時，自動去呼叫 Lua 函數進行清洗或計算，並把結果寫回實體欄位中。

* 實戰場景：當新會員寫入 users 表時，自動觸發 Lua 進行文字清洗，並永久存檔。
* 實戰 SQL 語法：
```sql
-- 在資料庫內建立一個永久的觸發器（這個會永久保存在檔案中）
CREATE TRIGGER IF NOT EXISTS auto_clean_user_tag
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    -- 當有新資料進來，自動叫 Lua 處理，並更新回實體欄位
    UPDATE users 
    SET cleaned_tag = lua('return string.upper(string.gsub(..., "[%s%p]", ""))', NEW.raw_tag)
    WHERE id = NEW.id;
END;
```

* 優勢：關閉資料庫後，觸發器依然在磁碟裡。下次開啟時，只要您有 .load ./sqlite-lua，新寫入的資料就會自動被 Lua 處理並永久存檔，查詢時直接讀取 cleaned_tag 欄位即可，不需要再呼叫 Lua。

## 方案二：環境設定檔（.sqliterc）法 —— 適合命令列 CLI
如果您平常主要是使用官方的 sqlite3 命令列工具，您不希望每次進去都要手動打那四五行註冊指令，您可以利用 SQLite 內建的啟動腳本機制。

1.在您的專案根目錄（或使用者家目錄）下，建立一個名為 .sqliterc 的隱藏檔案。
2.將所有初始化與註冊指令直接寫進去：
```text
-- .sqliterc 內容
.load ./sqlite-lua-luajit
SELECT createlua();
SELECT eval_lua(1, 'dofile("logic.lua")');
SELECT register_lua(1, 'CALC_SCORE', 'calc_advanced_score');
SELECT register_lua(1, 'CLEAN_TAG', 'clean_user_tag');
```

3.效果：當您下次在該目錄輸入 sqlite3 mydata.db 開啟資料庫時，SQLite 會在背景自動執行這幾行指令。您一進去就能直接下達 SELECT CALC_SCORE(...)，用起來就跟內建函數一模一樣！

## 方案三：應用程式端的「啟動回呼（Boot-Loader）」封裝 —— 適合後端開發
如果您是用 Python、Node.js 或是 Go 語言在寫系統，最標準的做法是將這段註冊邏輯封裝進資料庫連線池（Connection Pool）的初始化函式中。

以 Python 為例，我們可以寫一個標準的資料庫啟動器（Boot-Loader）：
```python
import sqlite3

def get_db_connection(db_path):
    # 1. 建立實體連線
    conn = sqlite3.connect(db_path)
    conn.enable_load_extension(True)
    cursor = conn.cursor()
    
    # 2. 封裝初始化與註冊邏輯 (每次連線建立時自動在內存中跑一次)
    cursor.execute("SELECT load_extension('./sqlite-lua-luajit');")
    cursor.execute("SELECT createlua();")
    cursor.execute("SELECT eval_lua(1, 'dofile(\"logic.lua\")');")
    cursor.execute("SELECT register_lua(1, 'CALC_SCORE', 'calc_advanced_score');")
    
    return conn

# ---- 之後在主程式裡直接使用，完全感受不到它消失過 ----
conn = get_db_connection("my_big_data.db")
cursor = conn.cursor()

# 隨時快樂調用自訂函數！
cursor.execute("SELECT id, CALC_SCORE(price, years, status) FROM orders")
print(cursor.fetchall())
```

## 💡 總結：為什麼 SQLite 要這樣設計？
SQLite 的本質是 一個乾淨的單一檔案（Single File）儲存庫。它在磁碟檔案中只記錄標準的 SQL 結構（資料表、索引、觸發器、視圖），而不希望檔案內部夾帶特定程式語言（如 Lua、Python）的內存虛擬機狀態，這也是它能保持絕對輕量與跨平台（100% Portable）的關鍵。

透過「方案一（永久觸發器）」或「方案三（連線自動載入）」，您就能完美繞過這個記憶體限制，讓您的自訂 Lua 函數達到「名義上的永久儲存」！

這三種自動化配置是業界對付 SQLite 內存函數的核心心法。
