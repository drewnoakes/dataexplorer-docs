---
title: Kusto query ingestion (set, append, replace) - Azure Data Explorer
description: This article describes Ingest from query (.set, .append, .set-or-append, .set-or-replace) in Azure Data Explorer.
ms.reviewer: orspodek
ms.topic: reference
ms.date: 03/30/2020
---
# Ingest from query (.set, .append, .set-or-append, .set-or-replace)

These commands execute a query or a control command and ingest the results of the query into a table. The difference between these commands is how they treat existing or nonexistent tables and data.

|Command          |If table exists                     |If table doesn't exist                    |
|-----------------|------------------------------------|------------------------------------------|
|`.set`           |The command fails                  |The table is created and data is ingested|
|`.append`        |Data is appended to the table      |The command fails                        |
|`.set-or-append` |Data is appended to the table      |The table is created and data is ingested|
|`.set-or-replace`|Data replaces the data in the table|The table is created and data is ingested|

To cancel an ingest from query command, see [`cancel operation`](../cancel-operation-command.md).

## Syntax

`.set` [`async`] *TableName* [`with` `(`*PropertyName* `=` *PropertyValue* [`,` ...]`)`] `<|` *QueryOrCommand*

`.append` [`async`] *TableName* [`with` `(`*PropertyName* `=` *PropertyValue* [`,` ...`])`] `<|` *QueryOrCommand*

`.set-or-append` [`async`] *TableName* [`with` `(`*PropertyName* `=` *PropertyValue* [`,` ...]`)`] `<|` *QueryOrCommand*

`.set-or-replace` [`async`] *TableName* [`with` `(`*PropertyName* `=` *PropertyValue* [`,` ...]`)`] `<|` *QueryOrCommand*

## Parameters

|Name|Type|Required|Description|
|--|--|--|--|
| *async* | string | | If specified, the command will return and continue ingestion in the background. Use the returned `OperationId` with the `.show operations` command to retrieve the ingestion completion status and results. |
| *TableName* | string | &check; | The name of the table to ingest data into. The *TableName* is always related to the database in context. |
| *PropertyName*, *PropertyValue* | string | | One or more [supported ingestion properties](#supported-ingestion-properties) used to control the ingestion process. |
| *QueryOrCommand* | string | &check; | The text of a query or a control command whose results will be used as data to ingest.|

> [!NOTE]
> Only `.show` control commands are supported.

## Supported ingestion properties

|Property|Type|Description|
|--|--|--|
|`creationTime` | string | The datetime value, formatted as an ISO8601 string, to use at the creation time of the ingested data extents. If unspecified, `now()` will be used. When specified, make sure the `Lookback` property in the target table's effective [Extents merge policy](../mergepolicy.md) is aligned with the specified value.|
|`extend_schema` | bool | If `true`, the command may extend the schema of the table. Default is `false`. This option applies only to `.append`, `.set-or-append`, and `set-or-replace` commands. |
|`recreate_schema` | bool | If `true`, the command may recreate the schema of the table. Default is `false`. This option applies only to the `.set-or-replace` command. This option takes precedence over the `extend_schema` property if both are set.|
|`folder` | string | The folder to assign to the table. If the table already exists, this property will overwrite the table's folder.|
|`ingestIfNotExists` | string | If specified, ingestion won't succeed if the table already has data tagged with an `ingest-by:` tag with the same value.|
|`policy_ingestiontime` | bool | If `true`, the [Ingestion Time Policy](../show-table-ingestion-time-policy-command.md) will be enabled on the table. The default is `true`.|
|`tags` | string | A JSON string that represents a list of [tags](../extents-overview.md#extent-tagging) to associate with the created extent. |
|`docstring` | string | A description used to document the table.|
|`distributed` | bool | If `true`, the command will ingest from all nodes executing the query in parallel. Default is `false`. See [performance tips](#performance-tips) below.|
|`persistDetails` |A Boolean value that, if specified, indicates that the command should persist the detailed results for retrieval by the [.show operation details](../operations.md#show-operation-details) command. Defaults to `false`. |`with (persistDetails=true)`|

## Schema considerations

* `.set-or-replace` will preserve the schema unless one of `extend_schema` or `recreate_schema` ingestion properties is set to `true`.
* `.set-or-append` and `.append` commands will preserve the schema unless the  `extend_schema` ingestion property is set to `true`.
* Matching the result set schema to that of the target table is based on the column types. There's no matching of column names. Make sure that the query result schema columns are in the same order as the table, else data will be ingested into the wrong columns.

> [!CAUTION]
> If the schema is modified, it happens in a separate transaction before the actual data ingestion. This means the schema may be modified even when there is a failure to ingest the data.

## Performance tips

* Data ingestion is a resource-intensive operation that might affect concurrent activities on the cluster, including running queries. Avoid running too many ingestion commands at the same time.
* Limit the data for ingestion to less than 1 GB per ingestion operation. If necessary, use multiple ingestion commands.
* Set the `distributed` flag to `true` if the amount of data being produced by the query is large, exceeds 1 GB, and doesn't require serialization. Then, multiple nodes can produce output in parallel. Don't use this flag when query results are small, since it might needlessly generate many small data shards.

[!INCLUDE [store-query-character-limitation.md](../../../includes/store-query-character-limitation.md)]

## Examples

Create a new table called :::no-loc text="RecentErrors"::: in the database that has the same schema as :::no-loc text="LogsTable"::: and holds all the error records of the last hour.

```kusto
.set RecentErrors <|
   LogsTable
   | where Level == "Error" and Timestamp > now() - time(1h)
```

Create a new table called "OldExtents" in the database that has a single column, "ExtentId", and holds the extent IDs of all extents in the database that has been created more than 30 days earlier. The database has an existing table named "MyExtents". Since the dataset is expected to be bigger than 1 GB (more than ~1 million rows) use the *distributed* flag

```kusto
.set async OldExtents with(distributed=true) <|
   MyExtents 
   | where CreatedOn < now() - time(30d)
   | project ExtentId
```

Append data to an existing table called "OldExtents" in the current database that has a single column, "ExtentId", and holds the extent IDs of all extents in the database that have been created more than 30 days earlier.
Mark the new extent with tags `tagA` and `tagB`, based on an existing table named "MyExtents".

```kusto
.append OldExtents with(tags='["TagA","TagB"]') <| 
   MyExtents 
   | where CreatedOn < now() - time(30d) 
   | project ExtentId
```

Append data to the "OldExtents" table in the current database, or create the table if it doesn't already exist. Tag the new extent with `ingest-by:myTag`. Do so only if the table doesn't already contain an extent tagged with `ingest-by:myTag`, based on an existing table named "MyExtents".

```kusto
.set-or-append async OldExtents with(tags='["ingest-by:myTag"]', ingestIfNotExists='["myTag"]') <|
   MyExtents
   | where CreatedOn < now() - time(30d)
   | project ExtentId
```

Replace the data in the "OldExtents" table in the current database, or create the table if it doesn't already exist. Tag the new extent with `ingest-by:myTag`.

```kusto
.set-or-replace async OldExtents with(tags='["ingest-by:myTag"]', ingestIfNotExists='["myTag"]') <| 
   MyExtents 
   | where CreatedOn < now() - time(30d) 
   | project ExtentId
```

Append data to the "OldExtents" table in the current database, while setting the created extent(s) creation time to a specific datetime in the past.

```kusto
.append async OldExtents with(creationTime='2017-02-13T11:09:36.7992775Z') <| 
   MyExtents 
   | where CreatedOn < now() - time(30d) 
   | project ExtentId     
```

**Return output**

Returns information on the extents created because of the `.set` or `.append` command.

**Example output**

|ExtentId |OriginalSize |ExtentSize |CompressedSize |IndexSize |RowCount | 
|--|--|--|--|--|--|
|23a05ed6-376d-4119-b1fc-6493bcb05563 |1291 |5882 |1568 |4314 |10 |
