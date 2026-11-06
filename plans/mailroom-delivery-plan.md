# Mailroom 投遞模組規劃

版本：草案 4  
日期：2026-11-06  
狀態：規格草案，尚未實作。已依使用者選定的方向與兩份 Phase 0 報告收斂；Claude 只讀複審與使用者審查均為 ACCEPTED（規格層級），可開始 Phase 1a。  
基準：BAT `fbbd02d`、agent-mailroom `d1c2a51`。

**2026-11-06 本次討論結果**：採第 3 項，Phase 1 先做 (a′)，成員憑證輪替另開一張票。新模式保留以 `session_key` rebind，停用無 key 的 reconnect；存活快照移出 Phase 1。鎖與 epoch 只限制同一個 `host_id` 的投遞者，不作為舊 runtime 已結束的證明。

探查依據：[Phase 0 報告](../../../deliverables/mailroom-phase0-20261106/report.md)、[Phase 0 補充報告](../../../deliverables/mailroom-phase0-supplement-20261106/report.md)。兩份報告均已由使用者接受；實測範圍為 macOS 15.7.9 x86_64、GUI-free headless BAT、Claude SDK 與 Rust Codex app-server。desktop、Linux 與真實對話 resume 的未驗證範圍，不能因原始碼或控制要求測試通過而視為成立。

## 目標

讓 BAT 內建一個 Mailroom 投遞模組，直接把 Mailroom 的新信通知送進 BAT 自己管理的 Claude 與 Codex agent session。

- **Mailroom 不依賴 BAT**：保存、`request_id` 去重、ack、`notification_key` 都留在 Mailroom server。沒有 BAT 的 CLI agent 仍可主動收送信。
- **BAT 原生認得 Mailroom**：投遞改在 BAT 內部完成，不再需要外部透過 Remote Access 的 token、憑證指紋與 terminal ID 定址。
- **多代理溝通是核心**：Claude 與 Codex 都要支援。

非目標：
- 不改 Mailroom 的訊息語意：保存即成功、ack 才算處理、同 ID 不同內容回 409。
- Phase 1 不輪替成員 `session_key`，也不承諾 rebind 會阻止持有同一成員 key 的舊 runtime 繼續操作。
- 新模式不提供無 key 的 reconnect，合法的此類請求一律回 409 `liveness_unknown`。不實作存活快照、workspace `complete` 或 host 的關閉完成事件。
- Phase 1 不支援 `claude-channel`，原因有三：仍受 `BAT_DEBUG` 限制、沒有排隊機制、`--strict-mcp-config` 會擋掉使用者的 MCP。
- Phase 1 也不支援純 PTY 的 `claude-cli`／`codex-cli` preset。
- 不支援 remote profile，只投遞給 host 本機的 session。
- Phase 1 只支援 macOS 與 Linux。Windows 的憑證 ACL 另行定義；在那之前，Windows 上的模組狀態固定為 `unsupported_platform`。

## 上游規範對照

依 `CLAUDE.md`、`AGENTS.md`：

| 規範 | 本設計的做法 |
| --- | --- |
| IPC 只能新增 | 只新增 mailroom 狀態與診斷用的 command 與 event。既有的 `host.*`、`window.batAppAPI.*`、`claude:*` 事件都不改，既有 UI 流程也不依賴新 IPC。 |
| 共用程式碼要追蹤所有呼叫端 | `requireIdle` 是新增的選用參數，預設值為 false，既有呼叫端的行為不變。修改前要列出 sidecar `claude.sendMessage` 與 Codex `send_message` 的全部呼叫端，並補上回歸測試。 |
| 每個 session 的執行權只歸一方 | 模組不擁有任何 session。Phase 2 為目前私有的 `ClaudeRuntimeRouter` 新增明確的 crate 內部建構／送出介面，由 `HostContext` 取得既有 runtime states；Claude SDK 仍交給 sidecar，Codex 仍交給 Rust。本機綁定核對不可省略。 |
| Remote 狀態由 host 擁有 | 模組是 host 端的背景工作，透過 `HostContext` 在桌面版與 headless `bat-server` 上都能運作，不依賴 renderer。 |
| 不退步 | 預設關閉，由 `settings.json` 開關啟用。未啟用時不啟動投遞或連線；一旦成功取得 host 鎖，即使之後停用模組也持有到 BAT 程序結束。 |
| 命名 | 一律使用 `mailroom`，避開既有的 `commands/notification.rs`（那是 BAT 自己的提醒清單）。 |
| Logging | 使用專案的 Rust logger 與 sidecar logger。 |
| 驗證 | `pnpm exec tsc --noEmit --pretty false`、`pnpm run compile`、`pnpm run test:sidecar`、`pnpm run check:tauri-rust`、`pnpm run test:tauri-rust`。新測試加入 `verify:tauri-pre-ci`。 |

## 現況（已查證）

### Mailroom 目前怎麼通知

- **連線**：`agent_mailroom/bat.py` 以 `wss://127.0.0.1:9876` 連 BAT Remote Access。流程依序是比對憑證指紋 → `auth`（`bat-remote/v2`、`profileContext==1`）→ `profile:open` → `workspace:load` 取得 terminal 清單。
- **輪詢**：每個綁定一條 worker thread，每 2 秒輪詢一次；延後時間隔倍增，最長 30 秒（`notifications.py:258-277`）。
- **投遞**：透過 `claude:send-message {sessionId, prompt, clientMessageId}`，Codex 也走同一個 channel（`notifications.py:338-346`）。
- **通知內容**：只含 `room_id`、`message_id`、`notification_key`，不含正文、session key 或 BAT token（`notifications.py:193-217`）。
- **結果判讀**：Claude 要 `ok` 與 `accepted` 都是 true；Codex 只要 `ok` 是 true（`notifications.py:355-372`）。

### Mailroom 現有的保證

1. **先記錄意圖再寫出**：先在 `notifications` 表寫入 `submitting`，才送出 frame。若在這之間當機，啟動時會把 `submitting` 轉為 `unknown`，**永不自動重送**（`store.py:725-767`）。
2. **每位成員同時最多一則通知**：下一則要等收件人 ack（`store.py:704-723`）。
3. **兩層憑證的撤銷範圍不同**：
   - `notification_key`：每次送出都產生新的隨機通知憑證，只存 SHA-256；只對單一綁定、單一訊息有效，只能回覆一次。現行 leave、實際改目標的 rebind、成功 reconnect，會使舊綁定的通知憑證永久失效；草案 4 對所有綁定世代遞增（含 rejected 同目標重試、無 binding 時重新建立）另訂一致的失效規則，見第 2、3、6 節，這部分尚未實作。
   - **pause 不撤銷通知憑證**：成員 paused 時，該憑證只允許查狀態，讀信、回信、ack 回 403；恢復 active 後，原通知憑證可以繼續使用。paused 成員不會被排入新通知（`store.py:551-566`、`:653-669`、`:709`）。
   - `session_key`：代表成員身分。`join_room` 沿用原 key；rebind 只刪除舊 binding 並建立新 binding，不輪替成員憑證（`bridge.py:237`、`store.py:598-612`）。持有同一成員 key 的舊 runtime 仍可讀信、回信、ack，這和現行行為相同。
   - **leave 也不撤銷成員 key**：只設為 left 並解除 binding，原 key 仍可恢復 active；要恢復通知綁定，再以同 key rebind。現行成功的 reconnect 才會替換成員身分，使舊憑證在該 room 失效（`store.py:375-395`）。新模式停用這條無 key 的恢復路徑。

4. **送出前重新檢查目標**：runtime、profile、cwd 在註冊時與每次送出前都要核對。
5. **暫停／離開與送出互斥**：`delivery_lock` 包住 socket 寫出（`store.py:726`），因此 pause 會等到送出的邊界。已送出的通知無法撤回。
6. **不喚醒 resting session**：Codex 有待回答的問題或權限詢問時也會延後。
7. **現行舊模式的 reconnect 要證明舊 terminal 已不在**：成功後，未 ack 的信重新排給新 terminal，舊通知憑證及該 room 的舊成員身分失效。這只描述 `d1c2a51` 的舊模式，不代表存活證明可靠；新模式一律回 `liveness_unknown`，不執行 takeover。
8. **註冊是原子的**：驗證失敗時，不會留下部分建立的房間或成員。

### BAT 端的事實

- **terminal ID 就是 agent session ID**（`workspace-store.ts:548`、`ClaudeAgentPanel.tsx:2552`）。
  - host 保存／載入會保留 terminal ID 與 sdkSessionId；補充探查已實測 terminal ID 與非空的 Codex thread ID 跨 BAT 重啟保持一致。Claude 的無回合初始化沒有非空 sdkSessionId，真實對話的 resume 仍未驗證。保存欄位恢復不等於 runtime 已成功 resume。
  - 關閉分頁後重開會得到新 ID。
  - `repairTerminalIdentityCollision` 可能重新指派 ID（`workspace-store.ts:~327`）。
