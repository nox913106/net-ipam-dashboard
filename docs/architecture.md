# 系統架構與資料流

## 整體架構

```
┌─────────────────────────────────────────────────────────────────┐
│                        phpIPAM 主機 (stwphpipam-p)              │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  MariaDB (phpipam DB)                    │   │
│  │                                                         │   │
│  │  [phpIPAM 原生表]          [本專案新增表]                │   │
│  │  ├── ipaddresses           ├── grafana_ip_snapshot       │   │
│  │  ├── subnets               └── grafana_ip_changes        │   │
│  │  ├── sections                                            │   │
│  │  ├── changelog                                           │   │
│  │  └── users                                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│         ↑ 讀取           ↑ 寫入異動紀錄                        │
│         │                │                                      │
│  ┌──────┴────────────────┴──────┐                              │
│  │  ip_change_detector.py       │  ← Cron 每 5 分鐘            │
│  │  (scripts/)                  │                              │
│  └──────────────────────────────┘                              │
│                                                                 │
│  ┌──────────────────────────────┐                              │
│  │  report_server.py (Port 8088)│  ← systemd 常駐              │
│  │  + db.py (SQLite)            │                              │
│  │  + archive/ (HTML 報表)      │                              │
│  └────────────┬─────────────────┘                              │
│               │ 提供歷史報表瀏覽                               │
│  ┌────────────┴─────────────────┐                              │
│  │  report_generator.py (CLI)   │  ← 手動 / Cron 觸發          │
│  │  + Claude AI API             │                              │
│  └──────────────────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
         ↑ MySQL 直連
┌────────┴────────────────────────┐
│         Grafana 伺服器           │
│  ├── 日巡檢 Dashboard            │  ← 查詢 grafana_ip_changes
│  ├── 週巡檢 Dashboard            │  ← 查詢 ipaddresses / subnets
│  └── 月巡檢 Dashboard            │  ← 查詢 ipaddresses / subnets
└─────────────────────────────────┘
         ↑ 提供報表分析 Prompt
┌────────┴────────────────────────┐
│  AI 分析工作流（人工觸發）       │
│  prompts/ipam_analyst_system_prompt.md → Claude AI
└─────────────────────────────────┘
```

---

## 核心資料流

### 1. IP 異動偵測流程（每 5 分鐘）

```
Cron 觸發
    │
    ▼
ip_change_detector.py
    │
    ├─ 讀取 phpIPAM 所有靜態 IP
    │   SQL: ipaddresses JOIN subnets JOIN sections
    │   排除: state=7(DHCP)、is_gateway=1、custom_DHCP_pool_range=1、TC_YueyuenHotel
    │
    ├─ 讀取 grafana_ip_snapshot（上次快照）
    │
    ├─ 比對差異
    │   ├─ 新 IP → 記錄 ADD 事件
    │   ├─ 欄位變更（MAC/Hostname/Owner）→ 記錄 MODIFY 事件
    │   └─ 快照有但已消失 → 記錄 DELETED 事件
    │
    ├─ 查詢操作帳號
    │   方法1: changelog JOIN users WHERE coid = ipam_id（10分鐘內）
    │   方法2: changelog.cdiff LIKE %ip_display%（24小時內，備援）
    │   找不到 → '(系統掃描)'
    │
    ├─ 寫入 grafana_ip_changes
    │
    ├─ 更新 grafana_ip_snapshot（UPSERT）
    │
    └─ 清理超過 30 天的舊紀錄
```

### 2. 報表產生流程（手動 / Cron 觸發）

```
Grafana API 匯出資料（JSON 格式）
    │
    ▼
report_generator.py --data data.json
    │
    ├─ 載入 JSON 資料
    │
    ├─ 呼叫 Claude AI API（claude-sonnet-4-6）
    │   輸入: 概覽數據 + 異動統計 + 高使用率子網
    │   輸出: summary + focus_points + suggestions（JSON）
    │
    ├─ Jinja2 渲染 HTML（template/daily_report.html）
    │
    ├─ 儲存至 archive/YYYY/Mon/DD/daily_report_YYYYMMDD.html
    │
    └─ 寫入 SQLite（reports.db）登錄索引
```

### 3. 歷史報表瀏覽流程

