# Mission 表欄位方案（舊版提案／僅供參考）

> **文件狀態：舊版提案，僅供參考。** 本文件是早期任務系統設計，尚未定案；欄位、型別、行為與架構可能已過時。不得視為現行規格、填表依據或直接實作依據；現行設計以後續確認文件為準。

本文件中的提案**不代表 PJA-Server 或 PJA-Config 已實作**。

任務配置由 `Mission.xlsx` 定義任務規則；`Activity.xlsx` 以 `MissionIds` 指定活動期次包含的任務。活動任務順序以 `MissionIds` 陣列順序為準，`Mission.xlsx` 不設 `SortOrder`。

## Mission.xlsx 欄位

| 欄位 | Luban 型別 | 說明 |
| --- | --- | --- |
| `Id` | `int` | 任務唯一 ID。 |
| `Title` | `int#ref=Shared.LocalizationTextTable` | 任務名稱文字 ID。 |
| `Description` | `int#ref=Shared.LocalizationTextTable` | 任務說明文字 ID。 |
| `Category` | `Shared.MissionCategory` | 任務清單分類：`Main`、`Side`、`Achievement`、`Routine`、`Activity`。 |
| `GroupId` | `int` | 主線章節、成就群組等分類 ID；不分組填 `0`。 |
| `Enabled` | `bool` | 是否啟用。停用任務不新建、不更新進度；既有進度保留策略須由發布流程明定。 |
| `VisibilityMode` | `Shared.MissionVisibilityMode` | `Always` 一直顯示、`AfterUnlock` 解鎖後顯示、`AfterComplete` 完成後揭露。 |
| `ActivationMode` | `Shared.MissionActivationMode` | `Auto` 解鎖後自動開始、`Manual` 等玩家接受後開始計時與累積。 |
| `PrerequisiteMissionIds` | `array,int` | 前置任務 ID 清單；空陣列表示無前置。 |
| `PrerequisiteMode` | `Shared.MissionPrerequisiteMode` | `All` 所有前置都符合、`Any` 任一前置符合。空清單時忽略。 |
| `PrerequisiteState` | `Shared.MissionPrerequisiteState` | 前置任務至少達到 `Completed` 或 `Claimed` 狀態。 |
| `UnlockConditionId` | `int` | 額外解鎖條件 ID；`0` 表示無。 |
| `Objectives` | `list,Shared.MissionObjective` | 成功目標清單；至少一項。各目標進度分開保存。 |
| `ObjectiveMode` | `Shared.MissionObjectiveMode` | `All` 所有成功目標達標、`Any` 任一成功目標達標。 |
| `ObjectiveOrderMode` | `Shared.MissionObjectiveOrderMode` | `Parallel` 同時累積、`Sequential` 按列表順序逐項開啟。 |
| `FailureObjectives` | `list,Shared.MissionObjective` | 失敗目標清單；任一項達標即失敗。空清單表示沒有失敗目標。 |
| `TimeLimitSeconds` | `int` | 任務啟動後的完成時限秒數；`0` 表示不限時。 |
| `ExpireMode` | `Shared.MissionExpireMode` | `FailPermanent` 失敗或逾時後永久結束、`RetryNextReset` 下次重置週期再開放、`RetryAfterCooldown` 冷卻後再開放。 |
| `RetryCooldownSeconds` | `int` | `ExpireMode` 為冷卻後再開放時的冷卻秒數；其他模式填 `0`。 |
| `RepeatLimit` | `int` | 每個進度週期可完成次數；`1` 為一次，`0` 為不限次數。 |
| `ResetPolicy` | `Shared.MissionResetPolicy` | `None` 不重置、`Daily` 每日重置、`Weekly` 每週重置。 |
| `OverflowMode` | `Shared.MissionOverflowMode` | 重複任務超過門檻的累加進度：`Discard` 丟棄、`Carry` 保留至下一輪。 |
| `RepeatCooldownSeconds` | `int` | 領獎後開始下一輪前的冷卻秒數；`0` 表示立即開始。 |
| `ClaimMode` | `Shared.MissionClaimMode` | `Manual` 手動領獎、`Auto` 完成後自動發獎。 |
| `ClaimConditionId` | `int` | 領獎當下額外檢查的條件 ID；`0` 表示無。 |
| `ClaimWindowSeconds` | `int` | 完成後可領獎的秒數；`0` 表示任務本身不限領獎時間。活動結束時間仍可限制領獎。 |
| `RewardItems` | `list,Shared.ItemEntry` | 固定獎勵清單。與 `RewardChoices` 擇一填寫。 |
| `RewardChoices` | `list,Shared.MissionRewardChoice` | 玩家擇一領取的獎勵選項。與 `RewardItems` 擇一填寫；只能搭配手動領獎。 |

## Bean 結構

### Shared.MissionObjective

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `MetricType` | `Shared.MissionMetricType` | 衡量項目，例如戰鬥勝利次數、道具持有數量、建築等級。程式依種類決定事件來源與進度算法。 |
| `TargetId` | `int` | 衡量對象 ID，例如道具或建築 ID；是否可填 `0` 由 `MetricType` 定義。 |
| `TargetValue` | `long` | 達標門檻，必須大於 `0`。 |

目標進度以清單位置識別，不設 `ObjectiveId`。Server 必須按位置保存 `Objectives` 與 `FailureObjectives` 的進度。已發布任務不得任意重排、插入或刪除目標；變更時須遷移或重置既有玩家進度。

### Shared.MissionRewardChoice

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `OptionId` | `int` | 任務內獎勵選項 ID，供領獎請求識別所選項目。 |
| `Items` | `list,Shared.ItemEntry` | 選擇此項後發放的道具與數量。 |

## 任務系統架構與處理方案

本節是依 PJA-Server 目前 World handler、service、store 的分層方式提出的新系統方案；**Mission domain service、Mission 進度資料表與玩法事件介面目前尚未實作**。專案現有的 Record 發送與 `RecordOutbox` 是分析／紀錄資料管線，不作為玩家任務進度的權威來源。

**推薦落地版本：**在產生權威玩法結果的資料庫交易中，同時保存該結果與 `gameplay_outbox` 事件；Mission worker 消費事件，在自己的交易中寫去重紀錄、更新所有符合的任務進度，並建立每一輪的待領獎紀錄。玩家領獎再走獨立的冪等交易。這符合目前戰鬥結算已由 `wallet.SettleCombatGold` 自行開啟 MySQL 交易的現況。只有玩法結果與 Mission 進度確定能共用**同一筆交易**時，才改為在來源交易內直接更新 Mission，省去該來源的 outbox。兩種做法都由同一個 Mission 規則服務處理目標，避免規則分叉。

### 元件分工

| 元件 | 職責 | 不負責的事情 |
| --- | --- | --- |
| Config 載入與驗證 | 載入 `Mission.xlsx`、`Activity.xlsx` 及相關條件、獎勵設定；啟動或熱更新時檢查引用與欄位規則。 | 不保存玩家進度。 |
| World message handler | 接受玩家的接受任務、查詢任務、領獎等指令；解析請求並呼叫應用服務。 | 不信任客戶端提交的目標進度或完成狀態。 |
| Mission application/domain service | 解鎖、啟動、處理目標更新、完成／失敗、重複週期、領獎等任務規則。 | 不直接依賴特定協定封包或玩法資料表實作細節。 |
| 玩法服務 | 在戰鬥、關卡、背包、角色、公會等權威結果的同一交易內保存玩法事件；可在提交後即時通知 Mission 處理。 | 不自行修改 Mission 進度欄位。 |
| Mission store | 保存與讀取玩家任務狀態、目標位置進度、去重事件及領獎結果；提供原子更新或交易能力。 | 不決定玩法事件是否合法。 |
| Reward／背包服務 | 在領獎交易中發放固定或選擇獎勵，並避免重送重複發放。 | 不自行推導任務是否完成。 |

### 玩法事件接線

`MetricType` 是 Server 端的 enum／識別值，不會自動監聽遊戲行為，也不會因為在 Excel 填入 `CombatWinCount` 就自行增加。每一種會推進任務的玩法都要由對應的權威服務接線；不必做客戶端遙測埋點，也不應讓客戶端直接送「我完成了幾次」。

玩法事件是已發生的業務事實，例如 `CombatSettled`；`CombatWinCount` 是 Mission 對該事實推導的指標。兩者不必同名。一筆結算可同時產生勝場、指定關卡勝場、傷害量等多個任務指標。建議提供內部介面概念，例如 `MissionService.ApplyFact(ctx, fact)`。來源事實至少包含：

| 資料 | 用途 |
| --- | --- |
| `Source + OperationId + FactIndex` | 穩定來源識別；同一操作可能產生多項事實，故不能只用操作 ID 去重。 |
| `PlayerId` | 進度所屬玩家。 |
| `FactType` | 業務事實類型，例如 `CombatSettled`、`ItemConsumed`。Mission adapter 從中推導 `MissionMetricType`。 |
| 玩法參數與權威結果 | 關卡、模式、道具、角色、勝負、實際數量等可驗證資料。 |
| `OccurredAt` 及週期／期次識別 | 判斷活動、每日／每週週期與事件歸屬。 |

此介面是方案用的概念契約，實際 Go 型別與方法名稱留待實作階段決定。累加指標（如勝場、消耗量）使用權威事實的增量；目前狀態指標（如角色等級、道具持有數）在啟動或狀態變動時讀取權威快照並重算；最佳紀錄（如最快時間、最高樓層）使用專屬比較方向，不能一律 `+1`。首次上線不需為每個 `MetricType` 建立獨立 listener；按來源玩法建立少量 adapter，讓多個任務共享同一事實。

### `CombatWinCount` 增加一次的端到端流程

以例一的每日戰鬥任務為例，Mission 目標為 `MetricType=CombatWinCount`、`TargetId=0`、`TargetValue=3`：

1. Server 載入 Mission 配置；玩家符合前置與解鎖條件後，建立當日任務進度。若為 `Auto`，解鎖時開始；若為 `Manual`，接受時才開始計時與累積。
2. 玩家進入一場戰鬥，戰鬥服務建立其自身的戰鬥／結算識別碼。Mission 不因進入戰鬥或客戶端送出結果而先行加次數。
3. 戰鬥服務驗證並產生權威結算結果，確認玩家是勝方，將結算結果持久化。只有結算成功的勝利結果才形成可用於任務的事件。
4. 戰鬥結算在同一交易內保存可靠的 `CombatSettled` 事實（或與結算紀錄可穩定對應的 outbox 事件）。提交後通知 Mission；Mission adapter 檢查權威結果是否為勝利，再推導 `CombatWinCount` 增量 1。失敗、取消、未完成或無效結算不產生勝場進度。
5. Mission service 依任務是否已解鎖／啟動、事件是否在有效週期、`TargetId` 篩選與目標規則，找到符合的任務目標。若 `TargetId=0` 被該指標定義為不限關卡，才套用此例；`0` 的語義必須在 MetricType 規格中固定。
6. Mission store 以來源、操作 ID、事實序號、玩家 ID 去重，原子地保存「事實已處理」與此事實命中的所有任務進度。即使當下沒有符合任務，也要記錄已處理，避免玩家稍後啟動任務時把舊事實補算進來。
7. 更新後檢查 `ObjectiveMode`；本例一個目標從 2 到 3 後，狀態由進行中轉成完成，保存完成時間及可領獎狀態。後續勝利不再推進已完成的一次性目標。
8. 玩家查詢任務時，handler 從 Mission service 讀出服務端進度；客戶端只呈現進度，不回寫權威值。
9. 玩家領獎時，Server 再驗證完成狀態、領獎窗口、活動期次與 `ClaimConditionId`；在交易中鎖定／確認待領狀態、發放獎勵、記錄已領取及冪等請求 ID。重送同一領獎請求不會重複發獎。

若玩家在第 3 步完成結算後斷線，可靠事件仍須由 worker 重試；玩家重新登入不應依賴再次上線才補任務進度。Outbox 非同步處理會有短暫延遲：結算回應後立即查詢任務可能仍是舊進度；若產品要求立即可見，可在提交後同步觸發一次 Mission 消費，但仍須保留 outbox 重試兜底。

### 寫入一致性與去重

| 情況 | 建議處理 |
| --- | --- |
| 玩法可直接共用 Mission 交易 | 在原業務交易內寫來源結果、Mission 去重紀錄與進度；全部成功或全部回滾。 |
| 玩法已有獨立交易，或跨服務 | 在來源結果的同一交易寫 gameplay outbox；worker 至少一次投遞，Mission 端冪等處理。僅在交易提交後直接呼叫 Mission，無法防止中途當機造成漏計。 |
| 同一事件併發處理 | 以資料庫唯一鍵、條件更新或鎖定玩家任務狀態，確保只能成功套用一次。 |
| 進度更新與完成狀態切換 | 原子更新各目標進度、任務狀態、完成時間及去重紀錄。 |
| 發放固定／擇一獎勵 | 同一交易或冪等發獎操作中保存選項、發獎結果與領取狀態；不可先回應成功再非冪等地補發。 |
| 過期或跨週期事件 | 依可信 `OccurredAt`、來源期次、任務啟動時間及固定配置版本判定歸屬；不得直接加到消費時的目前週期。 |

Outbox 若新增給玩法事件使用，應是 gameplay domain 的可靠管線或明確擴充現有可靠機制；`PublishRecord` 類分析事件是 best-effort 且可能丟棄，不能保證任務進度正確。

### 玩家任務狀態與進度鍵

每筆任務狀態使用 `PlayerId + MissionId + ScopeKey` 作唯一鍵；`ScopeKey` 是不可為空的穩定字串：永久任務用固定鍵、每日／每週任務用全域重置邊界產生的週期鍵、活動任務用 `ActivityInstanceId`，活動內每日／每週任務再把期次與日／週鍵組合。不要把 SQL 可為 `NULL` 的活動 ID 直接放入唯一索引，否則不同資料庫對 `NULL` 的唯一性處理可能讓重複資料漏過。

持久化建議先用三張核心表，避免把每個目標拆成一列造成大量寫入：

| 表 | 唯一鍵 | 主要資料 |
| --- | --- | --- |
| `player_mission_progress` | `PlayerId + MissionId + ScopeKey` | 配置版本、任務狀態、啟動／截止時間、目前輪次、成功／失敗目標按列表位置的 JSON 進度、完成／領取次數、冷卻時間。 |
| `player_mission_claim` | `PlayerId + MissionId + ScopeKey + RoundNo` | 每一輪的完成時間、待領／已領狀態、`OptionId`、發獎操作 ID；一次事件完成多輪時有多筆待領紀錄。 |
| `mission_fact_inbox` | `Source + OperationId + FactIndex + PlayerId` | 已消費事實的時間與結果；與該玩家所有匹配任務進度更新同交易寫入。 |

來源玩法另需一張 `gameplay_outbox`：以 `Source + OperationId + FactIndex` 唯一識別事實，保存玩家、事實類型、可信發生時間、玩法結果、投遞狀態及重試資訊。它要與戰鬥結算、交易或其他權威來源紀錄**同交易寫入**。`mission_fact_inbox` 是消費端去重資料；兩者用途不同。Worker 可至少一次投遞，同一事實重送時由 inbox 唯一鍵擋下，不能因網路逾時猜測「應該已處理」就直接丟棄。