- **預定投遞入口**是 `ClaudeRuntimeRouter`（`commands/claude.rs:2747`），目前 struct 與 `from_ctx` 為私有，後者標示 dead_code；mailroom 模組尚不能直接使用。Phase 2 必須提供內部介面，保留本機綁定核對，且不以直接略過 `ensure_local_agent_session_access` 取代授權檢查。
  - **Claude SDK**：由 sidecar `claude.sendMessage` 處理（`claude-send.mjs:1373`）。session 忙碌時，訊息會排進 `s.sendQueue`；送出成功回傳 `{ok:true, accepted:true, queued}`；以 `clientMessageId` 去重時，回傳 `{ok:true, duplicate:true}`。
  - **Codex**：由 Rust `send_message` 處理（`codex_app_server.rs:5627`），並以 `turn_operation_lock` 序列化（`:5657-5660`）。
    - 回合進行中收到訊息，會**取代**該回合（`:5739`）。
    - 回合還在啟動時，回傳 `{ok:false}`。
    - 送出成功只回傳 `{ok:true}`（`:6122`）。
- **多個 Codex session 共用同一個 app-server**。只有 reload 過的 session 會搬到獨立程序（`codex_app_server.rs:344`）。因此程序層級的環境變數不一定能對應到單一 session。
- **兩條送出路徑都會清除 resting**：Claude 在 sidecar 的 `if (s.isResting) s.isResting = false`（`claude-send.mjs:1239`），Codex 在 Rust 的 `session.is_resting = false`（`codex_app_server.rs:5847`）。因此直接送出會喚醒 resting session。
- **sidecar `claude.sendMessage` 的順序**：先做 session 存在與 reload 的預先檢查，再做 `clientMessageId` 去重記錄（`:1400-1413`），然後 user echo（`:1414-1416`），最後排隊。
- **`resume_after_rate_limit` 不是競態安全的**：它先查 `isResting`／`getSessionState`（`claude.rs:626-641`），再另外呼叫 `send_message`（`:648`），兩步之間沒有鎖。只能拿來參考設定開關與 router 的用法，不能當成閒置判斷的依據。
- **BAT 沒有把 terminal ID 交給 agent 程序**。唯一相關的環境變數是 channel 專用的 `BAT_CHANNEL_SESSION_ID`。Phase 0 已確認 Claude `queryOptions.env` 與 Codex thread 層級 MCP env 的傳遞能力；這不是 BAT 的全路徑注入已完成。
- **Codex context window 會覆寫整個 config**：`codex_app_server.rs:753` 目前執行 `params["config"] = json!({"model_context_window": window})`，只在選用 context window 時發生。新增 `mcp_servers` 時必須合併，不能讓任一設定覆蓋另一個，規則見第 7 節。
- **現有關閉動作不是程序樹結束證明**：UI 確認關分頁未 await `stopSession` 就移除 terminal；關閉 workspace 只移除狀態。Claude SDK close 先關 stdin、等 2 秒再送 SIGTERM，BAT 沒有等待程序退出。Codex 共用 app-server 的 stop 不送 `turn/interrupt`，只移除本機 session 紀錄；因此舊 thread 在有進行中回合時可能繼續用同一成員 key 操作。本次沒有模型回合測試，這是依終止機制指出的觸發情境。專用 Codex runtime 雖會 kill-and-wait 直接子程序，也不能據此宣稱所有後代已結束。見補充報告「關閉路徑與完成時點」。

## 設計

### 方向反轉：BAT 主動向 Mailroom 拉取

```
現在：Mailroom server ──wss(token, 指紋, terminal ID)──▶ BAT Remote Access ──▶ session
新版：BAT mailroom 模組 ──loopback HTTP（投遞憑證）──▶ Mailroom server
                    └──▶ ClaudeRuntimeRouter（requireIdle）──▶ session
```

BAT 成為 Mailroom 的投遞端。投遞紀錄（claim 與結果）仍由 Mailroom 保存，所以 BAT 重啟後不會遺失。

### 1. 原子的閒置送出：`requireIdle` 與 `notAfter`

閒置判斷和送出期限，都必須在 runtime **接受訊息的同一個臨界區**內完成。在外面先查狀態再送，中間仍可能有使用者開始新回合。

兩個 runtime 各新增兩個選用參數：
- `requireIdle`：session 不閒置就拒絕。
- `notAfter`：送出期限。BAT 在本機用 monotonic clock 換算成截止時間。

兩個參數預設都不帶，既有呼叫端的行為不變。

**判斷條件**：以下任一項成立，就拒絕送出，不排隊、不取代回合，也不清除 resting：
- 串流中，或有 active turn、`is_running`；
- 已有佇列（sidecar 的 `s.sendQueue`）；
- 有待處理的權限詢問或問題；
- **session 是 resting**（Claude 的 `s.isResting`、Codex 的 `is_resting`）；
- 已超過 `notAfter`。

**回傳**：
- 不閒置：`{ok:false, busy:true, reason}`，`reason` 例如 `streaming`、`queued`、`pending_permission`、`pending_ask_user`、`resting`。
- 超過期限：`{ok:false, expired:true}`。

**sidecar `claude.sendMessage` 內的順序**：
1. 既有的預先檢查（session 存在、reload 中）。
2. **只讀**查詢 `clientMessageId` 去重紀錄。命中就照舊回 `duplicate`。這一步必須在 busy 判斷之前，否則同一次送出被重新呼叫時，會因為「自己那一回合正在跑」而得到 busy，被誤判成沒送進去。
3. `requireIdle` 與 `notAfter` 的判斷。
4. 寫入去重紀錄、user echo、排隊。

第 2 到 4 步要在同一段同步程式中完成，中間不能有 `await`。Node 是單執行緒，所以不會有其他送出插進來。Phase 0 已確認相關狀態可同步讀取（A5）；判斷也要涵蓋 SDK 啟動時的 `runtimeStatus`。狀態可讀不代表未來的臨界區實作已通過競態驗證。

**Codex `send_message`**：取得 `turn_operation_lock` 之後才判斷，判斷通過後才寫入 `is_resting = false` 等狀態。使用者自己的送出也要取得同一把鎖，所以兩者會被序列化。

router 只負責傳遞參數。模組在 claim 前可以先查一次狀態，用來減少不必要的 claim；但那次查詢只是最佳化，不構成保證。

有了這兩個參數，規格才能寫成：**Mailroom 通知不會取代或插入進行中的回合，不會喚醒 resting session，也不會在送出期限之後才送進 session。**

### 2. 提交點、期限與撤回語意

新流程中，Mailroom 的 `delivery_lock` 無法跨程序涵蓋 BAT 的實際送出，因此明訂以下規則。

**claim 就是提交點**：claim 成功之後，leave、rebind、pause 都**無法阻止**這則提示進入 session；新模式的無 key reconnect 本來就不執行。這是原保證 5 的語意變更：提交點從「socket 寫出」提前到「claim」。

**兩個期限**，在 claim 時都會回傳：

| 期限 | 預設 | 意義 |
| --- | --- | --- |
| `send_deadline` | 30 秒 | BAT 最晚可以把提示送進 runtime 的時間，以 `notAfter` 帶進 runtime 的臨界區檢查。 |
| `report_deadline` | 10 分鐘 | Mailroom 等待結果回報的時間。超過時，Mailroom 把 claim 標成 `unknown`，原因為 `report_timeout`。 |

- **期限是給 BAT 的上限，server 不會因為期限到了就自動釋放**。lease 到期、BAT 離線或回報遺失，都不能證明提示沒送進去，因此只會走到 `unknown`。
- **期限內送出、期限後才回報的結果仍然要記錄**。原因為 `report_timeout` 的 `unknown`，之後如果收到該 claim 的回報，可以改成回報的結果一次。啟動恢復時由 `submitting` 轉成的 `unknown`，不接受任何回報改寫。

**`released` 只能根據明確的「未送入」證據**：
- runtime 回傳 `busy:true` 或 `expired:true`，因為判斷就在接受訊息的臨界區內；
- BAT 還沒呼叫 runtime，就已經確定不送，例如本機核對發現 terminal 不符。

只要不在上述情況內，包括例外、逾時、程序當機，一律是 `unknown`。

**`released` 的處理**：
- 保留這個 claim 作為終態，不刪除，並撤銷這次的 `notification_key`。
- 訊息回到候選；下一次 claim 會產生新的 `claim_id` 與新的 key。
- 重送同一個 `released` 回報是冪等的。這是為了處理「第一次 release 成功、但 HTTP 回覆遺失」的重試。

