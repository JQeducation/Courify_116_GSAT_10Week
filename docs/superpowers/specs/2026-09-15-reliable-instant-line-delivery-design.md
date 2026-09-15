# 可靠即時 LINE 預約通知設計

## 目標

使用者在 Webinar Kit 報名後，從共用 GitHub Pages 點擊「加入 LINE 領取直播連結」，進入 LINE 並完成身分確認或加好友後，於數秒內收到對應講座的預約成功訊息。

國文與英文十周班共用同一個 LINE Login Channel、LIFF App、LINE 官方帳號及 GitHub Pages；兩個活動仍使用各自的 Step 2 Webhook、Google 試算表與講座資料。

## 固定限制

- LINE Messaging API 的 Webhook URL 維持指向 Super 8，不更換、不轉送。
- 不新增 LINE `follow` Webhook 或自建中繼服務。
- Meta Pixel 的程式、載入順序、`PageView` 與 `CompleteRegistration` 事件維持原狀。
- Step 1、試算表欄位與既有 Email 配對規則維持不變。
- 不新增 LINE Channel、LIFF App 或 Google 試算表欄位。

## 流程設計

### 1．從外部瀏覽器進入 LINE

共用頁面的主按鈕不再執行外部瀏覽器中的 `liff.login()`。按鈕會將目前的 `campaign`、`t`、`r`、`first_name`、`email` 與 `phone_number` 編碼後，開啟共用 LIFF URL。

使用者進入 LINE 後留在 LIFF 頁面內完成後續流程，避免「LINE → 外部瀏覽器 → LINE」的往返。

### 2．好友狀態分流

LIFF 初始化並取得使用者資料後，以 `liff.getFriendship()` 判斷好友狀態。

- 舊好友：直接呼叫對應活動的 Step 2 Webhook。
- 新好友或已封鎖：在 LIFF 內呼叫 `liff.requestFriendship()`，顯示加好友或解除封鎖視窗；視窗結束後再次呼叫 `liff.getFriendship()`。
- 第二次確認為好友：立即呼叫 Step 2 Webhook。
- 第二次確認仍非好友：留在 LIFF 頁面，顯示「尚未完成加入好友」及重試按鈕，不觸發 Step 2。

### 3．Step 2 發送

Step 2 維持現有流程：依 Email 尋找「等待加LINE」的報名紀錄，寫入 LINE 暱稱與 LINE User ID，將配對狀態改為「配對成功」，並發送預約成功訊息。

移除目前 1 分鐘的 Sleep。共用頁必須等待 Step 2 Webhook 回應成功，才顯示預約完成狀態。Webhook 失敗時顯示可重試狀態，不寫入本機成功標記。

## 防止重複發送

- 瀏覽器端以 `campaign + r` 作為已成功送出的識別鍵。
- Make 端只處理配對狀態為「等待加LINE」的資料列；第一次成功後改為「配對成功」。
- 使用者重新整理或重複點擊時，不應再次發送相同活動、相同報名紀錄的通知。

## 錯誤處理

- 缺少或不支援 `campaign`：不呼叫任何 Make Webhook。
- 缺少 Email、LINE User ID 或無法取得好友狀態：顯示明確錯誤與重試按鈕。
- 使用者取消加好友：留在提示頁，不顯示預約訊息已送出。
- Step 2 回應失敗：允許重試，保留試算表原始報名資料。

## 驗收標準

### 舊好友

1. 點擊主按鈕後進入 LINE LIFF，不再跳回外部瀏覽器。
2. 於 10 秒內觸發正確活動的 Step 2。
3. 試算表更新 LINE 資料與配對狀態。
4. LINE 收到一次內容正確的預約成功訊息。

### 新好友

1. 點擊主按鈕後進入 LINE LIFF，顯示加好友確認視窗。
2. 完成加好友後於 10 秒內觸發正確活動的 Step 2。
3. 試算表更新 LINE 資料與配對狀態。
4. LINE 收到一次內容正確的預約成功訊息。

### 既有服務

1. Super 8 訊息中心、聊天機器人與好友事件維持正常。
2. Meta Pixel 的既有事件維持正常。
3. 國文與英文不會寫入錯誤試算表或使用錯誤講座連結。

## 測試順序

1. 使用已加入官方帳號的 LINE 測試帳號驗證舊好友流程。
2. 使用從未加入或已封鎖後解除的測試帳號驗證新好友流程。
3. 各自測試國文與英文活動代碼。
4. 檢查 Make History、Google 試算表、LINE 收件內容與 Super 8 訊息中心。
5. 重複開啟同一筆報名連結，確認不會重複發送。