目標 JSON 只保存進度，不作高頻跨玩家統計查詢；若實際查詢需求要求按目標篩選玩家，再拆目標表。已發布任務的目標按列表位置識別，故配置版本必須固定在任務進度上，改動目標順序時需遷移或重置。`mission_fact_inbox` 可按保留期清理，但不得早於來源事件可能重試的期限；對永久重播的來源則需長期保留或以來源結算 ID 提供等價的去重保障。

每次狀態轉移都需明確允許的方向，例如 `Available → Active → Completed → Claimed`；`Active → Failed/Expired`。重複任務的「目前輪次進度」與「已完成但未領獎輪次」要分開：例如目標 100、`Carry`、一次增加 250，建立兩筆待領紀錄並保存下一輪進度 50；單一 `Completed` 狀態無法代表兩筆待領獎勵。若設定領獎後冷卻，需另外定義冷卻期間的超額事件處理；第一版可限制 `Carry` 只搭配 `RepeatCooldownSeconds=0`。已完成或已領獎的歷史結果不因後續可變狀態降低而倒退；未完成的狀態型目標可依最新值重新計算。

### 查詢、登入與事件更新責任

- 登入或查詢任務時載入玩家任務進度，檢查週期切換、活動開關與解鎖條件；不要靠登入事件補算所有歷史玩法事件。
- `CurrentState` 目標在解鎖／啟動時讀取當前值，並在相關狀態變動時重算；`Event` 目標只從權威玩法結果事件累積。
- 一個玩法事件可推進同一玩家的多個任務，但每個匹配目標都要套用各自的啟動狀態、週期、條件與完成上限。
- 推進進度後可回傳變更的任務摘要給當次請求，以便 World handler 通知客戶端；離線玩家進度仍由資料庫保存，下次查詢取得。
- 任務設定熱更新或停用時，不默默刪除玩家進度；沿用文件所述發布遷移政策並明確處理舊目標列表位置的相容性。

### 目前欄位的表達邊界

本文件後段的大量例子是玩法目錄，不等於目前 `MissionObjective(MetricType, TargetId, TargetValue)` 已能直接配置所有情況。零漏怪需要等於 `0`，排名與速通需要小於等於比較，同一局內完成兩項目標需要場次關聯，不同對象數需要集合去重，複合限制可能需要額外參數。實作每個 `MetricType` 前，要定義事件來源、計量單位、比較方向、作用範圍、去重方式與所需參數；現有三欄表達不了的規則應明確擴充 bean 或引用專用條件，不能把多個值偷塞進 `TargetId`。只有已完成 adapter 與配置驗證的指標才算 Server 支援。

### PJA-Server 接線位置建議

目前專案可見的訊息處理方式由 World services 將協定訊息分派至 handler，再呼叫玩法 service/store。新任務的玩家指令可沿用此路徑；玩法事實則在**產生權威業務結果的交易**內可靠保存，例如戰鬥結算、道具交易或關卡結算。不要在單純的 protocol handler 根據客戶端結果直接加任務次數，也不要依賴 Record analytics consumer 反向更新遊戲狀態。

以目前 combat submit handler 為例，它將請求交給 `combatService.Submit`；後者經 `wallet.SettleCombatGold` 在 MySQL 交易內用 `SettlementID` 去重，之後才更新戰鬥 session 及通知。若新增戰鬥任務事實，可靠事件須與金幣結算紀錄同交易落庫，或由可重放的結算紀錄可靠導出；只在 `Submit` 回傳後呼叫 Mission 會留下崩潰空窗。現有戰鬥結果亦未直接提供本文件示例的通用勝負語義，需先確認可信的勝／負判定。此文件只定義接線原則，尚未修改程式或宣稱此事件已存在。

## 普通與 VIP 同目標獎勵：沿用現有欄位

若需求是「完成同一個目標後，普通玩家可領普通獎勵，有 VIP 資格者還可領一份 VIP 獎勵」，可配置**兩筆同時啟動的 Mission**，各自保存進度與領獎紀錄，不必新增 `Mission.xlsx` 欄位。下表不涉及活動或通行證等級。

| 欄位 | 普通任務 | VIP 任務 |
| --- | --- | --- |
| `Id` | `4001` | `4002` |
| `Category` | `Side` | `Side` |
| `ActivationMode` | `Auto` | `Auto` |
| `UnlockConditionId` | `0` | `0` |
| `Objectives` | `CombatWinCount`；`TargetId=0`；`TargetValue=3` | 與 4001 相同：勝利 3 場。 |
| `ObjectiveMode`／`ObjectiveOrderMode` | `All`／`Parallel` | `All`／`Parallel` |
| `RepeatLimit`／`ResetPolicy` | `1`／`None` | `1`／`None` |
| `ClaimMode` | `Manual` | `Manual` |
| `ClaimConditionId` | `0` | `9001`，示意「目前擁有 VIP 資格」。 |
| `ClaimWindowSeconds` | `0` | `0` |
| `RewardItems` | 金幣 ×100 | 寶石 ×10 |
| `RewardChoices` | 空清單 | 空清單 |

兩筆任務從開始就同時接收勝利事件。玩家勝利 3 場後，兩筆各為 `3/3`；沒有 VIP 時 4001 可領、4002 不可領。日後取得 VIP，4002 的既有 `3/3` 進度不變，重新檢查領獎資格後即可領取。VIP 資格若有期限，每次查詢和實際領獎都應以當下資格判斷；領過的獎勵不因資格變化而重發。

查詢任務時，Server 可根據完成狀態、是否已領、領獎期限與 `ClaimConditionId` 計算執行期 `CanClaim`（或等價狀態）給 Client：4002 已完成但 `CanClaim=false` 時顯示進度與獎勵、領取按鈕變灰。`CanClaim` 不是 Excel 欄位；即使 Client 也持有 VIP 狀態並自行顯示灰階，Server 接到領獎請求時仍須再次驗證條件。

**不可把 VIP 資格放在 4002 的 `UnlockConditionId`**：事件型目標在任務啟動後才累積，玩家購買前的勝場會漏計。`ClaimConditionId=9001` 需要 Condition 系統新增可查詢 VIP 權益的運算元；目前 PJA-Server 的 Condition 尚不支援。兩筆 Mission 的目標設定要保持一致。若 UI 要將兩筆任務顯示成同一列的左右獎勵格，還需由 UI 配置或明確的配對約定指定 4001 與 4002 的關係；現有 `GroupId` 只能分組，不能保證一對一配對。

`UnlockConditionId` 與 `ClaimConditionId` 是當下的真假判定；`Objectives` 則定義要累積並保存的 `0/3`、`1/3`、`3/3` 進度。因此可共用 Condition 判定資格，但不能直接用現有 Condition 取代事件型目標的進度保存。

## 填表示例

### 例一：每日戰鬥任務

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2001` | 任務 ID。 |
| `Category` | `Routine` | 每日常規任務。 |
| `ResetPolicy` | `Daily` | 每日重置進度。 |
| `ActivationMode` | `Auto` | 解鎖後自動開始。 |
| `PrerequisiteMissionIds` | 空陣列 | 沒有前置任務。 |
| `Objectives` | `CombatWinCount`；`TargetId=0`；`TargetValue=3` | 戰鬥勝利 3 次。 |
| `ObjectiveMode` | `All` | 所有目標都要達成。 |
| `ObjectiveOrderMode` | `Parallel` | 目標同時累積。 |
| `RepeatLimit` | `1` | 每日完成一次。 |
| `ClaimMode` | `Manual` | 玩家手動領獎。 |
| `RewardItems` | `ItemId=1`；`Quantity=100` | 固定獎勵。 |
| `RewardChoices` | 空清單 | 不使用擇一獎勵。 |

玩家完成 3 場勝利戰鬥後可手動領獎。每日進度週期由遊戲全域時區與重置時間決定，不在每筆任務重複設定。

### 例二：順序目標

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2002` | 任務 ID。 |
| `Objectives` 第 1 項 | `CombatWinCount`；`TargetId=0`；`TargetValue=3` | 先取得 3 場勝利。 |
| `Objectives` 第 2 項 | `ItemUseCount`；`TargetId=1001`；`TargetValue=5` | 再使用 5 個道具 1001。 |
| `ObjectiveMode` | `All` | 兩個目標都要達成。 |
| `ObjectiveOrderMode` | `Sequential` | 第二個目標等第一個達成後才開始累積。 |

先累積 3 場勝利；第一項完成後才開始累積使用道具 1001 的次數。第二階段開啟前的道具使用不回溯計算。其他欄位使用各自預設值。

### 例三：多前置與擇一獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2003` | 任務 ID。 |
| `PrerequisiteMissionIds` | `[2001, 2002]` | 兩個前置任務。 |
| `PrerequisiteMode` | `All` | 兩個前置任務都須符合。 |
| `PrerequisiteState` | `Claimed` | 前置任務都要已領獎。 |
| `ClaimMode` | `Manual` | 玩家手動選擇並領獎。 |
| `RewardItems` | 空清單 | 不使用固定獎勵。 |
| `RewardChoices` 第 1 項 | `OptionId=1`；`ItemId=1`；`Quantity=500` | 選項一：500 個道具 1。 |
| `RewardChoices` 第 2 項 | `OptionId=2`；`ItemId=1001`；`Quantity=10` | 選項二：10 個道具 1001。 |

玩家領取任務 2001 與 2002 的獎勵後，任務 2003 才解鎖；完成後從兩組獎勵中選一組領取。

### 例四：任一目標達成即可完成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2004` | 任務 ID。 |
| `Objectives` 第 1 項 | `CombatWinCount`；`TargetId=0`；`TargetValue=1` | 戰鬥勝利 1 場。 |
| `Objectives` 第 2 項 | `ItemUseCount`；`TargetId=1001`；`TargetValue=3` | 使用 3 個道具 1001。 |
| `ObjectiveMode` | `Any` | 勝利一場或使用三個道具，任一達標即可完成。 |
| `ObjectiveOrderMode` | `Parallel` | 兩個目標同時累積。 |

### 例五：目前狀態型目標

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2005` | 任務 ID。 |
| `Objectives` 第 1 項 | `InventoryItemCount`；`TargetId=1001`；`TargetValue=100` | 目前持有 100 個道具 1001。 |
| `ObjectiveMode` | `All` | 所有目標都要達成。 |
| `RepeatLimit` | `1` | 狀態型目標只完成一次。 |
| `OverflowMode` | `Discard` | 不保留超過目標值的進度。 |

啟用任務時讀取玩家目前持有數量，之後在背包變動時重算。若玩家已有 80 個，初始進度為 80；達到 100 後完成狀態鎖定，不因之後消耗道具而撤銷。

### 例六：重複累積並保留超額進度

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2006` | 任務 ID。 |
| `Category` | `Routine` | 常規任務。 |
| `Objectives` 第 1 項 | `ActivityPointEarned`；`TargetId=0`；`TargetValue=100` | 累積 100 活動積分完成一次。 |
| `RepeatLimit` | `0` | 每個週期不限完成次數。 |
| `ResetPolicy` | `Daily` | 每日重置完成次數與剩餘進度。 |
| `OverflowMode` | `Carry` | 一次增加 250 積分時完成兩次，留下 50 進入下一輪。 |
| `RepeatCooldownSeconds` | `0` | 完成後立即開始下一輪。 |

此設定只適用於單一累加型目標。重置週期到達時，當日未達標的剩餘進度清除。

### 例七：限時挑戰，失敗後冷卻重試

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2007` | 任務 ID。 |
| `Category` | `Activity` | 活動任務，另由 `Activity.MissionIds` 引用。 |
| `ActivationMode` | `Manual` | 玩家接受任務後開始計時。 |
| `Objectives` 第 1 項 | `CombatWinCount`；`TargetId=0`；`TargetValue=1` | 勝利 1 場即可完成。 |
| `FailureObjectives` 第 1 項 | `CombatLossCount`；`TargetId=0`；`TargetValue=1` | 戰敗 1 次即失敗。 |
| `TimeLimitSeconds` | `1800` | 接受後 30 分鐘內完成。 |
| `ExpireMode` | `RetryAfterCooldown` | 失敗或逾時後，冷卻結束才能再接受。 |
| `RetryCooldownSeconds` | `3600` | 失敗後冷卻 1 小時。 |
| `ClaimWindowSeconds` | `86400` | 完成後 24 小時內領獎；若活動更早結束，以活動結束時間為準。 |

每次重試建立新任務嘗試狀態並重新開始計時；活動結束後不再接受或重試。

### 例八：主線推進與章節解鎖

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2008` | 主線任務 ID。 |
| `Category` | `Main` | 顯示在主線任務分類。 |
| `GroupId` | `1` | 主線第一章。 |
| `VisibilityMode` | `AfterUnlock` | 未解鎖前不顯示。 |
| `PrerequisiteMissionIds` | `[2007]` | 前置為任務 2007。 |
| `PrerequisiteState` | `Completed` | 前置完成即可解鎖，不要求先領獎。 |
| `Objectives` 第 1 項 | `StageClear`；`TargetId=3005`；`TargetValue=1` | 通關關卡 3005。 |
| `RewardItems` | `ItemId=1`；`Quantity=200` | 通關獎勵。 |

任務 2007 完成後才顯示本任務；通關指定關卡後，章節後續任務可依此任務作為前置解鎖。

### 例九：角色養成與指定角色突破

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2009` | 角色養成任務 ID。 |
| `Category` | `Side` | 支線任務。 |
| `GroupId` | `20` | 角色養成任務群組。 |
| `Objectives` 第 1 項 | `CharacterLevel`；`TargetId=501`；`TargetValue=40` | 角色 501 達到 40 級。 |
| `Objectives` 第 2 項 | `CharacterBreakthrough`；`TargetId=501`；`TargetValue=2` | 角色 501 突破 2 次。 |
| `ObjectiveMode` | `All` | 等級與突破都要達成。 |
| `ObjectiveOrderMode` | `Parallel` | 兩個養成狀態同時檢查。 |

這類目標讀取角色目前狀態；玩家接任務前已達成的等級或突破也可計入，接任務後再由角色資料變動重新檢查。

### 例十：探索收集與地圖探索度

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2010` | 探索任務 ID。 |
| `Category` | `Side` | 支線探索任務。 |
| `Objectives` 第 1 項 | `RegionDiscoverCount`；`TargetId=12`；`TargetValue=8` | 探索區域 12 中的 8 個探索點。 |
| `Objectives` 第 2 項 | `ChestOpenCount`；`TargetId=12`；`TargetValue=3` | 在區域 12 開啟 3 個寶箱。 |
| `ObjectiveMode` | `All` | 探索點與寶箱都要完成。 |
| `ObjectiveOrderMode` | `Parallel` | 兩項探索進度同時累積。 |

同一事件可以更新多個符合條件的任務目標；探索點與寶箱的重複事件需由來源系統去重，避免重登或重送造成多算。

### 例十一：社交功能與好友互動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2011` | 社交任務 ID。 |
| `Category` | `Routine` | 每日社交任務。 |
| `Objectives` 第 1 項 | `GiftSendCount`；`TargetId=0`；`TargetValue=3` | 送出 3 次好友禮物。 |
| `Objectives` 第 2 項 | `CoopCompleteCount`；`TargetId=0`；`TargetValue=1` | 完成 1 次合作玩法。 |
| `ObjectiveMode` | `All` | 兩種社交行為都要完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

送禮目標應在成功送出後記錄，合作目標應在合作玩法結算成功後記錄；單純點擊按鈕不計次。