**撤銷與暫停**：
- leave，或任何使 binding_version 遞增的綁定變更，都會永久撤銷前一世代所有 claim 的 notification_key，包含 legacy 通知憑證；同目標的 rejected 重試也適用。agent 收到提示後，第一步以該通知憑證呼叫 `notification_status`；憑證失效時回 403，提示也指示它停止處理。新模式的無 key reconnect 不會建立新綁定或撤銷任何身分。
- **pause 維持現行語意**：`notification_key` 不撤銷，只是暫時限制使用。以該通知憑證在 paused 期間只能查狀態，其餘操作回 403；恢復 active 後可以繼續處理已通知但未 ack 的信。這不將成員 `session_key` 改成相同的訊息範圍授權。
- **「通知憑證撤銷後 agent 必須停止處理」是協定要求**。server 只能保證使用失效 `notification_key` 的讀信、回信、ack 被拒絕。rebind 不撤銷成員 `session_key`，不能阻止持有同一成員 key 的舊 runtime 繼續操作；新舊 runtime 可能並行處理。成員憑證輪替另票，Phase 1 不宣稱解決此限制。

### 3. claim 與 outcome 的定址

- **`POST /delivery/claims`**
  - 輸入：`binding_id`、`binding_version`、`message_id`、`host_epoch`（見第 5 節）。
  - 回傳：唯一的 `claim_id`、`send_deadline`、`report_deadline`、完整的通知提示。
  - 綁定版本不是目前版本、綁定不是 confirmed、成員協作狀態不是 active，或 `host_epoch` 已被取代時，回 409。這些條件都在寫入 claim 的同一交易內重新核對，不能沿用先前 candidates 的結果；例如取得 candidates 後成員才 pause，claim 必須被拒絕，不寫入 claim，也不產生通知憑證。
- **`POST /delivery/claims/{claim_id}/outcome`**
  - 輸入：`binding_id`、`binding_version`、`result`、`evidence`。`evidence` 是 runtime 的回傳摘要，例如 `busy:resting`。
  - **outcome 以 `claim_id` 定址，只會更新那一個 claim**。`binding_id` 與 `binding_version` 必須等於該 claim 建立時的記錄值，用來防止回報送錯 claim；它們**不必**等於目前的綁定。因此，綁定變更後，舊 terminal 那次送出的真實結果仍然可以記錄，也不會影響新的 claim。
  - **可接受的狀態轉換**：
    - `submitting` → `accepted`、`not_accepted`、`released`、`unknown`；
    - `unknown`（`report_timeout`）→ 上述任一結果，只能一次。
  - **冪等與拒絕**：
    - 終態下重送相同的結果，冪等成功。
    - 與終態矛盾的回報，回 409，記錄為診斷事件，不更新紀錄。
    - claim 不存在，回 404。
  - **`released` 的額外限制**：只有 `evidence` 是 `busy`、`expired` 或 `not_invoked` 時才接受。
  - **晚到的回報只更新它自己的 claim**：不會重新啟用已撤銷的 key，也不會改變目前綁定的投遞資格。唯一的例外是：claim 屬於目前的綁定版本，而且是由 `report_timeout` 的 `unknown` 改為 `released` 時，可以依阻擋規則放行。
- **`binding_version`**：代表該 room 成員的通知綁定世代，跨 binding_id 單調遞增；持久保存該成員歷來最高版本，不能因刪除 binding／leave 或遷移而重設。下列三種情況建立新世代：改目標的 rebind；rejected 綁定以原 key 重新要求確認；成員沒有現存 binding 時以原 key join_room 建立綁定（包含 leave 後恢復、orphan legacy）。新版本在同一交易中設為 `max(歷來最高版本, 該成員已保存的遷移世代基準) + 1`；沒有可用版本時以 0 為基準。新成員首個綁定同樣從基準加 1。
  - 新世代一律撤銷先前世代所有通知憑證，建立 pending，並持久保存最高版本。leave 後先以同一成員 key 恢復 active，再 join_room；不讓 join_room 隱式恢復成員狀態，維持現行授權規則。
  - confirm／reject 只改狀態、不遞增版本；回傳目前版本，claim 使用該版本。pending／confirmed 下同 metadata、同目標的 join_room 重試沿用現有 binding／版本／狀態。單純重啟或 host_epoch 改變不遞增。
- **Mailroom schema**：新增 `delivery_claims` 表，以 `claim_id` 為主鍵。欄位包含 `message_id`、`binding_id`、`binding_version`、`host_epoch`、`state`、`unknown_reason`、`send_deadline`、`report_deadline`、`notification_key_hash`、`evidence`。
  - 現有的 `notifications` 表以 `message_id` 為主鍵，無法區分多次投遞嘗試。
  - 「每位成員最多一則通知」改為以綁定為單位的**阻擋規則**：
    - 同一個綁定下，只投遞最舊的未 ack 訊息。
    - 這封訊息沒有任何 claim，或最新的 claim 是 `released`，才能被 claim。
    - 最新的 claim 是 `submitting`、`accepted`、`not_accepted` 或 `unknown` 時，這個綁定之後的通知全部暫停。`not_accepted` 與 `unknown` 都不會自動放行。
    - 解除阻擋只有三種方式：收件人 ack 這封信；agent 以原成員 key 依本節三種入口建立新綁定世代並完成兩段式確認，將未 ack 的信排給新目標；或者 `report_timeout` 的 `unknown` 收到晚到的 `released` 回報。
  - 升級沿用既有做法，先備份再以交易升級 schema。舊 `notifications` 的每筆紀錄必須保留為新規則可辨識的 **legacy 阻擋 claim**，不能因新的 delivery_claims 為空而將它當成從未通知。
    - accepted、not_accepted、unknown 保留原狀態；submitting 轉為 unknown。legacy unknown 不是 report_timeout，不接受新 outcome 改寫。
    - legacy 記錄標示來源，沒有新的 runtime 呼叫或可回報的投遞嘗試；host_epoch／兩個 deadline 不適用，不為遷移捏造期限或新的通知憑證。新模式的 outcome 端點拒絕修改 legacy 記錄。
    - 已通知未 ack 的信維持阻擋，直到 ack，或以原成員 key 建立新綁定世代並完成確認。首次回填時以該成員遷移當下的最高 binding_version 作為持久化的「遷移世代基準」，沒有可用版本時取 0；現行舊 schema 沒有版本的既有 binding 以 0 匯入，遷移本身不遞增版本。schema 重跑不得重算或提高已保存的基準。
    - legacy 阻擋保存當時的 binding_id／binding_version；舊 binding 無法對映時也保存該 room 的成員／收件人／訊息及上述基準，不當成沒有 claim。只有依本節三種新世代規則，以原 key 建立高於基準的版本、撤銷舊通知憑證並完成確認，才在新綁定放行。
    - 「初次 confirm 不放行」特指遷移時轉 pending、版本不變的既有 binding；沒有 binding 的成員依新世代規則建立綁定後，confirm 可以放行。leave 後須先恢復 active。單純遷移確認、重啟或 host_epoch 更新不能放行，host_epoch 不得替代 binding_version／遷移世代。
    - 保留既有通知憑證的 hash 與授權／撤銷結果，不新發 key，也不重新啟用已撤銷的通知憑證。成員憑證不變；未曾有通知紀錄的信才適用「沒有任何 claim」的分支。
    - schema 回填須可安全重跑，不因重跑新增第二次投遞嘗試或重設阻擋。

### 4. 依 runtime 判讀送出結果

| runtime | 回傳 | Mailroom 結果 |
| --- | --- | --- |
| Claude SDK | `ok:true` 且 `accepted:true` | `accepted` |
| Claude SDK | `duplicate:true`，且帶有原本那次的結果 | 依原本的結果判讀，例如帶有 `accepted:true` 就是 `accepted` |
| Claude SDK | `duplicate:true` 且 `pending:true` | 等原本那次送出的結果；無法確認時記為 `unknown` |
| Codex | `ok:true` | `accepted` |
| 兩者 | `ok:false` 且 `busy:true` | `released`（證據：`busy:<reason>`） |
| 兩者 | `ok:false` 且 `expired:true` | `released`（證據：`expired`） |
| 兩者 | `ok:false`，其他原因 | `not_accepted`，`last_error` 記錄原因 |
| 兩者 | 例外、逾時、非預期的回傳格式 | `unknown`，永不自動重送 |

- `accepted` 只代表 runtime 已經接受，**處理完成仍以收件人的 ack 為準**。
- 不在上表的回傳格式一律記為 `unknown`，並在 log 中保留原始回傳。
- **去重命中不一定代表已接受**：`duplicateClientMessageResult`（`claude-send.mjs:110-122`）有三種情況：
  - 原本那次還在 `pending`，回傳 `{ok:true, duplicate:true, pending:true}`。pending 紀錄是在實際送出之前建立的。
  - 原本那次已被拒絕，直接拋出錯誤。
  - 原本那次已完成，回傳 `{...原結果, duplicate:true}`。
