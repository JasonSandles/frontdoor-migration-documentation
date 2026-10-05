**RepositoryWrapper & Repository Architecture**

**Developer Implementation and Extension Guide**

How to consume existing repositories, create new domain repositories, and register them with RepositoryWrapper

Purpose  
This document defines the repository architecture developers should follow when implementing domain-specific business logic and managing interactions with the database through the application's standardized data-access layer. It explains the role of RepositoryWrapper, how domain-specific repositories receive shared infrastructure, how to add a new repository, and how environment-specific Oracle connection configuration is selected without exposing connection strings to repository developers.

# 1\. Architecture Overview

The repository layer separates application/domain code from Oracle data-access mechanics. RepositoryWrapper acts as the central access point for domain-specific repositories. It owns the shared data-access dependency and logger, while each domain repository contains only the operations for its own business area.

**Standard call path  
**Controller / Service → RepositoryWrapper → Domain-Specific Repository → OracleDAL → Oracle Database / PL/SQL

The supplied RepositoryWrapper follows a lazy-initialization pattern: a repository instance is created only when its property is first accessed, then the same instance is returned for subsequent accesses within that wrapper instance.

## 1.1 Responsibilities by Layer

|     |     |
| --- | --- |
| **Component** | **Primary Responsibility** |
| Controller / Service | Requests business/data operations through the repository layer; does not create OracleDAL or handle connection strings. |
| RepositoryWrapper | Centralizes repository access, owns shared infrastructure, and creates domain repository instances on demand. |
| Domain Repository | Implements domain-specific data operations and consumes the supplied OracleDAL/logger. |
| OracleDAL | Handles Oracle commands, parameters, connections, result retrieval, Dapper queries, and other low-level Oracle access. |
| Oracle / PL/SQL | Executes packages, stored procedures, SQL, and database-side business/data logic. |

# 2\. RepositoryWrapper Responsibilities

RepositoryWrapper is the composition point for the repository layer. Developers consume repository properties from the wrapper rather than constructing domain repositories manually. The current pattern maintains private backing fields for each repository and exposes a public property that creates the repository on first access.

private CustomerRepository? \_customer;  
<br/>public CustomerRepository Customer  
{  
get  
{  
if (\_customer == null)  
{  
\_customer = new CustomerRepository(\_logger, oDal);  
}  
<br/>return \_customer;  
}  
}

The important architectural rule is that RepositoryWrapper supplies shared dependencies to the repository. A domain repository should not read configuration, resolve a connection string, or instantiate its own OracleDAL.

## 2.1 Lazy Repository Creation

- The private backing field starts as null.
- The public repository property checks whether an instance already exists.
- On first access, RepositoryWrapper constructs the repository and passes the logger and OracleDAL.
- Later accesses return the previously created repository instance.

**Developer rule  
**When adding a new repository, follow the same backing-field + public-property pattern unless the repository architecture is intentionally refactored as a whole.

# 3\. Environment-Specific Connection Configuration

The Oracle connection string used by RepositoryWrapper comes from application configuration (appsettings). Repository developers do not receive or manage the connection string directly. RepositoryWrapper resolves the appropriate configured connection based on the build/environment condition and uses it to create the shared OracleDAL instance.

**Security boundary  
**Connection strings and credentials belong in application configuration or an approved secret provider. They should never be hard-coded in RepositoryWrapper, domain repositories, source comments, or application code.

## 3.1 Appsettings Configuration

A typical configuration structure can expose separate named connection strings for the supported environments:

{  
"ConnectionStrings": {  
"OracleDev": "&lt;development Oracle connection string&gt;",  
"OracleTest": "&lt;test Oracle connection string&gt;",  
"OracleProd": "&lt;production Oracle connection string&gt;"  
}  
}

Actual credentials should be supplied through the deployment/configuration mechanism appropriate to the environment. The values above are placeholders only.

## 3.2 Conditional Selection in RepositoryWrapper

RepositoryWrapper should select the configuration key using the project's compile-time environment symbols. The following illustrates the requested DEV / TEST / PROD conditional pattern.

