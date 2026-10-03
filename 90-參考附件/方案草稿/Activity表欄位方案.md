# Activity 表欄位與系統方案（舊版提案／僅供參考）

> **文件狀態：舊版提案，僅供參考。** 本文件是早期活動系統設計，尚未定案；欄位、型別、行為與架構可能已過時。不得視為現行規格、填表依據或直接實作依據；現行設計以後續確認文件為準。

本文件中的提案**不代表 PJA-Server 或 PJA-Config 已實作**。文件以獨立的 `Activity.xlsx` 管理活動定義，活動任務由 `Mission.xlsx` 管理，活動透過 `MissionIds` 引用任務。

此專案目前可見的 `Activity` 用途包含登入活躍紀錄等玩家活動時間資料，並非遊戲限時活動配置系統；兩者是不同概念。

## 設計責任

| 資料／系統 | 責任 |
| --- | --- |
| `Activity.xlsx` | 活動基本資訊、開放與領獎時程、顯示、資格、活動期次規則、活動任務清單及活動里程碑。 |
| `Mission.xlsx` | 每個任務的目標、前置、進度週期、領取方式與任務獎勵。 |
| 活動伺服器邏輯 | 判定活動是否可見／可參加／可領獎、建立期次、處理活動積分及活動狀態。 |
| 玩家活動進度 | 保存參與資格、期次、積分、里程碑領取與任務進度關聯。 |

不建議把活動任務複製進 `Activity.xlsx`：活動以 `MissionIds` 引用 `Mission.xlsx`。任務的排序由 `MissionIds` 陣列順序決定，不在 `Mission.xlsx` 加 `SortOrder`。活動總積分里程碑若存在，屬活動獨有獎勵，可放在 `ActivityMilestones`；一般任務獎勵仍留在 Mission。

## Activity.xlsx 建議欄位

| 欄位 | Luban 型別 | 說明 |
| --- | --- | --- |
| `Id` | `int` | 活動配置唯一 ID；同一活動定義跨次重開時保持不變。 |
| `Name` | `int#ref=Shared.LocalizationTextTable` | 活動名稱文字 ID。 |
| `Description` | `int#ref=Shared.LocalizationTextTable` | 活動說明文字 ID。 |
| `Type` | `Shared.ActivityType` | 活動玩法類型，例如 `Mission`、`Login`、`PointExchange`、`Ranking`、`Gacha`；由程式決定專屬規則。 |
| `Enabled` | `bool` | 是否啟用；停用後不向玩家開放。緊急下架及既有參與者處理規則另見生命週期。 |
| `VisibilityMode` | `Shared.ActivityVisibilityMode` | `Scheduled` 到顯示時間才顯示、`Condition` 符合資格後顯示、`Always` 時程內一直顯示。 |
| `DisplayStartAt` | `long` | 活動入口開始顯示的 UTC Unix timestamp（秒）；`0` 表示與參加時間相同。 |
| `StartAt` | `long` | 開始參加時間，UTC Unix timestamp（秒）。 |
| `EndAt` | `long` | 停止參與／累積進度的時間，採半開區間 `[StartAt, EndAt)`。 |
| `ClaimEndAt` | `long` | 活動獎勵最晚領取時間；`0` 表示與 `EndAt` 相同。不可早於 `EndAt`。 |
| `MissionIds` | `array,int` | 此活動包含的 Mission ID 清單；陣列順序即活動任務顯示順序。沒有任務時可為空。 |
| `UnlockConditionId` | `int` | 額外活動資格條件 ID；`0` 表示無。 |
| `EntryConditionId` | `int` | 每次參與／兌換前需檢查的條件；`0` 表示無。與解鎖資格不同，可用來檢查單次操作要求。 |
| `ProgressScope` | `Shared.ActivityProgressScope` | `PerPlayer` 每玩家共享一份活動進度；`PerMission` 各任務各自保存；`PerPeriod` 每活動期次各自保存。 |
| `RepeatPolicy` | `Shared.ActivityRepeatPolicy` | `Once` 單次活動、`ScheduledRecurring` 依排程建立多期。每一期必須有獨立實例 ID。 |
| `ScheduleId` | `int#ref=Shared.ActivityScheduleTable` | 重複活動引用排程表；單次活動填 `0`。 |
| `ActivityMilestones` | `list,Shared.ActivityMilestone` | 活動總積分里程碑；不使用時為空。每個里程碑定義門檻與獎勵。 |
| `CurrencyId` | `int` | 活動專用積分／代幣 ID；`0` 表示不使用活動貨幣。 |
| `Icon` | `string` | 活動列表圖示資源鍵。 |
| `Banner` | `string` | 活動頁面主視覺資源鍵。 |

