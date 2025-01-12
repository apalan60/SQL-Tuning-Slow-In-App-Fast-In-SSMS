#

## Problem definition

- DTU > 90%
- Query Timeout > 30s (查詢識別碼23707,23732,24073,prd)

> 重複執行(<20次)F006 API[產出的SQL](../F006User回報0108.sql)，可以明顯觀察到API飆高，消耗完後會下降
> ![alt text](image-2.png)

### 於local搭配prfiler模擬並取得高資源消耗的前幾筆SQL

- 模擬情境
  - ENV: TST
  - from local to tst db
  - response time: 24.29 s

- request payload

  {{APIUrl}}/api/bpm/Folder/F006?SortOrder=&Desc=false

  ```json
  {
    "filterApplyUnitCode": [],
    "filterApplyEmpCode": [],
    "filterDateTimeRange": [],
    "filterFormTypeId": [],
    "filterFormNo": [],
    "filterFlowState": [],
    "filterCurrentApprovalEmpCode": [],
    "containSubordinateUnits": false,
    "pageModel": {
        "pageCurrent": 1,
        "pageSize": 100
    }
  }
  ```

- SQL Events

| Event Sequence | EventClass     | StartTime              | TextData                                                                                                               | CPU(ms)      | Duration(ms) | Reads   | Writes | Row Count | HostName     | LoginName          | DatabaseID | ClientProcessID | ApplicationName                         |
|----------------|----------------|------------------------|------------------------------------------------------------------------------------------------------------------------|----------|----------|---------|--------|-----------|--------------|--------------------|------------|-----------------|------------------------------------------|
| 19094          | rpc_completed  | 2025-01-08T03:33:13.465Z | `exec sp_executesql N'SELECT [u2].[MainFlowRecordId], [u2].[FormObjectId]...`                                           | **703000**   | 699801   | 173367  | 0      | 593       | A2-2023-062 | mayoform_apiuser   | 6          | 27208           | Core Microsoft SqlClient Data Provider  |
| 19089          | rpc_completed  | 2025-01-08T03:33:12.700Z | `exec sp_executesql N'SELECT COUNT(*) FROM ( SELECT 1 AS empty FROM...`                                                 | **250000**   | 269496   | 47094   | 0      | 1         | A2-2023-062 | mayoform_apiuser   | 6          | 27208           | Core Microsoft SqlClient Data Provider  |
| 19071          | rpc_completed  | 2025-01-08T03:33:11.844Z | `exec sp_executesql N'SELECT [f].[MainFlowRecordId] FROM [FlowRecordObjs] AS [f] WHERE ...`                             | **125000**   | 122101   | 73882   | 0      | 1         | A2-2023-062 | mayoform_apiuser   | 6          | 27208           | Core Microsoft SqlClient Data Provider  |

---

## Tuning

> 目標: 查詢識別碼23707,23732,24073的SQL，平均CPU使用時間(22000ms/ 4514ms/ 3696ms)要降到1000ms以下，以緩解DTU飆高的情況
>
> *Note: 因為無法在Prd使用的Db上快速更改Index，故在TST測試。
> 但在Index各環境一致的前提下，前後Index的調整若能帶來效能差異，應該要能反映於各環境*
>
> 執行步驟
>
> 1. TST建立與Prd相同的Index，取得前側
> 2. 

### TST

#### Before

#### After

### Init

只保留Clustered Index, 筆數約50000筆

| Operation              | Object                                                                | Estimated Cost % | Estimated Subtree Cost | Estimated Rows | Actual Rows | Average Row Size | Actual Executions | Estimated Executions | Estimated CPU Cost | Actual CPU Cost | Estimated IO Cost | Estimated Data Size | Actual Data Size | Parallel | Ordered | Actual Rewinds | Estimated Rewinds | Actual Rebinds | Estimated Rebinds |
|------------------------|----------------------------------------------------------------------|------------------|------------------------|----------------|-------------|------------------|-------------------|----------------------|--------------------|----------------|-------------------|---------------------|------------------|----------|---------|----------------|-------------------|----------------|-------------------|
| Clustered Index Scan   | [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].[PK_FlowStatesRecord] [f14] | 28.12            | 1.5                    | 61             | 6899100     | 48               | 156               | 1                    | 0.077663          | 838            | 1.46831           | 2940.9168          | 331156800        | false    | false   | 0              | 0                 | 0              | 0                 |
| Clustered Index Scan   | [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].[PK_FlowStatesRecord] [f8]  | 28.12            | 1.5                    | 61             | 4422500     | 36               | 100               | 1                    | 0.077663          | 377            | 1.46831           | 2205.6876          | 159210000        | false    | false   | 0              | 0                 | 0              | 0                 |
| Clustered Index Scan   | [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].[PK_FlowStatesRecord] [f1]  | 28.12            | 1.5                    | 61             | 44225       | 36               | 1                 | 1                    | 0.077663          | 41             | 1.46831           | 2205.6876          | 1592100          | false    | false   | 0              | 0                 | 0              | 0                 |

Total execution time: 00:01:59.305

## Prd - before

有Non-Clustered Index, 筆數約80000筆

Total execution time: 00:00:05.849

- DTU 1/8 12:44~12:50 DTU飆高到90%
- 此時對應的SQL如下 ![CPU](image-3.png)