### 例十二：賽季積分與段位要求

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2012` | 賽季任務 ID。 |
| `Category` | `Activity` | 由賽季活動引用。 |
| `ActivationMode` | `Auto` | 賽季開放後自動開始。 |
| `Objectives` 第 1 項 | `SeasonPointEarned`；`TargetId=0`；`TargetValue=1000` | 本賽季取得 1000 積分。 |
| `Objectives` 第 2 項 | `CurrentRank`；`TargetId=0`；`TargetValue=5` | 段位達到 5。 |
| `ObjectiveMode` | `All` | 積分與段位都須達標。 |
| `RepeatLimit` | `1` | 每個賽季期次完成一次。 |

活動期次識別用於隔離不同賽季的進度；賽季積分是累加事件，段位是目前狀態，兩種指標可在同一任務中分別追蹤。

### 例十三：付費／消耗目標與退款處理

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2013` | 商店消耗任務 ID。 |
| `Category` | `Activity` | 限時商店活動任務。 |
| `Objectives` 第 1 項 | `PremiumCurrencySpent`；`TargetId=0`；`TargetValue=500` | 活動期間有效消耗 500 單位付費貨幣。 |
| `RepeatLimit` | `1` | 活動期次完成一次。 |
| `ClaimMode` | `Manual` | 玩家手動領取。 |

消耗目標應依已完成且有效的交易事件計算，並定義退款或撤銷交易如何沖回進度；不得以客戶端回報的消耗值直接加計。

### 例十四：連續登入與中斷失敗

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2014` | 連續登入挑戰 ID。 |
| `Category` | `Achievement` | 成就挑戰。 |
| `Objectives` 第 1 項 | `ConsecutiveLoginDays`；`TargetId=0`；`TargetValue=7` | 連續登入 7 天。 |
| `FailureObjectives` 第 1 項 | `LoginStreakBroken`；`TargetId=0`；`TargetValue=1` | 連續登入中斷即失敗。 |
| `RepeatLimit` | `1` | 一次性挑戰。 |

登入日的時區與每日切換時間必須和帳號登入紀錄一致；漏登後依失敗目標結束挑戰，避免任務與登入系統各自判定出不同連續天數。

### 例十五：建造與基地發展

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2015` | 基地建造任務 ID。 |
| `Category` | `Main` | 主線基地發展任務。 |
| `Objectives` 第 1 項 | `BuildingLevel`；`TargetId=7001`；`TargetValue=5` | 建築 7001 升至 5 級。 |
| `Objectives` 第 2 項 | `BuildingCount`；`TargetId=7002`；`TargetValue=3` | 擁有 3 座建築 7002。 |
| `ObjectiveMode` | `All` | 兩種建造條件都要達成。 |
| `ClaimMode` | `Auto` | 達成後自動發獎。 |

建築等級與數量屬目前狀態型指標；拆除建築是否撤銷未完成進度，需由該指標的狀態計算規則決定。任務完成後則保留已完成結果。

### 例十六：限次副本挑戰，禁止使用復活

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2016` | 副本挑戰任務 ID。 |
| `Category` | `Activity` | 限時副本活動任務。 |
| `ActivationMode` | `Manual` | 玩家接受挑戰後開始。 |
| `Objectives` 第 1 項 | `DungeonClear`；`TargetId=9001`；`TargetValue=1` | 通關副本 9001。 |
| `FailureObjectives` 第 1 項 | `ReviveUsed`；`TargetId=0`；`TargetValue=1` | 使用復活即失敗。 |
| `TimeLimitSeconds` | `3600` | 接受後 1 小時內通關。 |
| `ExpireMode` | `RetryNextReset` | 下次週期重置時才可重試。 |
| `ResetPolicy` | `Daily` | 每日為一個挑戰週期。 |

副本進入、復活與結算事件須使用同一場次識別，避免不同場次事件被錯誤合併；失敗或逾時後當日不可再次開始。

### 例十七：裝備取得、強化與穿戴

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2017` | 裝備養成任務 ID。 |
| `Category` | `Side` | 支線任務。 |
| `Objectives` 第 1 項 | `EquipmentEnhanceCount`；`TargetId=0`；`TargetValue=5` | 成功強化裝備 5 次。 |
| `Objectives` 第 2 項 | `EquipmentEquipCount`；`TargetId=0`；`TargetValue=3` | 穿戴裝備 3 次。 |
| `ObjectiveMode` | `All` | 強化與穿戴都要達成。 |

強化失敗是否算次數須由指標定義；穿戴目標通常應計成功的裝備變更事件，而非每次登入時掃描到已穿戴狀態。

### 例十八：圖鑑收集與套裝完成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2018` | 圖鑑任務 ID。 |
| `Category` | `Achievement` | 收集成就。 |
| `Objectives` 第 1 項 | `CollectionUnlockCount`；`TargetId=0`；`TargetValue=20` | 解鎖 20 個圖鑑項目。 |
| `Objectives` 第 2 項 | `CollectionSetComplete`；`TargetId=301`；`TargetValue=1` | 完成圖鑑套組 301。 |
| `ObjectiveMode` | `All` | 總收集數與指定套組都要完成。 |

圖鑑解鎖通常是永久狀態；若圖鑑內容會因版本更新調整，應明確定義總數目標是否跟著版本變動。

### 例十九：抽卡與指定結果

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2019` | 招募任務 ID。 |
| `Category` | `Activity` | 招募活動任務。 |
| `Objectives` 第 1 項 | `GachaPullCount`；`TargetId=10`；`TargetValue=20` | 在卡池 10 抽卡 20 次。 |
| `Objectives` 第 2 項 | `GachaRarityObtainCount`；`TargetId=5`；`TargetValue=1` | 從指定卡池獲得 1 個稀有度 5 角色。 |
| `ObjectiveMode` | `All` | 抽卡次數與結果條件都要滿足。 |

抽卡次數依成功扣款且完成抽取的伺服器結果計算；重複角色是否算「獲得」以及補償轉換是否算獲得，需在指標規格定義。

### 例二十：競技場對戰與排名

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2020` | 競技場任務 ID。 |
| `Category` | `Routine` | 每週競技場任務。 |
| `Objectives` 第 1 項 | `ArenaWinCount`；`TargetId=0`；`TargetValue=10` | 取得 10 場競技場勝利。 |
| `Objectives` 第 2 項 | `ArenaBestRank`；`TargetId=0`；`TargetValue=100` | 本週最高排名達前 100 名。 |
| `ObjectiveMode` | `All` | 勝場與排名條件都要達成。 |
| `ResetPolicy` | `Weekly` | 每週重置任務週期。 |

排名名次是越小越好，需由 `ArenaBestRank` 的比較方向定義為 `目前值 <= TargetValue`，不能沿用「數值達到或超過門檻」的通用算法。

### 例二十一：公會捐獻、貢獻與首領戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2021` | 公會週任務 ID。 |
| `Category` | `Routine` | 公會每週任務。 |
| `Objectives` 第 1 項 | `GuildDonateCount`；`TargetId=0`；`TargetValue=3` | 完成 3 次公會捐獻。 |
| `Objectives` 第 2 項 | `GuildContributionEarned`；`TargetId=0`；`TargetValue=500` | 取得 500 點公會貢獻。 |
| `Objectives` 第 3 項 | `GuildBossDamage`；`TargetId=8001`；`TargetValue=100000` | 對公會首領 8001 造成 100,000 傷害。 |
| `ObjectiveMode` | `All` | 三種公會行為都要完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

若玩家退會或轉會，任務歸屬應依「事件發生時公會」或「當前公會」擇一，避免不同公會的進度被合併。

### 例二十二：好友助戰與組隊遊玩

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2022` | 好友合作任務 ID。 |
| `Category` | `Routine` | 每日社交任務。 |
| `Objectives` 第 1 項 | `FriendAssistUseCount`；`TargetId=0`；`TargetValue=3` | 使用好友助戰 3 次。 |
| `Objectives` 第 2 項 | `PartyStageClearCount`；`TargetId=3005`；`TargetValue=2` | 與隊伍通關關卡 3005 兩次。 |
| `ObjectiveMode` | `All` | 助戰與組隊通關都要完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

要明確定義「好友」是在進入時或結算時檢查，以及同一場多人戰鬥只按玩家本人結算一次。

### 例二十三：體力消耗與副本遊玩

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2023` | 每日體力任務 ID。 |
| `Category` | `Routine` | 每日常規任務。 |
| `Objectives` 第 1 項 | `StaminaSpent`；`TargetId=0`；`TargetValue=120` | 有效消耗 120 點體力。 |
| `Objectives` 第 2 項 | `DungeonClearCount`；`TargetId=9001`；`TargetValue=3` | 通關副本 9001 三次。 |
| `ObjectiveMode` | `All` | 體力消耗與副本通關都要達成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

體力消耗依伺服器實際扣除量計算；退款、戰鬥失敗是否返還體力，以及返還時是否沖回任務進度，須採一致規則。

### 例二十四：商店購買與指定商品

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2024` | 商店活動任務 ID。 |
| `Category` | `Activity` | 限時商店活動。 |
| `Objectives` 第 1 項 | `ShopPurchaseCount`；`TargetId=0`；`TargetValue=5` | 完成 5 次商店購買。 |
| `Objectives` 第 2 項 | `ShopItemPurchaseCount`；`TargetId=6001`；`TargetValue=2` | 購買商品 6001 兩次。 |
| `ObjectiveMode` | `All` | 購買次數與指定商品都要達成。 |

只有交易成功才計數；訂單取消、退款與免費領取是否屬於購買，應明確區分其事件類型。

### 例二十五：技能升級與天賦配置

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2025` | 技能養成任務 ID。 |
| `Category` | `Side` | 支線養成任務。 |
| `Objectives` 第 1 項 | `SkillLevel`；`TargetId=50101`；`TargetValue=5` | 技能 50101 升至 5 級。 |
| `Objectives` 第 2 項 | `TalentNodeUnlockCount`；`TargetId=0`；`TargetValue=3` | 解鎖 3 個天賦節點。 |
| `ObjectiveMode` | `All` | 技能等級與天賦解鎖都要達成。 |

若洗點可以撤銷天賦，需確認任務檢查的是歷史解鎖次數或目前已解鎖數；兩者應使用不同指標。

### 例二十六：製作、合成與分解

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2026` | 製作任務 ID。 |
| `Category` | `Routine` | 每週製作任務。 |
| `Objectives` 第 1 項 | `CraftSuccessCount`；`TargetId=0`；`TargetValue=10` | 成功製作 10 次。 |
| `Objectives` 第 2 項 | `SynthesisResultCount`；`TargetId=7101`；`TargetValue=2` | 合成 2 個物品 7101。 |
| `Objectives` 第 3 項 | `EquipmentDisassembleCount`；`TargetId=0`；`TargetValue=3` | 分解 3 件裝備。 |
| `ObjectiveMode` | `All` | 三種操作都要完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

製作失敗是否計次、合成產物如何識別、批次操作按操作次數或產物件數計算，都需由各 `MetricType` 明定。

### 例二十七：郵件領取與系統互動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2027` | 郵件互動任務 ID。 |
| `Category` | `Achievement` | 系統功能成就。 |
| `Objectives` 第 1 項 | `MailClaimCount`；`TargetId=0`；`TargetValue=20` | 領取 20 封郵件附件。 |
| `Objectives` 第 2 項 | `FriendAddCount`；`TargetId=0`；`TargetValue=10` | 成功新增 10 位好友。 |
| `ObjectiveMode` | `All` | 郵件與好友功能都要完成。 |

郵件批次領取應按附件領取筆數或郵件數擇一計數並固定；好友任務只計成功建立關係，不計送出邀請數。

### 例二十八：登入活躍與每日任務點數

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2028` | 每日活躍任務 ID。 |
| `Category` | `Routine` | 每日常規任務。 |
| `Objectives` 第 1 項 | `DailyActivityPoint`；`TargetId=0`；`TargetValue=100` | 取得 100 點每日活躍度。 |
| `Objectives` 第 2 項 | `LoginCount`；`TargetId=0`；`TargetValue=1` | 當日登入一次。 |
| `ObjectiveMode` | `All` | 活躍度與登入條件都要滿足。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

登入事件通常不等同活躍度；若活躍度由其他任務獎勵累積，應避免領取活躍度獎勵又反向增加產生活躍度的任務，造成循環。

### 例二十九：賽事名次與無敗挑戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2029` | 競賽挑戰任務 ID。 |
| `Category` | `Activity` | 限時賽事任務。 |
| `ActivationMode` | `Manual` | 玩家報名接受後開始。 |
| `Objectives` 第 1 項 | `TournamentRank`；`TargetId=51`；`TargetValue=3` | 賽事 51 取得前三名。 |
| `FailureObjectives` 第 1 項 | `TournamentLossCount`；`TargetId=51`；`TargetValue=1` | 賽事中落敗即失敗。 |
| `ExpireMode` | `FailPermanent` | 失敗後本期不再重試。 |
| `RepeatLimit` | `1` | 每期只完成一次。 |

名次通常在賽事結算後才確定，因此排名目標需支援結算事件；不得在名次未定時以暫時排名提前完成。

### 例三十：生存時間與波次挑戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2030` | 生存玩法任務 ID。 |
| `Category` | `Activity` | 生存活動任務。 |
| `Objectives` 第 1 項 | `SurvivalWaveReach`；`TargetId=4001`；`TargetValue=20` | 在模式 4001 到達第 20 波。 |
| `Objectives` 第 2 項 | `SurvivalDurationSeconds`；`TargetId=4001`；`TargetValue=900` | 生存至少 900 秒。 |
| `ObjectiveMode` | `All` | 波次與生存時間都要達標。 |
| `TimeLimitSeconds` | `1800` | 最長挑戰時間 30 分鐘。 |

生存時間與波次應依同一有效對局結算；中途斷線、重連及掛機是否計入，應由模式規則決定。

### 例三十一：付費訂單與累積儲值

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2031` | 儲值活動任務 ID。 |
| `Category` | `Activity` | 限時儲值活動。 |
| `Objectives` 第 1 項 | `PaidOrderCount`；`TargetId=0`；`TargetValue=3` | 完成 3 筆有效付費訂單。 |
| `Objectives` 第 2 項 | `PaidAmountAccumulated`；`TargetId=USD`；`TargetValue=10` | 累積有效付費金額達門檻。 |
| `ObjectiveMode` | `All` | 訂單筆數與付費金額都達標。 |
| `RepeatLimit` | `1` | 每活動期次完成一次。 |

付費目標應使用伺服器驗證的支付回調與標準化幣別金額；需處理退款、拒付、重複回調與不同幣別換算，不能由客戶端直接提交累積值。

### 例三十二：廣告觀看與免費資源領取

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2032` | 免費資源任務 ID。 |
| `Category` | `Routine` | 每日任務。 |
| `Objectives` 第 1 項 | `RewardedAdCompleteCount`；`TargetId=0`；`TargetValue=5` | 完整觀看並成功驗證 5 次獎勵廣告。 |
| `Objectives` 第 2 項 | `FreeResourceClaimCount`；`TargetId=2`；`TargetValue=3` | 領取資源類型 2 的免費資源 3 次。 |
| `ObjectiveMode` | `All` | 廣告與免費資源都要完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

廣告須由廣告服務回傳的有效完成憑證確認；開啟廣告但未完成、重複憑證或發獎失敗不應多計。

