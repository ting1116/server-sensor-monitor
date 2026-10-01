# Server Sensor Monitor 專案規格

文件版本：0.1　｜　日期：2026-10-01　｜　目標產品版本：1.0.0

本文件定義一個用於學習工程自動化的 C++ 命令列專案。程式讀取模擬伺服器感測器資料，依測試規格檢查溫度與風扇轉速；工程流程涵蓋 Git 分支、Pull Request、Docker、建置、測試、靜態分析、報告保存與版本發布。

這是一份待實作的設計規格，不代表功能或 CI 已完成。以下流程是針對個人專案選定的工程做法，不宣稱所有公司採用相同工具。文件使用 **MUST** 表示 v1.0 必須完成，**SHOULD** 表示建議，**OPTIONAL** 表示延伸項目。

## 1 專案目的與成功標準

主要目標是實作可追溯的自動化驗證與交付流程。C++ 程式保持小而完整，讓開發者能理解每個失敗發生在哪一層，並從 CI 紀錄重現問題。

完成後應能展示：

1. 從需求及驗收條件建立 issue，在 feature branch 完成變更並提出 PR。
2. 本機與 CI 使用相同的 Docker 建置與測試環境定義。
3. PR 自動編譯、執行 unit tests、執行 CLI 整合驗證與靜態分析。
4. 不符合規格的變更無法通過合併門檻；失敗時仍可取得診斷資料。
5. 版本 tag 對應明確的 source commit，通過檢查後產生經測試的 release package。
6. 取得 package 的人能依文件執行，並確認版本、checksum 與測試結果。

不以安裝工具數量、程式碼行數或「所有 CI 都是綠色」作為成功標準。必須演示至少一次合理的失敗、定位、修正與重新通過。

## 2 範圍與使用情境

### 2.1 v1.0 必要範圍

| 領域 | 交付內容 |
|---|---|
| 應用程式 | C++17 CLI，一次分析一份 sensor snapshot |
| 感測器 | CPU_TEMP、GPU0_TEMP、GPU1_TEMP、FAN0_RPM、FAN1_RPM |
| 設定 | JSON profile 定義必備感測器、有效範圍與 warning／critical 門檻 |
| 輸出 | 人可閱讀的 text 與供 automation 使用的 JSON |
| 測試 | GoogleTest unit tests、CTest、Python CLI integration tests |
| 環境 | Docker 建置與測試 image；Linux x86_64 |
| CI | GitHub Actions、PR required check、報告與紀錄保存 |
| 交付 | Linux tar.gz、SHA256SUMS、build metadata、GitHub Release |
| 文件 | README、spec、測試計畫、故障排查紀錄、CHANGELOG |

### 2.2 v1.0 不涵蓋

不連接真實 BMC，不執行 ipmitool／Redfish，不控制風扇、不自動關機；不建立 Web UI、資料庫、即時告警服務或 Kubernetes。名稱保留 monitor，但 v1.0 是單次 snapshot 檢查；長時間 polling 與趨勢監控列為後續功能。

也不以此工具認證硬體安全或可靠度。下文門檻是模擬案例，真實設備必須依型號、元件規格與測試條件制定 profile；例如風扇 0 RPM 並不在所有產品模式下都代表故障。

### 2.3 三個主要角色

| 角色 | 需求 |
|---|---|
| 開發者 | 修改程式後快速確認沒有破壞既有行為 |
| 驗證工程師 | 用多組資料核對實際結果，取得 expected／actual 差異 |
| 工具使用者 | 下載可用版本，輸入 log，取得明確判斷與錯誤原因 |

個人專案由同一人扮演這些角色，但文件與自動化仍保留角色間的交付介面。

## 3 系統架構與責任分工

| 模組 | 責任 | 不應承擔的工作 |
|---|---|---|
| CLI | 解析參數、協調流程、設定 exit code | 不直接寫入所有門檻判斷 |
| Profile loader | 讀取 JSON、驗證 schema 與門檻關係 | 不讀取 sensor log |
| Sensor parser | 讀取 KEY=VALUE、定位行號、驗證數值 | 不決定整體 PASS／FAIL |
| Evaluator | 根據 profile 產生每顆 sensor 的狀態 | 不操作檔案或 stdout |
| Reporter | 產生 text／JSON | 不改變評估結果 |
| Python validator | 啟動 CLI、比對預期結果、產生測試報告 | 不重新實作一份門檻演算法當答案 |
| CI workflow | 安排環境、呼叫共用腳本、保存結果 | 不在 YAML 中複製整套測試邏輯 |

Evaluator 應能以資料結構作為輸入與輸出，方便不依賴檔案的 unit tests。Sensor、Profile、CheckResult 等型別的具體 C++ 設計由實作決定；外部 CLI、JSON 與退出碼遵守本文件。

## 4 CLI 介面契約

執行檔統一命名為 `server-sensor-monitor`，repository 也使用相同名稱。

```
server-sensor-monitor --input data/normal.txt --profile config/demo-server.json
server-sensor-monitor --input data/fan-failure.txt --profile config/demo-server.json --format json
server-sensor-monitor --help
server-sensor-monitor --version
```