| 查詢識別碼 | 物件識別碼 | 物件名稱 | 查詢 SQL 文字 | 平均值 CPU 時間 | 執行計數 | 計畫計數 |
|------------|------------|----------|----------------|---------------|---------|----------|
| 23707      | 0          |          | (@__appUserId_0 bigint,@__tenantId_1 bigint,@__filterCreatedTimeStart_Value_2 datetimeoffset(7),@__filterCreatedTimeEnd_Value_3 datetimeoffset(7),@__noFormNoStates_4 nvarchar(4000),@__hasFormNoIncludeStates_5 nvarchar(4000),@__hasFormNoExcludeStates_6 nvarchar(4000),@__8__locals1_executingRevokingFlowIds_7 nvarchar(4000),@__p_9 int,@__p_8 int) SELECT [s5].[MainFlowRecordId], [s5].[FormObjectId], [s5].[FormNo], [s5].[CreatedTime], [s5].[LatestUpdatedTime], [s5].[ProcessDay], [s5].[c], [s5].[ParentFlowRecordObjId], [s5].[IsArchived], [s6].[Id0], [s5].[c0], [s7].[ToEmployeeCode], [s5].[c1] FROM ( ... ) | 22000.34     | 3       | 1        |
| 23732      | 0          |          | (@__appUserId_0 bigint,@__tenantId_1 bigint,@__filterFormNos_2 nvarchar(4000),@__noFormNoStates_3 nvarchar(4000),@__hasFormNoIncludeStates_4 nvarchar(4000),@__hasFormNoExcludeStates_5 nvarchar(4000),@__8__locals1_executingRevokingFlowIds_6 nvarchar(4000),@__p_8 int,@__p_7 int) SELECT [s5].[MainFlowRecordId], [s5].[FormObjectId], [s5].[FormNo], [s5].[CreatedTime], [s5].[LatestUpdatedTime], [s5].[ProcessDay], [s5].[c], [s5].[ParentFlowRecordObjId], [s5].[IsArchived], [s6].[Id0], [s5].[c0], [s7].[ToEmployeeCode], [s5].[c1] FROM ( ... ) | 4514.09      | 5       | 1        |
| 24073      | 0          |          | (@__appUserId_0 bigint,@__tenantId_1 bigint,@__filterFormNos_2 nvarchar(4000),@__noFormNoStates_3 nvarchar(4000),@__hasFormNoIncludeStates_4 nvarchar(4000),@__hasFormNoExcludeStates_5 nvarchar(4000),@__8__locals1_executingRevokingFlowIds_6 nvarchar(4000),@__p_8 int,@__p_7 int) SELECT [s5].[MainFlowRecordId], [s5].[FormObjectId], [s5].[FormNo], [s5].[CreatedTime], [s5].[LatestUpdatedTime], [s5].[ProcessDay], [s5].[c], [s5].[ParentFlowRecordObjId], [s5].[IsArchived], [s6].[Id0], [s5].[c0], [s7].[ToEmployeeCode], [s5].[c1] FROM ( ... ) | 3696.38      | 6       | 1        |

## Prd - after

temp

## Index column v.s. Include column

StateFlag分布

| Output List                                                                                              | Reference                                                                                         |
|---------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].StateFlag, [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].StateType |                                                                                                   |
| [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].StateFlag                                         |                                                                                                   |
| [tstMAYOFormSharedDB].[dbo].[FlowStatesRecordObjs].StateType                                         |                                                                                                   |

## Note

### Query Plan

Property -> Output list 看到該操作(e.g. seek、lookup)在找那些值
![alt text](image.png)

Property -> Object 可看到該操作有沒有使用index
![alt text](image-1.png)

右鍵 -> Highlight expensive operation -> 可根據Metric類型快速篩出可能有問題或待優化的操作

- Metric: Actual Elapsed Time
這是最直接的量測：實際等候該查詢跑完的時間。若要優化最終使用者的體驗，這是你最該關注的時間。
但僅看 Elapsed Time 有時不夠，需要再深入了解 CPU 與等待 (wait) 時間的分佈。

- estimated number of rows 預計該次運算返回多少row，編譯時在查詢優化器查執行計畫時計算的，其值根據統計數據而來

### Top operations

- Estimated Cost 只是優化器的「演算模型」；不一定代表實際上會花費多少資源或時間。當統計資料 (Statistics) 不準確時，可能會導致誤判與估計偏差。

### 完整tuning歷程

https://chatgpt.com/share/677d1711-f4d8-800d-8799-929279229f38

### 索引選擇

- 可以考慮將最常被使用來「過濾」或「JOIN」的欄位放到索引鍵 (Key columns(B tree))，其他只是偶爾需要 SELECT 的欄位放到 INCLUDE(is not B tree)

- 「WHERE DeletedTime IS NULL」是 Filtered Index 的用法，若幾乎每次查詢都只要 DeletedTime IS NULL 的資料，可大幅縮小索引範圍、減少維護成本。

```sql
CREATE NONCLUSTERED INDEX [IX_FlowRecordObjs_AppUser_Tenant_CreatedTime]
ON FlowRecordObjs(
    AppUserId,
    TenantId,
    CreatedTime
)
INCLUDE(
    CurrentFlowStateCode,
    FormNo,
    BranchSerialId,
    MainFlowRecordId
)
WHERE DeletedTime IS NULL --Filtered Index
```

- 欄位常是 <> 或 NOT IN 或 IS NULL 這種條件，索引優化器在實務上未必會使用 Seek(或容易退化為 Range Scan)，需實際確認效能

## 完整profiler對應SQL