### 例三十三：功能教學與引導步驟

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2033` | 新手引導任務 ID。 |
| `Category` | `Main` | 主線引導任務。 |
| `VisibilityMode` | `AfterUnlock` | 前置完成後顯示。 |
| `ActivationMode` | `Auto` | 解鎖後自動開始。 |
| `Objectives` 第 1 項 | `TutorialStepComplete`；`TargetId=110`；`TargetValue=1` | 完成教學步驟 110。 |
| `PrerequisiteMissionIds` | `[2001]` | 前置戰鬥教學任務。 |
| `PrerequisiteState` | `Completed` | 前置完成即可解鎖。 |

教學完成應由伺服器可驗證的功能操作或教學狀態確認；純前端彈窗播放完成不應作為高價值獎勵條件。

### 例三十四：玩家回流與回歸階段獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2034` | 回歸活動任務 ID。 |
| `Category` | `Activity` | 回歸活動任務。 |
| `UnlockConditionId` | `5001` | 額外要求符合回歸玩家條件。 |
| `Objectives` 第 1 項 | `ReturnLoginDays`；`TargetId=0`；`TargetValue=3` | 回歸活動期間登入 3 天。 |
| `Objectives` 第 2 項 | `ReturnStageClear`；`TargetId=3005`；`TargetValue=1` | 通關關卡 3005。 |
| `ObjectiveMode` | `All` | 登入天數與關卡都要完成。 |
| `ClaimWindowSeconds` | `604800` | 完成後 7 天內領取。 |

回歸資格應由活動開啟時的帳號條件快照決定；若每次登入都動態重算，可能在玩家已開始活動後突然失去任務資格。

### 例三十五：排行榜累積與賽季結算獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2035` | 排行榜賽季任務 ID。 |
| `Category` | `Activity` | 賽季活動任務。 |
| `Objectives` 第 1 項 | `LeaderboardScore`；`TargetId=70`；`TargetValue=10000` | 排行榜 70 的賽季積分達 10,000。 |
| `Objectives` 第 2 項 | `LeaderboardFinalRank`；`TargetId=70`；`TargetValue=50` | 賽季結算排名前 50。 |
| `ObjectiveMode` | `All` | 積分與最終排名都達標。 |
| `ClaimMode` | `Manual` | 結算後手動領獎。 |

最終排名需等賽季鎖榜並完成結算後才可判定；活動停止期間與結算期間的領獎窗口要明確設定。

### 例三十六：跨系統綜合任務

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2036` | 綜合週任務 ID。 |
| `Category` | `Routine` | 每週任務。 |
| `Objectives` 第 1 項 | `StageClearCount`；`TargetId=0`；`TargetValue=5` | 通關任意關卡 5 次。 |
| `Objectives` 第 2 項 | `GuildContributionEarned`；`TargetId=0`；`TargetValue=300` | 取得 300 公會貢獻。 |
| `Objectives` 第 3 項 | `EquipmentEnhanceCount`；`TargetId=0`；`TargetValue=3` | 強化裝備 3 次。 |
| `ObjectiveMode` | `All` | 三個系統目標全部完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

跨系統任務可以由多個事件來源共同推進；各來源需用穩定的事件 ID 去重，並確保事件重試不會重複加進度。

### 例三十七：PvP 擊殺與助攻

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2037` | PvP 對戰任務 ID。 |
| `Category` | `Routine` | 每週 PvP 任務。 |
| `Objectives` 第 1 項 | `PvpKillCount`；`TargetId=0`；`TargetValue=20` | 在玩家對戰中擊敗 20 名對手。 |
| `Objectives` 第 2 項 | `PvpAssistCount`；`TargetId=0`；`TargetValue=10` | 累積 10 次有效助攻。 |
| `ObjectiveMode` | `All` | 擊敗與助攻都需達成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

只計正式結算且非同隊的有效 PvP 場次；訓練場、投降局與重複結算事件依玩法規則排除或去重。

### 例三十八：競技場防守成功

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2038` | 競技場防守任務 ID。 |
| `Category` | `Achievement` | 競技場成就。 |
| `Objectives` 第 1 項 | `ArenaDefenseWinCount`；`TargetId=0`；`TargetValue=10` | 防守競技場成功 10 次。 |
| `Objectives` 第 2 項 | `ArenaDefenseRank`；`TargetId=0`；`TargetValue=500` | 防守排名進入前 500 名。 |
| `ObjectiveMode` | `All` | 防守勝場與排名皆須達成。 |

防守勝利以伺服器對戰結果為準；排名為越小越好的狀態值，需使用小於等於門檻的比較方式。

### 例三十九：多人團隊首領傷害

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2039` | 團隊首領任務 ID。 |
| `Category` | `Activity` | 團隊首領活動。 |
| `Objectives` 第 1 項 | `RaidBossDamage`；`TargetId=8101`；`TargetValue=1000000` | 對首領 8101 累積造成 1,000,000 傷害。 |
| `Objectives` 第 2 項 | `RaidClearCount`；`TargetId=8101`；`TargetValue=1` | 完成首領戰 1 次。 |
| `ObjectiveMode` | `All` | 傷害與通關條件都要達成。 |
| `RepeatLimit` | `1` | 每個活動期次一次。 |

傷害需依個人有效貢獻累加，或明確指定按團隊結果共同計進度；避免同一份團隊傷害被所有玩家重複領作個人貢獻。

### 例四十：多人副本救援與支援

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2040` | 多人副本支援任務 ID。 |
| `Category` | `Routine` | 每週合作任務。 |
| `Objectives` 第 1 項 | `AllyReviveCount`；`TargetId=0`；`TargetValue=5` | 救援隊友 5 次。 |
| `Objectives` 第 2 項 | `SupportSkillUseCount`；`TargetId=4201`；`TargetValue=10` | 使用支援技能 4201 十次。 |
| `ObjectiveMode` | `All` | 救援與技能使用都需完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

救援需是成功復活其他玩家；自我復活、重放事件或同一救援被多個結算來源重複回報不可重複計次。

### 例四十一：交易市場上架與成交

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2041` | 市場交易任務 ID。 |
| `Category` | `Achievement` | 經濟系統成就。 |
| `Objectives` 第 1 項 | `MarketListingCount`；`TargetId=0`；`TargetValue=10` | 成功上架 10 次。 |
| `Objectives` 第 2 項 | `MarketSaleCount`；`TargetId=0`；`TargetValue=5` | 完成 5 次市場成交。 |
| `ObjectiveMode` | `All` | 上架與成交均需完成。 |

上架後取消不應算成交；是否計入上架次數由產品規則決定。成交事件需使用交易 ID 去重，退款或回滾需定義是否沖回進度。

### 例四十二：貨幣累積與資產持有

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2042` | 資產累積任務 ID。 |
| `Category` | `Achievement` | 長期經濟成就。 |
| `Objectives` 第 1 項 | `CurrencyEarned`；`TargetId=3`；`TargetValue=100000` | 累積取得貨幣 3 共 100,000。 |
| `Objectives` 第 2 項 | `CurrencyBalance`；`TargetId=3`；`TargetValue=50000` | 目前持有貨幣 3 至少 50,000。 |
| `ObjectiveMode` | `All` | 累積取得量與當前持有量都達標。 |

累積取得量是歷史事件總和，持有量是目前狀態；消耗貨幣不應減少前者，但會影響後者。

### 例四十三：收集角色與隊伍編成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2043` | 角色收集任務 ID。 |
| `Category` | `Achievement` | 收集成就。 |
| `Objectives` 第 1 項 | `CharacterObtainCount`；`TargetId=0`；`TargetValue=30` | 累積取得 30 名角色。 |
| `Objectives` 第 2 項 | `PartyElementCount`；`TargetId=2`；`TargetValue=3` | 目前隊伍中配置至少 3 名元素類型 2 的角色。 |
| `ObjectiveMode` | `All` | 累積收集及當前編隊都要達成。 |

角色取得數要定義重複角色是否計入；編隊目標則是目前狀態，換隊後未完成任務的進度可能下降。

### 例四十四：圖鑑完成度與稀有收藏

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2044` | 圖鑑完成任務 ID。 |
| `Category` | `Achievement` | 長期收藏成就。 |
| `Objectives` 第 1 項 | `CollectionCompletionRate`；`TargetId=1`；`TargetValue=80` | 圖鑑分類 1 完成度達 80%。 |
| `Objectives` 第 2 項 | `RareCollectionUnlockCount`；`TargetId=4`；`TargetValue=5` | 解鎖 5 個稀有度 4 收藏。 |
| `ObjectiveMode` | `All` | 完成度與稀有收藏數都要達標。 |

完成度目標應使用整數百分比或固定分母規則，並處理新增圖鑑項目時分母變動造成的完成度下降。

### 例四十五：寵物培養與出戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2045` | 寵物養成任務 ID。 |
| `Category` | `Side` | 寵物支線任務。 |
| `Objectives` 第 1 項 | `PetLevel`；`TargetId=2001`；`TargetValue=30` | 寵物 2001 達到 30 級。 |
| `Objectives` 第 2 項 | `PetBondLevel`；`TargetId=2001`；`TargetValue=5` | 寵物 2001 親密度達到 5 級。 |
| `Objectives` 第 3 項 | `PetBattleUseCount`；`TargetId=2001`；`TargetValue=10` | 使用寵物 2001 出戰 10 次。 |
| `ObjectiveMode` | `All` | 等級、親密度與出戰次數都需達成。 |

等級與親密度可按目前狀態檢查；出戰次數是歷史事件，兩種進度不可共用同一個計算方式。

### 例四十六：坐騎取得、培養與騎乘

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2046` | 坐騎系統任務 ID。 |
| `Category` | `Achievement` | 坐騎成就。 |
| `Objectives` 第 1 項 | `MountObtainCount`；`TargetId=0`；`TargetValue=5` | 取得 5 種坐騎。 |
| `Objectives` 第 2 項 | `MountLevel`；`TargetId=6001`；`TargetValue=10` | 坐騎 6001 達到 10 級。 |
| `Objectives` 第 3 項 | `MountRideDistance`；`TargetId=0`；`TargetValue=10000` | 騎乘移動累積 10,000 公尺。 |
| `ObjectiveMode` | `All` | 三項都完成。 |

騎乘距離只應累加有效移動，需排除傳送、掛機與非騎乘狀態，並考慮速度作弊及重複上報。

### 例四十七：家園拜訪與互動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2047` | 家園社交任務 ID。 |
| `Category` | `Routine` | 每週社交任務。 |
| `Objectives` 第 1 項 | `OtherHomeVisitCount`；`TargetId=0`；`TargetValue=5` | 拜訪 5 位不同玩家的家園。 |
| `Objectives` 第 2 項 | `HomeLikeCount`；`TargetId=0`；`TargetValue=3` | 獲得 3 個有效讚。 |
| `ObjectiveMode` | `All` | 拜訪與獲讚都要完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

不同訪客或按讚是否要求唯一玩家由指標規則定義；自刷帳號與撤銷按讚需有防重及回滾策略。

### 例四十八：釣魚、採集與生活技能

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2048` | 生活技能任務 ID。 |
| `Category` | `Routine` | 每日生活任務。 |
| `Objectives` 第 1 項 | `FishCatchCount`；`TargetId=0`；`TargetValue=10` | 成功釣起 10 條魚。 |
| `Objectives` 第 2 項 | `GatherResourceCount`；`TargetId=3001`；`TargetValue=20` | 採集資源 3001 共 20 個。 |
| `Objectives` 第 3 項 | `CookingSuccessCount`；`TargetId=0`；`TargetValue=5` | 成功料理 5 次。 |
| `ObjectiveMode` | `All` | 釣魚、採集與料理都需完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

採集按資源件數或採集操作次數計算需固定；料理失敗是否計次也需明確定義。

### 例四十九：隨機事件與世界事件參與

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2049` | 世界事件任務 ID。 |
| `Category` | `Activity` | 世界事件活動任務。 |
| `Objectives` 第 1 項 | `WorldEventParticipateCount`；`TargetId=901`；`TargetValue=3` | 參與世界事件 901 三次。 |
| `Objectives` 第 2 項 | `WorldEventContribution`；`TargetId=901`；`TargetValue=1000` | 累積貢獻達 1000。 |
| `ObjectiveMode` | `All` | 參與次數及貢獻都達標。 |
| `RepeatLimit` | `1` | 每個活動期次完成一次。 |

只有符合參與條件並完成結算才計入；世界事件需以場次 ID 區分，避免同一事件重連或重領獎重複計算。

### 例五十：劇情選擇與分支結局

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2050` | 劇情分支任務 ID。 |
| `Category` | `Achievement` | 劇情收集成就。 |
| `Objectives` 第 1 項 | `StoryChapterClear`；`TargetId=120`；`TargetValue=1` | 完成章節 120。 |
| `Objectives` 第 2 項 | `StoryEndingUnlock`；`TargetId=3`；`TargetValue=1` | 解鎖結局 3。 |
| `ObjectiveMode` | `All` | 章節與指定結局都要達成。 |
| `VisibilityMode` | `AfterComplete` | 達成揭露條件後才顯示任務。 |

故事分支若只能擇一且無法重玩，任務資料須說明互斥分支的可達成策略；`AfterComplete` 的具體揭露條件也要由 Server 規則明確定義。

### 例五十一：難度通關與限制條件

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2051` | 高難度挑戰任務 ID。 |
| `Category` | `Achievement` | 挑戰成就。 |
| `Objectives` 第 1 項 | `StageClearDifficulty`；`TargetId=3005`；`TargetValue=5` | 以難度 5 通關關卡 3005。 |
| `FailureObjectives` 第 1 項 | `BattleItemUseCount`；`TargetId=0`；`TargetValue=1` | 使用戰鬥道具即挑戰失敗。 |
| `TimeLimitSeconds` | `900` | 單次挑戰限時 15 分鐘。 |
| `ExpireMode` | `RetryAfterCooldown` | 失敗或逾時後冷卻再試。 |
| `RetryCooldownSeconds` | `1800` | 失敗後冷卻 30 分鐘。 |

難度需由伺服器結算資料確認；禁止操作類條件須能取得同場次事件，避免任務啟動前或其他戰鬥中的使用被誤判。

### 例五十二：戰鬥評價與連擊

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2052` | 戰鬥技巧任務 ID。 |
| `Category` | `Achievement` | 戰鬥成就。 |
| `Objectives` 第 1 項 | `BattleGradeCount`；`TargetId=5`；`TargetValue=10` | 取得評價 5 的戰鬥結算 10 次。 |
| `Objectives` 第 2 項 | `MaxComboReach`；`TargetId=0`；`TargetValue=50` | 單場最高連擊達 50。 |
| `ObjectiveMode` | `All` | 高評價次數與連擊條件都要完成。 |

連擊需以單場最高值判定，不能把不同戰鬥中的連擊相加；評價規則版本變更時應確認歷史進度相容性。

### 例五十三：每日／每週階梯里程碑

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2053` | 每日里程碑任務 ID。 |
| `Category` | `Routine` | 每日任務。 |
| `Objectives` 第 1 項 | `DailyMissionCompleteCount`；`TargetId=0`；`TargetValue=5` | 完成 5 個每日任務。 |
| `RepeatLimit` | `1` | 每日領取一次里程碑獎勵。 |
| `ResetPolicy` | `Daily` | 每日重置。 |
| `RewardItems` | `ItemId=1`；`Quantity=300` | 里程碑獎勵。 |