private static string GetConnectionString(IConfiguration configuration)  
{  
#if DEV  
const string connectionName = "OracleDev";  
#elif TEST  
const string connectionName = "OracleTest";  
#elif PROD  
const string connectionName = "OracleProd";  
#else  
#error A database environment symbol (DEV, TEST, or PROD) must be defined.  
#endif  
<br/>return configuration.GetConnectionString(connectionName)  
?? throw new InvalidOperationException(  
$"Connection string '{connectionName}' is not configured.");  
}

The exact preprocessor symbols must match those defined by the solution/build pipeline. If the project uses different symbol names, update the conditional names accordingly. This guide assumes DEV, TEST, and PROD as the intended environment symbols.

**Result  
**Only RepositoryWrapper needs configuration access. Domain repositories receive OracleDAL and remain environment-agnostic.

# 4\. Domain Repository Structure

Each repository should represent one domain or functional area. For example, CustomerRepository contains customer-specific operations, while InvoiceRepository contains invoice-specific operations. This keeps repositories focused and prevents a single data-access class from accumulating unrelated methods.

public class CustomerRepository  
{  
#region privates  
<br/>private readonly ILogger \_logger;  
private readonly OracleDAL \_oDal;  
<br/>#endregion  
<br/>#region constructors  
<br/>public CustomerRepository(  
ILogger logger,  
OracleDAL oDal)  
{  
\_logger = logger;  
\_oDal = oDal;  
}  
<br/>#endregion  
<br/>#region methods  
<br/>// Customer-specific repository methods.  
<br/>#endregion  
}

The supplied customer example demonstrates this intended dependency pattern: the repository receives a logger and DAL through its constructor rather than creating database infrastructure itself.

**Naming note  
**The supplied samples contain older/inconsistent DAL naming in a few places. This guide uses OracleDAL consistently because the target architecture described for this repository layer is Oracle-based. Align the concrete source code with the actual OracleDAL type used by the solution.

# 5\. How Developers Consume Existing Repositories

Application code should access a domain repository through RepositoryWrapper and then call the required domain method. It should not construct CustomerRepository, InvoiceRepository, or OracleDAL directly.

public class CustomerService  
{  
private readonly RepositoryWrapper \_repository;  
<br/>public CustomerService(RepositoryWrapper repository)  
{  
\_repository = repository;  
}  
<br/>public async Task&lt;List<CustomerDTO&gt;> GetCustomers(int companyId)  
{  
return await \_repository.Customer.GetCustomers(companyId);  
}  
}

**Do not do this  
**Do not retrieve a connection string in CustomerRepository, create a new OracleDAL there, or construct CustomerRepository directly from a controller/service.

# 6\. Creating a New Domain Repository

When a new functional area requires database access, create a dedicated repository and register it with RepositoryWrapper. The following procedure is the standard extension pattern.

## Step 1 - Create the Repository Class

public class PaymentRepository  
{  
private readonly ILogger \_logger;  
private readonly OracleDAL \_oDal;  
<br/>public PaymentRepository(  
ILogger logger,  
OracleDAL oDal)  
{  
\_logger = logger;  
\_oDal = oDal;  
}  
<br/>// Payment-specific methods go here.  
}

## Step 2 - Add Domain Methods

Implement only operations that belong to the repository's domain. Use the supplied OracleDAL instance for Oracle access.

public async Task&lt;List<PaymentDTO&gt;> GetPayments(long invoiceId)  
{  
var parameters = new  
{  
P_INVOICE_ID = invoiceId  
};  
<br/>return await \_oDal.GetList&lt;PaymentDTO&gt;(  
"PKG_PAYMENTS.GET_PAYMENTS",  
parameters);  
}

## Step 3 - Add a Backing Field to RepositoryWrapper

private PaymentRepository? \_payment;

## Step 4 - Add the Public Repository Property

public PaymentRepository Payment  
{  
get  
{  
if (\_payment == null)  
{  
\_payment = new PaymentRepository(  
\_logger,  
oDal);  
}  
<br/>return \_payment;  
}  
}

## Step 5 - Consume the New Repository

List&lt;PaymentDTO&gt; payments =  
await repositoryWrapper.Payment.GetPayments(invoiceId);

# 7\. Complete Extension Pattern

A new repository requires changes in only two architectural locations:

|     |     |
| --- | --- |
| **Location** | **Developer Change** |
| New domain repository file | Create the repository, accept ILogger and OracleDAL, and implement domain-specific methods. |
| RepositoryWrapper | Add one private backing field and one public lazy-loaded property for the new repository. |

No connection-string code should be added to the new repository. No environment-selection logic should be duplicated in the new repository. Those concerns remain centralized in RepositoryWrapper/application configuration.

# 8\. Repository Method Design Guidelines

- Keep methods domain-specific. Customer operations belong in CustomerRepository; payment operations belong in PaymentRepository.
- Use the OracleDAL instance supplied by the constructor.
- Prefer async methods for I/O operations when an appropriate OracleDAL async method is available.
- Use DTOs/models appropriate to the repository domain rather than exposing DataRow/DataTable unless the consuming contract specifically requires them.
- Use parameterized stored procedure/package calls; do not concatenate user-supplied values into SQL.
- Let the application's established exception/logging strategy handle failures; add repository logging only when it provides useful operational context.
- Do not store connection strings, credentials, or environment-specific database values in repository classes.
- Do not create a second OracleDAL inside a domain repository.

# 9\. Adding a Repository - Developer Checklist

☐ Create a repository class named for one domain/functional area.

☐ Add private readonly ILogger and OracleDAL fields.

☐ Accept ILogger and OracleDAL through the repository constructor.

☐ Add only domain-specific database methods.

☐ Use the supplied OracleDAL for all Oracle access.

☐ Add a nullable/private backing field to RepositoryWrapper.

☐ Add a public lazy-loaded property to RepositoryWrapper.

☐ Pass the existing wrapper logger and OracleDAL into the new repository.

☐ Do not add IConfiguration or connection-string lookup code to the domain repository.

☐ Do not duplicate DEV / TEST / PROD selection logic in the domain repository.

☐ Verify the repository can be accessed through RepositoryWrapper.

☐ Test the repository against the intended environment configuration.

# 10\. Common Mistakes

|     |     |     |
| --- | --- | --- |
| **Mistake** | **Why It Is a Problem** | **Correct Pattern** |
| Reading appsettings from every repository | Duplicates configuration concerns and exposes database configuration broadly. | RepositoryWrapper resolves configuration; repositories receive OracleDAL. |
| Hard-coding a connection string | Creates security, deployment, and environment-management risks. | Use named appsettings connection strings and approved secret/configuration mechanisms. |
| Creating OracleDAL inside each repository | Breaks centralized ownership and can create inconsistent configuration/lifetimes. | Accept OracleDAL in the repository constructor. |
| Forgetting to register a new repository in RepositoryWrapper | Consumers have no standard entry point for the new domain repository. | Add the backing field and lazy-loaded property. |
| Putting unrelated methods in an existing repository | Creates large, tightly coupled repository classes. | Create a new repository for a distinct domain. |
| Duplicating environment #if logic | Makes environment behavior harder to control and audit. | Keep environment selection centralized. |
| Constructing repositories directly in controllers/services | Bypasses RepositoryWrapper and its shared dependencies. | Access repositories through RepositoryWrapper. |

# 11\. Reference Architecture

Application / API  
|  
v  
RepositoryWrapper  
|  
+---- CustomerRepository ----+  
+---- InvoiceRepository -----+  
+---- ProductRepository -----+----> Shared OracleDAL ----> Oracle  
+---- EstimateRepository ----+  
+---- New Domain Repository -+  
<br/>Configuration boundary:  
appsettings / secret provider  
|  
+--> DEV / TEST / PROD conditional selection  
|  
+--> RepositoryWrapper only

The key design objective is centralized infrastructure with domain-specific repositories. Developers extend the repository layer by adding focused repository classes and registering them with RepositoryWrapper, while database configuration and environment selection remain centralized.

# 12\. Summary

**Developer contract  
**Use RepositoryWrapper as the entry point. Create one repository per domain. Accept ILogger and OracleDAL in the repository constructor. Register the repository with a lazy-loaded RepositoryWrapper property. Never retrieve or hard-code the Oracle connection string in a domain repository. RepositoryWrapper obtains the connection from appsettings using the configured DEV / TEST / PROD build condition.