### 建議枚舉

| Enum | 建議值 | 說明 |
| --- | --- | --- |
| `Shared.ActivityType` | `Mission`、`Login`、`PointExchange`、`Ranking`、`Gacha`、`Boss`、`BattlePass`、`Custom` | 活動業務處理種類；`Custom` 僅在有明確通用處理或擴充鍵時使用，不能讓未知類型靜默通過。 |
| `Shared.ActivityVisibilityMode` | `Scheduled`、`Condition`、`Always` | 決定列表顯示資格；不等同是否允許參與。 |
| `Shared.ActivityProgressScope` | `PerPlayer`、`PerMission`、`PerPeriod` | 定義活動共享資料的隔離範圍；任務本身週期仍依 `Mission.ResetPolicy`。 |
| `Shared.ActivityRepeatPolicy` | `Once`、`ScheduledRecurring` | 單次開放或按排程重複開放。 |
| `Shared.ActivityState` | `Disabled`、`Upcoming`、`Visible`、`Open`、`Ended`、`Claimable`、`Archived` | 執行期狀態，由配置、伺服器時間、資格與領獎窗口推導；不是必須逐筆存入 Excel 的欄位。 |

## ActivityMilestone Bean

### Shared.ActivityMilestone

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `MilestoneId` | `int` | 此活動定義內唯一的里程碑識別碼，供領獎請求與進度資料引用。 |
| `RequiredValue` | `long` | 達到此活動積分門檻後可領取。 |
| `RewardItems` | `list,Shared.ItemEntry` | 固定獎勵清單。 |
| `RewardChoices` | `list,Shared.MissionRewardChoice` | 玩家擇一的獎勵選項；與 `RewardItems` 擇一填寫，僅手動領取。 |
| `ClaimMode` | `Shared.MissionClaimMode` | `Manual` 手動領取或 `Auto` 自動發放。 |

里程碑進度使用 Activity 的 `CurrencyId` 或該活動定義的總積分值。若獎勵可擇一，必須保存玩家選擇及冪等領取結果。若遊戲不需要活動總積分獎勵，`ActivityMilestones` 可留空，不必建立空里程碑。

## ActivitySchedule.xlsx 排程欄位

重複活動需將「活動定義」與「每期時間排程」分開。`Activity.xlsx` 的 `ScheduleId` 引用 `ActivitySchedule.xlsx`；單次活動使用 `RepeatPolicy=Once`、`ScheduleId=0`，並直接設定自身起訖時間。

| 欄位 | Luban 型別 | 說明 |
| --- | --- | --- |
| `Id` | `int` | 排程規則唯一 ID。 |
| `FirstStartAt` | `long` | 第一期開始的 UTC Unix timestamp（秒）。 |
| `RepeatIntervalSeconds` | `int` | 每期開始間隔秒數；例如每週填 `604800`。 |
| `PeriodDurationSeconds` | `int` | 每期可參加時長，必須大於 `0`。 |
| `ClaimDurationSeconds` | `int` | 每期結束後可領獎時長；`0` 表示結束時立即關閉領獎。 |
| `RepeatCount` | `int` | 總期數；`0` 表示依排程持續產生，直到活動停用。 |

第 `n` 期開始為 `FirstStartAt + n * RepeatIntervalSeconds`；參加結束為開始加 `PeriodDurationSeconds`，領獎截止為參加結束加 `ClaimDurationSeconds`。所有計算由 Server 依 UTC 執行。若需要依本地週幾／夏令時間排程，應另擴充時區與日曆規則，不要用固定秒數假裝本地週期。

## 任務、活動與獎勵的歸屬

| 需求 | 配置位置 | 例子 |
| --- | --- | --- |
| 活動期間一個任務及其完成條件 | `Mission.xlsx` | 活動期間勝利 3 場、完成登入任務。 |
| 活動頁面展示哪些任務及順序 | `Activity.xlsx: MissionIds` | `[3001, 3003, 3002]` 即照此順序顯示。 |
| 任務完成後的固定／選擇獎勵 | `Mission.xlsx` | 每個 Mission 自己的 `RewardItems` 或 `RewardChoices`。 |
| 活動總積分門檻獎勵 | `Activity.xlsx: ActivityMilestones` | 累積 100／500 活動積分領取里程碑獎勵。 |
| 活動進入資格 | `Activity.xlsx: UnlockConditionId` | 帳號等級、伺服器開服天數或回歸資格。 |
| 每次兌換或參與成本／資格 | `Activity.xlsx: EntryConditionId` 或 ActivityType 專屬設定 | 每次挑戰消耗入場券。 |