若此任務由其他每日任務完成事件推進，需避免它自己也被計入完成數而形成循環；「完成」應定義為達標或已領獎其中一種。

### 例五十四：連續完成日常的連勝紀錄

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2054` | 日常連勝成就 ID。 |
| `Category` | `Achievement` | 長期成就。 |
| `Objectives` 第 1 項 | `DailyMissionStreak`；`TargetId=0`；`TargetValue=14` | 連續 14 天完成指定每日任務條件。 |
| `FailureObjectives` 第 1 項 | `DailyMissionStreakBreak`；`TargetId=0`；`TargetValue=1` | 任一日未達條件則本輪連勝中斷。 |
| `RepeatLimit` | `1` | 一次性成就。 |

連勝日切換必須使用統一的每日週期；每日任務完成資料需能在重置後供連勝檢查讀取。

### 例五十五：限時排行榜分段獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2055` | 排行榜分段活動任務 ID。 |
| `Category` | `Activity` | 限時排行榜活動。 |
| `Objectives` 第 1 項 | `LeaderboardTierReach`；`TargetId=71`；`TargetValue=3` | 排行榜 71 達到分段 3。 |
| `Objectives` 第 2 項 | `LeaderboardParticipationCount`；`TargetId=71`；`TargetValue=5` | 參與排行榜玩法 5 次。 |
| `ObjectiveMode` | `All` | 分段與參與次數都達標。 |
| `ClaimWindowSeconds` | `172800` | 完成後 48 小時內領取。 |

分段值是目前狀態還是歷史最高值需明確指定；排行榜關閉後的結算獎勵仍需受活動期次與領獎期限管理。

### 例五十六：資源田採集與運輸護送

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2056` | 資源運輸任務 ID。 |
| `Category` | `Routine` | 每週玩法任務。 |
| `Objectives` 第 1 項 | `ResourceGatherAmount`；`TargetId=3`；`TargetValue=50000` | 採集資源種類 3 共 50,000。 |
| `Objectives` 第 2 項 | `CaravanEscortCompleteCount`；`TargetId=0`；`TargetValue=3` | 完成 3 次運輸護送。 |
| `ObjectiveMode` | `All` | 採集量與護送次數都要完成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

採集量以實際入帳數量或採集產出擇一計算；被掠奪或運輸失敗是否影響進度須依玩法結算規則處理。

### 例五十七：拍賣競標與競標成功

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2057` | 拍賣玩法任務 ID。 |
| `Category` | `Activity` | 限時拍賣活動。 |
| `Objectives` 第 1 項 | `AuctionBidCount`；`TargetId=0`；`TargetValue=5` | 提交有效競標 5 次。 |
| `Objectives` 第 2 項 | `AuctionWinCount`；`TargetId=0`；`TargetValue=1` | 成功得標 1 次。 |
| `ObjectiveMode` | `All` | 競標次數與得標都達成。 |

流標、撤回、無效出價與退款需各自區分；出價請求重試應以競標交易 ID 去重。

### 例五十八：角色好感劇情與送禮

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2058` | 角色好感任務 ID。 |
| `Category` | `Side` | 角色支線任務。 |
| `Objectives` 第 1 項 | `CharacterGiftCount`；`TargetId=501`；`TargetValue=5` | 送給角色 501 五次禮物。 |
| `Objectives` 第 2 項 | `CharacterAffectionLevel`；`TargetId=501`；`TargetValue=4` | 角色 501 好感度達 4 級。 |
| `Objectives` 第 3 項 | `BondStoryComplete`；`TargetId=501`；`TargetValue=1` | 完成角色 501 好感劇情。 |
| `ObjectiveMode` | `All` | 送禮、好感度與劇情都要達成。 |

送禮次數屬事件累積，好感度屬目前狀態，劇情完成屬永久旗標，應分別採用適合的進度語義。

### 例五十九：染色、外觀與拍照分享

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2059` | 外觀玩法任務 ID。 |
| `Category` | `Achievement` | 外觀成就。 |
| `Objectives` 第 1 項 | `CostumeObtainCount`；`TargetId=0`；`TargetValue=10` | 取得 10 套外觀。 |
| `Objectives` 第 2 項 | `DyeApplyCount`；`TargetId=0`；`TargetValue=5` | 成功套用染色 5 次。 |
| `Objectives` 第 3 項 | `PhotoShareCount`；`TargetId=0`；`TargetValue=3` | 成功分享 3 張照片。 |
| `ObjectiveMode` | `All` | 三項外觀玩法都要完成。 |

取得外觀是否包含限時租用、染色重複套用是否計次、分享是否要求平台確認成功，均需由指標定義清楚。

### 例六十：載具競速與賽道紀錄

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2060` | 競速玩法任務 ID。 |
| `Category` | `Activity` | 限時競速活動。 |
| `Objectives` 第 1 項 | `RaceFinishCount`；`TargetId=6101`；`TargetValue=5` | 完成賽道 6101 五次。 |
| `Objectives` 第 2 項 | `RaceBestTimeMs`；`TargetId=6101`；`TargetValue=90000` | 最佳時間達到 90,000 毫秒以內。 |
| `ObjectiveMode` | `All` | 完賽次數與最佳時間都達成。 |

競速時間是越小越好，需支援 `目前最佳時間 <= TargetValue` 的比較；成績必須經伺服器驗證或反作弊檢查。

### 例六十一：防守配置與基地防禦

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2061` | 基地防守任務 ID。 |
| `Category` | `Achievement` | 防禦成就。 |
| `Objectives` 第 1 項 | `DefenseWinCount`；`TargetId=0`；`TargetValue=10` | 成功防守 10 次。 |
| `Objectives` 第 2 項 | `DefenseTrapTriggerCount`；`TargetId=0`；`TargetValue=20` | 防守陷阱觸發 20 次。 |
| `ObjectiveMode` | `All` | 防守勝利與陷阱觸發都要完成。 |

防守勝利與陷阱觸發需排除玩家自行測試、同場重複結算或無效攻擊事件。

### 例六十二：治療量與承傷量

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2062` | 戰鬥支援成就 ID。 |
| `Category` | `Achievement` | 長期戰鬥成就。 |
| `Objectives` 第 1 項 | `HealingAmount`；`TargetId=0`；`TargetValue=500000` | 累積有效治療量 500,000。 |
| `Objectives` 第 2 項 | `DamageTakenAmount`；`TargetId=0`；`TargetValue=1000000` | 累積承受傷害 1,000,000。 |
| `ObjectiveMode` | `All` | 治療與承傷都要達標。 |

有效治療需定義過量治療是否計入；承傷需排除自傷或無效傷害來源，並以結算事件去重。

### 例六十三：元素反應與戰鬥操作

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2063` | 戰鬥操作任務 ID。 |
| `Category` | `Achievement` | 戰鬥成就。 |
| `Objectives` 第 1 項 | `ElementReactionTriggerCount`；`TargetId=4`；`TargetValue=30` | 觸發元素反應 4 共 30 次。 |
| `Objectives` 第 2 項 | `UltimateSkillUseCount`；`TargetId=0`；`TargetValue=10` | 使用終極技能 10 次。 |
| `ObjectiveMode` | `All` | 反應與技能使用都要完成。 |

複合反應是否同時計入多種類型、一次技能命中多名敵人算幾次，需固定事件粒度避免版本差異。

### 例六十四：戰力門檻與隊伍評分

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2064` | 戰力成就 ID。 |
| `Category` | `Achievement` | 長期成就。 |
| `Objectives` 第 1 項 | `AccountPower`；`TargetId=0`；`TargetValue=100000` | 帳號戰力達到 100,000。 |
| `Objectives` 第 2 項 | `PartyPower`；`TargetId=1`；`TargetValue=50000` | 編隊 1 戰力達到 50,000。 |
| `ObjectiveMode` | `All` | 帳號與指定編隊戰力都達標。 |

戰力屬狀態指標，需明確定義裝備、增益、臨時效果是否計入，以及戰力下降是否影響未完成任務。

### 例六十五：帳號安全與綁定功能

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2065` | 帳號功能任務 ID。 |
| `Category` | `Achievement` | 帳號成就。 |
| `Objectives` 第 1 項 | `AccountBindProviderCount`；`TargetId=0`；`TargetValue=1` | 綁定至少一種外部登入提供者。 |
| `Objectives` 第 2 項 | `AccountSecuritySetup`；`TargetId=0`；`TargetValue=1` | 完成指定帳號安全設定。 |
| `ObjectiveMode` | `All` | 綁定與安全設定都完成。 |

帳號安全任務是否提供獎勵應審慎設計；不得在任務進度或日誌記錄敏感憑證，只保存完成狀態。

### 例六十六：使用者回饋與遊戲內評分

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2066` | 遊戲回饋活動任務 ID。 |
| `Category` | `Activity` | 限時回饋活動。 |
| `Objectives` 第 1 項 | `SurveySubmitCount`；`TargetId=700`；`TargetValue=1` | 完成問卷 700 一次。 |
| `Objectives` 第 2 項 | `GameReviewSubmit`；`TargetId=0`；`TargetValue=1` | 完成遊戲內評分流程。 |
| `ObjectiveMode` | `All` | 問卷與評分流程都完成。 |

只記錄問卷或評分流程完成旗標，不應把自由文字回饋內容放進任務進度資料。

### 例六十七：觀看劇情、動畫與過場

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2067` | 劇情觀看任務 ID。 |
| `Category` | `Main` | 主線任務。 |
| `Objectives` 第 1 項 | `CutsceneComplete`；`TargetId=1205`；`TargetValue=1` | 完整播放過場 1205。 |
| `Objectives` 第 2 項 | `DialogueChoiceSelect`；`TargetId=4`；`TargetValue=1` | 選擇對話選項 4。 |
| `ObjectiveMode` | `All` | 播放與指定選擇都要完成。 |

可略過過場是否算完成、劇情跳轉後是否補記完成旗標，需與劇情系統保持一致。

### 例六十八：每日商店刷新與免費兌換

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2068` | 每日商店任務 ID。 |
| `Category` | `Routine` | 每日任務。 |
| `Objectives` 第 1 項 | `ShopFreeRefreshCount`；`TargetId=2`；`TargetValue=1` | 使用商店 2 的免費刷新一次。 |
| `Objectives` 第 2 項 | `ShopFreeExchangeCount`；`TargetId=2`；`TargetValue=1` | 在商店 2 免費兌換一次。 |
| `ObjectiveMode` | `All` | 刷新與兌換都完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

免費刷新與付費刷新應是不同事件類型；任務領獎造成的商店資源變動不可反向觸發同一任務條件。

### 例六十九：跨服玩法與跨服排名

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2069` | 跨服競賽任務 ID。 |
| `Category` | `Activity` | 跨服活動。 |
| `Objectives` 第 1 項 | `CrossServerMatchCompleteCount`；`TargetId=81`；`TargetValue=10` | 完成跨服模式 81 十場。 |
| `Objectives` 第 2 項 | `CrossServerRank`；`TargetId=81`；`TargetValue=100` | 跨服排名前 100。 |
| `ObjectiveMode` | `All` | 對戰場次及排名皆達標。 |
| `RepeatLimit` | `1` | 每個活動期次完成一次。 |

跨服排名與場次事件需帶活動期次及伺服器區域識別，避免不同賽季或分區資料混算。

### 例七十：新手保護期與初期成長階段

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2070` | 新手成長任務 ID。 |
| `Category` | `Main` | 新手主線任務。 |
| `UnlockConditionId` | `5100` | 帳號處於新手成長階段才可解鎖。 |
| `Objectives` 第 1 項 | `AccountLevel`；`TargetId=0`；`TargetValue=20` | 帳號等級達 20。 |
| `Objectives` 第 2 項 | `FeatureUnlockCount`；`TargetId=0`；`TargetValue=5` | 解鎖 5 項指定功能。 |
| `ObjectiveMode` | `All` | 等級及功能解鎖都需完成。 |
| `ClaimWindowSeconds` | `0` | 任務完成後不另設領獎期限。 |

新手階段資格若會隨時間或等級失效，應定義已解鎖任務是否仍可完成與領獎，避免資格更新造成玩家進度消失。

### 例七十一：科技研究與科技樹解鎖

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2071` | 科技發展任務 ID。 |
| `Category` | `Main` | 主線科技任務。 |
| `Objectives` 第 1 項 | `TechnologyResearchCompleteCount`；`TargetId=0`；`TargetValue=5` | 完成 5 項科技研究。 |
| `Objectives` 第 2 項 | `TechnologyNodeUnlock`；`TargetId=1205`；`TargetValue=1` | 解鎖科技節點 1205。 |
| `ObjectiveMode` | `All` | 研究數量與指定節點都要完成。 |

研究完成是歷史事件，節點解鎖是持有狀態；若科技可重置，需定義任務是否保留已完成記錄。

### 例七十二：工廠生產與訂單交付

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2072` | 生產經營任務 ID。 |
| `Category` | `Routine` | 每日經營任務。 |
| `Objectives` 第 1 項 | `ProductionCompleteCount`；`TargetId=0`；`TargetValue=10` | 完成 10 次生產。 |
| `Objectives` 第 2 項 | `OrderDeliveryCount`；`TargetId=5002`；`TargetValue=3` | 完成交付訂單類型 5002 三次。 |
| `ObjectiveMode` | `All` | 生產與訂單都需達成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

生產佇列完成、領取產物與訂單交付是不同事件，需選定任務計數時點，避免同一產物重複計入。

### 例七十三：城市人口與建築繁榮度

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2073` | 城市發展成就 ID。 |
| `Category` | `Achievement` | 長期經營成就。 |
| `Objectives` 第 1 項 | `CityPopulation`；`TargetId=1`；`TargetValue=10000` | 城市人口達 10,000。 |
| `Objectives` 第 2 項 | `CityProsperity`；`TargetId=1`；`TargetValue=50000` | 城市繁榮度達 50,000。 |
| `ObjectiveMode` | `All` | 人口與繁榮度都需達標。 |

兩項皆為目前狀態值；人口或繁榮度下降時，未完成任務應重新判斷，已完成成就則保持完成。

### 例七十四：領地佔領與據點控制

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2074` | 領地爭奪任務 ID。 |
| `Category` | `Activity` | 領地戰活動。 |
| `Objectives` 第 1 項 | `TerritoryCaptureCount`；`TargetId=0`；`TargetValue=3` | 佔領 3 個據點。 |
| `Objectives` 第 2 項 | `TerritoryHoldSeconds`；`TargetId=7001`；`TargetValue=1800` | 控制領地 7001 累積 30 分鐘。 |
| `ObjectiveMode` | `All` | 佔領數與控制時間都需達標。 |
| `RepeatLimit` | `1` | 每個活動期次完成一次。 |

控制時間需定義按玩家個人、公會或隊伍歸屬計算；中立、爭奪及離線期間是否計時要固定。

### 例七十五：賽季通行證經驗與等級

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2075` | 賽季通行證任務 ID。 |
| `Category` | `Activity` | 通行證賽季任務。 |
| `Objectives` 第 1 項 | `BattlePassExpEarned`；`TargetId=31`；`TargetValue=5000` | 賽季 31 累積取得 5,000 通行證經驗。 |
| `Objectives` 第 2 項 | `BattlePassLevel`；`TargetId=31`；`TargetValue=20` | 通行證等級達 20。 |
| `ObjectiveMode` | `All` | 經驗累積與等級都需達標。 |
| `RepeatLimit` | `1` | 每賽季完成一次。 |

