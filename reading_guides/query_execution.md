# 查詢與執行流程

## Query 結構與預設值
`Query` 結構保存語句、參數、一致性、分頁、預設時間戳、觀察者、重試與猜測執行策略等資訊；建立時會從 Session 複製預設值（包含一致性、Tracer、Observer、RetryPolicy、DefaultTimestamp 等），並初始化度量資料結構。【F:session.go†L1077-L1170】呼叫端可以透過 `Consistency`／`SetConsistency`、`CustomPayload`、`Trace`、`Observer`、`PageSize`、`DefaultTimestamp`、`RoutingKey`、`WithContext` 等方法調整行為；同時也能設定 `Idempotent`、`SerialConsistency`、`PageState` 與 `NoSkipMetadata` 來控制重試策略、LWT、手動分頁與 metadata 行為。【F:session.go†L1204-L1510】了解這些欄位能幫助閱讀 `executeQuery` 時判斷每個旗標如何影響 frame 建構。

## 執行流程與 queryExecutor
`Query.Iter` 會在需要時自動執行查詢並建立迭代器，若啟用自動分頁會在伺服端回傳空頁時持續以新的 paging state 重試。【F:session.go†L1538-L1560】`Session.executeQuery` 負責檢查 Session 是否關閉或尚未就緒，然後委派給 `queryExecutor.executeQuery`，使所有查詢都走統一的重試／猜測執行流程。【F:session.go†L648-L660】`queryExecutor` 會根據 Query 是否指定 HostID 選擇對應的連線池或透過 HostSelectionPolicy 建立 host iterator；若 Query 被標記為非 idempotent 或未啟用 speculative execution，則直接執行一次；否則會在時間間隔後啟動額外嘗試並在任何一次成功時取消其他 goroutine。【F:query_executor.go†L31-L133】理解這段程式碼可掌握 Query 在多節點、多連線環境下的執行順序。

## 重試與猜測執行策略
RetryPolicy 介面允許根據 Query 嘗試次數與錯誤型別決定是否重試、改用其他節點或直接丟出錯誤；預設的 `SimpleRetryPolicy` 會在非 idempotent 已潛在執行成功時停止，否則改向下一個節點重試，LWT 情境下則偏好在同一節點重試避免 Paxos 爭用。【F:policies.go†L166-L198】SpeculativeExecutionPolicy 介面則控制額外嘗試次數與延遲，預設為 `NonSpeculativeExecution`，也可改用固定間隔的 `SimpleSpeculativeExecution`。【F:policies.go†L1224-L1240】Query 透過 `SetSpeculativeExecutionPolicy`、`RetryPolicy` 與 `Idempotent` 方法調整這些策略，讀程式碼時可對照 `queryExecutor` 如何根據設定啟動 goroutine，理解各種選項的效果。【F:session.go†L1428-L1467】 

## 結果迭代與度量
`Iter` 封裝查詢結果、欄位 metadata、自動分頁狀態與自訂 payload，提供 `Scanner` 介面以逐列讀取並在遇到錯誤時停止；同時也可取得警告、custom payload、paging state 等資訊。【F:session.go†L1701-L1960】`queryMetrics` 會紀錄嘗試次數與延遲，並在 `Query.attempt`／`Batch.attempt` 呼叫時更新；若掛載 Observer，會傳回各節點的統計資料以利監控。【F:session.go†L984-L1059】【F:session.go†L1315-L1333】閱讀這部分可了解 driver 如何將結果與度量緊密結合。

## 批次操作
`Batch` 與 `Query` 結構相似，除了維護語句列表外，也支援設定一致性、RetryPolicy、SpeculativeExecutionPolicy、Tracer、Observer、DefaultTimestamp、SerialConsistency 以及自訂 payload，並能判斷是否為 idempotent 或 LWT 批次。【F:session.go†L2004-L2160】【F:session.go†L2135-L2154】`Conn.executeBatch` 對每個語句執行 Prepare／序列化參數，建立 `BATCH` frame 後送出，並在回應中處理 warnings。【F:conn.go†L1703-L1780】若批次查詢需要 routing key，可使用 `Batch.GetRoutingKey` 觸發 metadata 推導，同樣依賴 `metadataDescriber` 快取。【F:session.go†L2278-L2287】閱讀批次路徑有助於理解 driver 如何重用 Query 邏輯並加入額外語句處理。