不要把活動任務另做一份 `ActivityMission.xlsx`，除非未來需要任務被多個活動期次獨立覆用、任務有獨立期次覆寫，且 `MissionIds` 已無法表達。活動任務進度仍由 Mission 系統保存，但鍵值需包含活動實例。

## 期次與執行期資料

`Activity.xlsx` 是靜態定義；單次活動由 `StartAt`／`EndAt` 描述時程，重複活動則由 `ScheduleId` 推導每期時程。玩家資料不能只以 `Activity.Id` 隔離。每次實際開放需建立穩定的 `ActivityInstanceId`，例如 `ActivityId + ScheduleVersion + PeriodStartAt`，並在活動任務與里程碑進度中保存該實例 ID。

建議執行期資料至少保存：

| 資料 | 唯一鍵／欄位 | 用途 |
| --- | --- | --- |
| 活動實例 | `ActivityInstanceId`、`ActivityId`、配置版本、開始／結束時間、狀態 | 固定某一期排程及使用的配置版本。 |
| 玩家活動狀態 | `PlayerId + ActivityInstanceId`、資格快照、首次參與時間、玩家總積分 | 防止跨期混算並保存個人活動總進度。 |
| 活動里程碑領取 | `PlayerId + ActivityInstanceId + MilestoneId`、狀態、選擇、領取時間、冪等鍵 | 每一里程碑最多成功領取一次。 |
| 活動事件去重 | 來源 `EventId`、`PlayerId`、`ActivityInstanceId`、處理結果 | 重送玩法事件時不重複累加活動積分。 |
| 任務進度 | 沿用 Mission 進度鍵並包含 `ActivityInstanceId` | 同一 Mission 被不同活動期次引用時分開保存。 |

重開活動時建立新實例；除非產品明確設計跨期累積，否則不複用上一期積分、任務進度或里程碑領取狀態。活動結束後保留歷史資料以支援補領、客服查詢與營運稽核，依資料保留政策清理。

## 時間與活動生命週期

全部配置時間建議使用 UTC Unix timestamp，展示時由客戶端依玩家／伺服器時區轉換；判斷活動狀態以 Server 時間為準。開始／結束採半開區間：`StartAt <= now < EndAt` 可參與；領獎時間則為 `EndAt <= now < ClaimEndAt`（若活動仍開放時即可領獎，可依狀態規則允許 `now < ClaimEndAt`）。

| 狀態 | 判定 | 玩家可做的事 |
| --- | --- | --- |
| `Disabled` | `Enabled=false` | 不顯示、不參加；既有已完成獎勵依下架政策保留或停止領取。 |
| `Upcoming` | 未到 `DisplayStartAt`／`StartAt` | 可選擇不展示，不能參與。 |
| `Visible` | 已到顯示時間，未到開始時間，且符合顯示資格 | 查看活動說明及倒數，不能產生進度。 |
| `Open` | `StartAt <= now < EndAt`、啟用且符合資格 | 參加活動、更新進度、完成／領取可在活動中領取的獎勵。 |
| `Ended` | 到達 `EndAt`，仍在領獎窗口 | 不可再參加或累積；可領已符合的獎勵。 |
| `Archived` | 超過 `ClaimEndAt` 或已按政策結檔 | 不可參加或領獎；僅查歷史／客服紀錄。 |

`ClaimEndAt=0` 表示採 `EndAt`。緊急停用不是單純把活動狀態改成結束：需由營運政策指定停用後是否停止累積、是否允許已完成領獎、是否延長補領；停用操作與配置版本要可稽核。

## 活動事件與積分流程

活動任務及活動積分需接在產生權威結果的玩法服務，而非客戶端或分析事件 consumer：

1. 玩家發起遊戲操作；玩法服務驗證請求、成本、資格及遊戲結果。
2. 玩法成功提交後，建立唯一來源 `EventId`，例如戰鬥結算 ID、交易 ID、登入日鍵值。
3. Mission service 更新所有符合的活動任務進度；Activity service 依活動實例、有效時段及積分規則更新總積分。
4. 事件去重紀錄與任務／積分更新以同一交易原子保存；跨服務時用可靠 gameplay outbox 加至少一次投遞，消費端按 EventId 冪等。
5. 重新檢查活動里程碑門檻，將新可領獎項目標記為可領取；不要在客戶端顯示進度時才認定達標。
6. 玩家領活動里程碑獎勵時，驗證活動期次、截止時間、門檻、未領狀態與選項，在冪等交易中發獎並標記已領。