| 參數 | 必要性 | 行為 |
|---|---|---|
| `--input PATH` | 檢查模式必填 | 指定一份 sensor snapshot |
| `--profile PATH` | 檢查模式必填 | 指定明確的 profile，避免依賴工作目錄中的隱藏預設值 |
| `--format text\|json` | 選填 | 預設 text |
| `--help` | 獨立模式 | 印出用法，exit 0，不讀取檔案 |
| `--version` | 獨立模式 | 印出產品版本與 commit，exit 0，不讀取檔案 |

MUST 拒絕未知旗標、重複旗標、缺少旗標值、額外 positional arguments、不支援的 format。help／version 必須各自單獨使用；與其他參數混用視為用法錯誤。檔案路徑可含空白，由 shell 引號及程式參數陣列正確處理。

參數解析成功後，依序驗證 profile，再讀取及驗證 input，最後評估與輸出。CLI 用法錯誤以 text 寫入 stderr；參數解析成功且指定 JSON 後，其餘可預期錯誤也必須產生 JSON。

### 4.1 退出碼

| Code | 分類 | 意義 |
|---|---|---|
| 0 | 正常完成 | overall_status 為 PASS，或 help／version 成功 |
| 1 | 正常完成但資料未達標 | overall_status 為 WARN 或 FAIL |
| 2 | 無法完成評估 | 參數錯誤、讀檔失敗、profile 錯誤、sensor 資料無效 |
| 3 | 程式內部錯誤 | 非預期且已攔截的例外 |

OS signal、程序被終止或 crash 可能不遵守以上退出碼，Python validator 必須另外識別，不能將其當成合法的「sensor 異常」。

### 4.2 stdout 與 stderr

text 模式：有效資料的逐 sensor 結果及總結寫入 stdout；輸入或執行錯誤寫入 stderr，stdout 留空。

JSON 模式：stdout 只輸出一份 JSON object 加換行，不混入 debug 訊息、進度、ANSI 顏色碼。錯誤時輸出結構化 error object，stderr 可補充診斷文字。automation 的判斷以 JSON 欄位及退出碼為準，不依賴 stderr 文句。

## 5 Sensor 輸入格式

一份檔案代表同一台模擬 server 的一個 snapshot，單位不寫在數值中。

```
# Demo server snapshot
CPU_TEMP=72
GPU0_TEMP=65
GPU1_TEMP=91
FAN0_RPM=8200
FAN1_RPM=0
```

### 5.1 語法規則

- MUST 支援 UTF-8 文字、LF／CRLF，以及檔尾沒有換行。
- 忽略空白行與去除前置空白後以 `#` 開頭的整行註解；不支援 inline comment。
- 去除 key、value 兩側的空格與 tab；key 區分大小寫。
- 每個資料行恰好一個 `=`，兩側不可為空。
- key 必須存在於指定 profile；重複 key、未知 key 與缺少必要 sensor 都是錯誤，不覆蓋、不忽略。
- 溫度語法為 `-?[0-9]+(\.[0-9]+)?`；RPM 為 `[0-9]+`。不接受科學記號、逗號小數點、單位後綴、NaN、Inf、前置加號或未完整解析的字串。
- 數值須可表示且為有限值；溫度以 C++ `double` 處理，RPM 以整數處理。比較前不得先將數值四捨五入。
- 檔案不得超過 1 MiB，單行不得超過 1,024 bytes（不含換行）。超過即回報 INPUT_LIMIT_EXCEEDED。
- 空檔案或只有註解的檔案回報 EMPTY_INPUT。
- UTF-8 BOM／NUL／非法 UTF-8 不支援並回報 INPUT_FORMAT_ERROR。

語法、資料範圍或完整性有錯時，不產生部分 PASS。先完成輸入驗證，再執行評估。輸入資料行按行號驗證並在第一個錯誤停止；缺少必要 sensor 在讀完整份檔案後依 profile 順序回報第一項。profile 自身錯誤優先於 input 錯誤。

### 5.2 固定的 v1.0 sensor inventory

profile 的 sensors array 必須包含下列五項，恰好各一次；不接受第六項。v1.0 可調整門檻，但尚未提供任意新增 sensor 的通用平台。

| Sensor | Kind | Unit | demo profile 有效數值範圍 |
|---|---|---|---|
| CPU_TEMP | temperature | C | -40 至 150，含端點 |
| GPU0_TEMP | temperature | C | -40 至 150，含端點 |
| GPU1_TEMP | temperature | C | -40 至 150，含端點 |
| FAN0_RPM | fan | RPM | 0 至 30000，整數、含端點 |
| FAN1_RPM | fan | RPM | 0 至 30000，整數、含端點 |

有效範圍是辨識輸入資料是否可信的規則，warning／critical 則是有效資料是否符合測試條件的規則。例如 120 C 是有效但 critical；999 C 是無效資料，整份 input 回報 ERROR。

## 6 Profile 與判斷規則

`config/demo-server.json` 必須完整列出五顆 sensor。以下是單顆 temperature sensor 的結構示意，並非可直接使用的完整 profile：

```json
{
  "schema_version": 1,
  "profile_id": "demo-server-v1",
  "sensors": [
    {
      "name": "CPU_TEMP",
      "kind": "temperature",
      "unit": "C",
      "valid_min": -40,
      "valid_max": 150,
      "warning_high": 85,
      "critical_high": 95
    }
  ]
}
```

fan 項目使用 `warning_low` 和 `critical_low`，不使用 high 欄位。所有頂層及 sensor 層級欄位均為必填；拒絕未知欄位、重複 JSON member、未知 schema version、錯誤型別、非有限或超出可表示範圍的數字。`profile_id` 為 1–64 字元的 `[A-Za-z0-9._-]+`。profile 檔案也以 1 MiB 為上限。

