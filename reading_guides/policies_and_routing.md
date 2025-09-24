# 節點選擇與策略模組

## Copy-on-write Host 列表
多數策略都依賴 `cowHostList` 儲存節點集合。這個結構透過 `atomic.Value` 搭配互斥鎖，在更新時複製陣列、在讀取時無鎖存取，避免頻繁加鎖影響挑選效能。【F:policies.go†L40-L113】瞭解這個基礎資料結構，有助於閱讀各策略如何維護本地與遠端節點列表。

## HostSelectionPolicy 介面
`HostSelectionPolicy` 定義 Session 初始化、Partitioner 設定、Keyspace 變更與 `Pick` 介面；返回的 `NextHost` 迭代器會在同一查詢的多次嘗試間序列使用，因此策略可以安全地使用內部狀態而不需額外同步。【F:policies.go†L352-L369】預設的 `RoundRobinHostPolicy` 僅依序遍歷 copy-on-write host list，適合無需考量 token 或拓撲的場景。【F:policies.go†L424-L462】閱讀時可先熟悉這個基準行為，再比較其他策略的額外邏輯。

## TokenAwareHostPolicy
`TokenAwareHostPolicy` 結合 fallback 策略與 token ring，依據 routing key 或 tablets 資訊挑選節點。初始化時會綁定 Session 的 metadata 查詢函式，Keyspace 變更或 Partitioners 更新時會重建 `clusterMeta` 內的 token ring 與 replicas 映射。【F:policies.go†L524-L667】`Pick` 會先嘗試從 tablets 路由或 token ring 找到 replicas，再依設定決定是否隨機洗牌、避開忙碌節點、或記錄遠端副本以供 fallback；若 replicas 耗盡，則回退到 fallback 策略，並避免重複節點。【F:policies.go†L724-L879】理解這段邏輯有助於追蹤 token-aware 查詢如何優先選擇本地且健康的節點。

## DCAware 與 RackAware 策略
`DCAwareRoundRobinPolicy` 讓應用優先使用特定資料中心的節點，並可選擇是否允許跨 DC fallback。初始化時會確認目標資料中心存在於目前拓撲中，避免設定錯誤；挑選節點時則以 round-robin 遍歷本地與（必要時）遠端列表。【F:policies.go†L900-L1007】`RackAwareRoundRobinPolicy` 則在 DC-aware 基礎上，進一步依據機架優先順序分層挑選，適合對機架感知的部署。【F:policies.go†L1009-L1038】在需要多層級優先順序時，可參考 `roundRobbin` 如何逐層回退。【F:policies.go†L960-L1006】 

## TokenAware 擴充選項
`TokenAwareHostPolicy` 提供多個選項調整行為：`ShuffleReplicas` 決定是否隨機排列 replicas，`AvoidSlowReplicas` 會依節點 in-flight 數量排序，`NonLocalReplicasFallback` 則控制是否在本地副本耗盡後改用遠端 replicas。這些選項透過函式參數注入策略結構，在 `Pick` 中改變排序與 fallback 行為。【F:policies.go†L463-L506】【F:policies.go†L782-L861】當查詢需要與 Scylla tablets 整合時，也可利用 tablets routing 路徑優先選擇特定 host。【F:policies.go†L751-L760】閱讀這些選項可以了解如何在程式碼中組合策略以符合部署需求。
