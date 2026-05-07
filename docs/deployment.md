# 部署與操作指南

## 部署環境

| 項目 | 說明 |
|------|------|
| 目標主機 | `stwphpipam-p`（phpIPAM 所在伺服器） |
| 部署路徑 | `/opt/GRAFANA-phpipam-dashboards/` |
| 作業系統 | Linux（systemd 環境） |
| Python | 3.10+ |

---

## 1. 初次部署

### 1.1 取得程式碼

```bash
cd /opt
git clone <repo-url> GRAFANA-phpipam-dashboards
cd GRAFANA-phpipam-dashboards
```

### 1.2 安裝 Python 依賴

```bash
pip3 install pymysql flask jinja2 anthropic
```

> 若環境使用內部 PyPI mirror，請調整 pip 設定後再安裝。

### 1.3 設定環境變數

建立 `/opt/GRAFANA-phpipam-dashboards/.env`（不會進 git）：

```bash
export PHPIPAM_DB_HOST=localhost
export PHPIPAM_DB_PORT=3306
export PHPIPAM_DB_USER=phpipam
export PHPIPAM_DB_PASSWORD=<your-password>
export PHPIPAM_DB_NAME=phpipam
export ANTHROPIC_API_KEY=<your-claude-api-key>
```

在需要執行腳本前 `source .env`，或設定至 systemd 服務的 `Environment=` 參數。

### 1.4 建立 MariaDB 資料表

資料表會在 `ip_change_detector.py` 第一次執行時自動建立。
也可手動執行：

```bash
mysql -u phpipam -p phpipam < database/grafana_ip_changes.sql
```

### 1.5 建立 IP 初始快照（首次執行）

```bash
cd /opt/GRAFANA-phpipam-dashboards
source .env
python3 scripts/ip_change_detector.py
```

首次執行會建立快照，不產生任何異動紀錄。確認 log 輸出含 `初始快照已建立` 即成功。

---

## 2. 設定 Cron（IP 異動偵測）

編輯 `/etc/crontab`，新增：

```cron
*/5 * * * * root cd /opt/GRAFANA-phpipam-dashboards && source .env && python3 scripts/ip_change_detector.py >> /var/log/grafana-ip-changes.log 2>&1
```

確認 Cron 正常執行：

```bash
tail -f /var/log/grafana-ip-changes.log
```

正常輸出範例：
```
2026-05-07 10:00:01 [INFO] 讀取 4521 筆靜態 IP
2026-05-07 10:00:02 [INFO] 偵測完成: ADD=0, MODIFY=2, DELETED=0
```

---

## 3. 部署報表伺服器（systemd）

### 3.1 安裝服務

```bash
cd /opt/GRAFANA-phpipam-dashboards/reports
sudo cp phpipam-report.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now phpipam-report
```

### 3.2 確認服務狀態

```bash
sudo systemctl status phpipam-report
```

### 3.3 查看服務 Log

```bash
sudo journalctl -u phpipam-report -f
```

### 3.4 服務管理指令

```bash
sudo systemctl start phpipam-report    # 啟動
sudo systemctl stop phpipam-report     # 停止
sudo systemctl restart phpipam-report  # 重啟
```

---

## 4. 匯入 Grafana Dashboard

### 週巡檢 / 月巡檢 Dashboard

1. 登入 Grafana
2. 左側選單 → Dashboards → Import
3. 點擊「Upload dashboard JSON file」
4. 選擇 `weekly_inspection.json` 或 `monthly_inspection.json`
5. 在「Options」中選擇 phpIPAM MariaDB 資料來源
6. 點擊「Import」

> Dashboard ID 可能與現有 Dashboard 衝突，匯入時 Grafana 會提示是否覆蓋。
> 如需保留兩個版本，匯入前先修改 JSON 中的 `"id"` 欄位。

---

## 5. 產生 HTML 報表

### 5.1 取得巡檢數據

從 Grafana 的「Share」功能或 API 匯出當日巡檢數據為 JSON，格式參考 `reports/sample_data.json`。

### 5.2 產生日巡檢報表

```bash
cd /opt/GRAFANA-phpipam-dashboards/reports
source ../.env
python3 report_generator.py --data /path/to/today_data.json
```

### 5.3 產生週巡檢 / 月巡檢報表

```bash
python3 report_generator.py --data data.json --type weekly
python3 report_generator.py --data data.json --type monthly
```

### 5.4 排版預覽（不寫入資料庫）

```bash
python3 report_generator.py --sample
# 報表輸出至 reports/output/daily_report_YYYYMMDD.html
```

### 5.5 跳過 AI 分析（節省 API 費用）

```bash
python3 report_generator.py --data data.json --no-ai
```

---

## 6. 瀏覽歷史報表

報表伺服器啟動後，在內網瀏覽器開啟：

```
http://<stwphpipam-p 的 IP>:8088
```

介面功能：
- 左側選擇年份 → 月份 → 報表類型（日 / 週 / 月）
- 右側列表點擊「開啟報表」在新分頁查看 HTML 報表

---

## 7. 更新部署

```bash
cd /opt/GRAFANA-phpipam-dashboards
git pull origin main
sudo systemctl restart phpipam-report
```

---

## 8. 日常維護

### 清理舊 log

```bash
# 保留最近 90 天的 log
find /var/log -name "grafana-ip-changes*.log" -mtime +90 -delete
```

### 確認異動紀錄自動清理

`ip_change_detector.py` 每次執行後會自動刪除超過 30 天的 `grafana_ip_changes` 記錄。
可在 log 中確認：

```
2026-05-07 10:00:02 [INFO] 清理 15 筆過期異動紀錄（>30 天）
```

### 備份 SQLite 報表索引

```bash
cp /opt/GRAFANA-phpipam-dashboards/reports/reports.db \
   /backup/reports_$(date +%Y%m%d).db
```

---

## 9. 常見問題

### Q: Cron 執行後 log 顯示「資料庫連線失敗」

確認環境變數是否正確 source，並確認 MariaDB 可從本機連線：
```bash
mysql -u phpipam -p phpipam -e "SELECT 1"
```

### Q: 報表伺服器啟動後從外部連不進去

確認防火牆是否開放 Port 8088，以及確認連線來源是內網 IP（否則會被內網過濾器 403 阻擋）。

### Q: AI 分析輸出 "需要安裝 anthropic SDK"

```bash
pip3 install anthropic
```

### Q: AI 分析輸出 "請設定環境變數 ANTHROPIC_API_KEY"

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

### Q: 首次執行後沒有任何異動紀錄是正常的嗎？

是正常的。首次執行只建立快照，第二次（5 分鐘後）才會開始偵測差異。

### Q: 報表中 `changed_by` 都是「(系統掃描)」

phpIPAM 的 `changelog` 表沒有找到對應的操作記錄，可能是：
1. 變更是透過 phpIPAM API 或腳本直接寫入資料庫（繞過 Web UI）
2. phpIPAM 的 changelog 功能未啟用
3. 操作時間超過 10 分鐘前（方法 1 查詢窗口）且 cdiff 未含 IP 字串