經驗值與等級可能互相推導；若保留兩項，需說明兩者代表不同規則，否則只配置其中一項即可。

### 例七十六：賽季段位晉級與保級

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2076` | 段位賽任務 ID。 |
| `Category` | `Activity` | 段位賽季任務。 |
| `Objectives` 第 1 項 | `RankTierReach`；`TargetId=32`；`TargetValue=6` | 賽季 32 曾達段位 6。 |
| `Objectives` 第 2 項 | `RankMatchCompleteCount`；`TargetId=32`；`TargetValue=20` | 完成 20 場段位賽。 |
| `ObjectiveMode` | `All` | 最高段位與對戰場次都要達成。 |
| `RepeatLimit` | `1` | 每賽季完成一次。 |

「曾達段位」應記錄賽季最高段位；若要檢查目前段位，應使用另一個狀態型指標，避免降段後任務倒退的歧義。

### 例七十七：戰鬥禁用治療挑戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2077` | 無治療挑戰任務 ID。 |
| `Category` | `Activity` | 限時挑戰。 |
| `ActivationMode` | `Manual` | 接受後開始挑戰。 |
| `Objectives` 第 1 項 | `DungeonClear`；`TargetId=9101`；`TargetValue=1` | 通關副本 9101。 |
| `FailureObjectives` 第 1 項 | `HealingItemUseCount`；`TargetId=0`；`TargetValue=1` | 使用治療道具即失敗。 |
| `TimeLimitSeconds` | `1200` | 挑戰限時 20 分鐘。 |
| `ExpireMode` | `RetryAfterCooldown` | 失敗後冷卻再挑戰。 |
| `RetryCooldownSeconds` | `3600` | 冷卻 1 小時。 |

限制事件需關聯本次挑戰場次；場外使用治療道具不可使任務失敗。

### 例七十八：單角色通關與隊伍限制

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2078` | 單角色通關挑戰 ID。 |
| `Category` | `Achievement` | 高難度成就。 |
| `Objectives` 第 1 項 | `StageClearWithPartySize`；`TargetId=3008`；`TargetValue=1` | 以指定隊伍條件通關關卡 3008。 |
| `Objectives` 第 2 項 | `StageClearDifficulty`；`TargetId=3008`；`TargetValue=7` | 難度 7 通關。 |
| `ObjectiveMode` | `All` | 隊伍限制與難度均須符合。 |

此處隊伍限制由 `MetricType` 配套條件表解讀，若種類與角色清單都需配置，應擴充 Objective bean 或引用專用條件 ID，而非把多個值塞進 `TargetId`。

### 例七十九：戰鬥中切換角色與連續技能

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2079` | 戰鬥操作成就 ID。 |
| `Category` | `Achievement` | 操作技巧成就。 |
| `Objectives` 第 1 項 | `BattleSwitchCharacterCount`；`TargetId=0`；`TargetValue=20` | 戰鬥中切換角色 20 次。 |
| `Objectives` 第 2 項 | `SkillComboCompleteCount`；`TargetId=12`；`TargetValue=5` | 完成連段配置 12 共 5 次。 |
| `ObjectiveMode` | `All` | 切換次數與指定連段都達成。 |

技能連段應由戰鬥結算或權威戰鬥事件確認，避免客戶端單純上報操作序列就增加進度。

### 例八十：任務鏈章節完成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2080` | 任務鏈章節總結任務 ID。 |
| `Category` | `Main` | 主線任務。 |
| `GroupId` | `2` | 主線第二章。 |
| `PrerequisiteMissionIds` | `[2081, 2082, 2083]` | 前置為三個章節任務。 |
| `PrerequisiteMode` | `All` | 三項都要完成。 |
| `PrerequisiteState` | `Completed` | 前置達成即可解鎖。 |
| `Objectives` 第 1 項 | `ChapterBossClear`；`TargetId=2`；`TargetValue=1` | 擊敗第二章首領。 |
| `RewardItems` | `ItemId=1`；`Quantity=1000` | 章節獎勵。 |

多前置任務可用來匯合分支任務；若分支互斥，需確保至少一條可行路徑能滿足 `All`，或改用 `Any`。

### 例八十一：地圖探索率與隱藏區域

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2081` | 地圖探索成就 ID。 |
| `Category` | `Achievement` | 探索成就。 |
| `Objectives` 第 1 項 | `MapExplorationPercent`；`TargetId=15`；`TargetValue=100` | 地圖 15 探索率達 100%。 |
| `Objectives` 第 2 項 | `SecretAreaDiscoverCount`；`TargetId=15`；`TargetValue=5` | 發現 5 個隱藏區域。 |
| `ObjectiveMode` | `All` | 探索率與隱藏區域都達成。 |

探索百分比需固定分母或版本化分母；新增探索點後是否重新計算舊玩家的百分比應有明確規則。

### 例八十二：傳送點與地標解鎖

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2082` | 地標解鎖任務 ID。 |
| `Category` | `Side` | 支線探索任務。 |
| `Objectives` 第 1 項 | `WaypointUnlockCount`；`TargetId=15`；`TargetValue=10` | 解鎖地圖 15 的 10 個傳送點。 |
| `Objectives` 第 2 項 | `LandmarkInteractCount`；`TargetId=0`；`TargetValue=5` | 啟動 5 個地標。 |
| `ObjectiveMode` | `All` | 傳送點與地標都要完成。 |

若重置世界狀態或開新週目，目標需限定目前週目、帳號永久解鎖或角色資料範圍。

### 例八十三：成就點數階梯獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2083` | 成就點數里程碑 ID。 |
| `Category` | `Achievement` | 成就系統里程碑。 |
| `Objectives` 第 1 項 | `AchievementPointTotal`；`TargetId=0`；`TargetValue=500` | 累積取得 500 成就點數。 |
| `ClaimMode` | `Manual` | 玩家手動領取。 |
| `RewardItems` | `ItemId=1`；`Quantity=500` | 里程碑獎勵。 |

成就點數來源應排除尚未領取或已回滾的成就，並定義成就重置時點數是否保留。

### 例八十四：賽季任務點數與階段解鎖

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2084` | 賽季任務階段 ID。 |
| `Category` | `Activity` | 賽季活動任務。 |
| `Objectives` 第 1 項 | `SeasonMissionPoint`；`TargetId=33`；`TargetValue=100` | 賽季 33 任務點數達 100。 |
| `PrerequisiteMissionIds` | `[2075]` | 先完成通行證任務。 |
| `PrerequisiteState` | `Claimed` | 前置獎勵領取後才開放。 |
| `RepeatLimit` | `1` | 每賽季完成一次。 |

若點數是由其他任務完成回饋，需防止此任務本身也產生該點數而形成循環。

### 例八十五：回合制戰鬥評價與剩餘回合

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2085` | 回合戰鬥挑戰 ID。 |
| `Category` | `Achievement` | 戰鬥成就。 |
| `Objectives` 第 1 項 | `TurnBattleClear`；`TargetId=5201`；`TargetValue=1` | 通關回合戰鬥 5201。 |
| `Objectives` 第 2 項 | `TurnBattleRemainingTurn`；`TargetId=5201`；`TargetValue=5` | 通關時至少剩 5 回合。 |
| `ObjectiveMode` | `All` | 通關及回合條件皆滿足。 |

剩餘回合是越大越好的結算值；需由指標明定比較方向，不能套用回合數越少越好的速度任務規則。

### 例八十六：速度通關與競速紀錄

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2086` | 速通成就 ID。 |
| `Category` | `Achievement` | 挑戰成就。 |
| `Objectives` 第 1 項 | `StageClear`；`TargetId=3010`；`TargetValue=1` | 通關關卡 3010。 |
| `Objectives` 第 2 項 | `StageClearTimeMs`；`TargetId=3010`；`TargetValue=180000` | 3 分鐘內通關。 |
| `ObjectiveMode` | `All` | 通關與時間條件都達成。 |

通關時間為越小越好，使用毫秒可避免小數精度差異；計時起訖點應由關卡結算事件統一提供。

### 例八十七：戰鬥中不倒地挑戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2087` | 無倒地挑戰 ID。 |
| `Category` | `Activity` | 限時挑戰活動。 |
| `Objectives` 第 1 項 | `StageClear`；`TargetId=3011`；`TargetValue=1` | 通關關卡 3011。 |
| `FailureObjectives` 第 1 項 | `PartyKnockoutCount`；`TargetId=0`；`TargetValue=1` | 任一隊員倒地即失敗。 |
| `TimeLimitSeconds` | `1800` | 限時 30 分鐘。 |
| `ExpireMode` | `RetryNextReset` | 下個每日週期才可重試。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

倒地、死亡、復活與戰鬥結束事件需定義先後順序；若通關與倒地同時到達，結算結果必須有固定優先規則。

### 例八十八：傷害排名與 MVP 評價

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2088` | 團隊貢獻任務 ID。 |
| `Category` | `Routine` | 每週團隊任務。 |
| `Objectives` 第 1 項 | `RaidPersonalDamageRank`；`TargetId=8102`；`TargetValue=3` | 在首領戰個人傷害排名前三。 |
| `Objectives` 第 2 項 | `MvpAwardCount`；`TargetId=0`；`TargetValue=2` | 獲得 MVP 評價兩次。 |
| `ObjectiveMode` | `All` | 傷害排名與 MVP 次數都達成。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

名次、平手規則與 MVP 頒發需由結算服務定義；不得根據客戶端顯示值計算。

### 例八十九：救援委託與懸賞目標

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2089` | 懸賞委託任務 ID。 |
| `Category` | `Activity` | 限時懸賞活動。 |
| `Objectives` 第 1 項 | `BountyTargetDefeat`；`TargetId=9901`；`TargetValue=3` | 擊敗懸賞目標 9901 三次。 |
| `Objectives` 第 2 項 | `BountyRescueCount`；`TargetId=0`；`TargetValue=2` | 完成兩次救援委託。 |
| `ObjectiveMode` | `All` | 擊敗目標與救援委託都要完成。 |
| `ClaimWindowSeconds` | `86400` | 完成後 24 小時內領獎。 |

目標重生或委託刷新需帶任務實例 ID，避免重複擊敗同一實例或跨期擊殺錯誤計入。

### 例九十：隨機副本詞綴挑戰

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2090` | 詞綴副本挑戰 ID。 |
| `Category` | `Activity` | 詞綴副本活動任務。 |
| `Objectives` 第 1 項 | `DungeonClearWithAffix`；`TargetId=9102`；`TargetValue=1` | 通關副本 9102 並符合詞綴條件。 |
| `Objectives` 第 2 項 | `DungeonAffixCount`；`TargetId=4`；`TargetValue=3` | 同場啟用指定類型的 3 個詞綴。 |
| `ObjectiveMode` | `All` | 通關與詞綴條件都達標。 |
| `RepeatLimit` | `1` | 每個副本活動期次一次。 |

若詞綴組合需要多值配置，應增加專用條件資料或複合目標 bean；目前 `TargetId` 只能表達單一識別值。

### 例九十一：限時生產與交付競賽

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2091` | 生產競賽任務 ID。 |
| `Category` | `Activity` | 限時經營活動。 |
| `Objectives` 第 1 項 | `ProductionItemAmount`；`TargetId=7102`；`TargetValue=100` | 活動期間生產物品 7102 共 100 個。 |
| `Objectives` 第 2 項 | `OrderDeliveryScore`；`TargetId=6002`；`TargetValue=5000` | 交付訂單類型 6002 累積 5000 分。 |
| `ObjectiveMode` | `All` | 生產數量與交付分數都達成。 |
| `TimeLimitSeconds` | `604800` | 開始後 7 天內完成。 |

生產活動窗口與個人任務時限需比較並採較早截止；已在佇列但活動結束後才完成的生產是否計入需明訂。

### 例九十二：好友邀請與新玩家加入

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2092` | 邀請好友活動任務 ID。 |
| `Category` | `Activity` | 邀請活動。 |
| `Objectives` 第 1 項 | `ReferralSignupCount`；`TargetId=0`；`TargetValue=3` | 3 位受邀新玩家完成註冊。 |
| `Objectives` 第 2 項 | `ReferralLevelReachCount`；`TargetId=10`；`TargetValue=3` | 3 位受邀玩家達到 10 級。 |
| `ObjectiveMode` | `All` | 註冊與等級條件都達標。 |
| `ClaimMode` | `Manual` | 手動領取。 |

邀請歸因需使用伺服器驗證的推薦關係，定義自邀、重複裝置、刪帳重建與受邀者更換推薦人的處理方式。

### 例九十三：公會招募與成員成長

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2093` | 公會招募任務 ID。 |
| `Category` | `Activity` | 公會活動任務。 |
| `Objectives` 第 1 項 | `GuildMemberRecruitCount`；`TargetId=0`；`TargetValue=5` | 招募 5 位新公會成員。 |
| `Objectives` 第 2 項 | `GuildMemberLevelUpCount`；`TargetId=0`；`TargetValue=10` | 公會成員累積升級 10 次。 |
| `ObjectiveMode` | `All` | 招募與成員成長都達成。 |
| `RepeatLimit` | `1` | 每活動期次一次。 |

成員離會後是否保留活動貢獻按活動規則決定；成員升級事件需去重，避免同步資料重放造成多算。

### 例九十四：分享邀請碼與社群任務

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2094` | 社群推廣任務 ID。 |
| `Category` | `Activity` | 社群活動任務。 |
| `Objectives` 第 1 項 | `InviteCodeShareCount`；`TargetId=0`；`TargetValue=3` | 透過遊戲內功能分享邀請碼 3 次。 |
| `Objectives` 第 2 項 | `InviteCodeRedeemCount`；`TargetId=0`；`TargetValue=1` | 有效邀請碼被新玩家使用一次。 |
| `ObjectiveMode` | `All` | 分享與有效兌換皆需完成。 |

外部平台是否真的送達通常無法由遊戲確認；較可靠的任務目標是伺服器成功建立分享請求或邀請碼被有效兌換。

### 例九十五：直播觀看與觀戰參與

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2095` | 觀戰活動任務 ID。 |
| `Category` | `Activity` | 賽事觀戰活動。 |
| `Objectives` 第 1 項 | `LiveWatchSeconds`；`TargetId=501`；`TargetValue=1800` | 觀看賽事頻道 501 累積 30 分鐘。 |
| `Objectives` 第 2 項 | `LivePredictionSubmitCount`；`TargetId=501`；`TargetValue=3` | 提交 3 次有效賽事預測。 |
| `ObjectiveMode` | `All` | 觀看時間與預測都要達成。 |

觀看時間需定義前景、心跳與閒置判定；若影音平台提供可信觀看回調，進度應以伺服器可驗證資料為準。