```SQL
---prd---
-- 請求參數:[{"pageModel":{"pageCurrent":1,"pageSize":30},"filterDateTimeRange":["2024-12-08T16:00:00.000Z","2025-01-09T15:59:59.998Z"]}]
-- 從local測試該版本錄到的F006參數
-- @__appUserId_0=590709188939776,@__tenantId_1=590709191925764 --需替換,
-- @__noFormNoStates_2=N'["PG0","PG1"]',
-- @__hasFormNoIncludeStates_3=N'["P1"]',
-- @__hasFormNoExcludeStates_4=N'["P2"]',
-- @__8__locals1_executingRevokingFlowIds_5=N'[6749830704793609]' --需替換,
-- @__p_6=0,
-- @__p_7=100

-- 會先查出撤銷流程
sp_executesql N'SELECT [f].[MainFlowRecordId]
FROM [FlowRecordObjs] AS [f]
WHERE [f].[TenantId] = @___tenantId_0 AND [f].[AppUserId] = @___appUserId_1 AND [f].[ParentFlowRecordObjId] IS NOT NULL AND [f].[FlowFlag] = 1',N'@___tenantId_0 bigint,@___appUserId_1 bigint',@___tenantId_0=6395217529958400,@___appUserId_1=6070292994734084


DECLARE @__appUserId_0 bigint = 6070292994734084; 
DECLARE @__tenantId_1 bigint = 6395217529958400;
DECLARE @__filterCreatedTimeStart_Value_2 datetimeoffset(7) = '2024-12-08T16:00:00.000Z';
DECLARE @__filterCreatedTimeEnd_Value_3 datetimeoffset(7) = '2025-01-09T15:59:59.998Z';
DECLARE @__noFormNoStates_4 nvarchar(4000) = N'["PG0","PG1"]'
DECLARE @__hasFormNoIncludeStates_5 nvarchar(4000) = N'["P1"]'
DECLARE @__hasFormNoExcludeStates_6 nvarchar(4000) = N'["P2"]'
DECLARE @__8__locals1_executingRevokingFlowIds_7 nvarchar(4000) = NULL; -- 這個來自撤銷查詢
DECLARE @__p_8 int = (1 - 1) * 30; -- 計算 Offset 值
DECLARE @__p_9 int = 100; -- 每頁大小


SELECT [s5].[MainFlowRecordId], 
       [s5].[FormObjectId], 
       [s5].[FormNo], 
       [s5].[CreatedTime], 
       [s5].[LatestUpdatedTime], 
       [s5].[ProcessDay], 
       [s5].[c], 
       [s5].[ParentFlowRecordObjId], 
       [s5].[IsArchived], 
       [s6].[Id0], 
       [s5].[c0], 
       [s7].[ToEmployeeCode], 
       [s5].[c1]
FROM (
    SELECT [s0].[MainFlowRecordId],
           [s0].[FormObjectId],
           [s0].[FormNo],
           [s0].[CreatedTime],
           [s0].[LatestUpdatedTime],
           [s0].[ProcessDay],
           MAX([s0].[LatestUpdatedTime0]) AS [c],
           COUNT(*) AS [c0],
           CASE WHEN [s0].[ParentFlowRecordObjId] IS NOT NULL THEN CAST(1 AS bit)
                ELSE CAST(0 AS bit)
           END AS [c1],
           [s0].[ParentFlowRecordObjId],
           [s0].[IsArchived]
    FROM (
        SELECT [f].[CreatedTime],
               [f].[FormNo],
               [f].[FormObjectId],
               [f].[LatestUpdatedTime],
               [f].[MainFlowRecordId],
               [f].[ParentFlowRecordObjId],
               [f].[ProcessDay],
               [s].[LatestUpdatedTime] AS [LatestUpdatedTime0],
               CAST(0 AS bit) AS [IsArchived]
        FROM [FlowRecordObjs] AS [f]
        INNER JOIN (
            SELECT [f0].[Id],
                   [f0].[LatestUpdatedTime],
                   [f0].[MainFlowRecordId]
            FROM [FlowRecordObjs] AS [f0]
            INNER JOIN (
                SELECT [f1].[FlowRecordObjsId],
                       [f1].[StateFlag],
                       [f1].[StateType]
                FROM [FlowStatesRecordObjs] AS [f1]
                WHERE [f1].[AppUserId] = @__appUserId_0
                  AND [f1].[TenantId] = @__tenantId_1
            ) AS [f2] ON [f0].[Id] = [f2].[FlowRecordObjsId]
            WHERE [f0].[AppUserId] = @__appUserId_0
              AND [f0].[TenantId] = @__tenantId_1
              AND [f0].[DeletedTime] IS NULL
              AND [f0].[FormObjectId] IS NOT NULL
              AND [f0].[CreatedTime] >= @__filterCreatedTimeStart_Value_2
              AND [f0].[CreatedTime] <= @__filterCreatedTimeEnd_Value_3
              AND (
                  (
                      [f0].[FormNo] IS NULL
                      AND [f0].[CurrentFlowStateCode] IN (
                          SELECT [n].[value]
                          FROM OPENJSON(@__noFormNoStates_4) WITH ([value] nvarchar(50) '$') AS [n]
                      )
                  )
                  OR (
                      [f0].[FormNo] IS NOT NULL
                      AND (
                           [f0].[CurrentFlowStateCode] IN (
                               SELECT [h].[value]
                               FROM OPENJSON(@__hasFormNoIncludeStates_5) WITH ([value] nvarchar(50) '$') AS [h]
                           )
                           OR [f0].[CurrentFlowStateCode] NOT IN (
                               SELECT [h0].[value]
                               FROM OPENJSON(@__hasFormNoExcludeStates_6) WITH ([value] nvarchar(50) '$') AS [h0]
                           )
                           OR [f0].[CurrentFlowStateCode] IS NULL
                      )
                  )
              )
              AND [f0].[CurrentFlowStateCode] IS NOT NULL
              AND [f0].[ParentFlowRecordObjId] IS NULL
              AND [f0].[Id] NOT IN (
                  SELECT [f3].[Id]
                  FROM [FlowRecordObjs] AS [f3]
                  WHERE [f3].[AppUserId] = @__appUserId_0
                    AND [f3].[TenantId] = @__tenantId_1
                    AND [f3].[MainFlowRecordId] IN (
                        SELECT [f4].[Id]
                        FROM [FlowRecordObjs] AS [f4]
                        WHERE [f4].[AppUserId] = @__appUserId_0
                          AND [f4].[TenantId] = @__tenantId_1
                          AND [f4].[ParentFlowRecordObjId] IS NOT NULL
                    )
              )
              AND [f0].[MainFlowRecordId] NOT IN (
                  SELECT [8].[value]
                  FROM OPENJSON(@__8__locals1_executingRevokingFlowIds_7) WITH ([value] bigint '$') AS [8]
              )
              AND [f0].[CurrentFlowStateCode] <> N'P2'
              AND [f0].[FlowFlag] <> 3
              AND (
                  ([f0].[CurrentFlowStateCode] = N'P1' AND [f0].[FormNo] IS NOT NULL AND [f0].[FormNo] NOT LIKE N'')
                  OR (
                      [f0].[CurrentFlowStateCode] = N'P9'
                      AND [f0].[BranchSerialId] IS NOT NULL
                      AND [f2].[StateFlag] = 5
                      AND [f2].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      [f0].[CurrentFlowStateCode] = N'P5'
                      AND [f0].[BranchSerialId] IS NOT NULL
                      AND [f0].[BranchSerialId] = 0
                  )
                  OR (
                      [f0].[CurrentFlowStateCode] = N'P6'
                      AND [f0].[BranchSerialId] IS NOT NULL
                      AND [f0].[BranchSerialId] = 0
                  )
                  OR [f0].[CurrentFlowStateCode] = N'R2'
                  OR [f0].[CurrentFlowStateCode] = N'R3'
                  OR (
                      [f0].[CurrentFlowStateCode] = N'E1'
                      AND [f2].[StateFlag] = 1
                      AND [f2].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      [f0].[CurrentFlowStateCode] IN (N'P3', N'P4')
                      AND [f2].[StateFlag] = 1
                      AND [f2].[StateType] = CAST(3 AS TINYINT)
                  )
              )
        ) AS [s] ON [f].[Id] = CASE
                                  WHEN [s].[MainFlowRecordId] = CAST(0 AS bigint) THEN [s].[Id]
                                  ELSE [s].[MainFlowRecordId]
                              END
    ) AS [s0]
    GROUP BY 
         [s0].[MainFlowRecordId],
         [s0].[FormObjectId],
         [s0].[FormNo],
         [s0].[CreatedTime],
         [s0].[LatestUpdatedTime],
         [s0].[ProcessDay],
         [s0].[ParentFlowRecordObjId],
         [s0].[IsArchived]
    ORDER BY [s0].[LatestUpdatedTime] DESC
    OFFSET @__p_8 ROWS FETCH NEXT @__p_9 ROWS ONLY
) AS [s5]
OUTER APPLY (
    SELECT DISTINCT [s1].[Id0]
    FROM (
        SELECT [f5].[CreatedTime],
               [f5].[FormNo],
               [f5].[FormObjectId],
               [f5].[LatestUpdatedTime],
               [f5].[MainFlowRecordId],
               [f5].[ParentFlowRecordObjId],
               [f5].[ProcessDay],
               [s2].[Id] AS [Id0],
               CAST(0 AS bit) AS [IsArchived]
        FROM [FlowRecordObjs] AS [f5]
        INNER JOIN (
            SELECT [f6].[Id],
                   [f6].[MainFlowRecordId]
            FROM [FlowRecordObjs] AS [f6]
            INNER JOIN (
                SELECT [f8].[FlowRecordObjsId],
                       [f8].[StateFlag],
                       [f8].[StateType]
                FROM [FlowStatesRecordObjs] AS [f8]
                WHERE [f8].[AppUserId] = @__appUserId_0
                  AND [f8].[TenantId] = @__tenantId_1
            ) AS [f7] ON [f6].[Id] = [f7].[FlowRecordObjsId]
            WHERE [f6].[AppUserId] = @__appUserId_0
              AND [f6].[TenantId] = @__tenantId_1
              AND [f6].[DeletedTime] IS NULL
              AND [f6].[FormObjectId] IS NOT NULL
              AND [f6].[CreatedTime] >= @__filterCreatedTimeStart_Value_2
              AND [f6].[CreatedTime] <= @__filterCreatedTimeEnd_Value_3
              AND (
                  (
                      [f6].[FormNo] IS NULL
                      AND [f6].[CurrentFlowStateCode] IN (
                          SELECT [n0].[value]
                          FROM OPENJSON(@__noFormNoStates_4) WITH ([value] nvarchar(50) '$') AS [n0]
                      )
                  )
                  OR (
                      [f6].[FormNo] IS NOT NULL
                      AND (
                          [f6].[CurrentFlowStateCode] IN (
                              SELECT [h1].[value]
                              FROM OPENJSON(@__hasFormNoIncludeStates_5) WITH ([value] nvarchar(50) '$') AS [h1]
                          )
                          OR [f6].[CurrentFlowStateCode] NOT IN (
                              SELECT [h2].[value]
                              FROM OPENJSON(@__hasFormNoExcludeStates_6) WITH ([value] nvarchar(50) '$') AS [h2]
                          )
                          OR [f6].[CurrentFlowStateCode] IS NULL
                      )
                  )
              )
              AND [f6].[CurrentFlowStateCode] IS NOT NULL
              AND [f6].[ParentFlowRecordObjId] IS NULL
              AND [f6].[Id] NOT IN (
                  SELECT [f9].[Id]
                  FROM [FlowRecordObjs] AS [f9]
                  WHERE [f9].[AppUserId] = @__appUserId_0
                    AND [f9].[TenantId] = @__tenantId_1
                    AND [f9].[MainFlowRecordId] IN (
                        SELECT [f10].[Id]
                        FROM [FlowRecordObjs] AS [f10]
                        WHERE [f10].[AppUserId] = @__appUserId_0
                          AND [f10].[TenantId] = @__tenantId_1
                          AND [f10].[ParentFlowRecordObjId] IS NOT NULL
                    )
              )
              AND [f6].[MainFlowRecordId] NOT IN (
                  SELECT [80].[value]
                  FROM OPENJSON(@__8__locals1_executingRevokingFlowIds_7) WITH ([value] bigint '$') AS [80]
              )
              AND [f6].[CurrentFlowStateCode] <> N'P2'
              AND [f6].[FlowFlag] <> 3
              AND (
                  ([f6].[CurrentFlowStateCode] = N'P1' AND [f6].[FormNo] IS NOT NULL AND [f6].[FormNo] NOT LIKE N'')
                  OR (
                      [f6].[CurrentFlowStateCode] = N'P9'
                      AND [f6].[BranchSerialId] IS NOT NULL
                      AND [f7].[StateFlag] = 5
                      AND [f7].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      [f6].[CurrentFlowStateCode] = N'P5'
                      AND [f6].[BranchSerialId] IS NOT NULL
                      AND [f6].[BranchSerialId] = 0
                  )
                  OR (
                      [f6].[CurrentFlowStateCode] = N'P6'
                      AND [f6].[BranchSerialId] IS NOT NULL
                      AND [f6].[BranchSerialId] = 0
                  )
                  OR [f6].[CurrentFlowStateCode] = N'R2'
                  OR [f6].[CurrentFlowStateCode] = N'R3'
                  OR (
                      [f6].[CurrentFlowStateCode] = N'E1'
                      AND [f7].[StateFlag] = 1
                      AND [f7].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      ([f6].[CurrentFlowStateCode] = N'P3' OR [f6].[CurrentFlowStateCode] = N'P4')
                      AND [f7].[StateFlag] = 1
                      AND [f7].[StateType] = CAST(3 AS TINYINT)
                  )
              )
        ) AS [s2] ON [f5].[Id] = CASE
                                   WHEN [s2].[MainFlowRecordId] = CAST(0 AS bigint) THEN [s2].[Id]
                                   ELSE [s2].[MainFlowRecordId]
                               END
    ) AS [s1]
    WHERE [s5].[MainFlowRecordId] = [s1].[MainFlowRecordId]
      AND (
          [s5].[FormObjectId] = [s1].[FormObjectId]
          OR ([s5].[FormObjectId] IS NULL AND [s1].[FormObjectId] IS NULL)
      )
      AND (
          [s5].[FormNo] = [s1].[FormNo]
          OR ([s5].[FormNo] IS NULL AND [s1].[FormNo] IS NULL)
      )
      AND [s5].[CreatedTime] = [s1].[CreatedTime]
      AND (
          [s5].[LatestUpdatedTime] = [s1].[LatestUpdatedTime]
          OR ([s5].[LatestUpdatedTime] IS NULL AND [s1].[LatestUpdatedTime] IS NULL)
      )
      AND (
          [s5].[ProcessDay] = [s1].[ProcessDay]
          OR ([s5].[ProcessDay] IS NULL AND [s1].[ProcessDay] IS NULL)
      )
      AND (
          [s5].[ParentFlowRecordObjId] = [s1].[ParentFlowRecordObjId]
          OR ([s5].[ParentFlowRecordObjId] IS NULL AND [s1].[ParentFlowRecordObjId] IS NULL)
      )
      AND [s5].[IsArchived] = [s1].[IsArchived]
) AS [s6]
OUTER APPLY (
    SELECT DISTINCT [s3].[ToEmployeeCode]
    FROM (
        SELECT [f11].[CreatedTime],
               [f11].[FormNo],
               [f11].[FormObjectId],
               [f11].[LatestUpdatedTime],
               [f11].[MainFlowRecordId],
               [f11].[ParentFlowRecordObjId],
               [f11].[ProcessDay],
               [s4].[ToEmployeeCode],
               CAST(0 AS bit) AS [IsArchived]
        FROM [FlowRecordObjs] AS [f11]
        INNER JOIN (
            SELECT [f12].[Id],
                   [f12].[MainFlowRecordId],
                   [f13].[ToEmployeeCode]
            FROM [FlowRecordObjs] AS [f12]
            INNER JOIN (
                SELECT [f14].[FlowRecordObjsId],
                       [f14].[StateFlag],
                       [f14].[StateType],
                       [f14].[ToEmployeeCode]
                FROM [FlowStatesRecordObjs] AS [f14]
                WHERE [f14].[AppUserId] = @__appUserId_0
                  AND [f14].[TenantId] = @__tenantId_1
            ) AS [f13] ON [f12].[Id] = [f13].[FlowRecordObjsId]
            WHERE [f12].[AppUserId] = @__appUserId_0
              AND [f12].[TenantId] = @__tenantId_1
              AND [f12].[DeletedTime] IS NULL
              AND [f12].[FormObjectId] IS NOT NULL
              AND [f12].[CreatedTime] >= @__filterCreatedTimeStart_Value_2
              AND [f12].[CreatedTime] <= @__filterCreatedTimeEnd_Value_3
              AND (
                  (
                      [f12].[FormNo] IS NULL
                      AND [f12].[CurrentFlowStateCode] IN (
                          SELECT [n1].[value]
                          FROM OPENJSON(@__noFormNoStates_4) WITH ([value] nvarchar(50) '$') AS [n1]
                      )
                  )
                  OR (
                      [f12].[FormNo] IS NOT NULL
                      AND (
                          [f12].[CurrentFlowStateCode] IN (
                              SELECT [h3].[value]
                              FROM OPENJSON(@__hasFormNoIncludeStates_5) WITH ([value] nvarchar(50) '$') AS [h3]
                          )
                          OR [f12].[CurrentFlowStateCode] NOT IN (
                              SELECT [h4].[value]
                              FROM OPENJSON(@__hasFormNoExcludeStates_6) WITH ([value] nvarchar(50) '$') AS [h4]
                          )
                          OR [f12].[CurrentFlowStateCode] IS NULL
                      )
                  )
              )
              AND [f12].[CurrentFlowStateCode] IS NOT NULL
              AND [f12].[ParentFlowRecordObjId] IS NULL
              AND [f12].[Id] NOT IN (
                  SELECT [f15].[Id]
                  FROM [FlowRecordObjs] AS [f15]
                  WHERE [f15].[AppUserId] = @__appUserId_0
                    AND [f15].[TenantId] = @__tenantId_1
                    AND [f15].[MainFlowRecordId] IN (
                        SELECT [f16].[Id]
                        FROM [FlowRecordObjs] AS [f16]
                        WHERE [f16].[AppUserId] = @__appUserId_0
                          AND [f16].[TenantId] = @__tenantId_1
                          AND [f16].[ParentFlowRecordObjId] IS NOT NULL
                    )
              )
              AND [f12].[MainFlowRecordId] NOT IN (
                  SELECT [81].[value]
                  FROM OPENJSON(@__8__locals1_executingRevokingFlowIds_7) WITH ([value] bigint '$') AS [81]
              )
              AND [f12].[CurrentFlowStateCode] <> N'P2'
              AND [f12].[FlowFlag] <> 3
              AND (
                  ([f12].[CurrentFlowStateCode] = N'P1' AND [f12].[FormNo] IS NOT NULL AND [f12].[FormNo] NOT LIKE N'')
                  OR (
                      [f12].[CurrentFlowStateCode] = N'P9'
                      AND [f12].[BranchSerialId] IS NOT NULL
                      AND [f13].[StateFlag] = 5
                      AND [f13].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      [f12].[CurrentFlowStateCode] = N'P5'
                      AND [f12].[BranchSerialId] IS NOT NULL
                      AND [f12].[BranchSerialId] = 0
                  )
                  OR (
                      [f12].[CurrentFlowStateCode] = N'P6'
                      AND [f12].[BranchSerialId] IS NOT NULL
                      AND [f12].[BranchSerialId] = 0
                  )
                  OR [f12].[CurrentFlowStateCode] = N'R2'
                  OR [f12].[CurrentFlowStateCode] = N'R3'
                  OR (
                      [f12].[CurrentFlowStateCode] = N'E1'
                      AND [f13].[StateFlag] = 1
                      AND [f13].[StateType] = CAST(3 AS TINYINT)
                  )
                  OR (
                      ([f12].[CurrentFlowStateCode] = N'P3' OR [f12].[CurrentFlowStateCode] = N'P4')
                      AND [f13].[StateFlag] = 1
                      AND [f13].[StateType] = CAST(3 AS TINYINT)
                  )
              )
        ) AS [s4] ON [f11].[Id] = CASE
                                    WHEN [s4].[MainFlowRecordId] = CAST(0 AS bigint) THEN [s4].[Id]
                                    ELSE [s4].[MainFlowRecordId]
                                END
    ) AS [s3]
    WHERE [s5].[MainFlowRecordId] = [s3].[MainFlowRecordId]
      AND (
          [s5].[FormObjectId] = [s3].[FormObjectId]
          OR ([s5].[FormObjectId] IS NULL AND [s3].[FormObjectId] IS NULL)
      )
      AND (
          [s5].[FormNo] = [s3].[FormNo]
          OR ([s5].[FormNo] IS NULL AND [s3].[FormNo] IS NULL)
      )
      AND [s5].[CreatedTime] = [s3].[CreatedTime]
      AND (
          [s5].[LatestUpdatedTime] = [s3].[LatestUpdatedTime]
          OR ([s5].[LatestUpdatedTime] IS NULL AND [s3].[LatestUpdatedTime] IS NULL)
      )
      AND (
          [s5].[ProcessDay] = [s3].[ProcessDay]
          OR ([s5].[ProcessDay] IS NULL AND [s3].[ProcessDay] IS NULL)
      )
      AND (
          [s5].[ParentFlowRecordObjId] = [s3].[ParentFlowRecordObjId]
          OR ([s5].[ParentFlowRecordObjId] IS NULL AND [s3].[ParentFlowRecordObjId] IS NULL)
      )
      AND [s5].[IsArchived] = [s3].[IsArchived]
) AS [s7]
ORDER BY [s5].[LatestUpdatedTime] DESC, 
         [s5].[MainFlowRecordId], 
         [s5].[FormObjectId], 
         [s5].[FormNo], 
         [s5].[CreatedTime], 
         [s5].[ProcessDay], 
         [s5].[ParentFlowRecordObjId], 
         [s5].[IsArchived], 
         [s6].[Id0];



-- 計算筆數
exec sp_executesql N'SELECT COUNT(*)
FROM (
    SELECT 1 AS empty
    FROM (
        SELECT [f].[FormObjectId], [f].[FormNo], [f].[ProcessDay], [f].[MainFlowRecordId], [f].[CreatedTime], [f].[LatestUpdatedTime], [f].[ParentFlowRecordObjId], CAST(0 AS bit) AS [IsArchived], [s].[LatestUpdatedTime] AS [SubFlowLatestUpdatedTime], [s].[Id] AS [SubFlowId], [s].[ToEmployeeCode]
        FROM [FlowRecordObjs] AS [f]
        INNER JOIN (
            SELECT [f0].[Id], [f0].[LatestUpdatedTime], [f0].[MainFlowRecordId], [f2].[ToEmployeeCode]
            FROM [FlowRecordObjs] AS [f0]
            INNER JOIN (
                SELECT [f1].[FlowRecordObjsId], [f1].[StateFlag], [f1].[StateType], [f1].[ToEmployeeCode]
                FROM [FlowStatesRecordObjs] AS [f1]
                WHERE [f1].[AppUserId] = @__appUserId_0 AND [f1].[TenantId] = @__tenantId_1
            ) AS [f2] ON [f0].[Id] = [f2].[FlowRecordObjsId]
            WHERE [f0].[AppUserId] = @__appUserId_0 AND [f0].[TenantId] = @__tenantId_1 AND [f0].[DeletedTime] IS NULL AND [f0].[FormObjectId] IS NOT NULL AND (([f0].[FormNo] IS NULL AND [f0].[CurrentFlowStateCode] IN (
                SELECT [n].[value]
                FROM OPENJSON(@__noFormNoStates_2) WITH ([value] nvarchar(50) ''$'') AS [n]
            )) OR ([f0].[FormNo] IS NOT NULL AND ([f0].[CurrentFlowStateCode] IN (
                SELECT [h].[value]
                FROM OPENJSON(@__hasFormNoIncludeStates_3) WITH ([value] nvarchar(50) ''$'') AS [h]
            ) OR [f0].[CurrentFlowStateCode] NOT IN (
                SELECT [h0].[value]
                FROM OPENJSON(@__hasFormNoExcludeStates_4) WITH ([value] nvarchar(50) ''$'') AS [h0]
            ) OR [f0].[CurrentFlowStateCode] IS NULL))) AND [f0].[CurrentFlowStateCode] IS NOT NULL AND [f0].[ParentFlowRecordObjId] IS NULL AND [f0].[Id] NOT IN (
                SELECT [f3].[Id]
                FROM [FlowRecordObjs] AS [f3]
                WHERE [f3].[AppUserId] = @__appUserId_0 AND [f3].[TenantId] = @__tenantId_1 AND [f3].[MainFlowRecordId] IN (
                    SELECT [f4].[Id]
                    FROM [FlowRecordObjs] AS [f4]
                    WHERE [f4].[AppUserId] = @__appUserId_0 AND [f4].[TenantId] = @__tenantId_1 AND [f4].[ParentFlowRecordObjId] IS NOT NULL
                )
            ) AND [f0].[MainFlowRecordId] NOT IN (
                SELECT [8].[value]
                FROM OPENJSON(@__8__locals1_executingRevokingFlowIds_5) WITH ([value] bigint ''$'') AS [8]
            ) AND [f0].[CurrentFlowStateCode] <> N''P2'' AND [f0].[FlowFlag] <> 3 AND (([f0].[CurrentFlowStateCode] = N''P1'' AND [f0].[FormNo] IS NOT NULL AND [f0].[FormNo] NOT LIKE N'''') OR ([f0].[CurrentFlowStateCode] = N''P9'' AND [f0].[BranchSerialId] IS NOT NULL AND [f2].[StateFlag] = 5 AND [f2].[StateType] = CAST(3 AS TINYINT)) OR ([f0].[CurrentFlowStateCode] = N''P5'' AND [f0].[BranchSerialId] IS NOT NULL AND [f0].[BranchSerialId] = 0) OR ([f0].[CurrentFlowStateCode] = N''P6'' AND [f0].[BranchSerialId] IS NOT NULL AND [f0].[BranchSerialId] = 0) OR [f0].[CurrentFlowStateCode] = N''R2'' OR [f0].[CurrentFlowStateCode] = N''R3'' OR ([f0].[CurrentFlowStateCode] = N''E1'' AND [f2].[StateFlag] = 1 AND [f2].[StateType] = CAST(3 AS TINYINT)) OR ([f0].[CurrentFlowStateCode] IN (N''P3'', N''P4'') AND [f2].[StateFlag] = 1 AND [f2].[StateType] = CAST(3 AS TINYINT)))
        ) AS [s] ON [f].[Id] = CASE
            WHEN [s].[MainFlowRecordId] = CAST(0 AS bigint) THEN [s].[Id]
            ELSE [s].[MainFlowRecordId]
        END
        UNION
        SELECT [f5].[FormObjectId], [f5].[FormNo], [f5].[ProcessDay], [f5].[MainFlowRecordId], [f5].[CreatedTime], [f5].[LatestUpdatedTime], [f5].[ParentFlowRecordObjId], CAST(1 AS bit) AS [IsArchived], [s0].[LatestUpdatedTime] AS [SubFlowLatestUpdatedTime], [s0].[Id] AS [SubFlowId], [s0].[ToEmployeeCode]
        FROM [FlowRecordObjsArchived] AS [f5]
        INNER JOIN (
            SELECT [f6].[Id], [f6].[LatestUpdatedTime], [f6].[MainFlowRecordId], [f8].[ToEmployeeCode]
            FROM [FlowRecordObjsArchived] AS [f6]
            INNER JOIN (
                SELECT [f7].[FlowRecordObjsId], [f7].[ToEmployeeCode]
                FROM [FlowStatesRecordObjsArchived] AS [f7]
                WHERE [f7].[AppUserId] = @__appUserId_0 AND [f7].[TenantId] = @__tenantId_1
            ) AS [f8] ON [f6].[Id] = [f8].[FlowRecordObjsId]
            WHERE [f6].[AppUserId] = @__appUserId_0 AND [f6].[TenantId] = @__tenantId_1 AND [f6].[DeletedTime] IS NULL AND [f6].[FormObjectId] IS NOT NULL AND (([f6].[FormNo] IS NULL AND [f6].[CurrentFlowStateCode] IN (
                SELECT [n0].[value]
                FROM OPENJSON(@__noFormNoStates_2) WITH ([value] nvarchar(50) ''$'') AS [n0]
            )) OR ([f6].[FormNo] IS NOT NULL AND ([f6].[CurrentFlowStateCode] IN (
                SELECT [h1].[value]
                FROM OPENJSON(@__hasFormNoIncludeStates_3) WITH ([value] nvarchar(50) ''$'') AS [h1]
            ) OR [f6].[CurrentFlowStateCode] NOT IN (
                SELECT [h2].[value]
                FROM OPENJSON(@__hasFormNoExcludeStates_4) WITH ([value] nvarchar(50) ''$'') AS [h2]
            ) OR [f6].[CurrentFlowStateCode] IS NULL))) AND [f6].[CurrentFlowStateCode] IS NOT NULL
        ) AS [s0] ON [f5].[Id] = [s0].[MainFlowRecordId]
    ) AS [u]
    GROUP BY [u].[MainFlowRecordId], [u].[FormObjectId], [u].[FormNo], [u].[CreatedTime], [u].[LatestUpdatedTime], [u].[ProcessDay], [u].[ParentFlowRecordObjId], [u].[IsArchived]
) AS [u0]',N'@__appUserId_0 bigint,@__tenantId_1 bigint,@__noFormNoStates_2 nvarchar(4000),@__hasFormNoIncludeStates_3 nvarchar(4000),@__hasFormNoExcludeStates_4 nvarchar(4000),@__8__locals1_executingRevokingFlowIds_5 nvarchar(4000)',@__appUserId_0=6070292994734084,@__tenantId_1=6395217529958400,@__noFormNoStates_2=N'["PG0","PG1"]',@__hasFormNoIncludeStates_3=N'["P1"]',@__hasFormNoExcludeStates_4=N'["P2"]',@__8__locals1_executingRevokingFlowIds_5=N'[6749830704793609]'

```

