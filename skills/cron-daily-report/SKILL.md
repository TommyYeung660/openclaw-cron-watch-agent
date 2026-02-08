---
name: cron-daily-report
version: 0.1.0
description: 生成昨日 OpenClaw cron 執行日總結（成功/失敗次數、主要錯誤類型），並發送摘要到 Telegram 群組。
user-invocable: true
---

# Cron Daily Report（昨日日總結）

## 目標
- 讀取「所有 OpenClaw cron jobs」與「昨日（Asia/Hong_Kong 00:00-23:59）」的 runs。
- 統計每個 job：成功/失敗次數。
- 額外分類失敗原因（特別係：
  - 429 overload
  - gateway timeout
  - path /workspace 不存在
  - read-only file system
  - tool validation failed
  - Telegram send failed / chat not found（屬於通知層，唔一定係工作本體失敗）
）。
- 產出一份精簡報告，寫入：`{baseDir}/../../reports/YYYY-MM-DD.md`（YYYY-MM-DD = 昨日日期）。
- 另外用 message tool 將「摘要」發到 Telegram 群組：`-5162606720`。

## 可用工具
- 使用 `cron` 工具：
  - `cron.status()`（可選）
  - `cron.list(includeDisabled:true)`
  - `cron.runs(jobId)`
- 使用 `write`/`edit`：寫 report 檔
- 使用 `message.send`：發送摘要到群組（只發摘要，唔好貼長 log；目標必須係數字 chat id）

## 操作步驟（建議）
1) 用 `cron.list(includeDisabled:true)` 取得 job 清單。
2) 對每個 job 呼叫 `cron.runs(jobId)`，取足夠多 entries（例如最近 200 次），然後用時間範圍過濾到「昨日 00:00-23:59」（Asia/Hong_Kong）。
3) 逐 job 統計：
   - runs_total、success、error
   - error 類型 top（可從 `error` / `summary` 字段做 keyword 分類）
4) 匯總全局：總 runs、成功、失敗、成功率。
5) 寫入 report 檔（Markdown）。
6) 組裝 10~20 行摘要並發到 Telegram 群組 `-5162606720`。

## Telegram 發送規則（必讀）
- 只可以用 `message.send`。
- `channel` 用 `telegram`。
- `to` **必須**係字面值 `-5162606720`（唔好用 "cron-watch"、唔好用群組名、唔好用 @username）。
- 如 `message.send` 失敗：
  - 仍然要把報告檔寫入成功；
  - 在 report 內加一段「通知層失敗」說明（含錯誤字串，但唔好貼 token）。

## 檔案層日誌（必做，用於追查）
除咗寫 `reports/YYYY-MM-DD.md`，你必須同時寫一份「執行日誌」到：
- `logs/daily-report-YYYY-MM-DD.log`（YYYY-MM-DD = 昨日日期）

日誌內容最少包含：
- 開始/結束時間（含時區 Asia/Hong_Kong）
- `cron.status` / `cron.list` / 每個 `cron.runs(jobId)` 是否成功
- 若工具調用失敗（例如 `gateway timeout`），必須原樣記錄錯誤字串
- 成功寫 report 檔、以及 `message.send` 成功/失敗結果

注意：
- 日誌只寫到檔案，唔好整段貼去 Telegram。
- 唔好寫入任何 secrets/token。

## 報告格式（Markdown 建議）
- 標題：`# Cron 日總結：YYYY-MM-DD（昨日）`
- **資料完整性聲明（必備）**：
  - `cron.list` 是否成功取得 job 清單
  - `cron.runs` 是否能取得足夠 runs（若 runs 記錄量太少，必須明確寫明「昨日可能未被覆蓋」，避免把「0 runs」誤判成故障）
- **昨日新增 Cron Jobs（必備）**：
  - 以 `createdAtMs` 落在昨日（Asia/Hong_Kong 00:00-23:59）為準
  - 列出：name、jobId、schedule、sessionTarget、enabled
- 總覽：總 runs / 成功 / 失敗 / 成功率
- 失敗原因 Top（分「通知層」同「工作本體層」）
- Job 列表（只列：昨日有 run 的 job；失敗 job 放前面）
  - job name (id)
  - success x / fail y
  - 最近一次失敗時間 + 一行錯誤摘要（最多 1-2 條 sample）

## 注意
- 全程使用繁體中文。
- 唔好輸出任何敏感 token。
- 唔好嘗試自動修復 cron；只提出建議。