目前 PJA-Server 的 `PublishRecord`／Record Outbox 是 Record 分析／紀錄用途，不能作為此流程的可靠 gameplay outbox；若未來活動跨服務處理，需新增合適的可靠領域事件管線或將進度處理放在可原子提交的同一服務邊界。

## 活動查詢與協定建議

World message handler 只負責解析協定並呼叫 Activity／Mission application service。可提供的玩家操作概念如下，實際協定 ID 與訊息型別留待實作：

| 操作 | Server 驗證／回傳 |
| --- | --- |
| 活動列表 | 回傳此玩家可見活動、當前狀態、期次、時程與顯示用進度摘要。 |
| 活動詳情 | 回傳任務 ID 順序、任務進度、活動積分、里程碑及可領狀態。 |
| 參與／接受活動 | 檢查 Enabled、時程、資格、每次參與條件及參與次數；建立玩家活動狀態。純展示型活動可不需單獨接受。 |
| 領活動里程碑 | 檢查活動期次、門檻、ClaimEndAt、獎勵選項與冪等鍵後發獎。 |
| 領任務獎勵 | 交由 Mission service 檢查任務自身 `ClaimMode`、`ClaimWindowSeconds` 與獎勵規則，並再受活動 `ClaimEndAt` 限制。 |

客戶端提供的 `ActivityId`、里程碑 ID 或任務 ID 只用來選擇操作對象；可參與、完成、積分和已領狀態必須由 Server 計算與保存。

## 設定驗證規則

- `Id` 大於 `0` 且唯一；名稱、說明、資源鍵與所有引用均有效。
- `RepeatPolicy=Once` 時必須 `StartAt < EndAt`；若 `DisplayStartAt` 非 `0`，不得晚於 `StartAt`；`ClaimEndAt` 為 `0` 或不早於 `EndAt`。`ScheduledRecurring` 時 `ScheduleId` 必須有效，Activity 的 `StartAt`、`EndAt`、`ClaimEndAt` 填 `0`，由排程表產生期次時間。
- `MissionIds` 不得重複；引用的任務必須存在。`Category=Activity` 的 Mission 應被至少一個活動引用；若允許未掛活動的活動任務，需明確例外規則。
- `ActivityMilestones` 的 `MilestoneId` 唯一、`RequiredValue > 0`、門檻不得重複；獎勵需恰有一種非空。
- `CurrencyId=0` 時不得配置需要活動貨幣的里程碑或兌換功能；非零 ID 必須引用有效貨幣配置。
- `RepeatPolicy=ScheduledRecurring` 必須配置有效的重複排程來源／期次產生規則；不能只反覆重用同一個玩家進度鍵。
- 排程的 `RepeatIntervalSeconds` 與 `PeriodDurationSeconds` 必須大於 `0`；`RepeatCount` 不得小於 `0`；排程期次不得因時間修改而覆用既有 `ActivityInstanceId`。
- 活動時間有重疊時，若同類活動會競爭同一資源、榜單或 UI 入口，需由 ActivityType 定義是否允許重疊。
- 活動引用任務的領獎時間上限採任務與活動限制中較早者：`min(Mission.ClaimWindowSeconds 絕對截止時間, Activity.ClaimEndAt)`。
- 停用、修改排程、變更任務清單及更換獎勵時，發布流程需記錄配置版本與遷移政策，不能靜默改寫進行中期次的規則。

## 填表示例

### 例一：限時任務活動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `501` | 活動定義 ID。 |
| `Name` | `85001` | 活動名稱文字 ID。 |
| `Type` | `Mission` | 以任務列表為主要內容。 |
| `Enabled` | `true` | 活動啟用。 |
| `VisibilityMode` | `Scheduled` | 到展示時間顯示。 |
| `DisplayStartAt` | `1798761600` | 開始前先展示入口。 |
| `StartAt` | `1798848000` | 開放參加時間。 |
| `EndAt` | `1799452800` | 停止累積進度。 |
| `ClaimEndAt` | `1799539200` | 結束後保留一天領獎。 |
| `MissionIds` | `[3001, 3002, 3003]` | 活動頁任務順序。 |
| `UnlockConditionId` | `0` | 無額外資格限制。 |
| `ProgressScope` | `PerPeriod` | 每活動期次獨立。 |
| `RepeatPolicy` | `Once` | 單次活動。 |
| `ActivityMilestones` | 空清單 | 本例沒有活動總積分獎勵。 |