溫度必須滿足 `valid_min < warning_high < critical_high <= valid_max`；fan 必須滿足 `0 <= valid_min <= critical_low < warning_low <= valid_max`，所有 fan 數值均為整數。name／kind／unit 必須與固定 inventory 對應。缺少欄位不能靜默套用預設值。

### 6.1 demo 門檻

| Sensor | OK | WARNING | CRITICAL |
|---|---|---|---|
| CPU_TEMP | value < 85 | 85 <= value < 95 | value >= 95 |
| GPU0_TEMP、GPU1_TEMP | value < 85 | 85 <= value < 95 | value >= 95 |
| FAN0_RPM、FAN1_RPM | value >= 3000 | 1000 < value < 3000 | value <= 1000 |

先檢查有效範圍，再依上述條件分類。所有門檻只由 profile 取得，程式中不得另外硬編碼一套數值。

### 6.2 整體狀態

| 條件 | overall_status | exit code |
|---|---|---|
| 全部 OK | PASS | 0 |
| 至少一個 WARNING，沒有 CRITICAL | WARN | 1 |
| 至少一個 CRITICAL | FAIL | 1 |
| profile／input 無效，無法評估 | ERROR | 2 |
| 已攔截的內部錯誤 | ERROR | 3 |

v1.0 固定採取 warning 也回傳非零的策略，不增加 warning-as-error 開關。缺資料不是 PASS；無法評估的 ERROR 也不應被描述為已證實硬體故障。

## 7 輸出格式與機器介面

### 7.1 Text

對第 5 節的有效 input，依固定 inventory 順序輸出：

```
[OK] CPU_TEMP: 72 C
[OK] GPU0_TEMP: 65 C
[WARNING] GPU1_TEMP: 91 C
[OK] FAN0_RPM: 8200 RPM
[CRITICAL] FAN1_RPM: 0 RPM
Overall: FAIL
```

顯示順序不因 input 行順序改變。text 模式可改善人類可讀性，但 JSON 才是穩定的 automation 介面。

### 7.2 JSON

下例是含一筆 sensor 的結構示意；實際成功評估必須輸出全部五筆：

```json
{
  "schema_version": 1,
  "tool_version": "1.0.0",
  "commit_sha": "<full-source-commit-sha>",
  "profile_id": "demo-server-v1",
  "overall_status": "FAIL",
  "sensors": [
    {
      "name": "FAN1_RPM",
      "value": 0,
      "unit": "RPM",
      "status": "CRITICAL",
      "reason_code": "AT_OR_BELOW_CRITICAL"
    }
  ],
  "error": null
}
```

所有頂層欄位必須存在。sensors 順序固定為 CPU、GPU0、GPU1、FAN0、FAN1。value 為 JSON number；RPM 的 value 為整數。正常 reason_code 為 `WITHIN_LIMITS`；temperature 的非正常代碼為 `AT_OR_ABOVE_WARNING`／`AT_OR_ABOVE_CRITICAL`；fan 為 `BELOW_WARNING`／`AT_OR_BELOW_CRITICAL`。

ERROR 時 sensors 必須為空 array，error 為 object；profile 驗證成功前 profile_id 為 null，之後使用實際值。例如：

```json
{
  "schema_version": 1,
  "tool_version": "1.0.0",
  "commit_sha": "<full-source-commit-sha>",
  "profile_id": "demo-server-v1",
  "overall_status": "ERROR",
  "sensors": [],
  "error": {
    "code": "INPUT_FORMAT_ERROR",
    "message": "Invalid numeric value for GPU1_TEMP",
    "sensor": "GPU1_TEMP",
    "line": 4
  }
}
```

error 的 code、message、sensor、line 欄位必須存在；sensor／line 不適用時為 null，line 採 1-based。message 供人閱讀，不作為精確字串相等的測試契約。

穩定錯誤代碼至少包含 PROFILE_READ_ERROR、PROFILE_INVALID、INPUT_READ_ERROR、INPUT_LIMIT_EXCEEDED、INPUT_FORMAT_ERROR、EMPTY_INPUT、DUPLICATE_SENSOR、UNKNOWN_SENSOR、MISSING_SENSOR、VALUE_OUT_OF_RANGE、INTERNAL_ERROR。無法辨識有效 key 的行回報 INPUT_FORMAT_ERROR；有效但不在 inventory 的 key 回報 UNKNOWN_SENSOR。

在 `docs/output-schema.json` 提供 JSON Schema；自動驗證需核對必填欄位、型別、enum、sensor 數量及 exit code 與狀態的一致性。不依賴 JSON object 的 key 順序或空白格式。

## 8 測試設計

### 8.1 測試層級

| 層級 | 工具 | 驗證目標 |
|---|---|---|
| Unit | GoogleTest、CTest | parser、profile validation、邊界判斷、overall aggregation、JSON serializer |
| Integration | Python subprocess | 真正執行 CLI，驗證參數、檔案、stdout、stderr、exit code |
| Automation 自身測試 | Python unittest | timeout、crash、錯誤 JSON、缺案例、報告失敗等 harness 行為 |
| Package smoke | Python 與乾淨 runtime container | 解壓後的正式 binary 與相對路徑 profile 能實際使用 |

