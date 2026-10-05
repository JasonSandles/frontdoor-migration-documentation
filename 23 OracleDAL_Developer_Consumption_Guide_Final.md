**OracleDAL**

**Developer Consumption Guide**

Standard patterns for accessing Oracle from .NET repositories and services

**Purpose  
**This guide explains how application developers should consume the OracleDAL class. It focuses on supported usage patterns, method selection, parameter handling, result retrieval, resource management, and common implementation mistakes.

Applies to: .NET • Oracle.ManagedDataAccess.Client • Dapper

Prepared as an internal developer reference

# 1\. Overview

OracleDAL is a reusable Oracle data-access helper intended to centralize common database operations. It wraps Oracle connection creation, OracleCommand configuration, parameter management, ADO.NET result retrieval, Dapper-based typed queries, non-query execution, and bulk insert operations.

OracleDAL is not instantiated directly by domain-specific repositories. The application repository infrastructure owns creation of the OracleDAL instance using the configured Oracle connection string and passes that initialized DAL instance to each domain-specific repository. This keeps connection-string access centralized in the application repository infrastructure and prevents individual repository developers from needing direct access to database credentials or configuration.

Recommended application flow  
Controller / API Endpoint → application repository infrastructure → Domain Repository → OracleDAL → Oracle Database / PL/SQL

## 1.1 What OracleDAL Handles for You

- Creates and disposes OracleConnection instances for normal operations.
- Configures commands for stored-procedure or text execution.
- Uses BindByName = true so Oracle parameters are matched by parameter name.
- Provides helper overloads for OracleParameter creation.
- Returns DataSet, DataTable, DataRow, DataColumn, scalar object, or strongly typed List&lt;T&gt; results.
- Executes stored procedures and raw SQL that do not return a result set.
- Supports OracleBulkCopy for batch insertion.
- Disposes the internally maintained OracleCommand when the OracleDAL instance is disposed.

## 1.2 Dependencies

|     |     |
| --- | --- |
| **Dependency** | **Purpose** |
| Oracle.ManagedDataAccess.Client | OracleConnection, OracleCommand, OracleParameter, OracleDataAdapter, OracleBulkCopy, and OracleDbType. |
| Dapper | Asynchronous strongly typed query mapping through QueryAsync&lt;T&gt;(). |
| System.Data | DataSet, DataTable, DataRow, DataColumn, CommandType, and ParameterDirection. |
| System.Text | Formats the current Oracle parameter collection in ToString(). |

# 2\. Creating and Disposing OracleDAL

Domain-specific repositories receive an initialized OracleDAL instance from the application's repository infrastructure. The creation and configuration of that infrastructure is outside the scope of this guide. Developers consuming OracleDAL from a domain repository should not retrieve, store, or manage the Oracle connection string themselves.

The consuming repository uses the supplied OracleDAL instance for parameter management and database operations. For normal OracleDAL methods, developers also do not manually create, open, or close OracleConnection objects.

Important  
application repository infrastructure should be scoped so its OracleDAL instance is scoped to the current application/request lifetime and is not shared concurrently across unrelated requests. OracleDAL maintains one mutable OracleCommand and parameter collection.

# 3\. Parameter Management

OracleDAL exposes three AddParameter overloads. Choose the overload based on how much Oracle-specific information the operation requires.

## 3.1 Basic Parameter

dal.AddParameter("P_USER_ID", userId);  

Use this overload when the Oracle provider can safely infer the parameter type from the supplied .NET value. A null value is automatically converted to DBNull.Value.

## 3.2 Explicit Oracle Type

dal.AddParameter(  
"P_USER_ID",  
userId,  
OracleDbType.Int32);  

Use this form when the Oracle type should be explicit. This is generally clearer for PL/SQL calls and reduces ambiguity.

## 3.3 Explicit Type and Direction

dal.AddParameter(  
"P_RESULT",  
null,  
OracleDbType.Varchar2,  
ParameterDirection.Output);  

Use this overload for output, input/output, return-value, or other direction-sensitive parameters.

## 3.4 BindByName Behavior

OracleDAL sets BindByName to true. Parameter names therefore matter: the command binds values by name instead of requiring parameters to be added in the exact ordinal order expected by the stored procedure.

## 3.5 Clearing Parameters