### 例九十六：季節活動收集與兌換

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2096` | 季節活動收集任務 ID。 |
| `Category` | `Activity` | 節慶活動任務。 |
| `Objectives` 第 1 項 | `EventTokenEarned`；`TargetId=41`；`TargetValue=1000` | 活動 41 累積取得代幣 1000 枚。 |
| `Objectives` 第 2 項 | `EventShopExchangeCount`；`TargetId=41`；`TargetValue=5` | 活動商店兌換 5 次。 |
| `ObjectiveMode` | `All` | 代幣取得與兌換都達成。 |
| `ResetPolicy` | `None` | 由活動期次隔離進度，不使用每日／每週週期。 |

活動代幣取得是歷史累積，兌換是成功交易事件；活動關閉後是否仍允許領獎由活動配置決定。

### 例九十七：看板委託刷新與完成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2097` | 委託看板任務 ID。 |
| `Category` | `Routine` | 每日委託任務。 |
| `Objectives` 第 1 項 | `BountyBoardCompleteCount`；`TargetId=0`；`TargetValue=5` | 完成 5 個看板委託。 |
| `Objectives` 第 2 項 | `BountyBoardRefreshCount`；`TargetId=0`；`TargetValue=2` | 刷新委託看板 2 次。 |
| `ObjectiveMode` | `All` | 完成與刷新都達成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

免費刷新與付費刷新可用 `TargetId` 或不同 `MetricType` 區分；刷新委託後完成舊委託是否計入應明確規範。

### 例九十八：角色派遣與遠征

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2098` | 角色派遣任務 ID。 |
| `Category` | `Routine` | 每日派遣任務。 |
| `Objectives` 第 1 項 | `ExpeditionStartCount`；`TargetId=0`；`TargetValue=3` | 派遣遠征 3 次。 |
| `Objectives` 第 2 項 | `ExpeditionCompleteCount`；`TargetId=0`；`TargetValue=2` | 領取完成的遠征結果 2 次。 |
| `ObjectiveMode` | `All` | 派遣與完成都需達成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

遠征開始、完成、領取結果是不同狀態事件；跨日遠征需定義以開始日或領取日計入日常進度。

### 例九十九：離線收益與回收

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2099` | 離線收益任務 ID。 |
| `Category` | `Routine` | 每日成長任務。 |
| `Objectives` 第 1 項 | `OfflineRewardClaimCount`；`TargetId=0`；`TargetValue=1` | 領取離線收益一次。 |
| `Objectives` 第 2 項 | `OfflineRewardItemAmount`；`TargetId=1`；`TargetValue=1000` | 領取至少 1000 個資源 1。 |
| `ObjectiveMode` | `All` | 領取次數與數量都達標。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

收益生成與領取分開計算；玩家使用加倍道具或回收功能時，須決定按原始收益還是實際入帳收益作為任務進度。

### 例一百：資料片／章節探索綜合里程碑

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2100` | 資料片探索里程碑 ID。 |
| `Category` | `Main` | 資料片主線任務。 |
| `GroupId` | `10` | 資料片章節群組。 |
| `PrerequisiteMissionIds` | `[2081, 2082]` | 先完成地圖探索與地標任務。 |
| `PrerequisiteMode` | `Any` | 任一前置完成即可開啟。 |
| `Objectives` 第 1 項 | `WorldBossClearCount`；`TargetId=9902`；`TargetValue=1` | 擊敗世界首領 9902。 |
| `Objectives` 第 2 項 | `RegionDiscoverCount`；`TargetId=16`；`TargetValue=12` | 發現區域 16 的 12 個地點。 |
| `ObjectiveMode` | `All` | 首領與探索目標都要完成。 |

此例示範任務前置可用 `Any`，成功目標仍可用 `All`；兩種模式分別控制解鎖條件與完成條件。

### 例一百零一：卡牌收集與牌組構築

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2101` | 卡牌玩法任務 ID。 |
| `Category` | `Achievement` | 卡牌收集成就。 |
| `Objectives` 第 1 項 | `CardObtainCount`；`TargetId=0`；`TargetValue=50` | 累積取得 50 張卡牌。 |
| `Objectives` 第 2 項 | `DeckPower`；`TargetId=1`；`TargetValue=20000` | 牌組 1 戰力達 20,000。 |
| `ObjectiveMode` | `All` | 收集數與牌組戰力都要達標。 |

卡牌取得數是否包含重複卡需明定；牌組戰力是目前狀態，換牌後未完成任務可重新判定。

### 例一百零二：卡牌升星與技能升級

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2102` | 卡牌養成任務 ID。 |
| `Category` | `Side` | 卡牌支線任務。 |
| `Objectives` 第 1 項 | `CardStarLevel`；`TargetId=30101`；`TargetValue=5` | 卡牌 30101 升至 5 星。 |
| `Objectives` 第 2 項 | `CardSkillLevel`；`TargetId=3010102`；`TargetValue=3` | 卡牌技能 3010102 升至 3 級。 |
| `ObjectiveMode` | `All` | 星級與技能都要達標。 |

升星與技能等級皆為狀態型目標；若卡牌可重置或轉化，需決定任務完成紀錄是否永久保留。

### 例一百零三：棋盤消除關卡與連鎖數

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2103` | 棋盤關卡任務 ID。 |
| `Category` | `Routine` | 每日玩法任務。 |
| `Objectives` 第 1 項 | `PuzzleStageClearCount`；`TargetId=0`；`TargetValue=5` | 通關任意消除關卡 5 次。 |
| `Objectives` 第 2 項 | `PuzzleMaxChain`；`TargetId=0`；`TargetValue=8` | 單次連鎖達 8 段。 |
| `ObjectiveMode` | `All` | 通關次數與連鎖目標都達成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

最高連鎖是單局最佳值，不應把多局連鎖相加；關卡結算資料需避免客戶端自行偽造。

### 例一百零四：塔防守住波次與防線耐久

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2104` | 塔防挑戰任務 ID。 |
| `Category` | `Activity` | 塔防限時活動。 |
| `Objectives` 第 1 項 | `DefenseWaveReach`；`TargetId=101`；`TargetValue=30` | 模式 101 到達第 30 波。 |
| `Objectives` 第 2 項 | `DefenseBaseHpPercent`；`TargetId=101`；`TargetValue=50` | 結束時基地剩餘耐久至少 50%。 |
| `ObjectiveMode` | `All` | 波次與基地耐久都符合。 |

基地耐久百分比為結算條件且越高越好；必須明確定義分母與結算時點。

### 例一百零五：Roguelike 單次通關與局內祝福

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2105` | Roguelike 挑戰任務 ID。 |
| `Category` | `Activity` | Roguelike 活動任務。 |
| `Objectives` 第 1 項 | `RoguelikeRunClear`；`TargetId=2`；`TargetValue=1` | 通關難度 2 的一場挑戰。 |
| `Objectives` 第 2 項 | `RoguelikeBlessingCollectCount`；`TargetId=0`；`TargetValue=10` | 單場取得 10 個祝福。 |
| `ObjectiveMode` | `All` | 通關與祝福數需在有效挑戰中達成。 |
| `TimeLimitSeconds` | `7200` | 單次挑戰最多 2 小時。 |

需以同一場 RunId 關聯局內取得與通關結果，防止把多次失敗場次的祝福累積成通關條件。

### 例一百零六：Roguelike 事件選擇與無傷首領

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2106` | Roguelike 特殊成就 ID。 |
| `Category` | `Achievement` | 長期挑戰成就。 |
| `Objectives` 第 1 項 | `RoguelikeEventChoiceCount`；`TargetId=301`；`TargetValue=3` | 單場選擇事件類型 301 三次。 |
| `Objectives` 第 2 項 | `RoguelikeBossClearNoDamage`；`TargetId=9003`；`TargetValue=1` | 無受傷擊敗首領 9003。 |
| `ObjectiveMode` | `All` | 事件選擇與無傷首領條件都完成。 |

局內目標須帶 RunId；受傷定義需排除護盾吸收或明確包含，避免不同戰鬥規則解讀不同。

### 例一百零七：賽車競速與漂移得分

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2107` | 賽車玩法任務 ID。 |
| `Category` | `Routine` | 每週競速任務。 |
| `Objectives` 第 1 項 | `RaceFinishCount`；`TargetId=6201`；`TargetValue=5` | 完成賽道 6201 五次。 |
| `Objectives` 第 2 項 | `DriftScoreEarned`；`TargetId=0`；`TargetValue=10000` | 累積漂移得分 10,000。 |
| `ObjectiveMode` | `All` | 完賽次數與漂移得分都達標。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

漂移分數應使用有效對局結算分，並說明單場最佳或跨場累積；例中定義為跨場累積。

### 例一百零八：賽車名次與指定車輛

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2108` | 賽車名次任務 ID。 |
| `Category` | `Activity` | 限時賽事任務。 |
| `Objectives` 第 1 項 | `RaceRankReach`；`TargetId=6202`；`TargetValue=3` | 賽道 6202 取得前三名。 |
| `Objectives` 第 2 項 | `RaceVehicleUseCount`；`TargetId=7101`；`TargetValue=3` | 使用車輛 7101 完賽三次。 |
| `ObjectiveMode` | `All` | 名次與指定車輛使用條件都要達成。 |

名次數值越小越好；車輛需在有效完賽時確認，僅在比賽開始時選擇但中途退出不計。

### 例一百零九：模擬經營客戶滿意與營收

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2109` | 店鋪經營任務 ID。 |
| `Category` | `Achievement` | 經營成就。 |
| `Objectives` 第 1 項 | `ShopCustomerSatisfaction`；`TargetId=1`；`TargetValue=90` | 店鋪 1 滿意度達 90。 |
| `Objectives` 第 2 項 | `ShopRevenueEarned`；`TargetId=1`；`TargetValue=100000` | 店鋪 1 累積營收 100,000。 |
| `ObjectiveMode` | `All` | 滿意度與累積營收皆達標。 |

滿意度是目前狀態，營收是歷史累積；營收需定義扣除退單、稅費或退款前後的計算口徑。

### 例一百一十：農場播種、收成與訂單

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2110` | 農場玩法任務 ID。 |
| `Category` | `Routine` | 每日農場任務。 |
| `Objectives` 第 1 項 | `CropHarvestCount`；`TargetId=8001`；`TargetValue=20` | 收成作物 8001 二十次。 |
| `Objectives` 第 2 項 | `FarmOrderCompleteCount`；`TargetId=0`；`TargetValue=3` | 完成 3 張農場訂單。 |
| `ObjectiveMode` | `All` | 收成與訂單都要完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

種植不代表收成；作物被偷取、枯萎或補領是否計入，需依實際產出事件決定。

### 例一百一十一：寵物競賽與訓練

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2111` | 寵物競賽任務 ID。 |
| `Category` | `Activity` | 寵物競賽活動。 |
| `Objectives` 第 1 項 | `PetRaceCompleteCount`；`TargetId=1`；`TargetValue=5` | 完成賽事類型 1 五次。 |
| `Objectives` 第 2 項 | `PetRaceRank`；`TargetId=1`；`TargetValue=3` | 賽事類型 1 取得前三名。 |
| `ObjectiveMode` | `All` | 完賽次數與名次條件都達成。 |
| `RepeatLimit` | `1` | 本活動期次一次。 |

名次目標需要結算排名資料；若可在同一場重複領取結果，應以賽事場次 ID 去重。

### 例一百一十二：音樂節奏準確度與連擊

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2112` | 音樂玩法任務 ID。 |
| `Category` | `Achievement` | 音樂成就。 |
| `Objectives` 第 1 項 | `RhythmSongClearCount`；`TargetId=5001`；`TargetValue=10` | 完成歌曲 5001 十次。 |
| `Objectives` 第 2 項 | `RhythmAccuracyPercent`；`TargetId=5001`；`TargetValue=95` | 歌曲 5001 準確率達 95%。 |
| `ObjectiveMode` | `All` | 完成次數與準確率都達標。 |

準確率按整數百分比、千分比或更高精度擇一固定；需使用有效結算資料，並處理同場重送。

### 例一百一十三：音樂玩法全連與難度通關

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2113` | 音樂挑戰任務 ID。 |
| `Category` | `Activity` | 音樂活動任務。 |
| `Objectives` 第 1 項 | `RhythmFullCombo`；`TargetId=5002`；`TargetValue=1` | 歌曲 5002 達成全連。 |
| `Objectives` 第 2 項 | `RhythmDifficultyClear`；`TargetId=5002`；`TargetValue=8` | 歌曲 5002 難度 8 通關。 |
| `ObjectiveMode` | `All` | 全連與難度通關均達成。 |

若全連本身已包含通關，兩個目標可保留作為獨立條件示例；實際配置應避免重複且沒有額外意義的目標。

### 例一百一十四：桌遊對局與任務目標完成

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2114` | 桌遊玩法任務 ID。 |
| `Category` | `Routine` | 每週桌遊任務。 |
| `Objectives` 第 1 項 | `BoardGameMatchCompleteCount`；`TargetId=3`；`TargetValue=10` | 完成桌遊模式 3 十場。 |
| `Objectives` 第 2 項 | `BoardGameWinCount`；`TargetId=3`；`TargetValue=5` | 桌遊模式 3 獲勝五場。 |
| `ObjectiveMode` | `All` | 對局與勝場皆達標。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

中途離局是否算完成對局需定義；勝利目標只在完整結算後更新。

### 例一百一十五：棋牌牌型與指定勝法

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2115` | 棋牌成就 ID。 |
| `Category` | `Achievement` | 棋牌成就。 |
| `Objectives` 第 1 項 | `CardHandTypeWinCount`；`TargetId=7`；`TargetValue=3` | 以牌型 7 獲勝三次。 |
| `Objectives` 第 2 項 | `CardGameWinCount`；`TargetId=0`；`TargetValue=10` | 棋牌總勝場達十場。 |
| `ObjectiveMode` | `All` | 指定牌型勝利及總勝場都完成。 |

牌型由伺服器結算識別；同一局符合多個牌型時，需確定採最高牌型或全部符合均計數。

### 例一百一十六：塔防塔種建造與升級

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2116` | 塔防養成任務 ID。 |
| `Category` | `Side` | 塔防支線任務。 |
| `Objectives` 第 1 項 | `TowerBuildCount`；`TargetId=301`；`TargetValue=10` | 建造塔種 301 十次。 |
| `Objectives` 第 2 項 | `TowerUpgradeCount`；`TargetId=301`；`TargetValue=20` | 升級塔種 301 二十次。 |
| `ObjectiveMode` | `All` | 建造與升級都達成。 |

升級是操作次數還是升級階數需固定；單次直接升多級應按實際階數或操作筆數擇一計算。

### 例一百一十七：塔防漏怪限制與完美防守

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2117` | 塔防完美防守挑戰 ID。 |
| `Category` | `Achievement` | 塔防成就。 |
| `Objectives` 第 1 項 | `DefenseStageClear`；`TargetId=102`；`TargetValue=1` | 通關塔防關卡 102。 |
| `Objectives` 第 2 項 | `DefenseLeakCount`；`TargetId=102`；`TargetValue=0` | 漏怪數為 0。 |
| `ObjectiveMode` | `All` | 通關且零漏怪。 |

目前 `TargetValue` 驗證規則要求大於 0，故零漏怪這種等於零的條件**不能直接用現有目標表示**；需改為專用布林指標，或擴充比較運算子與目標值驗證規則。