不能只測函式，也不能只印出 stdout 卻沒有 assertions。測試預期值必須由規格明確寫在 fixtures／manifest 中，不能呼叫被測程式自身的判斷邏輯產生 expected result。

### 8.2 最低測試矩陣

所有單一異常案例，其餘四個 sensor 使用正常值。溫度邊界參數化套用 CPU、GPU0、GPU1；fan 邊界套用 FAN0、FAN1。

| ID | 案例 | 預期 |
|---|---|---|
| TC-01 | 五顆正常 | PASS、exit 0、五筆 OK |
| TC-02 | 溫度 84.9／85／94.9／95 | OK／WARNING／WARNING／CRITICAL，exit 0／1／1／1 |
| TC-03 | fan 3000／2999／1001／1000／0 | OK／WARNING／WARNING／CRITICAL／CRITICAL |
| TC-04 | warning 與 critical 同時存在 | FAIL，兩者均被保留 |
| TC-05 | 只有 warning | WARN、exit 1 |
| TC-06 | 五顆 sensor 各自缺少 | MISSING_SENSOR、exit 2 |
| TC-07 | 重複 key，不論數值相同與否 | DUPLICATE_SENSOR、exit 2 |
| TC-08 | 未知 key／大小寫不符 | UNKNOWN_SENSOR、exit 2 |
| TC-09 | NaN、Inf、abc、72C、1e2、空 value | INPUT_FORMAT_ERROR、exit 2 |
| TC-10 | 負 RPM／小數 RPM | INPUT_FORMAT_ERROR、exit 2 |
| TC-11 | 溫度 -40、150 與 fan 30000 | 有效且依門檻分類 |
| TC-12 | 溫度 -40.1、150.1 或 fan 30001 | VALUE_OUT_OF_RANGE、exit 2 |
| TC-13 | 空檔案／只有註解 | EMPTY_INPUT、exit 2 |
| TC-14 | input 不存在／無法讀取 | INPUT_READ_ERROR、exit 2 |
| TC-15 | 空白行、註解、CRLF、檔尾無換行 | 與等價標準 input 相同結果 |
| TC-16 | 行順序改變 | 相同逐 sensor 結果及固定輸出順序 |
| TC-17 | profile 不存在／JSON 不合法 | PROFILE_READ_ERROR／PROFILE_INVALID、exit 2 |
| TC-18 | 門檻顛倒、欄位缺漏、未知欄位、重複 member | PROFILE_INVALID、exit 2 |
| TC-19 | 修改 profile warning 為 80，input 為 82 | WARNING，證明沒有硬編碼 85 |
| TC-20 | help／version／缺少參數／未知旗標 | 依第 4 節規定 |
| TC-21 | input／profile 路徑包含空白 | 正常解析並執行 |
| TC-22 | 超大檔案／超長行／NUL／非法 UTF-8 | 對應格式或大小錯誤，沒有 crash |
| TC-23 | text 與 JSON 使用同一份資料 | 分類與 exit code 相同 |
| TC-24 | JSON 欄位與型別、status／exit 一致性 | 符合 schema 及本規格 |
| TC-25 | stub 程序超時、crash、輸出壞 JSON | validator 回報 ERROR，自身非零退出 |
| TC-26 | 故意寫錯 expected status | validator 回報 FAIL，自身非零退出 |
| TC-27 | 空 manifest、重複 case ID、報告無法寫入 | validator 回報 infrastructure error，非零退出 |
| TC-28 | 解壓 release package 執行 normal／critical／invalid | 正確輸出與 0／1／2 退出碼 |

數值比較直接依規格的包含／不包含端點處理，不任意加 epsilon 改變門檻。另加高精度小數案例，防止顯示四捨五入反過來影響結果。

### 8.3 Python validator 契約

建議介面：

```
python3 scripts/run_validation.py \
  --binary build/server-sensor-monitor \
  --cases tests/integration/cases.json \
  --report-dir reports/integration \
  --timeout-seconds 5
```

cases.json 每筆包含唯一 id、input 相對路徑、profile 相對路徑、expected_exit_code、expected_overall_status、expected_sensor_statuses 或 expected_error_code。路徑以 manifest 所在目錄解析，不依賴呼叫者的目前目錄。無效 input 案例可故意指向不存在的 input；binary 與 manifest 本身不存在則是 harness 錯誤。

MUST 使用 subprocess 參數陣列與 `shell=False`，設定每案 timeout，擷取 stdout／stderr。對預期 exit 1／2 的案例不要直接使用 `check=True` 把它當成測試失敗。

每個案例必須分類為 PASS、FAIL 或 ERROR：符合所有 assertions 為 PASS；程式完成但結果不符為 FAIL；timeout、signal、無法啟動、JSON 無法解碼等為 ERROR。單案失敗後繼續執行其他可執行案例，不自動重試來把結果洗成綠色。

validator 自身退出碼：全部案例 PASS 且報告成功寫出為 0；任何案例 FAIL／ERROR 為 1；manifest 或 harness 基礎設施錯誤為 2。零案例絕不能算成功。

產生 summary.json、junit.xml 及每案 stdout／stderr。JUnit 的 assertion mismatch 對應 failure；timeout／crash 等對應 error。報告包含 case ID、預期及實際值、退出碼、耗時、run ID、source commit、工具版本及 profile／input checksum。不存在的 input checksum 為 null 並註明原因。

