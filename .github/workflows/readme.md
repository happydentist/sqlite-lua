# 📝 SQLite-Lua 延伸模組全平台靜態編譯技術筆記

## 🎯 最終編譯目標
1. **純靜態連結（Static Linking）**：編譯產出的 `.dll` 與 `.so` 必須完全獨立，內部直接鎖死 Lua 虛擬機引擎，部署到任何環境皆**免安裝 Lua 依賴庫**。
2. **自動化建構**：完全透過 **GitHub Actions** 實現跨平台手動觸發（`workflow_dispatch`）編譯與封裝。
3. **高效能雙版本**：同時提供「標準 Lua 5.3 版」與「極速 LuaJIT 版」。

---

## 🛠️ 過程中遭遇的核心問題與終極解決方案

### 問題一：預設 Makefile 跨平台相容性差，手動覆蓋參數時頻繁噴錯 (`Exit code 2`)
* **編譯情境**：嘗試直接在 GitHub Actions 中使用專案自帶的 `Makefile` 並透過變數傳遞靜態庫路徑。
* **原因分析**：專案自帶的 `Makefile` 結構較為老舊，且 Windows (MSYS2/MinGW) 與 Linux (Ubuntu GCC) 的工具鏈語法差異極大，強行修改變數會觸發 `make` 自身的語法攔截。
* **終極解法**：**直接跳過 Makefile，改用直球對決的 GCC 原生單行指令。**
  * 精準透過 `-I` 指定標頭檔目錄。
  * 手動串接各平台對應的核心參數（如 Windows 必備的 `-static-libgcc` 與 `-Wl,--export-all-symbols`），徹底擺脫 Makefile 的環境包袱。

### 問題二：Linux 平台在打包動態庫時被連結器攔截並拒絕 (`Exit code 1`)
* **錯誤日誌**：`relocation R_X86_64_PC32 against symbol can not be used when making a shared object; recompile with -fPIC`
* **原因分析**：Ubuntu APT 套件庫預設提供的 Lua 靜態庫（`liblua5.3.a`）在當初 Linux 官方編譯時**沒有加上 `-fPIC`（位置無關代碼）參數**。Linux 系統安全機制嚴格禁止將「非 -fPIC」的靜態庫壓入動態模組（`.so`）中。
* **終極解法**：**改採離線源碼即時編譯。**
  * 在 GitHub Actions 記憶體中透過官方 `actions/checkout` 直接拉取對應的 Lua/LuaJIT 官方儲存庫。
  * 在 Linux 環境下自行以 `gcc -O2 -fPIC -c *.c` 編譯出「100% 帶有 -fPIC」的專屬 `liblua.a` 靜態庫，連結器即可順利通過。

### 問題三：網路下載不穩定導致編譯斷頭 (`Exit code 23` / `No such file`)
* **錯誤日誌**：`tar (child): lua-5.3.6.tar.gz: Cannot open: No such file or directory`
* **原因分析**：早期使用 `curl` 或 `wget` 聯外下載 Lua 官方壓縮包，由於 GitHub Actions 的 Docker 安全隔離或 CDN 跳轉（301 Moved Permanently），導致只抓到首頁 HTML（`index.html`）而非真正的壓縮檔，後續解壓找不到檔案。
* **終極解法**：**全面改用 GitHub 基礎架構。**
  使用 `actions/checkout` 官方套件，直接將 `lua/lua` (v5.3.6) 或 `LuaJIT/LuaJIT` (v2.1) 以 Submodule/鏡像形式拉取到內部目錄，速度極快且 0% 斷網風險。

### 問題四：LuaJIT 高效能版本編譯時，API 嚴重衝突卡死
* **錯誤日誌**：`error: 'LUA_OPEQ' undeclared (first use in this function); did you mean 'LUA_NOREF'?`
* **原因分析**：這兩個 SQLite-Lua 專案的原始碼使用了 `lua_compare` 與 `LUA_OPEQ`，這是 Lua 5.2/5.3 引入的現代 API。而 LuaJIT 本質上是基於 Lua 5.1 語法開發的。即便 LuaJIT 原始碼內部有寫 5.2 相容開關，但往往會因為 Makefile 參數被外層 OS 環境剝離（如 Linux 的 `CFLAGS` 覆蓋）而失去作用。
* **終極解法**：**使用程式碼層級巨集補完（Polyfill）。**
  在 GCC 編譯延伸模組前，透過 `sed` 直接在專案的 `lua.c` 第一行，強行塞入兩行原生轉譯巨集：
  ```c
  #define LUA_OPEQ 0
  #define lua_compare(L,i1,i2,op) (op == LUA_OPEQ ? lua_equal(L,i1,i2) : 0)
  ```
  當 GCC 編譯到原作者的代碼時，會自動將 5.2 的比較語法展開成 LuaJIT 100% 完美的內建老函式 `lua_equal`。不需動用複雜的 Makefile 修改，一刀切斷所有相容性錯誤。

---

## 📈 總結：最佳實踐架構心法
透過這一連串的修補，我們歸納出一套**編譯老舊 C 語言動態延伸模組**的最穩架構：
1. **源碼不落地**：第三方依賴（如 Lua/LuaJIT）一律透過 GitHub Actions Checkout，不走外部 Wget。
2. **跳過建構工具**：不依賴專案自帶的 Makefile/CMake，改用精準的 GCC 單行指令手動控制。
3. **Polyfill 優先**：遇到編譯器版本衝突、未定義巨集時，直接在 Action 階段動態往原始碼頂端 `#define` 補完，避免手動重寫代碼，保持專案乾淨度。
