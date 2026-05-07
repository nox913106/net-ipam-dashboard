# 逐檔說明

本文件對專案中每個檔案做完整說明，包含用途、關鍵邏輯與注意事項。

---

## scripts/ip_change_detector.py

**類型：** Python 腳本（Cron 定期執行）
**用途：** 偵測 phpIPAM 中靜態 IP 的欄位變更，將 ADD / MODIFY / DELETED 事件寫入 MariaDB。

### 職責

1. 從 phpIPAM 的 `ipaddresses` 表讀取所有靜態 IP 的當前狀態
2. 與 `grafana_ip_snapshot` 快照比對，找出差異
3. 將差異寫入 `grafana_ip_changes` 並更新快照
4. 查詢 phpIPAM 的 `changelog` 表，找出操作帳號

### 監控欄位

只追蹤以下三個欄位的變更：
- `mac` — MAC 位址（統一轉小寫再比對，避免大小寫假異動）
- `hostname` — 主機名稱
- `owner` — 擁有者 / 備註

### 排除條件（與 Grafana Dashboard 一致）

| 排除對象 | 條件 |
|----------|------|
| DHCP 動態租約 | `state = 7` |
| Gateway IP | `is_gateway = 1` |
| DHCP Pool 範圍 | `custom_DHCP_pool_range = 1` |
| 外部租戶網段 | `sections.name = 'TC_YueyuenHotel'` |

### 首次執行行為

若 `grafana_ip_snapshot` 為空（首次執行），只建立初始快照，**不產生任何異動紀錄**，避免把所有現有 IP 都當作 ADD 事件。

### 操作帳號查詢邏輯

```
方法 1（精確）：
  changelog WHERE ctype='ip_addr' AND coid=ipam_id AND cdate >= NOW() - 10分鐘
  → 找到 → 回傳 username

方法 2（備援）：
  changelog WHERE ctype='ip_addr' AND cdiff LIKE '%ip_display%' AND cdate >= NOW() - 24小時
  → 找到 → 回傳 username

兩種方法都找不到 → 回傳 '(系統掃描)'
```

10 分鐘的時間窗口是為了只匹配「本次 Cron 週期內（5 分鐘）」的操作，避免把舊操作誤算進來。

### 設定常數

| 常數 | 預設值 | 說明 |
|------|--------|------|
| `TRACKED_FIELDS` | `['mac', 'hostname', 'owner']` | 監控的欄位 |
| `CHANGE_RETENTION_DAYS` | `30` | 異動紀錄保留天數 |

### 設定來源（環境變數）

| 環境變數 | 預設值 |
|----------|--------|
| `PHPIPAM_DB_HOST` | `localhost` |
| `PHPIPAM_DB_PORT` | `3306` |
| `PHPIPAM_DB_USER` | `phpipam` |
| `PHPIPAM_DB_PASSWORD` | `my_secret_phpipam_pass` |
| `PHPIPAM_DB_NAME` | `phpipam` |

### Cron 設定

```cron
*/5 * * * * root cd /opt/GRAFANA-phpipam-dashboards && python3 scripts/ip_change_detector.py >> /var/log/grafana-ip-changes.log 2>&1
```

---

## database/grafana_ip_changes.sql

**類型：** SQL DDL 腳本
**用途：** 在 phpIPAM 的 MariaDB 中建立本專案所需的兩個資料表。

### 包含的資料表

**`grafana_ip_snapshot`** — IP 快照表
- 儲存每個靜態 IP 最後一次已知的欄位值
- 以 `ip_addr`（IP 整數）為主鍵
- `snapshot_at` 欄位在每次 UPSERT 時自動更新

**`grafana_ip_changes`** — IP 異動紀錄表
- 儲存每次偵測到的異動事件
- 有三個索引：`detected_at`、`ip_addr`、`changed_field`，對應 Grafana 的常用篩選條件

### 使用方式

```bash
# 手動建立資料表
mysql -u phpipam -p phpipam < database/grafana_ip_changes.sql

# 腳本自動建立（若 SQL 檔案存在）
python3 scripts/ip_change_detector.py  # 腳本啟動時會自動執行 CREATE TABLE IF NOT EXISTS
```

> `ip_change_detector.py` 啟動時會自動讀取此 SQL 檔並執行 `CREATE TABLE IF NOT EXISTS`，
> 因此通常不需要手動執行。

---

## prompts/ipam_analyst_system_prompt.md

**類型：** AI System Prompt 文件（Markdown）
**用途：** 定義 Claude AI 作為 IPAM 分析師的角色、分析框架與輸出規範，供人工分析工作流使用。

### 內容結構