每案最多保存各 64 KiB stdout／stderr，超過時截斷並標記。為避免失控輸出造成記憶體耗盡，harness 應串流至暫存檔或採 bounded capture；不要無限制累積在 RAM。所有測試資料均使用合成資料。

## 9 建置與依賴管理

| 項目 | v1.0 決策 |
|---|---|
| 語言 | C++17，關閉 compiler extensions |
| 基準環境 | Ubuntu 22.04 x86_64 container、GCC 11 系列 |
| Build | CMake >= 3.22、Ninja；使用 target-based 設定 |
| Unit tests | GoogleTest，固定到明確版本／commit |
| JSON | nlohmann/json，固定到明確版本／commit |
| Python | 3.10 以上；標準函式庫為主，schema validator 依賴固定版本 |
| 靜態檢查 | cppcheck；指定 C++17 與 Linux 64-bit 平台 |
| 產物 | Linux x86_64 executable，動態依賴需列在 release 文件 |

這是相容於目前 Ubuntu 22.04 練習環境的基準，不是對最新工具版本的宣稱。SHOULD 本機繼續使用既有 g++、CMake、GDB 學習；正式 CI gate 以 Docker 中的基準環境為準。

依賴版本、commit／archive SHA256、來源 URL 與 license 記錄於 `docs/dependencies.md`。CMake FetchContent 若用 archive，必須驗證 hash；若用 Git 必須固定 commit，不追蹤 main。只在 image／configure 階段取得依賴，測試執行階段不需要網路。

針對本專案 targets 啟用 `-Wall -Wextra -Wpedantic`，CI 將 compiler warnings 視為 errors。不要把自身嚴格警告規則直接套到第三方函式庫。

建置目錄與原始碼分離。CMake 匯出 `compile_commands.json` 供分析使用。所有實際工具版本在 CI 開始時記錄到 `toolchain.txt`。

CTest 以非零退出表示失敗，必須設定「沒有找到測試」也失敗，並輸出 JUnit；可使用 `--output-on-failure --no-tests=error --output-junit`。目錄需由腳本先建立。參考來源 [S5]。

v1.0 不宣稱 bit-for-bit reproducible build。固定 base image digest 仍不足以固定後續 apt repository 的內容；每次 toolchain build 要記錄套件版本與 image ID。若需要更強重現性，再採套件 snapshot 或固定已發布的 toolchain image digest。

## 10 Docker 規格

Docker 的 v1.0 角色是提供一致的建置與測試工具環境；不必先把程式做成長期運行的服務，也不要求發布 image 到 registry。

MUST 提供 `docker/Dockerfile.toolchain`，包含編譯器、CMake、Ninja、Python、cppcheck 及必要測試依賴。base image 在實作時固定到經查核的完整 digest，旁邊標註可讀的版本。官方建議固定 digest 及透過 CI 測試 image 變更，見 [S2]。

容器執行時：

- 以 host UID／GID 或明確的非 root user 運行，讓輸出檔不被 root 佔有。
- source 與 output 的掛載位置清楚，input fixtures 不可由測試修改；需要產生暫存資料時使用個別暫存目錄。
- 不使用 privileged mode，不掛入 Docker socket，不將主機憑證或整個 home 掛入。
- Python validator 在容器內執行 CLI，不讓它每個案例再啟動一層 Docker。
- 提供 `scripts/ci_local.sh` 作為本機單一入口；GitHub Actions 使用同一套底層腳本。
- `.dockerignore` 排除 `.git`、build、reports、個人環境及機密檔案；source SHA 以明確 build metadata 傳入。

Package smoke test 使用另一個乾淨 runtime image，只有宣告的 runtime libraries，不含編譯器及專案 source。這能找出「在 build container 可用，但使用者下載後缺 shared library」的問題。

## 11 Git 與 PR 工作流程

採用 main 加短期 feature branch。分支可命名 `feat/profile-loader`、`fix/fan-boundary`、`ci/add-validation`、`docs/update-spec`。每個 PR 只解決一個合理範圍的問題。

Issue 必須包含問題、預期行為、驗收條件與關聯測試。PR 必須交代修改原因、行為變化、測試證據、對 CLI／JSON 的相容性影響；改動規格時同時更新 spec 與 fixtures。

main 保護規則 MUST 要求 PR 與 ci-gate 成功，禁止 force push／刪除，要求分支與 main 更新一致後再合併。使用管理員時也不要靠 bypass 通過故意失敗的 CI。

一人練習先不要求另一人的 approving review，避免無人能核准；仍需自我檢查 diff 與 PR checklist。兩人以上再加入至少一位 reviewer。自我檢查不宣稱等於獨立 code review。

GitHub 官方文件指出 GitHub Free 的 public repository 可用 protected branches，private repository 的可用性依方案而異 [S3]。若選 private 且沒有平台強制 gate，要在 README 明確記錄此限制；單純約定不要合併不等於平台保護已啟用。

## 12 CI Pipeline

### 12.1 觸發規則

| Event | 用途 | 是否發布 |
|---|---|---|
| pull_request → main | 合併前檢查；更新 PR 時重跑 | 否 |
| push → main | 合併後檢查實際 main commit | 否 |
| workflow_dispatch | 手動重現 CI 問題 | 否 |
| push tag vMAJOR.MINOR.PATCH | release workflow | 全部 gate 通過後才可 |

