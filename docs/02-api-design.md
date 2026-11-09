# 02 RESTful API Design：CramPlan

> Design Only｜對應 W04 RESTful API Design｜承接 [01-requirements.md](01-requirements.md)

思考順序：Requirement → Resource → Path → Method → Status Code → Mapping。

## 1. Resources

| Resource | 說明 | 角色 | 來源 |
|---|---|---|---|
| Exam | 一科考試：科目名稱與考試日期 | 主要 Resource，需儲存 | FR-01、FR-02 |
| StudyTask | 某一天要讀的一科，含完成狀態 | 主要 Resource，需儲存 | FR-04、FR-05 |
| Schedule | 「依規則產生一批讀書任務」這個動作的結果 | Controller Resource，本身不儲存 | FR-03、FR-06 |

**為什麼 Schedule 不是 Entity**：排程的實際內容就是一批 StudyTask。產生排程是「把未完成任務換成新的一批」，產生的結果會寫進 StudyTask，不需要另外保存一份 Schedule。

## 2. URI Design

```
/exams                       考試集合
/schedules                   產生排程（每次 POST 代表一次產生）
/study-tasks                 讀書任務集合，可用 ?date= 篩選某一天
/study-tasks/{task_id}       單一讀書任務
```

- Path 使用名詞與複數，命名一律小寫、以連字號分隔（`study-tasks`）。
- 版本放在 Server URL：`/api/v1`，Path 本身保持乾淨。
- 查詢某一天的任務屬於「篩選」，使用 Query Parameter `?date=`，而不是 `/study-tasks/2027-01-07`，因為日期不是任務的識別碼。

## 3. API Operations

| # | Method | Path | 用途 | 成功 | 失敗 | 對應 FR |
|---|---|---|---|---|---|---|
| 1 | POST | `/exams` | 登錄一科考試 | 201 Created + `Location` | 400、409 | FR-01 |
| 2 | GET | `/exams` | 查看考試清單（依考試日期排序） | 200 OK | — | FR-02 |
| 3 | POST | `/schedules` | 產生或重新產生排程 | 200 OK | 400 | FR-03、FR-06 |
| 4 | GET | `/study-tasks?date={date}` | 查看指定日期的任務 | 200 OK | 400 | FR-04 |
| 5 | PATCH | `/study-tasks/{task_id}` | 標記完成或取消完成 | 200 OK | 400、404 | FR-05 |

共 5 個 Operation，比作業建議的 2～4 個多一個。我保留第 2 條 `GET /exams`：登錄考試後，使用者需要能確認科目與日期是否輸入正確，再產生排程。

## 4. Status Codes

| 碼 | 使用情境 |
|---|---|
| 200 OK | 查詢成功、排程產生成功、任務狀態更新成功 |
| 201 Created | 考試登錄成功，Header `Location` 指向新考試，例如 `/api/v1/exams/3` |
| 400 Bad Request | 欄位缺漏或格式錯誤、考試日期不晚於今天（BR-02）、開始日期之後沒有考試（BR-04）、日期格式錯誤、任務狀態不是 PENDING／DONE |
| 404 Not Found | 指定的 `task_id` 不存在 |
| 409 Conflict | 科目名稱與尚未考完的考試重複（BR-03） |

### 狀態碼判斷說明

- **查詢某天沒有任務仍是 200**：`GET /study-tasks?date=2027-02-01` 回 `200` 加上空陣列。「那天沒排任務」不等於「資源不存在」。
- **重複科目用 409，不用 400**：Request 本身格式正確，是與既有資料衝突，409 Conflict 更能表達原因，Client 也能據此提示「這科已登錄」。
- **產生排程用 200，不用 201**：這次 POST 沒有建立一個可以用 URL 取得的新資源（Schedule 不儲存），而是更新了一批任務，所以回 200 並在 Body 回傳產生的任務。

## 5. Requirement-API Mapping

| Requirement | API／行為 |
|---|---|
| FR-01 登錄考試 | `POST /exams` |
| FR-02 查看考試 | `GET /exams` |
| FR-03 產生排程 | `POST /schedules` |
| FR-04 查看指定日期任務 | `GET /study-tasks?date=` |
| FR-05 標記完成 | `PATCH /study-tasks/{task_id}` |
| FR-06 重新排程並標示漏讀 | `POST /schedules` 的行為（BR-07～BR-09） |
| BR-01～BR-03 考試資料規則 | `POST /exams` 的驗證；違反時 400 或 409 |
| BR-04～BR-06 排程規則 | `POST /schedules` 的計算規則 |
| BR-09 漏讀天數 | `StudyTask.missed_days`，出現在 #3、#4、#5 的回應中 |
| BR-10 任務狀態 | `PATCH /study-tasks/{task_id}` 的 `status` enum |

FR-03 與 FR-06 共用同一個 Endpoint：第一次產生與重新產生，套用的是同一組規則，只是後者會保留已完成與漏讀的紀錄。不需要另外設計 `/schedules/regenerate`。

## 6. Design Notes

1. **為什麼不用 `PUT /study-tasks` 取代整批任務？**
   PUT 應該是冪等的：同一個 Request 送幾次結果都一樣。但產生排程的結果取決於「今天是哪天」與「哪些任務已完成」，同一個 Request 隔天再送，結果就會不同。因此選擇 POST，並把它命名為 `/schedules`。
2. **為什麼不設計 `/exams/{exam_id}/study-tasks`？**
   使用者每天的問題是「今天讀什麼」，不是「物理有哪些任務」。以日期篩選的 `/study-tasks?date=` 更貼近主要流程。
3. **`start_date` 放在 Request Body。**
   開始日期是產生排程的輸入條件，選填，未提供時為今天。這個選填欄位讓未來測試時可以固定日期（見 `05-ai-collaboration.md` 第 3 輪）。
4. **不提供修改、刪除考試的 API。**
   這是 Out of Scope 的決定。若要支援，需先決定「修改考試日期後，已產生的任務如何處理」，留待下一版。
