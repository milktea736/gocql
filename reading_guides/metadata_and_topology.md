# 拓撲、節點與中繼資料

## HostInfo 與節點狀態
`HostInfo` 封裝節點的連線位址、資料中心、rack、HostID、版本、分區器、shard aware port 等資訊，並以讀寫鎖保護欄位；同檔案也定義了 `cassVersion` 與節點狀態常數，提供版本比較與節點上/下線判斷。【F:host_source.go†L37-L200】閱讀 `HostInfo` 的 Getter/Setter 有助於理解 Session 如何在事件處理、連線池與拓撲策略間共享節點資訊。

## HostSource 與節點管理
`hostSource` 負責維護所有已知節點（以 HostID 與 IP 映射），提供查詢、加入、移除、標記狀態等功能；同時提供 `fetchHostInfoAndUpdate` 透過控制連線更新節點資料，並支援 AddressTranslator 與 Peer 資料驗證。【F:host_source.go†L201-L640】當 Session 透過事件接收到拓撲或狀態變動時，會呼叫這些方法更新內部狀態，因此閱讀 hostSource 能掌握節點生命周期如何被追蹤。

## RingDescriber 與 system.peers 查詢
`ringDescriber` 使用控制連線輪詢 `system.local` 與 `system.peers`，解析得到的資料後產生 `HostInfo` 列表與 Partitioners，並記錄前一次結果以供 fallback；同時維護 HostID 與 IP 的映射供事件處理快速查詢。【F:ring_describer.go†L11-L199】`GetHostsFromSystem`、`getClusterPeerInfo`、`getLocalHostInfo` 是理解節點探索流程的核心，建議沿著這三個函式閱讀如何從查詢結果轉換為 HostInfo。

## MetadataDescriber 與 schema 快取
`metadataDescriber` 將 schema metadata 與 tablets metadata 快取在 `Metadata` 結構中，提供 `getSchema`、`AddTablet`、`RemoveTabletsWithKeyspace/Table` 等方法；若 cache 未命中，會呼叫 `refreshSchema` 重新查詢 system tables。`refreshAllSchema` 會遍歷既有 keyspace，比對策略與 table 差異，必要時清除 tablets 快取，確保 schema 變動後的路由資訊一致。【F:metadata_scylla.go†L442-L567】閱讀時可留意 mutex 的使用方式與 tablets 快取的寫入/清除時機。

## Token 與拓撲策略
`tokenRing` 將 `HostInfo` 的 token 列表整理成排序後的 ring，並依分區器類型（Murmur3、Ordered、Random）解析 token。【F:token.go†L43-L199】`topology.go` 內的 `placementStrategy`（`simpleStrategy`、`networkTopology`）會根據 keyspace 的策略選擇副本，生成 `tokenRingReplicas` 映射，用於 TokenAware 策略挑選節點。【F:topology.go†L34-L181】理解這些結構可幫助追蹤 TokenAwareHostPolicy 如何選擇目標節點。

## Tablets 路由資訊
`tablets` 套件定義 `TabletInfo`、`TabletInfoList` 與相關建構/查詢方法，能夠維持排序列表、插入新 tablets、移除重疊區間並查詢特定 keyspace/table 的範圍。【F:tablets/tablets.go†L1-L195】`metadataDescriber` 會透過這些 API 快取 Scylla tablets，供 policies 或 query routing 使用，因此理解 tablets 資料結構有助於分析 Scylla 特有的路由邏輯。
