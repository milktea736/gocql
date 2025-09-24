# Cluster 與 Session 生命週期

## ClusterConfig 與預設設定
`ClusterConfig` 是驅動程式的核心進入點，用來設定節點列表、逾時、壓縮、認證、連線池大小與預設一致性等參數；同時也能配置警示處理、重新連線與主機篩選策略。【F:cluster.go†L57-L240】`NewCluster` 提供對應的預設值，包含 11 秒的請求/連線逾時、雙連線池大小、Quorum 一致性以及預設的 `WarningsHandler`、記錄器與重新連線策略，作為讀程式碼時的基準設定。【F:cluster.go†L401-L441】建立 Session 前，`Validate` 會檢查輸入是否合規（例如 Host 列表、認證配置、重新連線策略），確保後續初始流程不會因基本錯誤而失敗。【F:cluster.go†L501-L520】建議閱讀時先確認專案是否使用 `NewCluster` 預設值或自訂設定，以掌握下游組件的運作條件。

## Session 結構與觀察介面
`Session` 負責整個生命週期的管理，內部維護預設一致性、分頁與預取參數、查詢與批次觀察者、連線觀察者、Frame 監控器、Ring 描述器、Prepared Statement LRU 等資訊，是多個子系統的集合點。【F:session.go†L45-L110】配置時若提供 `WarningsHandlerBuilder`，會在 Session 建立時產生對應的警示處理器，以便在查詢執行結束後集中處理伺服端回傳的 warning。【F:cluster.go†L149-L149】【F:session.go†L205-L208】預設的處理器會寫入 Session 的 logger，也可以改成自訂實作。【F:warning_handler.go†L3-L28】這些欄位說明了 Session 為何能提供觀察性功能：查詢、批次、連線的 Observer 介面都是從這裡注入的。

## Session 建立流程與資源初始化
`newSessionCommon` 會先驗證 Cluster 設定，建立 `context`、預設一致性、Prepared Statement LRU 等資源，確保後續流程失敗時能正確釋放。【F:session.go†L139-L160】`NewSession` 會同步建立 `ConnConfig`、`policyConnPool` 與 `queryExecutor`，並啟動 goroutine 執行 `Session.init` 完成非阻塞初始化，同時回傳 `readyCh` 供等待完成。【F:session.go†L161-L254】初始流程會解析主機清單、建立控制連線、根據 `system.peers` 更新 Host 清單並設定 Partitioners，再填滿各節點連線池；若 `ReconnectInterval` > 0，亦會啟動背景任務定期嘗試重連下線節點。【F:session.go†L256-L445】閱讀初始流程時，可以依序追蹤：`addrsToHosts` → `control.connect` → `hostSource.GetHostsFromSystem` → `pool.addHost` → `hostConnPool.fill`，藉此掌握每個環節的責任。

## 控制連線與心跳
`controlConn` 介面負責執行系統查詢、協調 schema agreement 與偵測協定版本。它在 `heartBeat` 迴圈中週期性送出 `OPTIONS` frame，若失敗則觸發 `reconnect` 重建控制連線。【F:control.go†L61-L144】`Session.AwaitSchemaAgreement` 透過控制連線等待 schema version 一致，配合 `MaxWaitSchemaAgreement` 逾時，確保拓撲或 schema 變更後再繼續查詢。【F:session.go†L448-L463】閱讀控制連線時，可先理解 `createControlConn` 如何建立 retry policy，再進一步跟進 `discoverProtocol` 與 `querySystem` 等實作。

## 事件處理與拓撲更新
Session 建立兩個 `eventDebouncer` 分別負責 schema 與節點事件，透過緩衝與定時 flush 避免大量事件造成壓力。【F:events.go†L33-L109】schema 事件會清除 metadata 快取並通知負載策略；節點事件則整合拓撲與狀態更新，再呼叫 `handleNodeConnected` / `handleNodeDown` 等邏輯更新連線池與策略狀態。【F:events.go†L120-L200】閱讀時可先看 debouncer 如何聚合 frame，再對應到 `Session` 中哪些欄位被更新，理解事件 → metadata → pool → policy 的資料流。

## 可觀測性與追蹤
Session 提供多層級觀察：
- `QueryObserver`、`BatchObserver` 會在嘗試完成後收到延遲與節點資訊，可從 `Query.attempt`／`Batch.attempt` 看詳細內容。【F:session.go†L1315-L1333】【F:session.go†L2248-L2276】 
- `ConnectObserver` 在連線建立與結束時回報，方便記錄連線握手花費。【F:conn.go†L287-L313】
- `Tracer` 與 `TraceWriter` 透過控制連線查詢 `system_traces` 取得事件紀錄，適合除錯複雜查詢。【F:tracer.go†L10-L121】
- `WarningHandler` 會在迭代器取得 frame 後被呼叫，集中處理伺服端警告。【F:conn.go†L1454-L1462】【F:warning_handler.go†L3-L22】

閱讀時建議先確認需求是否依賴這些觀察點，再追蹤對應的欄位與回呼時機，便能理解如何在應用程式中掛載監控與追蹤邏輯。