- 依照第 9 節，每個 claim 只呼叫 runtime 一次，正常情況下不會命中去重。萬一命中，例如 router 內部重試：
  - 只有帶有原本那次的結果時，才依那個結果判讀。
  - `pending` 時可以等原本那次送出的結果；無法確認就保守記為 `unknown`。
  - 拋出錯誤一律記為 `unknown`。
  - 以上任何一種情況都**不能**判成 `released`，也**不能**重新呼叫 runtime。

### 5. 單一投遞者：固定位置的鎖與 host_epoch

A9 已不成立：在新 BAT 取得鎖的當下，舊 sidecar、Claude Code 或 Codex app-server 仍可能存活。補充探查也顯示關分頁、workspace 關閉與 collision repair 不能提供「runtime 及所有後代已停止」的既有 host 事件。這兩項結果決定本版不提供無 key 的 reconnect。

**同一個 `host_id` 同時最多一個投遞者**：
- BAT mailroom 模組先取得獨占鎖才能註冊；拿不到鎖時狀態為 `host_in_use`，不註冊、不投遞。這不禁止其他 BAT 程序啟動或管理自己的 runtime，但它們不能用同一個 `host_id` 投遞。
- **鎖檔位置只由 `host_id` 決定**：`<使用者 home>/.local/state/agent-mailroom/delivery-locks/<host_id>.lock`。
  - `<使用者 home>` 從 OS 使用者資料庫解析（Unix `getpwuid`），不讀環境變數，不隨憑證路徑、BAT 資料目錄、`$HOME` 或 `XDG_*` 改變。
  - 目錄權限 0700；`host_id` 必須符合 `host_` 加 hex 的固定格式，格式不符就拒絕，防止路徑穿越。
  - 同一份投遞憑證即使複製到不同目錄，仍對應同一把鎖。
- **取得後持有到 BAT 程序結束**：模組停用、投遞憑證輪替、連線失敗或 `instance_superseded` 都不解鎖、不關閉描述符。首次尚未啟用時，不為投遞啟動連線或取得鎖。
- **不交給子程序**：以 `O_CLOEXEC` 開啟；不在持鎖程序中呼叫沒有 exec 的 fork。A10 僅在受測的 macOS exec 路徑已確認，Linux 與未實作模組的生命週期仍須在實作階段驗證。
- **鎖檔不能刪除或替換**：以 `O_CREAT` 開啟既有檔案，不截斷、不刪除。取得鎖後及每輪投遞前，比對描述符與路徑上檔案的 device／inode；不一致就停止投遞，狀態為 `lock_replaced`。這只能偵測同一使用者的替換行為，沿用本機信任環境的前提。
- **推論的界線**：鎖只保證遵守上述約束的程序對同一鎖檔互斥，用於限制單一投遞者；不能證明舊 runtime 已經結束。即使前一個持鎖 BAT 已退出，其子程序仍可能存活。鎖、epoch 與 terminal 缺席都不得作為 reconnect 的存活證明。

**host instance 註冊與 epoch**：
- 持鎖後呼叫 `POST /delivery/hosts/register`，帶入投遞憑證內的 `host_id`。Mailroom 產生單調遞增的 `host_epoch`，不依賴 BAT 時鐘。
- 新註冊使舊 epoch 失效。取得 pending 綁定、confirm／reject、取得 candidates 及建立 claim 都必須帶目前 epoch；舊 epoch 回 409 `instance_superseded`。BAT 收到後停止新的輪詢、綁定確認與 claim／runtime 送出，且不得自動重新註冊，直到程序重啟；已有 claim 的 outcome 收尾例外見下項。
- 舊 instance 已建立的 claim 仍須依第 3 節回報 outcome：尚未呼叫 runtime 的回 released／not_invoked；已呼叫的取得結果後回報，例外或逾時為 unknown。已知結果即使晚到仍回報一次；回報失敗以同一內容重試至 report_deadline。這個例外只記錄該 claim 的真實結果，不得再次呼叫 runtime、不授予新 claim 資格、不重新啟用通知憑證。
- confirmed 綁定保留其 `host_id`，只有該 host 的目前投遞者可以處理。`host_epoch` 是投遞者版本，不是成員身分或 binding 版本；不得單憑 epoch 更新撤銷成員 key、舊通知憑證或既有綁定。

**BAT 重開不需要 reconnect**：
- host 保存／載入 terminal ID 與 sdkSessionId；單純重開 BAT 不改 terminal ID，所以既有綁定保持有效，不因重開自動轉 pending。新投遞者仍須逐個核對本機 terminal 的 profile、runtime、preset、cwd，且成功接回 runtime 後才能送入通知。
- 補充探查已確認 host 快照的 terminal ID 與非空 Codex sdkSessionId 跨程序保存／載入；Claude 無回合初始化沒有非空 sdkSessionId，真實對話 resume 尚未驗證，不能將欄位還原當成執行狀態還原。
- 尚未 ack 的已通知訊息依原 claim 阻擋規則處理，不因 BAT 重開自動重送。原 binding 未改變時，已送出通知的 key 不因新 epoch 失效；舊 runtime 若仍在處理，可以照原授權 ack。這不是它已停止的證明。

**新模式的無 key reconnect 固定停用**：
- 合法的 `reconnect_member`／HTTP reconnect 請求一律回 409 `liveness_unknown`，不進行 takeover；不以快照、profile ready、terminal 缺席、stop 成功或使用者未經另行定義的確認放行。
- key 遺失但 terminal ID 未變時，仍可嘗試既有 `recover_session`。key 遺失又換 terminal ID，且無法找回原 key 時，不能自動恢復原成員；只能用未被占用的新成員名稱加入，或由使用者手動處理。本版不新增手動 takeover／撤銷 API。

**移出 Phase 1**：`POST /delivery/snapshots`、快照 seq／時效、workspace `complete`、快照驅動的 reconnect 條件，以及 host 關閉完成契約均不實作。A6 記錄為已知限制：profile ready 不等於 workspace complete；日後重新設計存活證明時再處理。這不免除目前 pending 確認與投遞前的逐個 terminal 本機核對；無法核對就不確認、不 claim，不從缺席推論死亡。

### 6. 兩段式綁定

成員註冊（房間、成員）維持交易原子性。通知綁定另有自己的狀態：

```
pending ──confirm──▶ confirmed ──（以原成員 key rebind）──▶ pending（version+1）
   └──reject(reason)──▶ rejected ──（以原 key 再次 join_room）──▶ pending（version+1）
```

| 端點 | 用途 |
| --- | --- |
| `GET /delivery/bindings?state=pending` | BAT 發現待確認的綁定，回傳 `binding_id`、`binding_version`、宣告的 terminal ID、runtime、workspace。 |
| `POST /delivery/bindings/{binding_id}/confirm` | 帶入 binding_version 與實際觀察到的 preset、cwd、profile；只把該版本的 pending 改為 confirmed，版本不遞增並回傳目前版本，claim 沿用它。版本不符或與既有終態矛盾回 409；相同結果的重試冪等成功。 |
| `POST /delivery/bindings/{binding_id}/reject` | 只在 terminal 已存在且確實觀察到 runtime、preset 或 cwd 不符時，將該版本 pending 改為 rejected，版本不遞增。版本不符或與既有終態矛盾回 409，相同結果重試冪等。找不到 terminal 或無法核對時維持 pending，只記診斷，不以 terminal_not_found 拒絕。 |

- 只有 `confirmed` 的綁定會出現在 candidates 中。
- 註冊回傳與 `notification_status` 都會顯示 `binding_state`，以及拒絕的原因。
- pending 不會自動過期。BAT 沒有在執行、terminal 尚未載入或無法核對時，維持 pending，只記診斷；不把缺席當成 mismatch。只有對已存在 terminal 的明確不符才 reject。
- 以原 `session_key` 呼叫 `join_room` 並改變通知目標，是本版保留的 rebind：先核對原成員 metadata、workspace 及 terminal 是否被他人占用，再讓新綁定回 pending、版本加 1。BAT 確認之前不能 claim。
- 凡 binding_version 遞增都視為綁定變更，包括改目標的 rebind 與 rejected 後同目標的 join_room 重試。前一世代所有 claim 的 notification_key 永久失效，legacy 也算在內，不能再建立舊版本 claim；成員 session_key 不變。舊 claim 的真實結果仍只更新它自己，legacy 不接受 outcome 改寫（見第 3 節）。
- pending／confirmed 下，相同 key、metadata 與目標的 join_room 重試不重建綁定。rejected 下，以原 key 再次 join_room 是重新要求確認，即使同目標也依第 3 節建立新世代；之後相同呼叫已是 pending 重試，不再遞增。沒有 binding 時，原成員以 key join_room 也建立新世代，包含 orphan legacy；成員若是 left，先以同 key 恢復 active。兩種情況都先撤銷舊通知憑證，confirm 前不能 claim。
- 這些 join_room 呼叫就是狀態圖的重新綁定入口；recover_session 以同 key／目標呼叫 join_room 時同樣適用。回覆遺失後仍沿用原 key 重試，不輪替成員憑證。
- terminal ID 因關分頁重開或 collision repair 改變後，agent 必須以原 key rebind；沒有 key 就適用第 5 節的限制。collision repair 發生在 renderer，目前沒有可用的 host ID 變更事件，也不會證明舊 runtime 已停止。