不預設每個 feature branch push 與 PR 同時觸發兩套重複 CI。尚未開 PR 時可本機跑或手動觸發。同一 PR 新 commit 可取消舊 run；正式 release 不因其他分支的新 push 被取消。

### 12.2 CI Jobs 與通過條件

| Job | 工作 | 必須成功的條件 |
|---|---|---|
| build-test | 建立 toolchain image、configure、build、unit tests、harness tests、integration tests | 編譯無警告、所有預期測試執行且通過、必要報告存在 |
| static-analysis | 對專案 source 執行 cppcheck | 沒有未抑制的指定類型診斷，分析器自身成功 |
| ci-gate | 彙整上述 job 結果 | 所有必要 jobs 均為 success；skipped／cancelled 不當成成功 |

static-analysis 可與 build-test 平行執行，但它本身仍需正確 configure 或取得包含 include／defines 的 compile database。可在各 job 建置同一 Dockerfile 並利用 cache；cache 僅加速，清空 cache 後也必須能跑通。

build-test 中 unit tests 失敗後，在 build 成功且環境仍可用時 SHOULD 繼續跑 integration 以提供更多診斷，最終 job 仍失敗。shell 腳本要明確保存各步 exit status；不能用 `|| true` 或無條件 `continue-on-error` 吞掉結果。

ci-gate 必須在上游失敗時仍有機會執行並回報失敗，可採 `always()` 配合明確檢查 needs results。required workflow 不用 paths filter 略過整條 workflow，避免 required check 永遠 pending。與 release workflow 的 job 名稱保持可區別。

### 12.3 Static analysis policy

v1.0 啟用 cppcheck 的 error 及 warning、performance、portability 檢查；style／information 可先作為資訊輸出。所有啟用的 blocking finding 必須導致非零退出，明確設定 `--error-exitcode=1`，不可只列印警告卻讓 CI 通過。實作時核對選定版本的 `cppcheck --help`。

分析 src／include 中本專案程式碼；排除 third-party source 與 generated code，但保留正確的 include 及 macro 設定。抑制項必須限定診斷 ID／位置，記錄原因與 issue，不接受全面 suppress。

MUST 用一段明確會觸發所選規則的暫時性錯誤做演練，確認 workflow 真的會變紅，再修復；不把有缺陷程式碼合併進 main。

### 12.4 失敗、逾時與報告

- 每個 CI job timeout 設為 15 分鐘；冷建置若超時需先調查，再以 PR 調整。
- 每個 unit test 最長 10 秒；每個 CLI integration case 最長 5 秒。
- build／test 失敗後仍嘗試保存已有 reports，適用的上傳步驟使用 always 條件；runner 被強制終止時不能保證有完整報告。
- 正常成功路徑若缺少必要 junit.xml／summary.json，必須 gate 失敗；早期 build failure 的 missing test report 應標註「未執行」，不要造出零失敗報告。
- 每次 run 的摘要列出 commit、run URL、image ID、測試總數、PASS／FAIL／ERROR 與失敗案例。
- artifacts 名稱包含 run ID／attempt 或 commit，避免互相覆蓋。一般 CI 報告保留 14 天，並受 repository policy 上限限制 [S4]。
- 不自動重跑功能測試以掩蓋 flaky test。若是外部下載中斷，標記 infrastructure failure，再重跑整個 run，保留原失敗紀錄。

### 12.5 Workflow 權限

CI 預設 `contents: read`；只有 release 的 publish job 取得 `contents: write`。外部 Actions 固定到完整 commit SHA，註解標出所對應版本，更新需經 PR。GitHub 官方文件建議以完整 SHA 固定 Action [S1]。

使用 `pull_request` 執行未信任的 PR 程式碼；不以 `pull_request_target` 搭配 checkout PR code 取得高權限。一般 PR 不注入 secrets，不執行於可連線到私人設備的 self-hosted runner。第三方報告上傳服務不屬於 v1.0 必要項目。

## 13 Release 與 Continuous Delivery

這個專案的 CD 定義為產出並發布經驗證的工具版本；不是自動部署到實際 server。建立版本 tag 是人工決定發布的動作，後續自動執行。

### 13.1 發布前置條件

1. 發布相關程式、VERSION、CHANGELOG 已透過 PR 合併到 main。
2. tag 必須精確符合 `vMAJOR.MINOR.PATCH`；例如 `v1.0.0`，版本數字除 0 外不使用前置零。
3. tag 解引用後的 commit 必須可從 origin/main 追溯到；不得在未合併 feature commit 上發布。
4. tag 的版本與 VERSION 完全一致。正式包的 `--version` 不可顯示 unknown／dirty。
5. release workflow checkout tag 的 commit，對同一 commit 重新執行完整必要檢查。不能只信任 main 曾經綠過。

### 13.2 發布步驟

驗證 tag 後，使用 Release build 編譯並跑 unit／integration／static gates。打包此輪已通過測試的 binary，不在 publish 階段另行重編。接著解壓 package，在乾淨 runtime container 執行 package smoke tests；成功後計算 checksum、上傳同一 package 並發布 GitHub Release。

必要時使用 draft Release 先上傳完整資產，確認所有資產成功才公開；上傳一半失敗時不得留下看似完整的正式版。可重跑未發布的 draft，已正式發布的相同版本不得靜默替換檔案或移動 tag；修改應發布新的 patch version。

### 13.3 正式交付物