## Parameter sniffing

### PRD問題重現

![alt text](image-4.png)

![alt text](image-5.png)

> 18:43執行一次  --> DTU 飆到100% --> 18:46降回0

### 查詢參數差異

以下是兩個查詢主要差異的表格化比較：

| 比較項目              | 第一個查詢                                               | 第二個查詢                                                |
|-----------------------|----------------------------------------------------------|-----------------------------------------------------------|
| **日期範圍 - 開始時間** | `@__filterCreatedTimeStart_Value_2 = '2024-11-06T16:00:00Z'` | `@__filterCreatedTimeStart_Value_2 = '2024-09-08T16:00:00.000Z'` |
| **日期範圍 - 結束時間** | `@__filterCreatedTimeEnd_Value_3 = '2025-01-07T15:59:59.998Z'` | `@__filterCreatedTimeEnd_Value_3 = '2025-01-09T15:59:59.998Z'` |
| **無 FormNo 狀態**      | `@__noFormNoStates_4 = '[]'`                             | `@__noFormNoStates_4 = N'["PG0","PG1"]'`                   |
| **有 FormNo 包含狀態**   | `@__hasFormNoIncludeStates_5 = '[]'`                     | `@__hasFormNoIncludeStates_5 = N'["P1"]'`                  |
| **有 FormNo 排除狀態**   | `@__hasFormNoExcludeStates_6 = '[]'`                     | `@__hasFormNoExcludeStates_6 = N'["P2"]'`                  |
| **撤銷 Flow IDs**       | `@__8__locals1_executingRevokingFlowIds_7 = '[]'`        | `@__8__locals1_executingRevokingFlowIds_7 = NULL`           |
| **最外層 WHERE 條件**    | 使用 `WHERE [CreatedTime] > @__filterCreatedTimeStart_Value_2 AND [CreatedTime] < @__filterCreatedTimeEnd_Value_3` | 無該最外層日期篩選條件                                             |
| **ORDER BY 子句**      | 排序條件包含多個欄位：`[s5].[LatestUpdatedTime] DESC, [s5].[MainFlowRecordId], ... , [s6].[Id0]` | 僅排序 `[s5].[LatestUpdatedTime] DESC`，後面無其他排序欄位            |

此表格概述了兩個查詢在參數值、篩選條件和排序邏輯上的主要不同點。