### 例一百一十八：麻將／牌局番數與自摸

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2118` | 牌局玩法任務 ID。 |
| `Category` | `Achievement` | 牌局成就。 |
| `Objectives` 第 1 項 | `TileGameWinCount`；`TargetId=0`；`TargetValue=20` | 牌局獲勝 20 次。 |
| `Objectives` 第 2 項 | `TileGameSelfDrawWinCount`；`TargetId=0`；`TargetValue=3` | 自摸獲勝 3 次。 |
| `ObjectiveMode` | `All` | 總勝場與自摸勝場均達標。 |

牌局變體規則可能不同；番型與勝利判定應按伺服器使用的玩法規則版本解析。

### 例一百一十九：解謎嘗試次數與無提示通關

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2119` | 解謎挑戰任務 ID。 |
| `Category` | `Achievement` | 解謎成就。 |
| `Objectives` 第 1 項 | `PuzzleClear`；`TargetId=8101`；`TargetValue=1` | 解開謎題 8101。 |
| `FailureObjectives` 第 1 項 | `PuzzleHintUseCount`；`TargetId=8101`；`TargetValue=1` | 使用提示即挑戰失敗。 |
| `TimeLimitSeconds` | `600` | 解謎限時 10 分鐘。 |
| `ExpireMode` | `FailPermanent` | 失敗後不重試。 |

若任務不是單次挑戰而是長期成就，提示使用應改為一般目標，不應使用失敗狀態或時限欄位。

### 例一百二十：釣魚稀有度與尺寸紀錄

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2120` | 釣魚收藏成就 ID。 |
| `Category` | `Achievement` | 生活技能成就。 |
| `Objectives` 第 1 項 | `FishRarityCatchCount`；`TargetId=5`；`TargetValue=10` | 釣起稀有度 5 的魚十條。 |
| `Objectives` 第 2 項 | `FishRecordSize`；`TargetId=9001`；`TargetValue=100` | 魚種 9001 的最大尺寸紀錄達門檻 100。 |
| `ObjectiveMode` | `All` | 稀有魚數量與尺寸紀錄都達標。 |

最大尺寸為越大越好的最佳紀錄，應保存歷史最大值；尺寸單位與精度必須固定。

### 例一百二十一：挖礦深度與稀有礦物

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2121` | 挖礦探索任務 ID。 |
| `Category` | `Side` | 探索支線任務。 |
| `Objectives` 第 1 項 | `MineDepthReach`；`TargetId=1`；`TargetValue=100` | 礦區 1 到達深度 100。 |
| `Objectives` 第 2 項 | `RareOreGatherCount`；`TargetId=7005`；`TargetValue=5` | 採集稀有礦物 7005 五個。 |
| `ObjectiveMode` | `All` | 深度與稀有礦物都要達成。 |

最深紀錄與當前深度語義不同；若採最佳深度需保存歷史最高值，採當前深度則回到淺層後可能不再符合。

### 例一百二十二：餐廳菜單研究與服務顧客

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2122` | 餐廳經營任務 ID。 |
| `Category` | `Routine` | 每日餐廳任務。 |
| `Objectives` 第 1 項 | `RecipeResearchCompleteCount`；`TargetId=0`；`TargetValue=3` | 完成 3 道食譜研究。 |
| `Objectives` 第 2 項 | `CustomerServeCount`；`TargetId=0`；`TargetValue=50` | 服務 50 位顧客。 |
| `ObjectiveMode` | `All` | 研究與顧客服務都完成。 |
| `ResetPolicy` | `Daily` | 每日重置。 |

顧客到店、下單、完成服務與付款是不同事件；任務需選擇實際代表玩法完成的事件。

### 例一百二十三：房屋裝潢與家具評分

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2123` | 家園裝潢成就 ID。 |
| `Category` | `Achievement` | 家園成就。 |
| `Objectives` 第 1 項 | `FurniturePlaceCount`；`TargetId=0`；`TargetValue=30` | 放置 30 件家具。 |
| `Objectives` 第 2 項 | `HomeDecorationScore`；`TargetId=1`；`TargetValue=5000` | 房間 1 裝潢評分達 5,000。 |
| `ObjectiveMode` | `All` | 放置數與評分都達標。 |

放置次數是歷史事件，評分是目前狀態；移除家具後評分可下降，但已完成的放置次數不應倒扣。

### 例一百二十四：攝影收集與指定景點

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2124` | 攝影探索任務 ID。 |
| `Category` | `Activity` | 攝影季活動任務。 |
| `Objectives` 第 1 項 | `PhotoCaptureCount`；`TargetId=0`；`TargetValue=10` | 拍攝 10 張有效照片。 |
| `Objectives` 第 2 項 | `PhotoSpotCaptureCount`；`TargetId=1201`；`TargetValue=3` | 在景點 1201 拍攝 3 次。 |
| `ObjectiveMode` | `All` | 照片總數與指定景點都達標。 |

若拍照內容由客戶端保存，伺服器可驗證的通常是拍攝動作與景點識別，而非照片品質；需定義重拍是否算新次數。

### 例一百二十五：寵物照護與狀態維持

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2125` | 寵物照護任務 ID。 |
| `Category` | `Routine` | 每日寵物任務。 |
| `Objectives` 第 1 項 | `PetFeedCount`；`TargetId=0`；`TargetValue=3` | 餵食寵物三次。 |
| `Objectives` 第 2 項 | `PetHappiness`；`TargetId=0`；`TargetValue=80` | 寵物目前快樂度達 80。 |
| `ObjectiveMode` | `All` | 餵食次數及當前快樂度均達標。 |
| `ResetPolicy` | `Daily` | 每日重置事件次數。 |

餵食是事件累積，快樂度是可變狀態；每日重置事件次數不應重置寵物自身狀態。

### 例一百二十六：飛行、滑翔與移動距離

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2126` | 移動探索任務 ID。 |
| `Category` | `Achievement` | 探索成就。 |
| `Objectives` 第 1 項 | `GlideDistance`；`TargetId=0`；`TargetValue=50000` | 累積滑翔 50,000 公尺。 |
| `Objectives` 第 2 項 | `FlightTimeSeconds`；`TargetId=0`；`TargetValue=3600` | 累積飛行 3,600 秒。 |
| `ObjectiveMode` | `All` | 距離與時間均達標。 |

移動距離和時間應由伺服器驗證位置、速度與移動狀態；傳送、離線及異常速度不可計入。

### 例一百二十七：傳送網路使用與快速移動

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2127` | 快速移動任務 ID。 |
| `Category` | `Routine` | 每週探索任務。 |
| `Objectives` 第 1 項 | `FastTravelCount`；`TargetId=0`；`TargetValue=20` | 使用快速移動 20 次。 |
| `Objectives` 第 2 項 | `DistinctWaypointUseCount`；`TargetId=0`；`TargetValue=10` | 使用 10 個不同傳送點。 |
| `ObjectiveMode` | `All` | 使用次數與不同地點數皆達標。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

不同傳送點數需保存週期內集合或等效去重資料；只靠累積計數無法判斷是否為不同地點。

### 例一百二十八：世界首領貢獻排名與參與門檻

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2128` | 世界首領活動任務 ID。 |
| `Category` | `Activity` | 世界首領活動。 |
| `Objectives` 第 1 項 | `WorldBossParticipateCount`；`TargetId=9903`；`TargetValue=3` | 參與首領 9903 三次。 |
| `Objectives` 第 2 項 | `WorldBossContributionRank`；`TargetId=9903`；`TargetValue=10` | 單場貢獻排名前十。 |
| `ObjectiveMode` | `All` | 參與次數與排名條件都要達成。 |
| `RepeatLimit` | `1` | 每期完成一次。 |

若排名是單場排名，應保存符合條件的最高排名結果；若是活動總排名，則須在活動結算後才判定。

### 例一百二十九：戰鬥資源取得與技能施放

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2129` | 戰鬥資源任務 ID。 |
| `Category` | `Routine` | 每週戰鬥任務。 |
| `Objectives` 第 1 項 | `BattleEnergyEarned`；`TargetId=0`；`TargetValue=1000` | 戰鬥中累積取得能量 1,000。 |
| `Objectives` 第 2 項 | `BattleSkillUseCount`；`TargetId=4202`；`TargetValue=30` | 施放技能 4202 三十次。 |
| `ObjectiveMode` | `All` | 能量取得與技能施放都達標。 |
| `ResetPolicy` | `Weekly` | 每週重置。 |

能量取得與技能施放須以戰鬥有效事件判定；自動戰鬥重播或結算重送不可重複計數。

### 例一百三十：低血量通關與剩餘生命值

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2130` | 戰鬥風險挑戰 ID。 |
| `Category` | `Achievement` | 戰鬥挑戰成就。 |
| `Objectives` 第 1 項 | `StageClear`；`TargetId=3012`；`TargetValue=1` | 通關關卡 3012。 |
| `Objectives` 第 2 項 | `BattleClearHpPercent`；`TargetId=3012`；`TargetValue=20` | 通關時剩餘生命不高於 20%。 |
| `ObjectiveMode` | `All` | 通關且生命條件符合。 |

此例的生命條件是越低越符合，和通常「達到以上」方向相反；需使用專用指標或後續擴充比較運算子，不能直接沿用通用進度算法。

### 例一百三十一：無限塔層數與週目最高紀錄

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2131` | 無限塔成就 ID。 |
| `Category` | `Achievement` | 長期挑戰成就。 |
| `Objectives` 第 1 項 | `TowerFloorReach`；`TargetId=1`；`TargetValue=100` | 無限塔 1 到達 100 層。 |
| `Objectives` 第 2 項 | `TowerSeasonBestFloor`；`TargetId=31`；`TargetValue=80` | 賽季 31 最高到達 80 層。 |
| `ObjectiveMode` | `All` | 總紀錄與指定賽季紀錄都達標。 |

樓層紀錄是最高值；重置塔進度時需保留歷史最高紀錄或使用獨立賽季鍵值。

### 例一百三十二：分歧路線與多結局收集

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2132` | 多結局收集任務 ID。 |
| `Category` | `Achievement` | 劇情收藏成就。 |
| `Objectives` 第 1 項 | `StoryRouteCompleteCount`；`TargetId=50`；`TargetValue=3` | 完成路線 50 三種分支。 |
| `Objectives` 第 2 項 | `StoryEndingUnlockCount`；`TargetId=50`；`TargetValue=5` | 解鎖路線 50 的五個結局。 |
| `ObjectiveMode` | `All` | 分支與結局收集都達標。 |

需定義分支重玩與結局旗標是否可重複取得；若同一結局只能解鎖一次，使用唯一旗標數而非事件次數。

### 例一百三十三：活動貨幣兌換指定獎品

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2133` | 活動商店兌換任務 ID。 |
| `Category` | `Activity` | 節慶活動任務。 |
| `Objectives` 第 1 項 | `EventCurrencySpend`；`TargetId=42`；`TargetValue=2000` | 消耗活動 42 代幣 2,000。 |
| `Objectives` 第 2 項 | `EventShopItemExchangeCount`；`TargetId=8008`；`TargetValue=1` | 兌換指定商品 8008 一次。 |
| `ObjectiveMode` | `All` | 消耗量與指定商品都完成。 |
| `ClaimWindowSeconds` | `86400` | 完成後 24 小時內領獎。 |

退貨或交易回滾應同步影響有效消耗量；若指定商品兌換本身已消耗同一貨幣，需避免同一交易被重複計入總消耗兩次。

### 例一百三十四：限時登入時間段獎勵

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2134` | 時段登入任務 ID。 |
| `Category` | `Activity` | 限時登入活動。 |
| `Objectives` 第 1 項 | `LoginInTimeWindowCount`；`TargetId=900`；`TargetValue=3` | 在時間窗 900 內登入三個不同活動日。 |
| `Objectives` 第 2 項 | `TimeWindowRewardClaimCount`；`TargetId=900`；`TargetValue=3` | 領取時間窗 900 的每日獎勵三次。 |
| `ObjectiveMode` | `All` | 登入日數與獎勵領取都達標。 |
| `RepeatLimit` | `1` | 活動期次一次。 |

時間窗的時區、跨日邊界與不同裝置登入去重需由全域時間規則處理。

### 例一百三十五：客服回報與問題排除任務

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2135` | 新手問題排除任務 ID。 |
| `Category` | `Side` | 功能引導支線。 |
| `Objectives` 第 1 項 | `HelpArticleViewComplete`；`TargetId=33`；`TargetValue=1` | 完成閱讀說明頁 33。 |
| `Objectives` 第 2 項 | `PracticeScenarioComplete`；`TargetId=4`；`TargetValue=1` | 完成教學演練 4。 |
| `ObjectiveMode` | `All` | 說明閱讀與實作演練都完成。 |

閱讀頁面事件僅代表開啟或完成閱讀流程，不應推斷玩家已理解內容；演練結果較適合作為能力確認。

### 例一百三十六：賽季資料重置前的最終領獎

| 欄位 | 設定值 | 說明 |
| --- | --- | --- |
| `Id` | `2136` | 賽季結算領獎任務 ID。 |
| `Category` | `Activity` | 賽季收尾任務。 |
| `Objectives` 第 1 項 | `SeasonSettlementReady`；`TargetId=34`；`TargetValue=1` | 賽季 34 已完成結算。 |
| `Objectives` 第 2 項 | `SeasonRewardUnclaimedCount`；`TargetId=34`；`TargetValue=1` | 存在至少一項未領賽季獎勵。 |
| `ObjectiveMode` | `All` | 結算完成且有待領獎勵。 |
| `ClaimWindowSeconds` | `604800` | 結算後 7 天內領取。 |

此類任務應在賽季結算服務完成後建立或解鎖，並確保任務獎勵與原賽季獎勵不會重複發放。

## 設定驗證規則

- `Id` 必須唯一且大於 `0`；`GroupId`、條件 ID、前置任務及獎勵引用必須有效。
- 前置任務不得重複或形成循環。`PrerequisiteMode` 對空清單不生效。
- `Sequential` 必須搭配 `ObjectiveMode=All`。
- `Objectives` 至少一項；所有 `TargetValue` 必須大於 `0`。
- `Carry` 僅適用於單一累加型成功目標的重複任務。`CurrentState` 指標不得用 `Carry`，且必須限制為一次性完成。
- 第一版若支援 `Carry` 一次跨越多輪，要求 `RepeatCooldownSeconds=0`，並為每個完成輪次建立獨立待領紀錄；若要同時支援冷卻，須先定義冷卻期間超額事件的處理規則。
- `RewardItems` 與 `RewardChoices` 必須恰有一種非空；擇一獎勵只能手動領取，`OptionId` 在同一任務內唯一。
- `RetryCooldownSeconds` 只可搭配冷卻重試模式且必須大於 `0`。`RepeatCooldownSeconds` 只對可重複任務有作用。
- `Category=Activity` 的任務必須由 `Activity.MissionIds` 引用；活動任務的玩家進度需包含活動期次。每日或每週重置還需併入對應週期。
- `ClaimWindowSeconds` 與活動領獎截止時間同時存在時，採用較早的截止時間。
- `ClaimMode=Auto` 不支援 `RewardChoices` 或需要玩家互動的領獎條件。
- `RetryNextReset` 必須搭配 `Daily` 或 `Weekly`；`RetryAfterCooldown` 必須設定正數 `RetryCooldownSeconds`；其他模式的 `RetryCooldownSeconds` 填 `0`。

## 玩家進度儲存需求

玩家進度至少需保存玩家 ID、任務 ID、任務週期、任務狀態、啟動與截止時間、完成及領獎次數、冷卻時間、已選獎勵，以及成功／失敗目標的各位置進度。活動任務須包含活動期次；每日與每週任務須包含週期識別，避免不同期進度互相覆蓋。

領獎、記錄已選 `OptionId` 與發放道具必須在同一筆資料庫交易內完成，並以冪等方式處理重送請求。