`server-sensor-monitor-v1.0.0-linux-x86_64.tar.gz` 包含：

| 路徑 | 用途 |
|---|---|
| bin/server-sensor-monitor | 正式 executable，保留執行權限 |
| config/demo-server.json | 完整示範 profile |
| examples/normal.txt | 可立即驗證的資料 |
| README.md | 安裝、執行、退出碼、相容環境 |
| CHANGELOG.md | 此版本異動 |
| build-info.json | version、commit、toolchain image ID、工具版本、UTC build time、target、profile SHA256 |
| LICENSE、THIRD_PARTY_NOTICES | 專案選定授權與依賴授權資訊 |

Release 額外附 SHA256SUMS、驗證報告與 release notes。checksum 用於偵測檔案變更，不宣稱等同來源簽章。GitHub Actions artifact 是工作流程暫存結果，GitHub Release asset 是版本交付檔，兩者不可混為一談。

只宣稱支援實際測試過的 Linux x86_64 runtime 基準，初版為 Ubuntu 22.04；不宣稱 Linux executable 能在 Windows／macOS 原生執行，也不宣稱所有 Linux 發行版都相容。

### 13.4 發布失敗與回復

若 tag 驗證、測試或 smoke test 失敗，停止 publish 並保存診斷。已發布版本發現問題時，在 release notes 註明、指引使用上一個已知正常版本，再透過修復 PR 發布 patch。未部署到 server，因此此處的 rollback 是回到上一版工具，不是恢復伺服器系統狀態。

## 14 Repository 結構

使用以下路徑分工；初期里程碑不需要先建立所有空檔案。

| 路徑 | 內容 |
|---|---|
| src/main.cpp | CLI 入口 |
| src/sensor_parser.cpp | 文字輸入解析 |
| src/profile.cpp | profile 載入與驗證 |
| src/evaluator.cpp | sensor 狀態判斷 |
| src/reporter.cpp | text／JSON 輸出 |
| include/server_sensor_monitor/ | 對應 headers 與資料型別 |
| tests/unit/ | C++ unit tests |
| tests/integration/cases.json | 預期結果 manifest |
| tests/integration/fixtures/ | 正常、異常、錯誤格式測試資料 |
| tests/automation/ | validator 自身測試與 stub 程序 |
| data/ | 供使用者手動操作的例子 |
| config/demo-server.json | 完整 demo profile |
| scripts/build.sh | configure 與 build |
| scripts/test.sh | unit／harness tests |
| scripts/run_validation.py | CLI integration validation |
| scripts/static_analysis.sh | cppcheck policy |
| scripts/ci_local.sh | 本機 Docker 驗證入口 |
| scripts/package.sh | 打包已驗證的 binary |
| scripts/smoke_package.py | 對解壓產物驗證 |
| docker/Dockerfile.toolchain | 建置與測試環境 |
| docker/Dockerfile.runtime-test | 乾淨 runtime smoke 環境 |
| .github/workflows/ci.yml | PR／main／手動 CI |
| .github/workflows/release.yml | tag 發布流程 |
| .github/pull_request_template.md | PR checklist |
| docs/spec.md | 本文件 |
| docs/test-plan.md | 測試與需求對照 |
| docs/output-schema.json | CLI JSON schema |
| docs/dependencies.md | 依賴來源、版本、hash、授權 |
| docs/runbook.md | 失敗排查與重現命令 |
| docs/decisions/ | 重要設計決策及原因 |
| CMakeLists.txt、VERSION | 建置與產品版本 |
| README.md、CHANGELOG.md、LICENSE | 專案入口、版本異動及授權 |
| .gitignore、.dockerignore | 排除 build、reports、暫存與個人環境 |

build、reports、dist 由腳本產生且不提交 Git。正式 release 檔案由發布流程保存，不把每次 executable commit 進 source repository。

## 15 非功能需求與可追溯性

| ID | 要求 | 驗收方法 |
|---|---|---|
| NFR-01 | 相同 input／profile 產生相同分類與退出碼 | 重複執行與改變行順序 |
| NFR-02 | JSON stdout 不受 locale 或 debug logs 汙染 | parse JSON、驗證 schema |
| NFR-03 | 沒有網路也能執行已安裝工具與測試階段 | 容器斷網測試；依賴取得階段除外 |
| NFR-04 | 無效輸入不造成未處理例外或 crash | negative fixtures |
| NFR-05 | 每份 report 能追溯 source 與測試資料 | 核對 commit、run、checksum metadata |
| NFR-06 | 不依賴 /home/某人 的絕對路徑 | fresh clone／不同工作目錄驗證 |
| NFR-07 | demo snapshot 在基準 CI 環境 5 秒內完成 | 每案 timeout；不當作高精度效能 benchmark |
| NFR-08 | output 僅寫指定 reports／build／dist 區域 | fixtures 前後 hash 與目錄檢查 |

日常文件與 automation case ID 保持可讀、固定。報告時間使用 UTC；README 可用繁體中文說明，code identifiers／machine enum 使用英文。任何 JSON 欄位刪除、改名或語意變更都必須更新 schema_version 並評估產品 major version。

## 16 實作里程碑

各里程碑可使用多個短 PR 完成。以下是依賴順序，不是承諾每天完成一階段；C++ 基礎學習與專案同步進行。

