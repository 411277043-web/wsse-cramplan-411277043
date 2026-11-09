# 04 Architecture & Data Model：CramPlan

> Design Only｜對應 W07 Architecture & Data Modeling｜承接 [02-api-design.md](02-api-design.md)、[03-openapi.yaml](03-openapi.yaml)

## 1. Architecture Overview

採用課程的簡化分層架構：**Client → Router → Service → Repository**。

```
┌───────────────────────────┐
│ Client                    │  瀏覽器／App／Swagger UI
└─────────────┬─────────────┘
              │ HTTP + JSON               ▲ Response（符合 03-openapi.yaml）
              ▼                           │
┌───────────────────────────┐
│ Router（API Layer）       │  exams_router、schedules_router、study_tasks_router
└─────────────┬─────────────┘
              │ 呼叫 Service               ▲ 回傳結果或業務錯誤
              ▼                           │
┌───────────────────────────┐
│ Service（Business Layer） │  ExamService、ScheduleService、StudyTaskService
│                           │  └─ Clock（提供「今天」的日期）
└─────────────┬─────────────┘
              │ 讀寫資料                   ▲ 回傳 Entity
              ▼                           │
┌───────────────────────────┐
│ Repository（Data Layer）  │  ExamRepository、StudyTaskRepository
└─────────────┬─────────────┘
              ▼
          Database（尚未實作）
```

每一層只呼叫下一層：Router 不直接存取 Repository，Repository 不知道 HTTP 與排程規則。

**Clock 元件**：排程規則（BR-02、BR-04、BR-09）都依賴「今天」。把取得今天日期集中在一個 Clock 元件，未來測試時可以換成固定日期，與 `start_date` 選填欄位的目的相同（見 05 第 3 輪）。

## 2. Component Responsibilities

### Router

| 元件 | 負責的 API | 職責 |
|---|---|---|
| exams_router | `POST /exams`、`GET /exams` | 依 `ExamCreate` Schema 驗證格式；成功回 201 + `Location` |
| schedules_router | `POST /schedules` | 讀取選填的 `start_date`；回傳 `ScheduleResult` |
| study_tasks_router | `GET /study-tasks`、`PATCH /study-tasks/{task_id}` | 讀取 `date` Query 或 `task_id` Path Parameter；驗證 `status` enum |

所有 Router 共同負責：把 Service 拋出的業務錯誤轉成 `ErrorResponse` 與 Status Code（400、404、409）。

### Service

| 元件 | 職責 | 對應規則 |
|---|---|---|
| ExamService | 檢查考試日期晚於今天、科目不與尚未考完的考試重複 | BR-02、BR-03 |
| ScheduleService | 決定排程範圍、刪除舊的 PENDING 任務、逐日套用最緊急規則、建立新任務 | BR-04～BR-08 |
| StudyTaskService | 更新任務狀態；計算 `days_until_exam` 與 `missed_days` | BR-09、BR-10 |

`days_until_exam` 與 `missed_days` 由 StudyTaskService 統一計算。三條會回傳任務的 API（#3、#4、#5）都透過它組裝回應，確保算法只有一份。

### Repository

| 元件 | 職責 |
|---|---|
| ExamRepository | 新增考試；依日期排序列出；查詢某日期之後的考試；查詢科目是否已有尚未考完的考試 |
| StudyTaskRepository | 依日期查詢任務；依 id 查詢與更新；刪除某日期（含）之後的 PENDING 任務；批次新增任務；依科目統計某日期之前的 PENDING 任務數 |

### 驗證責任分工

| 規則 | 檢查位置 | 理由 |
|---|---|---|
| BR-01 科目不可空白、長度、日期格式、status enum | Router（Schema） | 只看這一筆 Request 就能判斷 |
| BR-02 考試日期晚於今天 | ExamService | 需要知道「今天」 |
| BR-03 科目不可重複 | ExamService | 需要查詢既有考試 |
| BR-04～BR-08 排程規則 | ScheduleService | 需要考試與任務資料 |

## 3. Request Flow

### 核心流程：`POST /schedules`（重新產生排程）

這條 API 承載系統的兩個亮點（每天一科、漏讀提醒），因此選它說明完整流程。

