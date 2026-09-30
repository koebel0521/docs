# Go Benchmark 逐次新增提示詞

複製以下提示詞，設定 `N` 後交給代理執行：

```text
請在目前 Go 專案與目前分支，逐次新增最多 N 個有實際價值、且目前尚未覆蓋的效能 benchmark。

每次依序完成以下流程，完成一輪才開始下一輪：
1. 閱讀專案指示，限定檢查與本輪候選情境及其 CI 範圍相關的程式碼、既有 benchmark 和 pipeline 設定；確認相同工作負載尚未被量測。
2. 選擇能代表實際生產工作量的不同情境，新增最小必要的 benchmark。不得只改名稱、資料大小或參數，重複既有情境。
3. 遵守專案格式規範，執行相關 benchmark，使用適當的 `go test -bench`、`-benchmem` 等選項，確認成功並記錄有效結果。
4. 根據數據使用 CPU 或記憶體 profile 找出主要成本，實作最小且有證據支持的效能調整。不要只憑猜測微優化。
5. 以相同機器、Go 版本、benchmark 與參數重跑，重複比較修改前後的 `ns/op`、`B/op`、`allocs/op`。確認結果正確且改善可重複，才保留調整；若無改善、結果不穩定或找不到明確瓶頸，撤回本輪效能調整，但保留有價值的 benchmark，並如實回報。
6. 檢查本輪 diff 與 `git diff --check`，確認只含本輪必要的 benchmark 與經驗證的效能調整；不可新增一般單元測試。Benchmark 如需驗證輸出，只加入必要檢查。
7. 以中文 Conventional Commit 建立一個 commit，且每輪恰好一個 commit；同一 commit 包含本輪 benchmark 與保留的效能調整。
8. 推送該 commit 至目前分支的對應遠端分支。
9. 只查詢該 commit 對應的 pipeline，確認新 pipeline 已建立並記錄編號與當下狀態；不必重複盤點其他 pipelines。

執行限制：
- 不要重複盤點整個專案或 CI。只檢查足以判斷 benchmark 是否重複，以及 pipeline 是否會執行該 benchmark 的相關範圍。
- 若 pipeline 不會執行新增 benchmark，明確記錄此事，不要擴大修改範圍去調整 CI。
- 若找不到有價值且尚未覆蓋的情境，立即停止；不要為了達到 N 次製造變更。回報已完成次數與停止原因。
- 只有同一情境、相同環境與參數下取得可重複的前後比較，才能宣稱效能改善；分別回報 `ns/op`、`B/op`、`allocs/op` 的變化。
- 保留工作區既有變更，只 stage 本輪新增或修改的必要檔案；不得把其他人的變更帶進 commit。
- 一旦某輪的 benchmark、diff 檢查、commit、push 或 pipeline 建立確認失敗，先處理該輪失敗，不可略過後繼續建立下一輪 commit。

全部完成或停止後，逐次回報每個 benchmark 的情境、執行指令、調整前後數據與改善結論、commit、push 結果、pipeline 編號與狀態；若只新增 benchmark 而未保留效能調整，說明原因。若提前停止，說明已完成幾次及原因。不要聲稱 pipeline 已通過，除非已確認其最終狀態。
N = 200
```