### 7. 身分交接與 Codex config 合併

1. 為每個 session 的 Mailroom MCP bridge 注入 `BAT_AGENT_SESSION_ID`，值為該 terminal ID。bridge 在註冊時自動帶入 `bat.session_id`，不把程序共用環境當成單一 session 身分。
2. Claude 使用該次 SDK query 的 `queryOptions.env`；Codex 使用該 thread 的 `config.mcp_servers` entry env。Phase 0 已確認兩種傳遞能力，但尚未完成 BAT 的產品接線。
3. 注入必須涵蓋 **start、resume、reload、replacement thread、collision repair 後的新 session**，並保留原本需要的 resume／thread 設定。不能只修改首次 start 或依賴 reload 專用程序。
4. `recover_session` 繼續依 terminal ID 查回 key；bridge 能取得注入值時使用該值。相同 terminal ID 的 BAT 重啟不要求 rebind；collision repair 會清掉舊 conversation handles，取得新 ID 後要以原成員 key rebind，重新走 pending／confirmed，不能假設存在 host rekey 事件。
5. 找不到可辨識的 Mailroom MCP entry，或 bridge 未取得正確 ID 時，要明確回報未完成自動身分注入；不得宣稱隔離已成立。仍可由持有 key 的 agent 明確提供正確 terminal ID 來 rebind，但本機確認不可省略。

**Codex config 合併規則（Phase 2／3 共用契約）**：
- `codex_app_server.rs:753` 現有的整體 config 指派必須改成欄位合併；所有建立／恢復 thread 的呼叫端都使用同一規則，不改 renderer 的既有 IPC 契約。
- 保留既有 config 的其他欄位。使用者選用 context window 時，只更新 `model_context_window`；未選用時，不為身分注入刪掉既有值。
- 對已明確辨識的 Mailroom MCP entry，合併其 env，將 `BAT_AGENT_SESSION_ID` 設為當前 terminal ID；該欄位的舊值由 BAT 的實際 ID 取代。保留 entry 的 command、args、其他 env，以及其他 MCP servers；不得以整個 `mcp_servers` 或整個 config 取代既有設定。
- config／entry 形狀不符、無法識別目標或有不能安全合併的衝突時，明確回報，不能靜默遺失設定或誤用別的 session ID。
- Phase 2 補上兩種設定同時存在、合併順序不影響結果、沒有選 context window、保留其他欄位與其他 MCP entry，以及兩個共用 app-server session 不混用 ID 的測試。Phase 3 再驗證上述五條生命週期路徑均套用規則。

### 8. 投遞憑證

- Mailroom 執行 `serve --delivery-host` 時，產生 `delivery-credential.json`，內容包含 `host_id` 與 token，權限設為 0600。BAT 設定只保存這個檔案的路徑。
- **輪替**：`agent-mailroom rotate-delivery-credential` 會寫入新檔，舊 token 立即失效。BAT 收到 401 時，重新讀檔一次；仍然失敗，就把模組狀態設為 `credential_rejected`，並停止投遞。
- **檔案遺失、無法讀取，或權限比 0600 寬鬆**：模組狀態設為 `credential_unavailable` 或 `credential_insecure`，不投遞，並顯示在診斷資訊中。
- 投遞憑證與成員的 `session_key`、`notification_key` 完全分開，不能用來讀信或寄信。

### 9. BAT mailroom 模組的每輪流程

位置在 `src-tauri/src/mailroom/`（Rust）。

1. 讀取 `settings.json` 的 `mailroomDelivery.enabled` 與憑證路徑。未啟用，就不做任何事。
2. 先依第 5 節取得並持有固定位置的 host 鎖，再呼叫 `POST /delivery/hosts/register` 取得 `host_epoch`。收到 instance_superseded 後停止新的輪詢、綁定確認、claim 與 runtime 送出，直到程序重啟；既有 claim 仍依第 5 節收尾回報，並非直接丟棄 outcome。
3. 以目前 `host_epoch` 處理 pending 綁定（確認或拒絕）；不送出存活快照，也不產生 workspace `complete`。
4. 取得 candidates。對每個候選，在本機核對 terminal：必須存在於本機 profile，preset、runtime 與 cwd 都要和綁定一致；找不到或無法核對時只記診斷、不確認也不 claim；已確認的綁定本輪略過，不藉此宣告 terminal 死亡。
5. 選擇性預先查詢一次閒置狀態，只用來減少不必要的 claim。
6. `POST /delivery/claims` 取得 `claim_id`、期限與提示。收到時，以本機 monotonic clock 換算送出截止時間，並預留一段安全邊界。
7. claim 之後才發現 terminal 不符而決定不送時，回報 `released`，證據為 `not_invoked`。
8. 透過 router 送出，參數帶 `requireIdle: true` 與 `notAfter`，`clientMessageId` 設為 `mailroom-<claim_id>`。每個 claim 只呼叫 runtime 一次。
9. 依第 4 節回報 outcome；instance_superseded 後仍依第 5 節完成既有 claim 的回報。回報失敗以相同內容重試至 report_deadline，不再呼叫 runtime；晚到的已知結果仍可依第 3 節回報一次。
10. 輪詢間隔從 2 秒起，退避到最長 30 秒。

### 10. 遷移與相容

- Mailroom 的 `serve` 二選一：舊模式（`--bat-fingerprint`）或新模式（`--delivery-host`）。兩者同時啟用時會拒絕啟動，以免重複投遞。
- 既有綁定的 terminal ID 就是 BAT 的 session ID；切換新模式時所有綁定一次性轉 pending，等 BAT 確認，同時依第 3 節回填 legacy 阻擋 claim。轉 pending／confirm 不清掉舊通知狀態、不放行 unknown 或已送出未 ack 的信，不能把新表沒有紀錄當成可重送。
- 相同 terminal ID 下單純重開 BAT 不要求 reconnect；初始化遷移把既有綁定轉 pending 是一次性的模式切換措施，不能在每次重開重做。
- 新模式保留以原 key rebind 與兩段式確認，無 key 的 reconnect 一律回 `liveness_unknown`。不讓新舊 adapter 同時投遞，也不以舊 adapter 的缺席推論替新模式放行 reconnect。
- 確認新模式穩定後，再另開一張票移除舊 adapter。成員憑證輪替是另一張獨立票，見估算章節；本版的投遞憑證輪替（第 8 節）不等於成員 `session_key` 輪替。

## 驗證策略

