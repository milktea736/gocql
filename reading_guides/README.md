# gocql 導讀索引

本資料夾提供閱讀 gocql 原始碼時的導覽指南，依照元件與職責分成多份文件，方便逐步掌握整體架構。建議的閱讀順序如下：

1. [Cluster 與 Session 生命週期](cluster_and_session.md)
2. [連線與連線池管理](connection_management.md)
3. [查詢與執行流程](query_execution.md)
4. [拓撲、節點與中繼資料](metadata_and_topology.md)
5. [節點選擇與策略模組](policies_and_routing.md)
6. [Scylla 專屬擴充功能](scylla_and_extensions.md)
7. [通訊協定與序列化層](protocol_and_serialization.md)

每份文件都包含：
- 該模組的功能與設計原則
- 重要資料結構／流程如何在程式碼中協作
- 建議閱讀程式碼的路徑與注意事項

搭配索引閱讀，可快速掌握驅動程式主要構件與延伸功能的關聯性。
