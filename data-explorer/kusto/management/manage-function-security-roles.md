---
title: Manage function roles - Azure Data Explorer
description: This article describes how to use management commands to view, add, and remove function admins on the function level in Azure Data Explorer.
ms.topic: reference
ms.date: 01/25/2023
---

# Manage function roles

Azure Data Explorer uses a role-based access control model in which principals get access to resources according to the roles they're assigned. On functions, the only security role is `admins`. Function `admins` have the ability to view, modify, and remove the function.

In this article, you'll learn how to use management commands to [view existing admins](#view-existing-admins) as well as [add and remove admins](#add-and-remove-admins) on functions.

> [!NOTE]
>
> * You must be an AllDatabasesAdmin, a Database Admin, or a Function Admin to control function access.
> * A principal must have access on the database or table level to be a Function Admin.
> * For more information, see [role-based access control](access-control/role-based-access-control.md).

## View existing admins

Before you add or remove principals, you can use the `.show` command to see a table with all of the principals that already have admin access on the function.

### Syntax

`.show` `function` *FunctionName* `principals`

### Parameters

|Name|Type|Required|Description|
|--|--|--|--|
| *FunctionName* | string | &check; | The name of the function for which to list principals.|

### Example

The following command lists all security principals that have access to the `SampleFunction` function.

```kusto
.show function SampleFunction principals
```

**Example output**

|Role |PrincipalType |PrincipalDisplayName |PrincipalObjectId |PrincipalFQN|
|---|---|---|---|---|
|Function SampleFunction Admin |Azure AD User |Abbi Atkins |cd709aed-a26c-e3953dec735e |aaduser=abbiatkins@fabrikam.com|

## Add and remove admins

This section provides syntax, parameters, and examples for adding and removing principals.

### Syntax

*Action* `function` *FunctionName* `admins` `(` *Principal* [`,` *Principal*...] `)` [`skip-results`] [ *Description* ]

### Parameters

|Name|Type|Required|Description|
|--|--|--|--|
| *Action* | string | &check; | The command `.add`, `.drop`, or `.set`.<br/>`.add` adds the specified principals, `.drop` removes the specified principals, and `.set` adds the specified principals and removes all previous ones.|
| *FunctionName* | string | &check; | The name of the function for which to add principals.|
| *Principal* | string | &check; | One or more principals. For how to specify these principals, see [principals and identity providers](./access-control/principals-and-identity-providers.md#examples-for-azure-ad-principals).|
| `skip-results` | string | | If provided, the command won't return the updated list of function principals.|
| *Description* | string | | Text to describe the change that will be displayed when using the `.show` command.|

> [!NOTE]
> The `.set` command with `none` instead of a list of principals will remove all principals.

### Examples

In the following examples, you'll see how to [add admins](#add-admins-with-add), [remove admins](#remove-admins-with-drop), and [add and remove admins in the same command](#add-new-admins-and-remove-the-old-with-set).

#### Add admins with .add

The following example adds a principal to the `admins` role on the `SampleFunction` function.

```kusto
.add function SampleFunction admins ('aaduser=imikeoein@fabrikam.com')
```

#### Remove admins with .drop

The following example removes all principals in the group from the `admins` role on the `SampleFunction` function.

```kusto
.drop function SampleFunction admins ('aadGroup=SomeGroupEmail@fabrikam.com')
```

#### Add new admins and remove the old with .set

THe following example removes existing `admins` and adds the provided principals as `admins` on the `SampleFunction` function.

```kusto
.set function SampleFunction admins ('aaduser=imikeoein@fabrikam.com', 'aaduser=abbiatkins@fabrikam.com')
```

#### Remove all admins with .set

The following command removes all existing `admins` on the `SampleFunction` function.

```kusto
.set function SampleFunction admins none
```