**Required when reusing the same OracleDAL instance  
**The OracleCommand is stored on the OracleDAL instance, so parameters added for one operation remain in its parameter collection until they are explicitly cleared, replaced by a method that clears them, or the DAL is disposed.

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>dal.AddParameter("P_USER_ID", userId, OracleDbType.Int32);  
DataTable user = dal.GetTable("PKG_USERS.GET_USER");  
<br/>dal.ClearSQLParameters();  
<br/>dal.AddParameter("P_ROLE_ID", roleId, OracleDbType.Int32);  
DataTable role = dal.GetTable("PKG_ROLES.GET_ROLE");  

# 4\. Choosing the Correct Read Method

Select the method based on the shape of the data your repository needs. New code that maps naturally to DTOs can use the Dapper GetList&lt;T&gt; methods; legacy or schema-flexible code can use the DataTable/DataSet methods.

|     |     |     |
| --- | --- | --- |
| **Method** | **Returns** | **Use When** |
| GetDataSet() | DataSet | A stored procedure result that should be represented as a DataSet. |
| GetTable() | DataTable | Multiple rows/columns when a DataTable is the desired result. |
| GetTableBySQL() | DataTable | A text SQL statement that returns tabular data. |
| GetRow() | DataRow? | Only the first returned row is needed. |
| GetColumn() | DataColumn? | The first DataColumn definition is needed. |
| GetObject() | object? | Only the first value from row 1 / column 1 is needed. |
| GetList&lt;T&gt;() | Task&lt;List<T&gt;> | Strongly typed asynchronous DTO/model results through Dapper. |

## 4.1 DataTable from a Stored Procedure

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>dal.AddParameter(  
"P_COMPANY_ID",  
companyId,  
OracleDbType.Int32);  
<br/>DataTable customers =  
dal.GetTable("PKG_CUSTOMERS.GET_CUSTOMERS");  

## 4.2 First Row

DataRow? customer =  
dal.GetRow("PKG_CUSTOMERS.GET_CUSTOMER");  
<br/>if (customer != null)  
{  
string name = customer\["CUSTOMER_NAME"\]?.ToString() ?? string.Empty;  
}  

GetRow returns the first DataRow when at least one row exists; otherwise it returns null.

## 4.3 First Value / Scalar-Style Result

object? value =  
dal.GetObject("PKG_CUSTOMERS.GET_CUSTOMER_COUNT");  
<br/>int count = value == null  
? 0  
: Convert.ToInt32(value);  

GetObject is useful when the stored procedure returns a tabular result but the caller only needs the value at the first row and first column.

## 4.4 DataSet

DataSet data =  
dal.GetDataSet("PKG_REPORTING.GET_REPORT_DATA");  

## 4.5 Raw SQL Returning a DataTable

DataTable result =  
dal.GetTableBySQL(  
"SELECT USER_ID, USER_NAME FROM USERS");  

**Use raw SQL deliberately  
**GetTableBySQL is supported by the class, but application standards may require stored procedures or packages. Do not concatenate untrusted values into SQL strings.

# 5\. Strongly Typed Async Queries with Dapper

OracleDAL provides two generic asynchronous methods for mapping query results directly into .NET objects. These methods create an OracleConnection, open it asynchronously, execute Dapper QueryAsync&lt;T&gt;(), and return List&lt;T&gt;.

## 5.1 Without Parameters

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>List&lt;CustomerDTO&gt; customers =  
await dal.GetList&lt;CustomerDTO&gt;(  
"PKG_CUSTOMERS.GET_CUSTOMERS");  

## 5.2 With Parameters

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>var parameters = new  
{  
P_COMPANY_ID = companyId,  
P_STATUS = "ACTIVE"  
};  
<br/>List&lt;CustomerDTO&gt; customers =  
await dal.GetList&lt;CustomerDTO&gt;(  
"PKG_CUSTOMERS.GET_CUSTOMERS_BY_STATUS",  
parameters);  

**DTO mapping  
**Dapper maps returned columns to properties on T. Developers should ensure that the result-set column names and DTO property names are compatible with the project's Dapper mapping conventions.

# 6\. Executing Procedures and SQL Without Results

## 6.1 Stored Procedure

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>dal.AddParameter(  
"P_USER_ID",  
userId,  
OracleDbType.Int32);  
<br/>bool success =  
dal.ExecuteProcedure("PKG_USERS.DELETE_USER");  

