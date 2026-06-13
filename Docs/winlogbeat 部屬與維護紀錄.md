# B311 教室教師機 Winlogbeat 部署與維護紀錄

## 📋 一、 環境配置摘要

* **客戶端系統：** Windows 11 Pro (主機名稱: `Teacher`)
* **Winlogbeat 版本：** `9.4.2-windows-x86_64`
* **Elasticsearch / Kibana 伺服器 IP：** `172.16.1.4`
* **通訊協定與安全設定：** 強制採用 **HTTPS (9200)** 傳輸，並啟動 **SSL 憑證豁免 (Ignore CA)**。

---

## 🛠️ 二、 黃金設定檔 (`winlogbeat.yml`)

此檔案已通過語法縮排校正，並成功解決「大寫索引被拒」與「HTTPS/EOF 斷線」地雷。

```yaml
###################### Winlogbeat Configuration ########################

# ======================== Winlogbeat 特定日誌監控 =========================
winlogbeat.event_logs:
  - name: Security
    processors:
      - drop_event.when.or:
          - equals.winlog.event_id: 4672  # 攔截過濾：特殊權限指派（減少雜訊）
          - equals.winlog.event_id: 4662
    ignore_older: 72h

  - name: Application
    ignore_older: 72h

  - name: System
    ignore_older: 72h

  - name: Setup
    ignore_older: 72h

  - name: Microsoft-Windows-Sysmon/Operational

  - name: Windows PowerShell
    event_id: 400, 403, 600, 800

  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106

  - name: ForwardedEvents
    tags: [forwarded]

# ====================== 索引範本優化設定 =======================
setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression

# ================================== 標籤欄位 (General) ===================================
# 自訂資安分析欄位，全教室大量佈署時的核心識別標籤
fields:
  class: teacher-computer-test
  pc_role: teacher-computer
  deployed_year: 2026
fields_under_root: true

# =================================== Kibana 端點 ===================================
setup.kibana:
  host: "http://172.16.1.4:5601"

# ================================== 輸出設定 (Outputs) ===================================
# ---------------------------- Elasticsearch Output ----------------------------
output.elasticsearch:
  # 採用 HTTPS 安全通道連線
  hosts: ["https://172.16.1.4:9200"]
  username: "elastic"
  password: "your_password" # 👈 請替換為實際的 elastic 密碼

  # 關鍵修正：索引名稱強制全小寫
  index: "b311-teacher-computer-test-log-%{+yyyy.MM.dd}"

  # 核心大絕招：無視自簽憑證限制，強行建立安全加密連線
  ssl.verification_mode: "none"

# -------------------------- 全域設定（絕對不能有縮排） --------------------------
# 徹底關閉官方內建 ILM 機制，強迫啟用上方自訂的 index 命名規則
setup.template.enabled: false
setup.ilm.enabled: false

# ================================== 紀錄檔設定 ===================================
logging.level: info
logging.to_files: true
logging.files:
  path: C:\ProgramData\winlogbeat\Logs
  name: winlogbeat
  keepfiles: 7
  permissions: 0640

# ================================= 處理器 (Processors) =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~

```

---

## 💻 三、 PowerShell 本地端驗證與部署指令

在 Windows 11 上以**系統管理員身分**開啟 PowerShell，切換至 `C:\Program Files\winlogbeat-9.4.2-windows-x86_64` 目錄後，依序執行以下實戰指令：

```powershell
# 1. 語法結構檢查（確認 YAML 縮排與空白無誤）
.\winlogbeat.exe test config -c .\winlogbeat.yml
# 💡 預期輸出：Config OK

# 2. 輸出連線測試（確認 HTTPS 通道、帳密、以及 SSL 豁免是否成功聯動）
.\winlogbeat.exe test output -c .\winlogbeat.yml
# 💡 預期輸出：talk to server... OK (並顯示後端 ES 版本號)

# 3. 將 Winlogbeat 安裝為 Windows 內建系統服務（初次佈署才需要）
.\install-service-winlogbeat.ps1

# 4. 重新啟動背景服務（套用最新 yml 檔）
Restart-Service winlogbeat

# 5. 檢查服務狀態
Get-Service winlogbeat
# 💡 預期輸出：Running

```

---

## 📊 四、 後端驗證與 Kibana 設定步驟

### 步驟 1：至 Dev Tools 檢查實體資料庫狀態

開啟 Kibana 網頁，導航至 **Management ➔ Dev Tools**，在 Console 輸入以下指令並點擊執行（▶）：

```text
GET _cat/indices?v

```

* **驗證標準：** 右側清單必須成功蹦出全小寫的新索引：
`yellow open b311-teacher-computer-test-log-2026.06.13`

### 步驟 2：建立 Kibana Data View

1. 導航至 **Stack Management ➔ Data Views** ➔ 點擊 **Create data view**。
2. **Name (顯示名稱)：** `B311教師機日誌`
3. **Index pattern (索引匹配)：** 輸入 `b311-teacher-computer-test-log-*`
4. **Timestamp field (時間戳記)：** 選擇 `@timestamp`。
5. 點擊 **Save data view to Kibana**。

### 步驟 3：Discover 查看客製化成果

進入 **Discover** 介面，切換至剛建立的 Data View，展開日誌確認：

* 欄位成功帶有 `class: teacher-computer-test`。
* 成功排除 `event_id: 4672` 雜訊，達成最完美的硬碟空間節省配置。

---

## ⚠️ 五、 核心除錯歷史筆記 (Lessons Learned)

1. **縮排地雷 (YAML indentation)：** `setup.ilm.enabled` 與 `setup.template.enabled` 在 `yml` 檔案中**絕對不能有任何前置空格（必須靠左對齊）**。若被誤縮排進 `output.elasticsearch` 中會導致停用 ILM 失敗，使自訂索引名稱失效。
2. **Elasticsearch 命名鐵律：** 索引名稱（Index Name）**嚴禁大寫字母**。如設定為 `B311-...` 會遭到 Elasticsearch 拒絕寫入。
3. **安全加密通訊 (HTTPS & EOF)：** 若後端 Elasticsearch 開啟安全加密，前端若使用 `http` 連線會直接被伺服器斷開並噴出 `EOF` 錯誤。必須將連線改為 `https://` 並追加 `ssl.verification_mode: "none"` 方可順利連通。