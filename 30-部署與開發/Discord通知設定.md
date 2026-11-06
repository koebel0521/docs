# Discord 通知設定（CI/CD 建置完成）

建置完成後，以 Discord Webhook 推送通知到頻道。本文件說明設定步驟與 `.gitlab-ci.yml` 範例。

## 方式一：Discord Webhook 搭配 GitLab CI（建議）

### 1. 建立 Discord Webhook

Discord 頻道設定 → 整合 → Webhook → 新增 Webhook，複製 URL。

### 2. 在 GitLab 新增變數

Settings → CI/CD → Variables → 新增：

| 變數 | 值 | 選項 |
|------|----|------|
| `DISCORD_WEBHOOK_URL`（已刪，目前程式與 CI 都不再使用） | Webhook URL | Masked、Protected（視需要） |

### 3. 加入 CI job

在 `.gitlab-ci.yml` 末端加入。`.post` 階段一定最後執行，不需改既有 stage。

```yaml
notify-success:
  stage: .post
  rules:
    - when: on_success
  script:
    - 'curl -sS -H "Content-Type: application/json" -X POST -d "{\"content\": \"✅ 建置成功 $CI_PROJECT_NAME $CI_COMMIT_REF_NAME $CI_COMMIT_SHORT_SHA\\n$CI_PIPELINE_URL\"}" "$DISCORD_WEBHOOK_URL"'

notify-failure:
  stage: .post
  rules:
    - when: on_failure
  script:
    - 'curl -sS -H "Content-Type: application/json" -X POST -d "{\"content\": \"❌ 建置失敗 $CI_PROJECT_NAME $CI_COMMIT_REF_NAME $CI_COMMIT_SHORT_SHA\\n$CI_PIPELINE_URL\"}" "$DISCORD_WEBHOOK_URL"'
```

說明：

- `on_success` / `on_failure` 分兩個 job，因 GitLab 沒有內建的 pipeline 狀態變數可在 `.post` 內判斷成功或失敗。
- `$CI_PROJECT_NAME`、`$CI_COMMIT_REF_NAME`、`$CI_COMMIT_SHORT_SHA`、`$CI_PIPELINE_URL` 為 GitLab 預設變數。
- 若要通知建置版本號，改用 `$CI_COMMIT_TAG`（僅 tag 觸發時有值）。

## 方式二：GitLab 內建 Discord 整合

Settings → Integrations → Discord Notifications，貼上 Webhook URL，勾選 Pipeline events。

- 不需改 `.gitlab-ci.yml`。
- 訊息格式固定，無法加入自訂版本號或下載連結。

## 建議

需要版本號或自訂內容時使用方式一；只要求簡單通知時使用方式二。
