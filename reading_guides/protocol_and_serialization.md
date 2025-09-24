# 通訊協定與序列化層

## CQL Frame 結構
`frame.go` 定義了 CQL 原生協定的版本、方向、frame operation 與旗標常數，例如 `flagValues`、`flagPageSize`、`flagCustomPayload` 等，並提供 `protoVersion` 與 `frameOp` 的輔助函式，用於判斷封包方向、結果型別與序列化時應帶的旗標。【F:frame.go†L63-L191】了解這些常數能幫助閱讀 `Conn.executeQuery` 時如何設定 frame，以及伺服端回應時如何解讀結果種類。

## Framer 與協定擴充
`framer` 負責讀寫 frame、管理緩衝區與自訂 payload，建立時會根據 compressor 與協定版本決定 header 旗標；若協議支援 Scylla 擴充，還會記錄 LWT bitmask、速率限制錯誤代碼與 tablets routing 旗標，供後續解析回應時使用。【F:frame.go†L360-L449】這層結構與 `Conn.exec`、`Conn.recv` 等方法合作，完成請求/回應的序列化流程。

## 型別系統與 TypeInfo
`marshal.go` 定義了 `TypeInfo` 介面與多種實作：`NativeType` 用於基本型別、`CollectionType` 對應 map/list/set、`TupleTypeInfo` 與 `UDTTypeInfo` 則描述複合型別。每個型別都實作 `NewWithError` 以建立對應 Go 型別，並提供 `String` 呈現人類可讀名稱；`Type` 常數列出 Cassandra 支援的內建型別編號。【F:marshal.go†L1482-L1700】透過這些結構，driver 能在解析 metadata 時推導欄位型別、並於 Marshaling/Unmarshaling 時做適當轉換。

## Marshal/Unmarshal 行為
`Marshal` 會依照 `TypeInfo` 與輸入值決定如何序列化，包括處理 `Marshaler` 介面、指標、自訂型別與多種內建對映（字串、整數、浮點、集合、UDT、Tuple、Duration 等）；也詳細說明 `UnsetValue`、`nil`、空 slice 等特別情境的編碼方式。`Unmarshal` 則反向操作，支援自訂 `Unmarshaler`、結構 tag 解析以及 tuple 展開邏輯。【F:marshal.go†L66-L199】【F:marshal.go†L1482-L1700】在閱讀查詢結果轉換流程或自訂型別時，這段程式碼是核心參考。

## 錯誤型別
`errors.go` 實作了對應 CQL 協定錯誤碼的結構（例如 `RequestErrUnavailable`、`RequestErrReadTimeout`、`RequestErrWriteFailure` 等），每個型別都包含一致性、節點數等診斷資訊，並實作 `Error()` 方法以供上層處理。【F:errors.go†L29-L196】這些型別不僅對應 frame 中的 error code，也提供 RetryPolicy 判斷錯誤性質時使用（例如檢查是否為 `RequestErrUnavailable`）。

## UUID、Duration 等輔助型別
`cqltypes.go` 定義了 `Duration` 等補充型別，並在 `marshal.go` 中提供對應的序列化支援，使得 driver 能直接與 Cassandra 的 duration、UUID、日期/時間等類型互換。【F:cqltypes.go†L1-L30】【F:marshal.go†L1501-L1700】熟悉這些型別有助於編寫自訂結構與欄位時選擇正確的 Go 代表型別。
