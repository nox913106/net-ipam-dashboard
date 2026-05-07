# GRAFANA-phpipam-dashboards 專案文件

## 專案概述

本專案提供一套基於 phpIPAM 的 IP 位址管理監控系統，整合 Grafana Dashboard、自動異動偵測腳本與 HTML 報表產生器，協助網路團隊進行日常 IP 管理與巡檢。

**核心功能：**
- 自動偵測靜態 IP 的新增 / 異動 / 刪除，並記錄操作帳號
- Grafana 日 / 週 / 月巡檢 Dashboard（直連 phpIPAM MariaDB）
- Claude AI 驅動的 HTML 報表產生與分析
- 歷史報表瀏覽 Web 伺服器

**部署主機：** `stwphpipam-p`（phpIPAM 所在伺服器）
**部署路徑：** `/opt/GRAFANA-phpipam-dashboards/`

---

## 目錄結構

```
GRAFANA-phpipam-dashboards/
├── docs/                          # 專案文件（本資料夾）
│   ├── README.md                  # 專案總覽（本文件）
│   ├── architecture.md            # 系統架構與資料流
│   ├── files-reference.md         # 逐檔說明
│   └── deployment.md              # 部署與操作指南
│
├── scripts/
│   └── ip_change_detector.py      # IP 異動偵測腳本（Cron 每 5 分鐘執行）
│
├── database/
│   └── grafana_ip_changes.sql     # MariaDB 資料表 DDL
│
├── prompts/
│   └── ipam_analyst_system_prompt.md  # AI 分析師 System Prompt
│
├── reports/
│   ├── report_generator.py        # HTML 報表產生器（CLI 工具）
│   ├── report_server.py           # 歷史報表瀏覽 Web 伺服器（Port 8088）
│   ├── db.py                      # SQLite 報表索引資料庫
│   ├── sample_data.json           # 範例資料（開發 / 排版預覽用）
│   ├── phpipam-report.service     # systemd 服務設定檔
│   ├── template/
│   │   └── index.html             # 報表瀏覽器前端（SPA）
│   └── static/
│       └── js/
│           ├── chart.umd.min.js               # Chart.js（離線版）
│           └── chartjs-plugin-datalabels.min.js  # Chart.js 標籤插件
│
├── weekly_inspection.json         # Grafana 週巡檢 Dashboard JSON
├── monthly_inspection.json        # Grafana 月巡檢 Dashboard JSON
└── .gitignore
```

---

## 相關文件

| 文件 | 說明 |
|------|------|
| [architecture.md](./architecture.md) | 系統架構圖、資料流、元件關係 |
| [files-reference.md](./files-reference.md) | 每個檔案的完整說明 |
| [deployment.md](./deployment.md) | 部署步驟、Cron 設定、服務管理 |

---

## 快速參考

### 啟動報表伺服器
```bash
cd /opt/GRAFANA-phpipam-dashboards/reports
python3 report_server.py
# 或使用 systemd：
sudo systemctl start phpipam-report
```

### 產生今日報表
```bash
cd /opt/GRAFANA-phpipam-dashboards/reports
python3 report_generator.py --data /path/to/today_data.json
```

### 手動執行 IP 異動偵測
```bash
cd /opt/GRAFANA-phpipam-dashboards
python3 scripts/ip_change_detector.py
```

### 在 Grafana 匯入 Dashboard
1. Grafana → Dashboards → Import
2. 上傳 `weekly_inspection.json` 或 `monthly_inspection.json`
3. 選擇 phpIPAM MariaDB 資料來源

---

## 環境需求

| 元件 | 版本需求 |
|------|----------|
| Python | 3.10+ |
| pymysql | 連接 phpIPAM MariaDB |
| Flask | 報表伺服器 |
| Jinja2 | HTML 報表模板渲染 |
| anthropic | Claude AI 分析（需 `ANTHROPIC_API_KEY`） |
| Grafana | 連接 MariaDB MySQL 資料來源 |

### 必要環境變數

| 變數 | 用途 | 預設值 |
|------|------|--------|
| `PHPIPAM_DB_HOST` | phpIPAM MariaDB 主機 | `localhost` |
| `PHPIPAM_DB_PORT` | MariaDB 連接埠 | `3306` |
| `PHPIPAM_DB_USER` | 資料庫使用者 | `phpipam` |
| `PHPIPAM_DB_PASSWORD` | 資料庫密碼 | _(需設定)_ |
| `PHPIPAM_DB_NAME` | 資料庫名稱 | `phpipam` |
| `ANTHROPIC_API_KEY` | Claude AI API 金鑰 | _(需設定)_ |