**輸入**：今天是 2027-01-07。已登錄物理 1/9（id 1）、微積分 1/11（id 2）、英文 1/13（id 3）；1/5 物理任務為 DONE，1/6 物理任務為 PENDING。

```
POST /api/v1/schedules
{ "start_date": "2027-01-07" }
```

| 步驟 | 元件 | 處理內容 | 失敗時 |
|---|---|---|---|
| 1 | schedules_router | 依 `ScheduleRequest` 檢查 `start_date` 格式；未提供則留空 | 400 `INVALID_REQUEST` |
| 2 | ScheduleService | 未提供 `start_date` 時，向 Clock 取得今天作為開始日期 | |
| 3 | ScheduleService → ExamRepository | 取得考試日期晚於 1/7 的考試：物理、微積分、英文；`end_date` = 最晚考試 1/13 的前一天 = 1/12（BR-04） | 沒有考試：400 `NO_UPCOMING_EXAMS` |
| 4 | ScheduleService → StudyTaskRepository | 刪除 1/7（含）之後所有 PENDING 任務；保留 1/5（DONE）與 1/6（PENDING）（BR-07、BR-08） | |
| 5 | ScheduleService → StudyTaskRepository | 取得 1/7～1/12 已是 DONE 的日期，這些日子跳過（BR-08）。本例沒有 | |
| 6 | ScheduleService | 逐日決定科目（BR-05、BR-06）：1/7、1/8 → 物理；1/9、1/10 → 微積分；1/11、1/12 → 英文 | |
| 7 | ScheduleService → StudyTaskRepository | 批次新增 6 筆 PENDING 任務 | |
| 8 | StudyTaskService | 為每筆任務計算 `days_until_exam`（例：1/7 物理 = 2）與 `missed_days`（物理早於今天的 PENDING 任務只有 1/6 一筆 = 1；微積分、英文 = 0）（BR-09） | |
| 9 | schedules_router → Client | 回傳 200，Body 符合 `ScheduleResult` | |

步驟 6 的逐日規則，以虛擬碼表示：

```
for day in start_date .. end_date:
    if day 已有 DONE 任務: continue                  # BR-08
    candidates = 考試日期 > day 的考試
    exam = candidates 中 exam_date 最早者；同日取 id 最小者   # BR-06
    建立 StudyTask(exam, day, PENDING)                # BR-05
```

### 補充流程：`PATCH /study-tasks/{task_id}`（標記完成）

1. study_tasks_router 讀取 `task_id`，依 `StudyTaskUpdate` 檢查 `status` 屬於 PENDING／DONE，不符回 400。
2. StudyTaskService 透過 StudyTaskRepository 查詢任務，不存在回 404 `NOT_FOUND`。
3. StudyTaskService 更新狀態（BR-10），並計算 `days_until_exam`、`missed_days`。
4. Router 回傳 200 與 `StudyTask`。

## 4. Data Model（Entity／Attribute／Key）

需要儲存的 Entity 有兩個：**Exam** 與 **StudyTask**。Schedule 是動作，不是 Entity。

### Exam

| Attribute | 型別 | Key／限制 | 來源 |
|---|---|---|---|
| `id` | integer | **PK**，系統自動遞增；同日考試時決定優先順序（BR-06） | `Exam.id` |
| `subject` | string（1～30） | 必填 | `ExamCreate.subject`（BR-01） |
| `exam_date` | date | 必填 | `ExamCreate.exam_date`（BR-02） |
| `created_at` | datetime | 必填，系統寫入 | 不對外回傳，供追蹤登錄時間 |

### StudyTask

| Attribute | 型別 | Key／限制 | 來源 |
|---|---|---|---|
| `id` | integer | **PK**，系統自動遞增 | `StudyTask.id` |
| `exam_id` | integer | **FK → Exam.id**，必填 | 排程時由 ScheduleService 決定 |
| `study_date` | date | 必填 | 排程時由 ScheduleService 決定 |
| `status` | string（enum） | 必填，PENDING／DONE，新建時為 PENDING | `TaskStatus`（BR-10） |

### 不儲存的欄位