ExecuteProcedure calls ExecuteNonQuery and returns true when the call completes successfully. Database failures are raised as exceptions; callers should not interpret false as an error path because the current implementation does not return false on execution failure.

## 6.2 Raw SQL

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>bool success =  
dal.ExecuteSQL(  
"UPDATE SOME_TABLE SET SOME_FLAG = 1");  

# 7\. Output Parameters

The Parameters property exposes the OracleCommand's OracleParameterCollection. After ExecuteProcedure completes, developers can inspect output parameter values through this collection.

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>dal.AddParameter(  
"P_NEW_ID",  
null,  
OracleDbType.Int64,  
ParameterDirection.Output);  
<br/>dal.ExecuteProcedure("PKG_USERS.CREATE_USER");  
<br/>object? rawValue =  
dal.Parameters\["P_NEW_ID"\]?.Value;  
<br/>long newId =  
rawValue == null || rawValue == DBNull.Value  
? 0  
: Convert.ToInt64(rawValue.ToString());  

# 8\. Using an Existing OracleConnection

GetTable and GetObject each provide an overload that accepts an OracleConnection by reference. These overloads are intended for scenarios where the caller needs to control and reuse an existing connection.

using OracleConnection connection =  
new OracleConnection(connectionString);  
<br/>using OracleDAL dal =  
new OracleDAL(connectionString);  
<br/>dal.AddParameter(  
"P_USER_ID",  
userId,  
OracleDbType.Int32);  
<br/>object? result =  
dal.GetObject(  
"PKG_USERS.GET_STATUS",  
ref connection);  

**Connection ownership  
**When a connection is supplied by the caller, the caller owns its lifetime and is responsible for disposing it. The DAL will open the supplied connection if it is not already open.

# 9\. Bulk Insert

BulkInsert uses OracleBulkCopy to write a DataTable to a target Oracle table. This is intended for high-volume insert operations where row-by-row execution would be inefficient.

using OracleDAL dal = new OracleDAL(connectionString);  
<br/>bool success =  
dal.BulkInsert(  
dataTable,  
"TARGET_TABLE");  

The destination table name is assigned directly to OracleBulkCopy.DestinationTableName and the bulk-copy timeout is set to 60 seconds. The DataTable supplied by the caller must be compatible with the destination table expected by OracleBulkCopy.

# 10\. Repository Consumption Pattern

Domain-specific repositories receive an initialized OracleDAL instance through the application's repository infrastructure. How that instance is created and passed to repositories is intentionally outside the scope of this document. The developer's responsibility begins with consuming the supplied OracleDAL instance.

public class CustomerRepository : ICustomerRepository  
{  
private readonly OracleDAL \_dal;  
<br/>public CustomerRepository(OracleDAL dal)  
{  
\_dal = dal;  
}  
<br/>public async Task&lt;List<CustomerDTO&gt;> GetCustomers(int companyId)  
{  
var parameters = new  
{  
P_COMPANY_ID = companyId  
};  
<br/>return await \_dal.GetList&lt;CustomerDTO&gt;(  
"PKG_CUSTOMERS.GET_CUSTOMERS",  
parameters);  
}  
}

## 10.1 Consumption Flow

|     |
| --- |
| API Controller / Application Service |
| ↓   |
| Domain-Specific Repository |
| ↓   |
| OracleDAL |
| ↓   |
| Oracle Database / PL/SQL Package or Stored Procedure |

# 11\. Error Handling

Most ADO.NET methods catch database or provider exceptions and throw a new Exception containing operation-specific context while preserving the original exception as InnerException. Repository or service code should allow these exceptions to flow to the application's established logging/error-handling layer unless it can add meaningful context or perform a deliberate recovery.

try  
{  
return await repository.GetCustomers(companyId);  
}  
catch (Exception ex)  
{  
// Log according to the application's standard logging policy.  
// Re-throw or translate at the appropriate application boundary.  
throw;  
}  

# 12\. Best Practices