| 層級 | 方法 | 驗證的保證 |
| --- | --- | --- |
| Mailroom 單元測試 | pytest，比照 `test_notifications.py` | 下列每一項都要有一個測試：<br>• 先寫入 claim 才回傳提示<br>• `unknown` 不重送<br>• 每個綁定只處理最舊的未 ack 訊息，最新 claim 未解除時不往後投遞<br>• outcome 的 binding 與 claim 記錄值不符時被拒絕<br>• 綁定變更後，舊 claim 的真實結果仍可記錄，且不影響新 claim<br>• 矛盾的回報回 409<br>• `released` 沒有允許的證據時被拒絕<br>• `released` 保留為終態並撤銷 key，重送冪等，下一次 claim 拿到新的 `claim_id`<br>• 超過 `report_deadline` 轉為 `unknown` 而不是釋放，之後的晚到回報只能改寫一次<br>• 啟動恢復產生的 `unknown` 不接受回報<br>• pause 不撤銷通知憑證，恢復 active 後仍可使用；leave、rebind 後舊 notification_key 回 403<br>• rebind 後原成員 key 的讀信、回信、ack 仍有效，並核對獨立授權路徑，照實保留現有限制<br>• leave 不撤銷成員 key，同 key 可恢復 active 並重新綁定<br>• 新模式合法的無 key reconnect 一律回 409 liveness_unknown，不建立新綁定或替換成員身分<br>• 舊 epoch 的 pending／confirm／reject／candidates／claim 請求回 instance_superseded；舊 claim 的 outcome 仍依第 3 節處理<br>• rebind 後回 pending，完成確認前不能 claim；pending／confirmed 的同目標重試不重建 binding<br>• 找不到或無法核對 terminal 時仍為 pending；明確不符才 reject；rejected 後同 key／同目標 join_room 可重新進 pending 並只增加一次版本<br>• 遷移 accepted 未 ack 通知經 reject、同 key／同目標 join_room 重新進 pending 並 confirm 後，legacy 通知 key 回 403，新 claim 只有新通知 key 有效，原成員 key 仍有效<br>• orphan legacy 成員以原 key／同目標 join_room 建立高於遷移基準的新版本，confirm 前不能 claim、confirm 後可 claim；原為 left 時先同 key 恢復 active<br>• 現存 binding 的成員另有 orphan legacy：遷移轉 pending 的同版本初次 confirm 不放行；透過改目標、先 reject 再重新確認，或 leave 後恢復再 join_room 建立更新世代，並確認後才放行；同目標的 pending／confirmed 重試不增版，不能用 host_epoch 更新替代<br>• 遷移基準為首次回填時的最高版本、無版本取 0；schema 重跑不重算，leave／刪 binding 不重設歷來最高版本<br>• confirm／reject 不改版本，成功重試冪等；相反終態或舊版本回報回 409，claim 使用確認回傳的版本<br>• 舊 binding 有 accepted 未 ack claim 時，key rebind 並確認後，新 binding 可 claim 該信；舊 outcome 仍只更新舊 claim<br>• 升級保留舊 notifications 的 accepted／not_accepted／unknown 阻擋；submitting 轉 unknown；遷移／confirm／重啟／重跑 schema 都不自動重送，legacy unknown 不接受 outcome 改寫<br>• BAT 重啟與 epoch 更新不改既有 binding 版本、狀態或通知憑證，也不自動重送已通知而未 ack 的訊息<br>• 同一綁定的最新 claim 是 `unknown`、`not_accepted` 或 `accepted` 未 ack 時，後續訊息不能被 claim；`released` 時最舊的訊息可以重新被 claim<br>• 晚到的回報不重新啟用已撤銷的 key<br>• pending 綁定不出現在 candidates<br>• 取得 candidates 後成員才轉為 paused：之後的 claim 回 409，不寫入 claim、不產生通知憑證；恢復 active 後同一則訊息可正常 claim<br>• 投遞憑證不能讀寫信件 |
| sidecar 測試 | `pnpm run test:sidecar` | `requireIdle` 為 true 時，以下每一種狀態都回傳 `busy`、不排隊，並且沒有 user echo、沒有寫入去重紀錄：串流中、有佇列、有待處理的權限或問題、resting。另外：resting 狀態不會被清除；超過 `notAfter` 時回傳 `expired`；已接受的 `clientMessageId` 重新呼叫時，回傳 `duplicate` 而不是 `busy`；`duplicate` 加 `pending`，以及原本那次被拒絕的情況，都對應到 `unknown`；預設呼叫的行為不變（回歸測試）。 |
| Rust 單元測試 | `#[cfg(test)]`，使用假的 Mailroom HTTP 與假的 router | Codex `require_idle` 遇到 active turn 或 resting 時，不取代回合，`is_resting` 也不變。在等待 `turn_operation_lock` 期間超過 `notAfter` 時，回傳 `expired`。各種回傳都對應到正確的 outcome。收到 instance_superseded 後不再輪詢、確認或建立 claim／呼叫 runtime，也不重新註冊；已建立 claim 的結果回報與同內容重試仍執行至 report_deadline，未呼叫的回 released／not_invoked，不能漏掉收尾。第二個程序拿不到鎖時，狀態為 `host_in_use`，也不註冊；憑證被複製到不同路徑時，兩個程序仍然對應到同一個鎖檔。模組停用或憑證輪替後，鎖仍然持有。鎖檔被替換時，狀態為 `lock_replaced`，並停止投遞。`host_id` 格式不符時拒絕。未啟用時不連線。透過新的內部 router 介面在 desktop／headless 取得 runtime，保留本機綁定核對；無法核對 terminal 時不確認、不 claim。Codex config 合併須同時保留 model_context_window 與每個 thread 的 Mailroom MCP env，並保留其他設定與其他 MCP entry；反向合併順序與兩個共用 session 也須驗證。 |
| 競態測試 | 讓模組送出與使用者送出交錯（sidecar 用同步排程，Rust 用執行緒） | 只要使用者的回合先取得臨界區，Mailroom 送出就回傳 `busy`。 |
| 身分交接測試 | BAT runtime／bridge 測試，使用明確的測試注入點 | start、resume、reload、replacement thread、collision repair 後的新 session 都注入正確 ID；兩個 Claude 與兩個共用 Codex session 的 bridge ID 不混用。持 key rebind 保留成員身分並重新確認；無 key 時回 liveness_unknown。測試注入點的成功不能替代產品路徑驗收。 |
| 既有測試 | 上游規範列出的全部指令 | 不退步。 |
| 實機驗證 | 測試用 Mailroom 實例（另外的 state dir 與 port）加上本機 BAT dev build | Claude 與 Codex 各收到一則通知，處理後 ack。Codex 忙碌時，通知延後到回合結束才送。BAT 重啟後，以相同 terminal ID／保存的 sdkSessionId 接回真實對話，bridge 讀到正確 ID，綁定不要求 reconnect，已通知而未 ack 的信不因重啟重送；進行中的原通知仍可依原授權 ack。另測以原 key 換 terminal 後走 pending／confirmed，以及新模式無 key reconnect 固定回 liveness_unknown。每份結果記錄平台、BAT commit、desktop／headless 與 runtime 路徑；Linux 尚未驗證，必須另取得其結果才能宣告 Linux 支援已驗收。 |

無法自動驗證：模型是否遵守通知提示中的指示，或持同一成員 key 的新舊 runtime 是否避免並行處理。server 可測的是不同憑證的授權結果；Phase 1 不宣稱已撤銷舊 runtime 的成員權限。這沿用現行 rebind 的限制。

## 階段與估算

0.1 pt = 1 小時實際工程量，不含會議、context switch 與等候。以下依草案 4 重新估算，尚未實作；不是把探查的實際經過時間換算成工時。單票上限 2.0 pt，不同 owner／性質分票；後續估算超過上限時必須再拆。

| 階段／票 | 範圍與可審查結果 | 估算 |
| --- | --- | --- |
| Phase 0 探查與補充 | 已完成，兩份報告均已接受；不計入下列待辦總額。未驗證的真實 resume 與平台路徑保留到後續驗收，不因接受報告而視為通過。 | 不回推實際 pt |
| Phase 1a Mailroom claim／outcome | delivery_claims、期限、阻擋規則、schema 備份升級（含 legacy 通知阻擋轉換）與對應測試。agent-mailroom repo。 | 0.9 至 1.2 pt |
| Phase 1b Mailroom host／綁定與投遞憑證 | host_epoch、投遞憑證、兩段式確認、既有 key rebind、無 key reconnect 固定拒絕、相容性測試；不含存活快照、成員憑證輪替。 | 0.8 至 1.1 pt |
| Phase 2a BAT router 與原子送出 | 內部 router 介面、本機綁定核對、Claude／Codex requireIdle／notAfter、去重結果及競態／回歸測試；先補 Codex config 合併規則與測試。 | 1.1 至 1.5 pt |
| Phase 2b BAT 投遞模組 | 固定位置鎖及生命週期、epoch 停止規則、背景輪詢、claim／outcome、settings 與診斷；不做存活證明或關閉完成契約。 | 1.2 至 1.6 pt |
| Phase 3a BAT 身分注入 | start、resume、reload、replacement thread、collision repair 後的新 session；Claude query env、Codex per-thread MCP env、跨路徑及隔離測試。 | 0.7 至 1.0 pt |
| Phase 3b Mailroom bridge 身分交接 | 自動帶入 terminal ID、recover_session、以原 key rebind、pending 狀態及無 key 限制的回傳；更新 recover_session／reconnect_member 的工具說明，新模式找不到 key 時指向新成員名稱加入或手動處理，不再引導自動 takeover。agent-mailroom repo，與 BAT 改動分票。 | 0.4 至 0.6 pt |
| Phase 4a 整合與實機驗收 | 測試用 Mailroom；Claude／Codex 真實對話重啟恢復、bridge ID、ack、busy 延後、rebind；分開記錄 macOS／Linux、desktop／headless 結果。 | 0.8 至 1.2 pt |
| Phase 4b 遷移與操作文件 | 新舊模式切換、手動恢復限制、已知風險及診斷說明。Settings UI 不在本次已確認需求內。 | 0.3 至 0.5 pt |

待辦工程量合計 **6.2 至 8.7 pt（62 至 87 小時）**，不含已完成探查、成員憑證輪替與等候審查的日曆時間。Phase 1 合計 1.7 至 2.3 pt、Phase 2 合計 2.3 至 3.1 pt，因此已分別拆票；router 內部介面及 config／身分路徑測試的工作均已納入。各票驗收需包含其測試證據與後續 peer review；等待審查不算工程量，但會延長排程。沒有受測平台環境時須列為未驗證，不能以其他平台通過替代。

**獨立票名：Mailroom rebind 成員憑證輪替與恢復相容性評估。** 範圍僅列授權模型、回覆遺失後的重試與冪等性、`recover_session`、與現行「重試時沿用同一把 key」規則的相容性。另票估算與審查，不納入上述投遞工程量；本草案不指定輪替協定或恢復流程。

