# CramPlan：期末考讀書排程

> **Design Only**：本 Repository 只包含規劃與設計文件，尚未實作程式、資料庫或測試。

**一句話價值主張**：登錄考試日期後，系統每天告訴你「今天只要讀這一科」，以及為什麼是它、這科之前漏讀了幾天。

- Student ID：411277043
- GitHub Username：411277043-web
- 課程：網際服務軟體工程（Web Service Software Engineering）個人 AI 協作小而美系統規劃

## 使用者與問題

**使用者**：準備期末考的大學生。

**問題**：期末考週好幾科集中在一兩週內，學生每天都要重新想「今天該讀哪一科」。常見的結果是挑喜歡的科目讀，或被最難的科目拖住，等到某科考試前一天才發現根本沒讀；前幾天沒讀完的進度，也沒有地方提醒自己。

## 主要使用情境

小明期末有三科：物理（1/9）、微積分（1/11）、英文（1/13）。1/5 他登錄三科考試並產生排程，系統排出：1/5～1/8 讀物理、1/9～1/10 讀微積分、1/11～1/12 讀英文。之後每天打開系統，只看到一件事：今天讀哪一科、距離考試幾天。1/6 他沒讀，1/7 按下重新排程，系統提醒他「物理已漏讀 1 天」。

## 系統亮點

1. **每天只給一個答案**：以「當天之後最早考的科目」決定今天讀什麼，不需要自己比較。
2. **附上判斷依據**：每筆任務都附「距離考試幾天」，讓使用者知道為什麼是這一科。
3. **漏讀提醒**：重新排程時，標示每一科先前漏讀了幾天，提醒使用者這科要加強。

亮點如何被設計支援，見 [01-requirements.md](docs/01-requirements.md) 第 4 節。

## 核心使用流程

1. 登錄每一科考試：科目名稱與考試日期（`POST /exams`）
2. 產生讀書排程，系統從今天排到最後一科考試前一天（`POST /schedules`）
3. 每天查看今天要讀的科目、距離考試天數（`GET /study-tasks?date=...`）
4. 讀完後把當天任務標記為已完成（`PATCH /study-tasks/{task_id}`）
5. 有幾天沒讀時重新產生排程，系統標示各科漏讀天數（`POST /schedules`）

## Client–Server 情境

使用者的手機或瀏覽器是 Client，透過 HTTP 呼叫 CramPlan Web Service；資料以 JSON 交換，Server 負責排程規則並保存考試與任務資料。

```
Client（瀏覽器／App） ── HTTP + JSON ──▶ CramPlan API（/api/v1） ──▶ Database
```

## 本次不做什麼

- 不依難度或每日可讀時數分配時間，每天只排一科
- 不處理同一天有兩科考試的公平分配（已知限制，見需求文件）
- 不做登入、多使用者、提醒推播、行事曆同步
- 不使用 AI 自動判斷讀書內容

## 文件與閱讀順序

| 順序 | 文件 | 對應週次 |
|---|---|---|
| 1 | [01-requirements.md](docs/01-requirements.md)：問題、需求、規則與驗收條件 | W03 |
| 2 | [02-api-design.md](docs/02-api-design.md)：RESTful API 設計 | W04 |
| 3 | [03-openapi.yaml](docs/03-openapi.yaml)：OpenAPI 3.1.0 契約 | W06 |
| 4 | [04-architecture-data-model.md](docs/04-architecture-data-model.md)：架構與資料模型 | W07 |
| 5 | [05-ai-collaboration.md](docs/05-ai-collaboration.md)：AI 協作與決策紀錄 | — |

## 設計狀態

| 項目 | 狀態 |
|---|---|
| Requirements、API Design、OpenAPI、Architecture／Data Model | 已完成 |
| OpenAPI 語法驗證 | 已通過（openapi-spec-validator、swagger-cli） |
| 程式實作、資料庫、測試 | 尚未開始（本作業不要求） |