| 區塊 | 說明 |
|------|------|
| Role Definition | 定義 AI 為企業 IP 資產管理分析師，列出管轄的 7 個站點 |
| Report Types | 定義日 / 週 / 月報告的時間窗口與分析側重點 |
| Key Metric Definitions | IP 狀態碼、異動類型、不合規定義、容量閾值 |
| Analysis Framework | 四大輸出區塊（Executive Summary / Detailed Analysis / Recommendations / Action Items） |
| Health Score Rubric | 滿分 10 分制評分標準（Severity / Capacity / Compliance / Change Risk） |
| Report-Specific Logic | 日 / 週 / 月報各自的分析重點細節 |
| Data Exclusion Rules | 必須從分析中排除的資料（與 SQL 查詢排除條件一致） |
| Output Rules | 語言（繁體中文）、簡潔要求、數據完整性規範 |
| Guardrails | 禁止虛構數據、禁止猜測缺失資料 |

### 使用方式

此 Prompt 為人工手動觸發的分析工作流使用：

1. 從 Grafana API 或報表匯出巡檢數據
2. 將此 System Prompt 貼入 Claude 對話
3. 貼入巡檢數據，Claude 會依格式輸出分析報告

> 與 `report_generator.py` 使用的 AI 提示不同：
> - `ipam_analyst_system_prompt.md` — 給人工互動使用，輸出完整的繁體中文報告
> - `report_generator.py` 內的 `build_analysis_prompt()` — 給自動化管線使用，輸出 JSON（摘要 + 重點 + 建議）

---

## reports/report_generator.py

**類型：** Python CLI 腳本
**用途：** 接收 JSON 格式的巡檢數據，呼叫 Claude AI 進行分析，渲染成 HTML 報表後存檔並登錄至 SQLite。

### 命令列用法

```bash
# 使用範例資料預覽（輸出至 output/，不寫入 DB）
python3 report_generator.py --sample

# 使用真實資料產生報表（輸出至 archive/，寫入 DB）
python3 report_generator.py --data /path/to/data.json

# 跳過 AI 分析（測試排版用）
python3 report_generator.py --data data.json --no-ai

# 指定報表類型
python3 report_generator.py --data data.json --type weekly
```

### 執行流程

```
1. 載入 JSON 資料（--sample 使用 sample_data.json，--data 使用指定檔案）
2. AI 分析（呼叫 claude-sonnet-4-6，max_tokens=1024）
   輸入: 概覽統計 + 異動數據 + 高使用率子網
   輸出: { "summary": "...", "focus_points": [...], "suggestions": [...] }
3. Jinja2 渲染 HTML（template/daily_report.html）
4. 儲存：
   --sample → reports/output/daily_report_YYYYMMDD.html（不寫 DB）
   --data   → reports/archive/YYYY/Mon/DD/daily_report_YYYYMMDD.html（寫 DB）
```

### 輸出路徑格式

```
reports/archive/2026/Apr/17/daily_report_20260417.html
reports/archive/2026/Apr/17/weekly_report_20260417.html
reports/archive/2026/Apr/17/monthly_report_20260417.html
```

### AI 分析輸入資料

從 JSON 的以下欄位擷取：
- `overview.total_ip`、`overview.active_ip`、`overview.dhcp_pool_ip`
- `changes.total`、`changes.modify`、`changes.add`、`changes.deleted`、`changes.operators`
- `hot_ips`（高頻異動 IP）
- `subnet_changes`（子網路異動）
- `high_usage_detail`（高使用率子網）

---

## reports/report_server.py

**類型：** Python Flask Web 伺服器
**用途：** 提供歷史報表的瀏覽介面，對外開放 Port 8088（限內網）。

### API 端點

| 端點 | 說明 |
|------|------|
| `GET /` | 回傳 SPA 前端（index.html） |
| `GET /api/years` | 所有有報表的年份清單 |
| `GET /api/stats` | 報表統計摘要（總數、各類型數量、最新日期） |
| `GET /api/reports` | 報表列表（支援 `?year=&month=&type=` 篩選） |
| `GET /archive/<path>` | 回傳靜態 HTML 報表檔 |

### 安全機制

- **內網限制：** 每個請求前驗證 `remote_addr`，非 `172.*`/`192.168.*`/`10.*`/`127.*` 一律 403
- **路徑穿越防護：** `/archive` 路由使用 `Path.resolve()` 確認路徑在 `archive/` 目錄內

### 啟動方式

```bash
# 直接執行
python3 report_server.py

# systemd 服務
sudo systemctl start phpipam-report
sudo systemctl enable phpipam-report  # 設定開機自啟
```

---

## reports/db.py

**類型：** Python 模組（SQLite 操作）
**用途：** 管理 `reports.db` SQLite 資料庫，提供報表索引的讀寫操作。

### 主要函式

