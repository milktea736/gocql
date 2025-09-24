# 連線與連線池管理

## ConnConfig 與 TLS 設定
`ConnConfig` 彙整了協定版本、讀寫逾時、Dialer/HostDialer、壓縮器與認證等細節，建立連線時會直接套用；其中 TLS 相關設定來自 `SslOptions`，支援從憑證路徑載入或整合既有的 `tls.Config`，並可開啟主機名稱驗證。【F:conn.go†L70-L214】`connConfig` 工廠會根據 Cluster 設定建立預設的 `ScyllaShardAwareDialer`，必要時套用使用者自訂的 Dialer 或 HostDialer。【F:connectionpool.go†L65-L103】閱讀時可先確認應用程式是否覆寫這些欄位，以理解後續握手流程的差異。

## Conn 結構與握手流程
`Conn` 結構封裝了底層 socket、讀寫緩衝、streams 產生器、呼叫表、壓縮器、認證器與主機資訊，同時保存 frame/stream 觀察者與超時設定，顯示每個連線同時處理多重請求的能力。【F:conn.go†L191-L360】`dialShard`／`dialWithoutObserver` 會透過 HostDialer 建立 TCP 連線、初始化 Reader/Writer、設定壓縮器與 streams，並保存 host 資訊；完成初始查詢與認證後會呼叫 `finalizeConnection` 將逾時改回營運值，並依 Session 設定啟用 tablets routing 支援。【F:conn.go†L296-L360】【F:conn.go†L259-L267】【F:session.go†L307-L334】想理解握手細節時，可從 `dialShard` 進入，依序觀察 dial、`writeStartupFrame`、`authenticateHandshake`、`registerWithSchema` 等私有方法。

## 連線池與 hostConnPool
`policyConnPool` 依據 Session 設定維護每個 Host 的連線池，`SetHosts` 會比對現有主機、建立或關閉 `hostConnPool`，並在背景填滿連線池，確保至少有一條連線可用。【F:connectionpool.go†L54-L170】`hostConnPool` 以 `ConnPicker` 管理實際連線集合，`fill` 會避免重複填補、並行建立多條連線；若建立失敗還會利用 Debouncer 緩衝重試，避免對宕機節點過度打擾。【F:connectionpool.go†L255-L496】`ConnPicker` 預設實作為循環挑選最空閒的連線，同時支援 shard-aware 建立連線所需的 NextShard 介面。【F:connpicker.go†L9-L139】閱讀時可以先看 `policyConnPool.SetHosts` 如何根據拓撲變化新增/刪除池，再進入 `hostConnPool.fill` 了解實際建立流程，最後看 `defaultConnPicker` 如何挑選連線。

## 請求執行與 Streams 管理
`Conn.executeQuery` 會根據 Query 是否需要 Prepare 決定送出 `EXECUTE` 或 `QUERY` frame；對於 Prepared Statement 會驗證參數數量、執行自訂 binding，並在成功後更新 routing metadata。它同時負責寫入 custom payload（例如 tablets 路由資訊）與在迭代器上附加 warnings，最後依照伺服端回應型別決定是否重試或回傳錯誤。【F:conn.go†L1454-L1652】批次執行則由 `Conn.executeBatch` 完成，會逐筆 Prepare 語句、序列化參數後送出 `BATCH` frame，並同樣在結束時觸發 warning 處理。【F:conn.go†L1703-L1780】可搭配 `Conn.AvailableStreams`、`Conn.Close` 等方法理解連線壽命與併發控制。【F:conn.go†L1662-L1701】閱讀這段程式碼能掌握 driver 如何交錯處理 prepare、execute 與錯誤回復。

## 錯誤處理與重連互動
當建立連線或填滿連線池失敗時，`hostConnPool.fillingStopped` 會記錄錯誤、依需求標記節點為 down，並在下一次事件觸發前等待隨機時間以避免雪崩式重試。【F:connectionpool.go†L437-L466】若 `ReconnectInterval` 有設定，Session 會定期呼叫 `pool.addHost` 嘗試把 down 節點重新加入，由連線池自行判斷是否可建立新連線。【F:session.go†L420-L490】了解這些互動有助於除錯：連線數歸零時是否會觸發節點下線？重新連線策略是否與 Session 背景協程互相配合？建議在閱讀 `hostConnPool.connect` 時關注 `ReconnectionPolicy` 的使用方式與 Net 錯誤分類，掌握重試條件。【F:connectionpool.go†L497-L520】 