| 欄位 | 出現在 | 計算方式 | 不儲存的理由 |
|---|---|---|---|
| `subject`、`exam_date`（任務中） | `StudyTask` | 經由 `exam_id` 從 Exam 取得 | 避免同一份資料存兩次 |
| `days_until_exam` | `StudyTask` | `exam_date − study_date` | 由既有欄位直接算出 |
| `missed_days` | `StudyTask` | 同一 `exam_id`、`study_date` 早於今天、`status = PENDING` 的筆數 | 每天都會改變，存起來很快就過期 |
| `end_date` | `ScheduleResult` | 最晚考試日期的前一天 | 每次產生排程時才有意義 |

## 5. Relationships & Design Notes

### Relationship

```
Exam                       StudyTask
───────────────            ─────────────────
id (PK)        1 ───── N   id (PK)
subject                    exam_id (FK)
exam_date                  study_date
created_at                 status
```

- 一科 Exam 對應多筆 StudyTask（1:N），每筆 StudyTask 只屬於一科 Exam。
- 一天最多一筆 StudyTask（BR-05）。實作時可在 `study_date` 加上唯一限制，由資料庫防止同一天出現兩筆任務。

### Design Notes

1. **`missed_days` 即時計算，不存在資料庫。**
   漏讀天數會隨著「今天」往後推而改變，也會因使用者補標 DONE 而減少。若存成欄位，每次狀態變動都要同步更新，容易出錯。改成查詢時計算，結果永遠正確。

2. **過去的 PENDING 任務不刪除。**
   重新排程只刪除開始日期之後的 PENDING 任務（BR-07）。過去未完成的任務保留下來，正是計算漏讀天數的資料來源。如果重新排程時把它們一起刪掉，漏讀提醒這個亮點就無法成立。

3. **用 `Exam.id` 決定同日考試的順序。**
   BR-06 需要一個穩定的排序依據，`id` 由系統遞增、代表登錄先後，因此不另外增加優先順序欄位。這也保證 NFR-01：相同資料產生相同排程。

4. **已知限制：同日考試。**
   依 BR-06，同一天考的第二科在考前分不到任何一天。我在第 3 輪已決定本版不處理；資料模型不需要為此修改，未來若要改成輪流安排，只需修改 ScheduleService 的逐日規則。

5. **待實作前確認的事項**
   - 「今天」使用哪個時區：建議以使用者所在的 UTC+8 為準。
   - 考試結束後，相關任務是否要封存或刪除。目前保留不動。

## 6. Traceability

### 完整對應表

| Requirement | API | OpenAPI Schema | Component | Entity |
|---|---|---|---|---|
| FR-01 | `POST /exams` | `ExamCreate` → `Exam` | exams_router → ExamService → ExamRepository | Exam |
| FR-02 | `GET /exams` | `Exam[]` | exams_router → ExamRepository（經 ExamService） | Exam |
| FR-03 | `POST /schedules` | `ScheduleRequest` → `ScheduleResult` | schedules_router → ScheduleService → Exam／StudyTaskRepository | Exam、StudyTask |
| FR-04 | `GET /study-tasks?date=` | `StudyTask[]` | study_tasks_router → StudyTaskService → StudyTaskRepository | StudyTask、Exam |
| FR-05 | `PATCH /study-tasks/{task_id}` | `StudyTaskUpdate` → `StudyTask` | study_tasks_router → StudyTaskService → StudyTaskRepository | StudyTask |
| FR-06 | `POST /schedules` | `StudyTask.missed_days` | ScheduleService、StudyTaskService | StudyTask（計算） |

### 追蹤範例：漏讀提醒（亮點）從需求到資料

| 層次 | 內容 |
|---|---|
| Requirement | **FR-06**：重新產生排程，並標示該科先前漏讀的天數；規則 BR-07～BR-09；驗收 AC-06-01 |
| API | `POST /schedules`，成功 200 |
| OpenAPI Schema | `ScheduleResult.tasks[]` → `StudyTask.missed_days`（integer, minimum 0） |
| Component | ScheduleService 保留過去的 PENDING 任務（BR-07）→ StudyTaskService 計算 `missed_days`（BR-09）→ StudyTaskRepository 依科目統計 |
| Entity | StudyTask：`exam_id`、`study_date`、`status` 三個欄位即可算出漏讀天數，不需新增欄位 |