| 函式 | 說明 |
|------|------|
| `init_db()` | 建立 `reports` 資料表（若不存在） |
| `register(...)` | 寫入一筆報表記錄（使用 INSERT OR REPLACE，同日期同類型會覆蓋） |
| `query(year, month, report_type)` | 查詢報表列表，所有參數可選 |
| `get_years()` | 查詢所有有報表的年份（降序） |
| `get_stats()` | 查詢首頁摘要（總數、各類型數、最新日期） |

### 資料庫位置

```
reports/reports.db  （與 report_server.py 同目錄）
```

### 唯一約束

`(report_date, report_type)` 組合唯一，同一天同類型的報表重新產生時會覆蓋舊記錄。

---

## reports/template/index.html

**類型：** HTML + CSS + JavaScript（SPA 前端）
**用途：** 報表歷史瀏覽介面，透過 Ajax 呼叫 Flask API 動態載入報表列表。

### 功能

- 左側側欄：年份標籤、月份格子（灰色代表該月無報表）、報表類型篩選（全部 / 日 / 週 / 月）
- 右側主區：報表卡片列表，點擊或按「開啟報表」在新分頁顯示 HTML 報表
- 頂欄：顯示各類型報表總數統計

### 設計特點

- 純原生 JavaScript（無外部框架依賴），所有 CSS 內嵌
- 月份格子會根據 API 回傳的資料動態標示哪些月份有資料
- 換年份時自動清除月份篩選狀態

---

## reports/phpipam-report.service

**類型：** systemd 服務設定檔
**用途：** 讓 `report_server.py` 作為系統服務在主機啟動時自動執行。

### 關鍵設定

| 設定 | 值 | 說明 |
|------|-----|------|
| `User` | `root` | 執行帳號 |
| `WorkingDirectory` | `/opt/GRAFANA-phpipam-dashboards/reports` | 工作目錄 |
| `ExecStart` | `/usr/bin/python3 .../report_server.py` | 啟動命令 |
| `Restart` | `on-failure` | 程序異常時自動重啟 |
| `RestartSec` | `5` | 重啟等待秒數 |

### 部署指令

```bash
sudo cp phpipam-report.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now phpipam-report
```

---

## reports/sample_data.json

**類型：** JSON 資料範例
**用途：** 開發與排版預覽時使用的假資料，結構與 Grafana API 匯出格式完全相同。

### 包含的資料欄位

| 欄位 | 說明 |
|------|------|
| `report_date` | 報表日期 |
| `time_range` | 分析時間窗口 |
| `generated_at` | 產生時間 |
| `overview` | 總 IP / Active IP / DHCP Pool IP |
| `site_subnets` | 各站點子網數量（按類型） |
| `site_ip_capacity` | 各站點 IP 容量（按類型） |
| `changes` | 異動統計（total / modify / add / deleted / operators） |
| `change_type_dist` | 異動類型分布（用於圓餅圖） |
| `operator_ranking` | 操作者排行（用於橫條圖） |
| `section_heat` | 站點異動熱度 |
| `hot_ips` | 高頻異動 IP 清單 |
| `subnet_changes` | 子網路異動清單 |
| `high_usage_detail` | 高使用率子網詳情 |
| `ai_analysis` | 預先填好的 AI 分析結果（示範用） |

---

## reports/static/js/chart.umd.min.js

**類型：** 第三方 JavaScript 函式庫（離線版）
**版本：** Chart.js UMD 版本
**用途：** 在 HTML 報表中繪製各類圖表（圓餅圖、橫條圖、折線圖等）。

離線打包至 repo，確保報表在無外網環境下也能正常顯示圖表。

---

## reports/static/js/chartjs-plugin-datalabels.min.js

**類型：** Chart.js 插件（離線版）
**用途：** 為 Chart.js 圖表添加資料標籤顯示（在圖表元素上直接顯示數值）。

與 `chart.umd.min.js` 配對使用，同樣離線打包。

---

## weekly_inspection.json

**類型：** Grafana Dashboard 匯出 JSON（Dashboard ID: 1921）
**用途：** 匯入 Grafana 的週巡檢 Dashboard 定義，直接查詢 phpIPAM MariaDB。

包含 Panel 定義、SQL 查詢、圖表配置等。匯入 Grafana 後即可使用，無需額外設定。

---

## monthly_inspection.json

**類型：** Grafana Dashboard 匯出 JSON（Dashboard ID: 1922）
**用途：** 匯入 Grafana 的月巡檢 Dashboard 定義，直接查詢 phpIPAM MariaDB。

---

## .gitignore

```
extract_output.txt     # 臨時匯出檔案
*.png / *.jpg          # 截圖（不需要版本控制）
reports/output/        # --sample 模式的預覽輸出（本機暫存）
__pycache__/ *.pyc     # Python 編譯快取
.env                   # 環境變數檔案（含密碼，不能進 repo）
```
