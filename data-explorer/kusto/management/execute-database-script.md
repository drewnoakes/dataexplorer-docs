---
title: .execute database script - Azure Data Explorer
description: This article describes the .execute database script functionality in Azure Data Explorer.
ms.reviewer: alexans
ms.topic: reference
ms.date: 10/11/2021
---
# .execute database script

Executes batch of control commands in scope of a single database.

## Syntax

`.execute` `database` `script`  
[`with` `(` *PropertyName* `=` *PropertyValue* [`,` ...] `)`]   
`<|`  
 *Control-commands-script*

### Parameters

* *Control-commands-script*: Text with one or more control commands.
* *PropertyName*: Optional properties from the following list:

#### Optional properties

  | PropertyName            | Type            | Description                          |
  |---------------------|-----------------|---------------------------------------------------------------------------------------------------|
  | `ContinueOnErrors`            | `bool`        | If set to `false` - the script will stop on the first error. If set to `true` - the script execution continues. Default: `false`. |
  | `ThrowOnErrors`            | `bool`        | If set to `true` - the script will throw an error (fail) on the first error. Does not work together with `ContinueOnErrors`, only one is allowed. Default: `false`. |

## Output

Each command appearing in the script will be reported as a separate record in the output table. Each record has the following fields:

|Output parameter |Type |Description
|---|---|--- 
|OperationId  |Guid |Identifier of the command.
|CommandType  |String |The type of the command.
|CommandText  |String |Text of the specific command.
|Result|String|Outcome of the specific command execution.
|Reason|String|Detailed information about command execution outcome.

>[!NOTE]
>* The script text may include empty lines and comments between the commands.
>* Commands are executed sequentially, in the order they appear in the input script.
>* Script execution is non-transactional, and no rollback is performed upon error. It's advised to use the idempotent form of commands when using `.execute database script`.
>* Default behavior of the command - fail on the first error, it can be changed using property argument.
>* Read-only control commands (`.show` commands) aren't executed and are reported with status `Skipped`.

>[!Tip]
>* This command is useful if you want to "clone"/"duplicate" an existing database. You can use the [`.show database schema command`](show-schema-database.md) on the existing database (the source database), and use its output as the *Control-commands-script* of ".execute database script".
>* If you want to "clone"/"duplicate" the cluster, you can use export its [ARM template](/azure/azure-resource-manager/templates/export-template-portal#export-template-from-a-resource) and recreate the resource. 

## Example

```kusto
.execute database script with (ContinueOnErrors=true)
<|
//
// Create tables
.create-merge table T(a:string, b:string)
//
// Apply policies
.alter-merge table T policy retention softdelete = 10d 
//
// Create functions
.create-or-alter function
  with (skipvalidation = "true") 
  SampleT1(myLimit: long) { 
    T1 | take myLimit
}
```

|OperationId|CommandType|CommandText|Result|Reason|
|---|---|---|---|---|
|1d28531b-58c8-4023-a5d3-16fa73c06cfa|TableCreate|.create-merge table T(a:string, b:string)|Completed||
|67d0ea69-baa4-419a-93d3-234c03834360|RetentionPolicyAlter|.alter-merge table T policy retention softdelete = 10d|Completed||
|0b0e8769-d4e8-4ff9-adae-071e52a650c7|FunctionCreateOrAlter|.create-or-alter function  with (skipvalidation = "true")SampleT1(myLimit: long) {T1 \| take myLimit}|Completed||