## Phase 0 探查項目（已完成，保留查核範圍）

1. **兩個 session 的身分隔離**：同時啟動兩個 Claude SDK session，以及兩個 Codex session（走共用 app-server 的路徑），確認各自的 MCP bridge 讀到的 ID 不同且正確。接著分別測試 resume、Codex reload，以及 `repairTerminalIdentityCollision`。只確認單一 session 讀得到環境變數是不夠的。
2. **Codex 每個 session 的注入點**：`thread/start` 的 `config` 能否帶入 MCP server 設定或環境變數；如果不行，找出其他做法。
3. **sidecar 閒置狀態能否同步取得**：在 `claude.sendMessage` 內部，能否同步讀到串流狀態、待處理權限、待處理問題與 `isResting`（對應 A5）。
   - 另外確認：Codex 的 resting 是否也經過 sidecar 路徑；以及除了 `claude.sendMessage` 與 Codex `send_message`，還有哪些地方會清除 resting。
4. **workspace 還原完成的訊號**：BAT 是否有可靠的「profile 的 workspace 已還原完成」事件或狀態（對應 A6）。
5. **BAT 結束時 runtime 是否一起結束**（對應 A9 與 A10）：
   - **判定時點**：先記下舊 BAT 的 sidecar、Codex app-server 與 Claude Code 子程序的 PID，然後正常關閉或強制終止（`kill -9`）BAT。新程序在迴圈中嘗試取得同一個鎖，在**取得鎖的同一刻**檢查那些 PID 是否都已不存在。只要有任何一個還在，即使幾秒後才消失，A9 也不成立。
   - **鎖的行為**：子程序是否繼承了鎖的描述符；模組停用、憑證輪替時鎖是否仍然持有。
   - 確認 desktop 與 headless 是否共用資料目錄（對應 A8）。
6. **headless 的可用性**：在 headless `bat-server` 中能否取得 `ClaudeRuntimeRouter`，以及是否必須繞過 `ensure_local_agent_session_access`。

Phase 0 與補充報告已交付並接受，連結見本文件開頭。上述清單保留當時的查核目標，不代表每一條實機情境都已通過；實際判定與未執行項目以報告為準。以下表格列出本版採用的結果及剩餘驗證。

## 假設查核結果與剩餘驗證

以下「已確認」只限證據所列平台與路徑，不能外推為產品接線已完成。P0 指文件開頭的 Phase 0 報告，P0-S 指補充報告。

| 項目 | 判定與本版處理 | 證據／仍需驗證 |
| --- | --- | --- |
| A1 Claude query env → MCP | 已確認傳遞能力。 | P0「A1」；SDK 0.3.316＋Claude Code 2.1.318。start／resume／reload 等 BAT 產品路徑仍須驗證。 |
| A2 Codex 每個 session 注入 | 已確認共用 app-server 的 thread config MCP env 可各自隔離。 | P0「第 2 項、A2」；CLI 0.166.0。依第 7 節合併 config 並補全生命週期。 |
| A3 terminal 身分保存 | 原始碼／store 及 host 保存載入已確認；改 ID 後改走以 key rebind。 | P0「A3」、P0-S「第 2、3 項」。非空 Codex sdkSessionId 已實測保存；Claude 非空 sdkSessionId 與真實對話 resume 未實測，不能宣告已恢復執行。 |
| A4 headless router | 需新增明確的內部介面；runtime states 與無視窗 remote 路徑存在。 | P0「第 6 項、A4」；目前 router／from_ctx 私有，Phase 2a 處理並保留本機核對。 |
| A5 同步讀取閒置狀態 | 已確認狀態可讀，包含 streaming／queue／permission／question／resting；另須涵蓋 runtimeStatus。 | P0「第 3 項、A5」；原子接受、期限及競態保證尚待 Phase 2 測試。 |
| A6 可靠的 workspace 還原完成訊號 | **不成立／移出 Phase 1（不能作為本版可用前提）**。已知限制是 ready 不等於 complete，本版不建立完整性旗標或以它作存活證明。 | P0「第 4 項、A6」：ready 同時 snapshot 為 null；原報告對是否另有可靠 host 訊號的判定為需改設計，並未證明所有路徑均無訊號。日後設計存活證明時再查。 |
| A7 記憶體去重 | 已確認現有 clientMessageId Map 不是持久化憑證。unknown 不自動重送。 | P0「A7」；不能以單元測試替代真實對話跨重啟驗收。 |
| A8 desktop／headless 並存 | 原始碼已確認可由同一使用者執行，預設資料目錄不一定相同，也可指定相同目錄。 | P0「A8」；同一 host_id 以固定位置鎖限制投遞者，不限制其他程序的 runtime，也不宣告其已死亡。 |
| A9 新 BAT 取得鎖時舊 runtime 全部結束 | **不成立／存活證明移出 Phase 1**。新模式無 key reconnect 固定回 liveness_unknown。 | P0「A9：退出機制與鎖取得時點」、P0-S「關閉路徑與完成時點」。反例在 macOS headless 成立：取得鎖當下，`ps` 顯示 Claude Code、Codex 程序仍為 R（執行中），SIGSTOP 情境為 T，不只是 `kill(pid, 0)` 把 zombie 算成存在（P0 `headless-lifecycle-sdk*.json` 的 `pid_status_after_acquire`）。不靠鎖、epoch 或既有關閉事件推論死亡。 |
| A10 鎖描述符不被繼承 | **已確認，僅限受測 macOS exec 路徑**。 | P0「A10」與 rust-fd-results.json；Linux、實際模組停用／投遞憑證輪替時持鎖的行為仍待實作驗證。 |

P0-S 另確認 collision repair 在 renderer，沒有現成 host rekey／關閉完成事件。Phase 1 不新增這類生命週期契約；依第 6、7 節以原成員 key rebind，無 key 時保留無法自動恢復的限制。

## 風險

- **上游變動快**：Tauri 遷移仍在進行，Codex 正從 sidecar 移到 Rust。模組只依賴 router 與 `HostContext`，`requireIdle` 也只是新增參數，以降低合併衝突。
- **改到共用的送出路徑**：`requireIdle` 會改動 sidecar 與 Codex 的送出程式，即使預設值不改變行為，也要追蹤所有呼叫端並補上回歸測試。
- **提交點提前**：claim 後無法撤回提示。撤銷 notification_key 只會拒絕使用該通知憑證的操作；成員 session_key 仍有效，舊 runtime 可能繼續操作。模型是否遵守停止處理指示，也無法由 server 保證。
- **rebind 不隔離舊 runtime**：新舊 runtime 持同一成員 key 時可能並行讀信、回信、ack；Codex 共用 app-server 的 stop 不中斷 thread，是已知可能觸發情境。這與現行行為相同，Phase 1 沒有改善或隱藏此限制。
- **無 key 的恢復受限**：key 遺失又換 terminal ID 時，新模式不能自動恢復原成員。不能用新成員名稱冒稱原身分；使用者手動處理的流程也不在本版新增授權 API。
- **實際 resume 尚未驗證**：保存／載入身分欄位不代表對話已成功 resume；必須在實機驗收補上，且不得向尚未接回的 runtime 宣稱投遞成功。
- **`report_timeout` 會讓 `unknown` 變多**：BAT 當機時，通知會停在 `unknown`，需要人工判斷；這是「不能證明沒送就不重送」的代價。
- **Claude 的通知會晚一點到**：從「排在回合後面」改為「等閒置才送」。
- **`unknown` 仍會卡住該成員之後的通知**：這是現有保證帶來的副作用。`released` 消除了「確定沒送進去」的那一類，但真正不明的情況仍無法自動處理。
- **兩段式綁定改變註冊語意**：成員可能已經註冊，卻還收不到通知。
- **單一使用者的本機信任環境**：投遞憑證只防止誤用，不防同一使用者的其他程序，這和 Mailroom 現有的前提相同。

## 已決事項（2026-11-06 本次討論結果）

| 事項 | 決定 |
| --- | --- |
| 投遞憑證交接 | 採檔案方式，BAT 設定只保存路徑；Unix 用 0600；Windows 待定義 ACL 後才支援。輪替、遺失與讀取失敗的行為見第 8 節。 |
| Claude 忙碌時 | 和 Codex 一樣等閒置，並以 `requireIdle` 在 runtime 臨界區內判斷。 |
| 兩段式綁定 | 採用。成員建立維持交易原子性；通知綁定另有 pending／confirmed 狀態，註冊回傳就顯示 pending。 |
| 上游 PR 形式 | 直接以 `mailroom` 命名。目前只有一個來源，沒有實際需求需要通用抽象，上游 maintainer 的意向也尚未確認。 |
| reconnect 方向 | 採第 3 項，Phase 1 先做 (a′)：保留以 session_key rebind 與兩段式確認；新模式無 key reconnect 固定回 liveness_unknown。 |
| 存活證明 | 快照、complete 與 host 關閉完成契約移出 Phase 1；鎖及 epoch 只限制投遞者。 |
| 成員憑證輪替 | 另開獨立票，只列授權、重試／冪等性、recover_session 與相容性範圍，不混入投遞實作。 |
| 既有授權限制 | rebind、leave 均不撤銷 session_key；舊 notification_key 失效不等於舊 runtime 失去成員權限。 |