任務各自的目標與任務獎勵由 Mission 3001–3003 定義；活動期次截止後不可再累積，但完成任務可在 `ClaimEndAt` 前依 Mission 領獎規則領取。

### 例二：活動積分與階梯里程碑

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `502` | 活動定義 ID。 |
| `Type` | `PointExchange` | 活動含積分／兌換玩法。 |
| `StartAt` | `1801440000` | 活動開始。 |
| `EndAt` | `1802044800` | 活動結束。 |
| `ClaimEndAt` | `1802131200` | 延後一天停止領獎。 |
| `MissionIds` | `[3101, 3102]` | 積分任務列表。 |
| `CurrencyId` | `42` | 活動積分貨幣 ID。 |
| `ProgressScope` | `PerPeriod` | 每期積分分開計算。 |
| `RepeatPolicy` | `Once` | 此活動只開一次。 |
| `ActivityMilestones` 第 1 項 | `MilestoneId=1`；`RequiredValue=100`；獎勵道具 1×100 | 100 分里程碑。 |
| `ActivityMilestones` 第 2 項 | `MilestoneId=2`；`RequiredValue=500`；獎勵道具 2×5 | 500 分里程碑。 |

Mission 3101、3102 完成後可依任務設定發任務獎勵；活動積分達門檻另可領活動里程碑獎勵。兩種獎勵各自冪等與記錄。

### 例三：每週重複開放的挑戰活動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `503` | 穩定的活動定義 ID。 |
| `Type` | `Boss` | 首領挑戰活動。 |
| `Enabled` | `true` | 排程可正常建立活動期次。 |
| `StartAt` | `0` | 循環活動的期次時間由 `ScheduleId` 產生。 |
| `EndAt` | `0` | 循環活動的期次時間由 `ScheduleId` 產生。 |
| `MissionIds` | `[3201, 3202]` | 每期任務清單。 |
| `ProgressScope` | `PerPeriod` | 每期獨立保存任務與積分。 |
| `RepeatPolicy` | `ScheduledRecurring` | 每週建立新 ActivityInstanceId。 |
| `ScheduleId` | `10` | 引用排程規則 10，例如每七天開一期。 |

排程 10 的 `FirstStartAt` 設定首期時間；後續期次由排程間隔與期數產生。週期性活動的 `StartAt`／`EndAt` 填 `0`，避免與排程表重複定義時間；每一期仍計算出有效的起訖與領獎時間。

### 例四：活動任務的唯一資料來源

| 表 | 欄位 | 設定值 |
| --- | --- | --- |
| `Activity.xlsx` | `Id` | `504` |
| `Activity.xlsx` | `MissionIds` | `[3302, 3301]` |
| `Mission.xlsx` | `Id=3301` | 任務目標與任務獎勵定義一次。 |
| `Mission.xlsx` | `Id=3302` | 任務目標與任務獎勵定義一次。 |

活動畫面先顯示 3302 再顯示 3301。活動只管理引用關係與期次，不複製任務欄位；玩家進度使用 `PlayerId + ActivityInstanceId + MissionId + PeriodKey` 隔離。

## 實作落地順序建議

1. 新增 Activity／Milestone Luban schema、enum 與表格，先完成靜態引用與時程驗證。
2. 建立 ActivityInstance 與玩家活動進度 store，定義唯一鍵、期次建立、UTC 時間及過期保留方式。
3. 實作 Activity application service：列表／詳情、資格檢查、活動狀態、里程碑領取及停用行為。
4. 實作 Mission application service 與目標處理 registry；由權威玩法服務接入，不接受客戶端上報進度。
5. 把事件去重、進度更新、任務完成與獎勵發放做成原子或冪等流程；跨服務時加入可靠 gameplay outbox。
6. 新增 World 協定 handler 與客戶端資料回傳；最後再擴展榜單、商店兌換、通行證等 ActivityType 專屬流程。

以上是方案，不包含本次新增程式碼或資料表。各 `ActivityType` 與活動玩法指標需在實際玩法確定後落到 Server enum、事件來源及設定驗證，不應只因文件列出就視為已支援。
