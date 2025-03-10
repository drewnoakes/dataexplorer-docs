---
title: table() (scope function) - Azure Data Explorer
description: Learn how to use the table() (scope function) function to reference a table.
ms.reviewer: alexans
ms.topic: reference
ms.date: 02/01/2023
zone_pivot_group_filename: data-explorer/zone-pivot-groups.json
zone_pivot_groups: kql-flavors
---
# table() (scope function)

The table() function references a table by providing its name as an expression of type `string`.

```kusto
table('StormEvent')
```

## Syntax

`table` `(` *TableName* [`,` *DataScope*] `)`

## Arguments

::: zone pivot="azuredataexplorer"

* *TableName*: An expression of type `string` that provides the name of the table
  being referenced. The value of this expression must be constant at the point
  of call to the function (i.e. it can't vary by the data context).

* *DataScope*: An optional parameter of type `string` that can be used to restrict
  the table reference to data according to how this data falls under the table's
  effective [cache policy](../management/cachepolicy.md). If used, the actual argument
  must be a constant `string` expression having one of the following possible values:

  * `"hotcache"`: Only data that is categorized as hot cache will be referenced.
  * `"all"`: All the data in the table will be referenced.
  * `"default"`: This is the same as `"all"`, except if the cluster has been
      set to use `"hotcache"` as the default by the cluster admin.

::: zone-end

::: zone pivot="azuremonitor"

* *TableName*: An expression of type `string` that provides the name of the table
  being referenced. The value of this expression must be constant at the point
  of call to the function (i.e. it can't vary by the data context).

* *DataScope*: An optional parameter of type `string` that can be used to restrict
  the table reference to data according to how this data falls under the table's
  effective cache policy. If used, the actual argument
  must be a constant `string` expression having one of the following possible values:

  * `"hotcache"`: Only data that is categorized as hot cache will be referenced.
  * `"all"`: All the data in the table will be referenced.
  * `"default"`: This is the same as `"all"`.

::: zone-end

## Returns

`table(T)` returns:

* Data from table *T* if a table named *T* exists.
* Data returned by function *T* if a table named *T* doesn't exist but a function named *T* exists. Function *T* must take no arguments and must return a tabular result.
* A semantic error is raised if there's no table named *T* and no function named *T*.

## Examples

### Use table() to access table of the current database

<!-- csl: https://help.kusto.windows.net/Samples -->
```kusto
table('StormEvent') | count
```

**Output**

|Count|
|---|
|59066|

### Use table() inside let statements

The query above can be rewritten as a query-defined function (let statement) that receives a parameter `tableName` - which is passed into the table() function.

<!-- csl: https://help.kusto.windows.net/Samples -->
```kusto
let foo = (tableName:string)
{
    table(tableName) | count
};
foo('help')
```

**Output**

|Count|
|---|
|59066|

### Use table() inside Functions

The same query as above can be rewritten to be used in a function that 
receives a parameter `tableName` - which is passed into the table() function.

```kusto
.create function foo(tableName:string)
{
    table(tableName) | count
};
```

::: zone pivot="azuredataexplorer"

> [!NOTE]
> Such functions can be used only locally and not in the cross-cluster query.

::: zone-end

### Use table() with non-constant parameter

A parameter, which isn't a scalar constant string, can't be passed as a parameter to the `table()` function.

Below, given an example of workaround for such case.

```kusto
let T1 = print x=1;
let T2 = print x=2;
let _choose = (_selector:string)
{
    union
    (T1 | where _selector == 'T1'),
    (T2 | where _selector == 'T2')
};
_choose('T2')

```

**Output**

|x|
|---|
|2|