```
使用者瀏覽器 http://<server>:8088
    │
    ├─ GET /              → Flask 回傳 index.html（SPA）
    │
    ├─ GET /api/years     → SQLite 查詢所有年份
    ├─ GET /api/stats     → SQLite 查詢統計摘要
    ├─ GET /api/reports   → SQLite 查詢報表列表（支援 year/month/type 篩選）
    │
    └─ GET /archive/...   → 直接回傳靜態 HTML 報表檔
```

---

## 資料庫設計

### MariaDB（phpIPAM 同庫）

#### `grafana_ip_snapshot` — IP 狀態快照
| 欄位 | 型別 | 說明 |
|------|------|------|
| `ip_addr` | INT UNSIGNED (PK) | IP 整數值（phpIPAM 原生格式） |
| `ipam_id` | INT UNSIGNED | `ipaddresses.id`，供 changelog 查詢用 |
| `subnet_id` | INT | 子網路 ID |
| `mac` | VARCHAR(20) | MAC 位址（統一小寫） |
| `hostname` | VARCHAR(255) | 主機名稱 |
| `owner` | VARCHAR(128) | 擁有者 |
| `state` | TINYINT | IP 狀態碼 |
| `ip_display` | VARCHAR(45) | 可讀 IP（INET_NTOA 結果） |
| `subnet_cidr` | VARCHAR(32) | 子網路 CIDR |
| `subnet_desc` | VARCHAR(255) | 子網路描述 |
| `section_name` | VARCHAR(128) | Section 名稱（站點） |
| `snapshot_at` | TIMESTAMP | 最後更新時間 |

#### `grafana_ip_changes` — IP 異動紀錄
| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | INT AUTO_INCREMENT (PK) | 主鍵 |
| `detected_at` | TIMESTAMP | 偵測時間（索引） |
| `ip_addr` | INT UNSIGNED | IP 整數值（索引） |
| `ip_display` | VARCHAR(45) | 可讀 IP |
| `subnet_cidr` | VARCHAR(32) | 子網路 CIDR |
| `subnet_desc` | VARCHAR(255) | 子網路描述 |
| `section_name` | VARCHAR(128) | Section 名稱 |
| `changed_field` | VARCHAR(32) | 異動類型：ADD / MODIFY / DELETED（索引） |
| `old_value` | VARCHAR(255) | 舊值 |
| `new_value` | VARCHAR(255) | 新值 |
| `changed_by` | VARCHAR(128) | 操作帳號 |

### SQLite（`reports/reports.db`）

#### `reports` — 報表索引
| 欄位 | 說明 |
|------|------|
| `report_date` | 報表日期 YYYY-MM-DD |
| `report_type` | daily / weekly / monthly |
| `year` / `month` / `day` | 拆分欄位（篩選用） |
| `file_path` | 相對於 archive/ 的路徑 |
| `file_size` | 檔案大小（bytes） |
| `created_at` | 產生時間 |

---

## 安全設計

### 內網存取限制（report_server.py）
Flask 伺服器在每個請求前檢查 `remote_addr`，只允許以下 IP 段存取：
- `172.*`（私有網段）
- `192.168.*`（私有網段）
- `10.*`（私有網段）
- `127.*`（本機）

非內網 IP 一律回傳 `403 Forbidden`。

### 路徑穿越防護（`/archive` 路由）
使用 `Path.resolve()` 正規化後比對是否在 `ARCHIVE_DIR` 目錄內，防止 `../` 路徑穿越攻擊。

### 資料庫查詢
所有 SQL 使用參數化查詢（pymysql 的 `%s` placeholder），防止 SQL Injection。

---

## Grafana Dashboard 說明

### 週巡檢 Dashboard（`weekly_inspection.json`）
- Dashboard ID: 1921
- 資料來源：phpIPAM MariaDB（MySQL 資料來源）
- 分析時間窗口：7 天
- 主要 Panel：IP 容量趨勢、合規性分布、子網使用率

### 月巡檢 Dashboard（`monthly_inspection.json`）
- Dashboard ID: 1922
- 資料來源：phpIPAM MariaDB（MySQL 資料來源）
- 分析時間窗口：30 天
- 主要 Panel：長期離線設備、跨站點容量評估、資產淨增減

> **日巡檢 Dashboard** 未包含在此 repo 的 JSON 中（已直接部署在 Grafana），
> 其 Panel 會查詢 `grafana_ip_changes` 表顯示「最近 24 小時 IP 異動」。