|     |     |
| --- | --- |
| **Practice** | **Why** |
| Use the OracleDAL instance supplied to the domain repository. | Domain repository developers should consume the supplied DAL without accessing the connection string. |
| Follow the application's established DAL lifetime ownership. | Domain repositories consume the supplied instance and do not take responsibility for database configuration. |
| Prefer explicit OracleDbType values for PL/SQL parameters. | This makes the database contract clearer and avoids unnecessary inference. |
| Clear parameters before a second command on the same DAL instance. | Parameters otherwise remain attached to the internal OracleCommand. |
| Prefer GetList&lt;T&gt; for strongly typed asynchronous repository reads. | It avoids manual DataRow mapping when the result maps cleanly to a DTO. |
| Use GetTable/GetDataSet when tabular ADO.NET results are actually required. | Do not convert to DataTable merely to map immediately into DTOs. |
| Treat ExecuteProcedure failures as exceptions. | The method returns true on success and throws on failure. |
| Avoid concatenating user input into raw SQL. | Use parameterized database operations and follow the application's database-access standards. |
| Keep OracleDAL and connection strings out of controllers/UI and domain repository construction. | Database configuration and DAL construction are outside the domain repository's responsibility. |
| Do not share one OracleDAL instance across concurrent requests. | The internal command and parameter state are mutable. |

# 13\. Common Mistakes

|     |     |     |
| --- | --- | --- |
| **Mistake** | **Impact** | **Correct Pattern** |
| Reusing a DAL without clearing parameters | Old parameters can remain on the internal OracleCommand. | Call ClearSQLParameters() before building the next parameter set, or create a new DAL instance. |
| Forgetting to dispose OracleDAL | The internal OracleCommand remains undisposed longer than necessary. | Use a using declaration. |
| Expecting ExecuteProcedure to return false | Its failure path throws an exception. | Handle failures through the application's exception strategy. |
| Using GetObject when a full row is required | GetObject returns only row 1, column 1. | Use GetRow or GetTable. |
| Assuming GetColumn returns column values | It returns the first DataColumn object/definition, not the first column's row values. | Use GetTable when actual row values are required. |
| Sharing OracleDAL as a singleton | The internal command/parameter collection is mutable. | Keep DAL instances short-lived and operation-scoped. |

# 14\. Quick Reference

|     |     |
| --- | --- |
| **Member** | **Purpose** |
| AddParameter(name, value) | Add parameter with inferred Oracle type. |
| AddParameter(name, value, OracleDbType) | Add parameter with explicit Oracle type. |
| AddParameter(name, value, OracleDbType, direction) | Add typed input/output parameter. |
| ClearSQLParameters() | Clear the internal command parameter collection. |
| Parameters | Access the current OracleParameterCollection. |
| GetDataSet(...) | Execute stored procedure and return DataSet. |
| GetTable(...) | Execute stored procedure and return DataTable. |
| GetTableBySQL(sql) | Execute text SQL and return DataTable. |
| GetRow(...) | Return first row or null. |
| GetColumn(...) | Return first DataColumn or null. |
| GetObject(...) | Return first row / first column value or null. |
| ExecuteProcedure(...) | Execute stored procedure with ExecuteNonQuery. |
| ExecuteSQL(...) | Execute text SQL with ExecuteNonQuery. |
| GetList&lt;T&gt;(...) | Asynchronously return typed List&lt;T&gt; through Dapper. |
| BulkInsert(...) | Bulk-copy a DataTable into an Oracle table. |
| ToString() | Return a text representation of current parameter names and values. |

# 15\. Developer Checklist

☐ I am consuming OracleDAL from a repository or data service rather than directly from a controller/UI component.

☐ My domain repository receives an initialized OracleDAL instance; I do not access the connection string directly.

☐ I follow the application's established OracleDAL lifetime and disposal ownership.

☐ I selected the result method that matches the data shape I need.

☐ I used explicit OracleDbType values where appropriate.

☐ I used the correct parameter names expected by the PL/SQL procedure/package.

☐ I cleared existing parameters before issuing another command on the same DAL instance.

☐ I understand whether I own the OracleConnection lifetime for the overload I selected.

☐ My DTO properties are compatible with the columns returned to Dapper.

☐ I allow database failures to flow through the application's logging/error-handling strategy.

☐ I do not share one OracleDAL instance across concurrent requests.

# 16\. Summary

OracleDAL provides a single, reusable entry point for common Oracle access patterns. Domain-specific repositories receive an initialized OracleDAL instance from the application's repository infrastructure and use it without direct access to the Oracle connection string. The construction of that infrastructure is outside the scope of this guide. Developers should focus on correct parameter handling, method selection, result mapping, and exception behavior when consuming the supplied DAL instance.

Default recommendation  
Domain-specific repository → supplied OracleDAL instance → parameterized stored procedure/package call → strongly typed DTO result when practical.