## 修訂紀錄

草案 1 至 3.2 的條目保留歷史決策，其中已被草案 4 取代的快照、reconnect 與未區分兩層 key 的文字，不作為本版契約。

- **草案 1**（2026-11-06）：初版。
- **草案 2**（2026-11-06）：依審查意見修正六項缺口，並記錄四項決定。
  1. 閒置判斷移進 runtime 臨界區，新增 `requireIdle`。也確認 `resume_after_rate_limit` 的判斷與送出之間沒有鎖，不再引用它作為保證。
  2. 明訂 claim 為提交點，加入 lease、key 撤銷與 `released`。不再宣稱原本的保證 5 不變。
  3. 新增 `claim_id`、`binding_version` 與 `delivery_claims` 表；outcome 會拒絕過期或矛盾的回報。
  4. 存活快照加入 host、instance、序號、有效期與 profile 載入狀態，「未知」時不允許 reconnect。
  5. 依 runtime 判讀結果：Codex 成功只回 `{ok:true}`，非預期的格式一律記為 `unknown`。
  6. 補上兩段式綁定的發現、確認、拒絕與重新確認流程。
  7. Phase 0 加入兩個 session 的身分隔離驗證，並註明共用 Codex app-server 的限制。
- **草案 3**（2026-11-06）：依第二次審查修正。
  1. lease 拆成 `send_deadline` 與 `report_deadline`，修正「過期就回報 `released`，卻又被 409 拒絕」的矛盾。期限內送出、期限後才回報的結果，仍然可以記錄。
  2. 送出期限以 `notAfter` 帶進 runtime 的臨界區檢查，涵蓋等待鎖與排程的時間。
  3. `released` 只接受明確的未送入證據（`busy`、`expired`、`not_invoked`），保留為終態，不刪除；重送冪等，下一次 claim 使用新的 `claim_id`。server 不會因為期限到了就自動釋放。
  4. `requireIdle` 補上 resting 條件。sidecar 的判斷放在去重紀錄寫入與 user echo 之前，但放在只讀的去重查詢之後。
  5. 更正 pause 語意：現行原始碼中 pause 不撤銷 key，草案 2 寫錯了。新版保留現行的暫停語意，只有 leave 與綁定變更才永久撤銷。另註明「撤銷後停止處理」是協定要求，無法保證模型一定遵守。
  6. outcome 改為比對 claim 建立時記錄的綁定版本，而不是目前的綁定，讓舊 terminal 那次送出的真實結果仍然可以記錄。
  7. 存活快照加入由 Mailroom 產生的 `host_epoch`、`seq` 不得倒退、舊 instance 被取代後停止投遞，以及同一個 `host_id` 只能有一個投遞者；並註明有效期內可能是過時資料。
  8. 狀態改為可以先進行 Phase 0，Phase 1 前再收斂協定。
- **草案 3.1**（2026-11-06）：依第三次審查修正 Phase 1 前的問題。
  1. 阻擋規則改以綁定為單位：最新的 claim 是 `released` 時，最舊的未 ack 訊息才能重新競爭；`unknown`、`not_accepted` 都不會自動放行。修正與「`unknown` 會卡住通知」說法的矛盾。
  2. 去重命中依原本那次的狀態判讀：`pending` 與被拒絕的情況都不判為已接受；無法確認時記為 `unknown`，不能 `released`，也不能重新呼叫 runtime。
  3. 加入反例：同一個 `host_id` 下兩個 BAT 同時存活時，新 instance 的快照不能代表舊 terminal。新增 `flock` 獨占與綁定記錄 `host_id` 兩條規則；新增待驗證假設 A9 與 Phase 0 第 5 項；Phase 0 初估因此從 0.4 至 0.6 pt 調為 0.5 至 0.7 pt。
  4. 第 6 節對齊第 3 節：綁定變更會撤銷舊 key，舊 claim 的真實結果仍然可以記錄。晚到的回報不重新啟用 key，也不改變目前綁定的投遞資格。
- **草案 3.2**（2026-11-06）：依第四次審查修正鎖的定義。
  1. 鎖檔位置改為只由 `host_id` 決定，路徑從使用者資料庫解析，不再放在憑證檔旁邊。這樣即使同一份憑證被複製到不同目錄，仍會對應到同一個鎖。
  2. 明訂鎖持有到程序結束、不交給子程序繼承、鎖檔不能被刪除或替換（以 device 與 inode 比對偵測）。
  3. 更正推論：取得鎖本身不等於前一個程序已結束，只有在上述約束都成立時才能推論。epoch 檢查只能擋下舊投遞者之後的請求。
  4. A9 的判定時點改為新 BAT 取得鎖的當下；新增 A10（描述符不被繼承），並擴充 Phase 0 第 5 項。

- **草案 4**（2026-11-06，**本次討論結果**）：依使用者選定的第 3 項及兩份已接受探查報告修訂。
  1. Phase 1 採 (a′)：保留原 key rebind 與兩段式確認，新模式無 key reconnect 固定回 liveness_unknown，不執行 takeover。
  2. 重寫第 5 節，移出 delivery snapshots、seq／時效、workspace complete 與快照 reconnect 條件；保留固定位置 flock、host_epoch、instance_superseded，明訂只限制投遞者，不證明舊 runtime 已停。
  3. 分清 notification_key 與 session_key：rebind／leave 使舊通知憑證失效，但不撤銷成員 key；成功 reconnect 才替換該 room 的成員身分，且新模式停用此路徑。補入舊 runtime 可繼續操作與 Codex stop 不中斷回合的限制。
  4. 明訂 BAT 重開不要求 reconnect、不因 epoch 更新重建 binding 或重送已通知未 ack 的信；區分 host 身分欄位保存／載入與仍未驗證的真實對話 resume。
  5. A6 記錄為 ready 不等於 complete 的已知限制，移出 Phase 1；A9 記為不成立；A10 僅在受測 macOS exec 路徑已確認，均附報告位置。
  6. 增加 Phase 2 的內部 router 介面與本機綁定核對範圍，不修改 renderer IPC 或假設私有 router 已可直接使用。
  7. 身分注入涵蓋 start、resume、reload、replacement thread、collision repair 後的新 session；明訂 Codex config 欄位合併、保留 model_context_window／其他 MCP 設定，增加兩種設定並存與隔離測試。
  8. 記錄 collision repair 在 renderer 且沒有 host 事件；改 ID 後以 key rebind，無 key 且無法 recover 時以新成員加入或由使用者手動處理。
  9. 移除存活快照與存活 reconnect 的驗證，新增 rebind 後通知憑證 403／原成員憑證仍有效，以及新模式無 key reconnect 固定拒絕的測試；保留租期、outcome、unknown 與原子送出測試。
  10. 重新估算待辦工程量為 6.2 至 8.7 pt，Phase 1、2 及跨 repo 身分工作分票；另列成員憑證輪替票名與範圍，不在本草案設計輪替。
  11. 依 Claude 第一輪複審補齊一致性：找不到 terminal 保留 pending；rejected 可用原 key／同目標 join_room 重新確認；版本只在新綁定世代遞增，confirm／reject 不遞增並明訂重試與終態衝突。
  12. 升級時把舊 notifications 保留為 legacy 阻擋 claim，避免切換新表造成重送；補上每個綁定的阻擋測試、rebind 確認後新綁定可 claim、被取代 instance 的 outcome 收尾，以及 bridge 恢復說明更新。
  13. 依 Claude 第二輪複審，統一所有綁定世代遞增都撤銷前一世代的通知憑證（包含 rejected 同目標重試及 legacy）；補明 orphan legacy 的遷移世代基準與對應測試，成員 key 維持不變。
  14. 依 Claude 第三輪複審，補明無 binding／leave 恢復的世代建立路徑、遷移基準取值與不可重算規則；區分既有 binding 的同版本遷移確認與明確建立新世代後的確認，並新增兩種 orphan legacy 測試情境。
  15. Claude 最終只讀複審結論為 ACCEPTED；納入解除阻擋入口及 orphan legacy 測試的非阻擋措辭建議，並將現行保證與本版新增的通知世代規則分開標示。這是規格審查結果，不代表產品實作或各平台驗收通過。
  16. 使用者審查結論為 ACCEPTED（規格層級）。依非阻擋建議：claim 的 409 條件補上成員非 active，並明訂在寫入 claim 的同一交易內重新核對；Phase 1a 新增「取得 candidates 後成員才 pause」的競態測試；A9 補上取得鎖時仍為 R／T 的程序狀態證據。
