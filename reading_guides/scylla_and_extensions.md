# Scylla 專屬擴充功能

## CQL Protocol Extensions
Scylla 在 SUPPORTED/STARTUP frame 中交換多種協定擴充：`TABLETS_ROUTING_V1` 用於傳送 tablets 路由資訊、`SCYLLA_RATE_LIMIT_ERROR` 提供速率限制錯誤代碼、`SCYLLA_LWT_ADD_METADATA_MARK` 則在 Prepared Statement metadata 上標記 LWT 狀態。這些擴充都實作 `cqlProtocolExtension` 介面，負責序列化與反序列化；`parseSupported` 也會同步解析 Shard-aware 設定與分區器資訊。【F:scylla.go†L17-L188】`framer` 在建立時會根據協商結果開啟對應功能，例如記錄 tablets routing 狀態、儲存 LWT bitmask 與 rate limit error code，為後續 frame 處理提供旗標。【F:frame.go†L360-L449】

## 連線層對 Scylla 支援
`Conn` 結構保留 `scyllaSupported` 欄位與 `tabletsRoutingV1` 旗標，透過 `setTabletSupported`／`isTabletSupported` 控制是否啟用 tablets 路由；Session 初始化時會檢查第一條控制連線是否支援此擴充，並將結果寫回 Session 供後續查詢與 metadata 取得使用。【F:conn.go†L218-L238】【F:conn.go†L677-L686】【F:session.go†L307-L334】若 Session 啟用了 tablets routing，`Session.TabletsMetadata` 才會允許取得快取資料，避免在不支援的叢集上誤用。【F:session.go†L689-L700】此外，`Conn.executeQuery` 會在 custom payload 中檢查 `tablets-routing-v1`，解析後寫入 `metadataDescriber` 的 tablets 快取。【F:conn.go†L1557-L1569】【F:metadata_scylla.go†L486-L509】這些步驟顯示 driver 如何在連線層動態啟用 Scylla 特有功能。

## Tablets 與 TokenAware 整合
啟用 tablets routing 後，`TokenAwareHostPolicy` 會在 `Pick` 時優先使用 tablets metadata 決定 replicas，而非僅依 token ring；若找不到對應 tablets，才回退到傳統 token 演算法，並且可以繼續套用避免慢副本等額外策略。【F:policies.go†L751-L804】【F:policies.go†L820-L879】這使得查詢能直接命中擁有對應 tablet 的主機，提高 Scylla tablet 化叢集的效率。

## CDC 與自訂分區器
Scylla CDC 使用專屬的 `CDCPartitioner`，`scylla_cdc.go` 定義了對應的 `scyllaCDCPartitioner`，從 CDC partition key 解析 token，並在 debug 模式下驗證版本與格式。`scyllaIsCdcTable` 會檢查 table metadata 是否使用 CDC partitioner，以決定是否應用此雜湊邏輯。【F:scylla_cdc.go†L11-L95】這個分區器可透過 Query 的 `GetCustomPartitioner` 介面被 TokenAware 策略使用，在讀取 CDC log 時仍能正確推導 replicas。