| Milestone | 工作 | 完成證據 |
|---|---|---|
| M0 規格與 repository | 建立 repo、README、spec、第一批 issues | 可說明輸入、輸出、gate 與範圍 |
| M1 最小 C++ 與 CI | 固定數值判斷、CMake、最小 unit test、Actions build-test | PR 中測試紅→修正→綠的紀錄 |
| M2 可使用的 CLI | parser、profile、exit codes、text／JSON、邊界 tests | TC-01 至 TC-24 適用案例通過 |
| M3 Python automation | manifest、結果比對、JUnit、timeout、harness tests | 故障案例正確 PASS，錯誤程式被判 FAIL |
| M4 Docker 與完整 gate | 相同環境、cppcheck、reports、ci-gate、main protection | 本機重現 CI，失敗 PR 無法合併 |
| M5 Release | tag 驗證、package、runtime smoke、checksum、Release | 在乾淨環境執行下載後 package |
| M6 故障演練與整理 | 失敗診斷紀錄、README 示範、版本說明 | 可完整解釋一次變更如何進入正式版本 |

M1 可先採固定門檻作為教學過渡，但 M2 必須移至 profile。不要把 M1 的部分功能當成 v1.0 已完成。工具版本 SHA／digest、開源授權的具體選擇在各自實作 PR 中確認並記錄；其值不在本文件中虛構。

## 17 故障演練

所有故意錯誤在專用分支進行，修復後才合併。

| 演練 | 注入方式 | 應觀察到的結果 |
|---|---|---|
| 編譯失敗 | 引入語法錯誤 | build job 失敗，test 標示未執行，無法合併 |
| 邊界回歸 | 溫度 `>=` 改成 `>` | 等於 85／95 的 unit 或 integration case 失敗 |
| 靜態分析失敗 | 引入經本版本 cppcheck 確認可偵測的缺陷 | static-analysis 與 ci-gate 失敗 |
| 逾時 | harness stub 超過 5 秒 | case ERROR，繼續其他案例，保存診斷 |
| 壞 JSON | stub 輸出非 JSON 或錯誤 enum | schema／解析失敗，不能只因 exit code 正確而通過 |
| 沒有測試 | 暫時移除所有 CTest 註冊或清空 manifest | gate 失敗，不可顯示 0 tests passed 即成功 |
| 版本錯誤 | tag 與 VERSION 不同 | release preflight 失敗，沒有正式 Release |
| 缺 runtime library | package smoke 環境缺少必要依賴 | smoke 失敗，publish 被阻擋 |

每次演練記錄症狀、定位的 job／case、根因、修正、預防測試與 run URL。這些紀錄可以作為面試時說明實際 automation 工作的證據。

## 18 v1.0 Definition of Done

- [ ] 五種感測器的 parsing、profile、邊界與 aggregation 符合規格。
- [ ] CLI 與 JSON 契約、錯誤代碼、exit codes 均有測試覆蓋。
- [ ] 最低測試矩陣全部完成，必要測試沒有被 skipped，零案例會失敗。
- [ ] Python harness 可正確分辨預期 sensor 故障、程式回歸與執行環境錯誤。
- [ ] Docker 本機入口與 GitHub Actions 使用相同底層腳本。
- [ ] unit／integration／static gates 都能因刻意缺陷失敗。
- [ ] 失敗 run 能取得已有報告，成功 run 缺報告不能通過。
- [ ] main required check 有實際設定，並以失敗 PR 驗證無法合併。
- [ ] Release 對精確 tag commit 驗證，package smoke 通過後才發布。
- [ ] 已發布 binary 可查到版本、commit、工具環境與 checksum。
- [ ] 從新目錄解壓 package，依 README 執行即可重現示範結果。
- [ ] README 清楚說明合成資料、模擬門檻、支援環境、限制與本專案的 CD 定義。

## 19 後續擴充

完成 v1.0 後，依學習方向挑一項加入：

| 方向 | 下一步 | 新增的工程問題 |
|---|---|---|
| Server validation | 透過 adapter 讀取 ipmitool／Redfish 輸出 | 權限、真實 schema、設備離線、資料新鮮度 |
| Monitoring | 定時 sampling、timestamps、stale data | 排程、signal cleanup、時間與遺漏資料 |
| Container delivery | 發布 runtime image 到 GHCR | image tag／digest、掃描、registry 權限 |
| C++ 品質 | Sanitizers、coverage、GCC／Clang matrix | 記憶體錯誤、測試缺口、跨工具相容性 |
| Supply chain | SBOM、簽章、artifact attestations | 依賴清單、來源證明與驗證 |
| 實驗室自動化 | 專用 self-hosted runner 與真實 SUT | 隔離、互斥鎖、憑證、設備排程 |

上述不作為 v1.0 完成的前置條件。真實 SUT workflow 需獨立設計，不讓外部 PR 任意控制實驗室設備。

## 20 官方參考資料

查核日期：2026-10-01。來源支援工具能力及特定實作做法；本專案的 sensor 門檻、schema、退出碼、timeout 與里程碑均為此規格的設計決策。

- [S1] GitHub Actions Secure use reference
- [S2] Docker Building best practices
- [S3] GitHub About protected branches
- [S4] GitHub Store and share data with workflow artifacts
- [S5] CMake 3.22 CTest reference
- [S6] Cppcheck manual

本文件只定義規格；Dockerfile、workflow、程式、tests 與報告範例必須在後續實作及執行後，才能宣稱已驗證完成。
