**Domain-Specific API Controller Architecture**

**Developer Implementation and Usage Guide**

How to create, structure, expose, and extend controllers consistently across application domains

**Purpose  
**This document defines the standard architecture developers should follow when creating domain-specific API controllers. Controllers expose HTTP endpoints, accept and validate request models, delegate domain-specific business operations to the appropriate repository through RepositoryWrapper, and translate repository results into HTTP responses. Controllers should remain thin and should not contain Oracle connection-string handling or direct database-access logic.

# 1\. Controller Architecture Overview

Each application domain should have a dedicated ASP.NET Core API controller. Examples include EstimateController, InvoiceController, CustomerController, ProductController, and future controllers created for additional business domains.

**Standard request flow  
**HTTP Client → Domain Controller → RepositoryWrapper → Domain-Specific Repository → OracleDAL → Oracle Database / PL/SQL

The controller is the HTTP boundary of the application. Its responsibility is to receive a request, hand the request to the correct domain repository, evaluate the returned result, and return an appropriate HTTP response. Domain business logic and database interaction belong below the controller layer.

## 1.1 Responsibilities by Layer

| **Layer** | **Primary Responsibility** |
| --- | --- |
| API Controller | Defines routes/endpoints, accepts request DTOs, delegates work, and returns HTTP status/results. |
| --- | --- |
| RepositoryWrapper | Provides the controller with the correct domain-specific repository entry point. |
| --- | --- |
| Domain Repository | Implements domain-specific business/data operations and uses the standardized OracleDAL. |
| --- | --- |
| OracleDAL | Handles low-level Oracle access, parameters, command execution, result retrieval, and connection mechanics. |
| --- | --- |
| Oracle / PL/SQL | Executes packages, stored procedures, queries, and database-side operations. |
| --- | --- |

# 2\. Standard Controller Definition

Every domain controller should inherit from ControllerBase and use the ApiController and Route attributes. The controller name becomes part of the route when api/\[controller\] is used.

\[Route("api/\[controller\]")\]  
\[ApiController\]  
public class EstimateController : ControllerBase  
{  
// Controller implementation  
}

## 2.1 Route Convention

With \[Route("api/\[controller\]")\], EstimateController is exposed under the base route "api/Estimate". Action-level HttpPost/HttpGet attributes append the operation route.

| **Controller** | **Base Route Example** |
| --- | --- |
| EstimateController | api/Estimate |
| --- | --- |
| InvoiceController | api/Invoice |
| --- | --- |
| CustomerController | api/Customer |
| --- | --- |
| PaymentController | api/Payment |
| --- | --- |

## 2.2 ApiController Behavior

The \[ApiController\] attribute enables ASP.NET Core API-controller conventions, including automatic model-binding behavior and request validation for invalid models. Developers should still perform domain/result validation required by the operation.

# 3\. Standard Private Dependencies

A domain controller should keep only the dependencies it actually needs. Under the current pattern, the main dependency is RepositoryWrapper, with ILogger available for controller-level diagnostics. IConfiguration may be retained when the controller genuinely needs non-database application configuration.

#region privates  
<br/>private readonly RepositoryWrapper repository;  
private readonly IConfiguration configuration;  
private readonly ILogger \_logger;  
<br/>#endregion

**Connection-string boundary  
**Controllers must not read, select, store, log, or pass Oracle connection strings. Environment-specific DEV / TEST / PROD connection selection belongs to RepositoryWrapper/application infrastructure, as defined by the repository architecture.

## 3.1 RepositoryWrapper

RepositoryWrapper is the controller's entry point into the repository layer. The controller uses the domain property that corresponds to itself. EstimateController calls repository.Estimate; CustomerController calls repository.Customer; InvoiceController calls repository.Invoice, and so on.

## 3.2 ILogger

ILogger is available for controller-specific diagnostics such as malformed requests, unexpected result states, or request-processing failures when the application's logging standard requires it. Do not duplicate repository/database logging unnecessarily.

## 3.3 IConfiguration

The supplied EstimateController constructor receives IConfiguration. Configuration should only be retained in a controller when that controller needs application configuration unrelated to database connection management. If a controller does not use IConfiguration, it should be removed from that controller rather than stored without purpose.

# 4\. Controller Constructor Pattern

The constructor establishes the controller's dependencies. In the supplied example, the logger is provided by ASP.NET Core dependency injection and RepositoryWrapper is created for the controller.

public EstimateController(  
IConfiguration config,  
ILogger&lt;EstimateController&gt; logger)  
{  
configuration = config;  
repository = new RepositoryWrapper(logger);  
\_logger = logger;  
}

The RepositoryWrapper implementation determines how the OracleDAL and environment-specific connection configuration are established. The controller does not pass or inspect an Oracle connection string.

**Future DI refactoring  
**If RepositoryWrapper is later registered directly with ASP.NET Core dependency injection, the constructor can accept RepositoryWrapper instead of constructing it. That is an infrastructure change; the controller's endpoint and repository-consumption responsibilities remain the same.

# 5\. Endpoint Design Pattern

Each endpoint should perform four basic steps: receive a request DTO, delegate the operation to the matching domain repository, evaluate the result, and return the appropriate IActionResult.

| **Step** | **Developer Responsibility** |
| --- | --- |
| 1\. Receive | Use \[FromBody\], route, or query binding as appropriate for the operation. |
| --- | --- |
| 2\. Delegate | Call the matching method on repository.&lt;Domain&gt;. |
| --- | --- |
| 3\. Evaluate | Check the repository result or domain outcome, not merely the already-bound request object. |
| --- | --- |
| 4\. Respond | Return Ok, NotFound, BadRequest, or another intentional HTTP response. |
| --- | --- |

# 6\. Search/List Endpoint

Search/list operations receive a search DTO and delegate it directly to the domain repository. The repository owns the domain-specific search logic.

\[HttpPost("GetEstimates")\]  
public IActionResult GetEstimates(  
\[FromBody\] EstimateSearchDTO searchRequest)  
{  
var estimates =  
repository.Estimate.Search(searchRequest);  
<br/>if (estimates == null)  
{  
return NotFound();  
}  
<br/>return Ok(estimates);  
}

**List semantics  
**Teams should consistently decide whether 'no matching records' returns an empty collection with 200 OK or null/404. The current example uses NotFound when the repository returns null; new controllers should follow the application's established convention.

# 7\. Get-by-ID Endpoint

When the application standard uses RequestHeaderDTO to identify the target record, the controller accepts that DTO and passes it unchanged to the domain repository.

\[HttpPost("GetEstimateById")\]  
public IActionResult GetEstimateById(  
\[FromBody\] RequestHeaderDTO requestHeader)  
{  
var estimate =  
repository.Estimate.Get(requestHeader);  
<br/>if (estimate == null)  
{  
return NotFound();  
}  
<br/>return Ok(estimate);  
}

# 8\. Create Endpoint

Create endpoints receive the domain DTO, pass it to the domain repository, and evaluate the repository operation result.

\[HttpPost("CreateEstimate")\]  
public IActionResult CreateEstimate(  
\[FromBody\] EstimateDTO estimate)  
{  
var result =  
repository.Estimate.Create(estimate);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}

**Correction to the supplied sample  
**The supplied CreateEstimate example checks 'estimate == null' after calling repository.Estimate.Create(estimate). At that point the meaningful operation outcome is 'result'. The standard controller pattern should evaluate the repository result unless a specific request-validation rule requires otherwise.

# 9\. Update Endpoint

\[HttpPost("UpdateEstimate")\]  
public IActionResult UpdateEstimate(  
\[FromBody\] EstimateDTO estimate)  
{  
var result =  
repository.Estimate.Update(estimate);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}

As with create operations, evaluate the repository's returned result. Request DTO validation should occur before or as part of model/domain validation rather than by rechecking the same request after the repository call.

# 10\. Domain-Specific Action Endpoint

Controllers may expose operations that are specific to the domain and are not standard CRUD operations. These endpoints should still follow the same thin-controller pattern.

\[HttpPost("ConvertEstimateToInvoice")\]  
public IActionResult ConvertEstimateToInvoice(  
\[FromBody\] RequestHeaderDTO requestHeader)  
{  
var result =  
repository.Estimate  
.ConvertEstimateToInvoice(requestHeader);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}

The conversion business logic belongs in EstimateRepository (and the underlying BL/database layer), not in EstimateController. The controller simply exposes the operation over HTTP and converts the outcome into an HTTP response.

# 11\. Creating a New Domain Controller

When a new business domain is introduced, create a dedicated controller after the corresponding domain repository has been added to RepositoryWrapper.

## Step 1 - Create the Controller Class

\[Route("api/\[controller\]")\]  
\[ApiController\]  
public class PaymentController : ControllerBase  
{  
}

## Step 2 - Add Controller Dependencies

private readonly RepositoryWrapper repository;  
private readonly ILogger \_logger;

Add IConfiguration only if this controller uses non-database application configuration. Do not add IConfiguration solely to obtain the database connection string.

## Step 3 - Add the Constructor

public PaymentController(  
ILogger&lt;PaymentController&gt; logger)  
{  
repository = new RepositoryWrapper(logger);  
\_logger = logger;  
}

Use the actual RepositoryWrapper constructor defined by the application. Regardless of its internal constructor signature, the controller must not resolve or supply an Oracle connection string.

## Step 4 - Add Domain Endpoints

\[HttpPost("GetPayments")\]  
public IActionResult GetPayments(  
\[FromBody\] PaymentSearchDTO searchRequest)  
{  
var payments =  
repository.Payment.Search(searchRequest);  
<br/>if (payments == null)  
{  
return NotFound();  
}  
<br/>return Ok(payments);  
}

## Step 5 - Use Only the Matching Repository Domain

PaymentController should normally delegate to repository.Payment. If an operation requires coordination across multiple domains, that orchestration should be intentionally designed rather than having controllers freely reach into unrelated repositories.

# 12\. Full Controller Template

The following template can be used as the starting point for new domain controllers.

\[Route("api/\[controller\]")\]  
\[ApiController\]  
public class &lt;Domain&gt;Controller : ControllerBase  
{  
#region privates  
<br/>private readonly RepositoryWrapper repository;  
private readonly ILogger \_logger;  
<br/>#endregion  
<br/>#region constructors  
<br/>public &lt;Domain&gt;Controller(  
ILogger&lt;<Domain&gt;Controller> logger)  
{  
repository = new RepositoryWrapper(logger);  
\_logger = logger;  
}  
<br/>#endregion  
<br/>#region service methods  
<br/>\[HttpPost("Search")\]  
public IActionResult Search(  
\[FromBody\] &lt;Domain&gt;SearchDTO searchRequest)  
{  
var result =  
repository.&lt;Domain&gt;.Search(searchRequest);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}  
<br/>\[HttpPost("GetById")\]  
public IActionResult GetById(  
\[FromBody\] RequestHeaderDTO requestHeader)  
{  
var result =  
repository.&lt;Domain&gt;.Get(requestHeader);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}  
<br/>\[HttpPost("Create")\]  
public IActionResult Create(  
\[FromBody\] &lt;Domain&gt;DTO request)  
{  
var result =  
repository.&lt;Domain&gt;.Create(request);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}  
<br/>\[HttpPost("Update")\]  
public IActionResult Update(  
\[FromBody\] &lt;Domain&gt;DTO request)  
{  
var result =  
repository.&lt;Domain&gt;.Update(request);  
<br/>if (result == null)  
{  
return NotFound();  
}  
<br/>return Ok(result);  
}  
<br/>#endregion  
}

# 13\. Synchronous vs. Asynchronous Endpoints

The supplied EstimateController uses synchronous repository calls and returns IActionResult. That is correct when the underlying repository operation is synchronous. When a repository method is asynchronous, the controller endpoint should also be asynchronous and await it rather than blocking.

\[HttpPost("GetPayments")\]  
public async Task&lt;IActionResult&gt; GetPayments(  
\[FromBody\] PaymentSearchDTO searchRequest)  
{  
var payments =  
await repository.Payment.SearchAsync(searchRequest);  
<br/>if (payments == null)  
{  
return NotFound();  
}  
<br/>return Ok(payments);  
}

**Rule  
**Match the controller method to the repository method. Synchronous repository → IActionResult. Asynchronous repository → async Task&lt;IActionResult&gt; and await.

# 14\. HTTP Response Guidelines

| **Result** | **Typical Response** | **Use** |
| --- | --- | --- |
| Successful result | 200 OK | The operation completed and a result is returned. |
| --- | --- | --- |
| Record not found | 404 Not Found | A requested domain record does not exist, when this matches the API contract. |
| --- | --- | --- |
| Invalid request/domain input | 400 Bad Request | The request is syntactically valid but fails required input/domain validation. |
| --- | --- | --- |
| Created resource | 201 Created | Optional REST-style response when the API contract uses resource creation semantics. |
| --- | --- | --- |
| Unhandled failure | Central error handling | Do not convert every exception to 200/false; allow standardized exception middleware/handling to respond. |
| --- | --- | --- |

The exact status-code policy should be consistent across controllers. Developers should follow the established API contract rather than inventing different semantics per domain.

# 15\. Controller Design Rules

- Create one controller per business/domain area.
- Inherit from ControllerBase and apply \[ApiController\].
- Use \[Route("api/\[controller\]")\] unless a different application-wide route convention is explicitly defined.
- Use RepositoryWrapper as the repository-layer entry point.
- Delegate to the repository that matches the controller domain.
- Keep domain-specific BL and database interaction out of the controller.
- Do not read or manage Oracle connection strings in controllers.
- Do not create OracleConnection, OracleCommand, or OracleDAL directly in a controller.
- Use request/response DTOs as the API contract.
- Evaluate the repository result when deciding the HTTP response.
- Match async controller methods to async repository methods.
- Keep endpoint methods small and focused on HTTP concerns.
- Use ILogger for meaningful controller-level diagnostics, not duplicate logging noise.
- Remove unused injected dependencies from a controller.

# 16\. Common Mistakes

| **Mistake** | **Why It Is a Problem** | **Correct Pattern** |
| --- | --- | --- |
| Putting BL logic in the controller | Mixes HTTP concerns with domain implementation and makes reuse/testing harder. | Move domain-specific BL into the matching repository/service layer. |
| --- | --- | --- |
| Accessing the Oracle connection string | Breaks the configuration/security boundary. | RepositoryWrapper/application infrastructure owns environment-specific DB configuration. |
| --- | --- | --- |
| Creating OracleDAL directly | Bypasses the standardized repository architecture. | Use repository.&lt;Domain&gt; through RepositoryWrapper. |
| --- | --- | --- |
| Calling an unrelated repository from a controller | Creates cross-domain coupling. | Use the controller's matching repository or intentionally design orchestration. |
| --- | --- | --- |
| Checking the request DTO after the repository call | Does not validate the operation outcome. | Check the returned result/status from the repository. |
| --- | --- | --- |
| Using async repository calls without await | Can block or mishandle asynchronous work. | Use async Task&lt;IActionResult&gt; and await. |
| --- | --- | --- |
| Keeping unused IConfiguration | Adds unnecessary dependency/noise. | Inject configuration only when actually needed. |
| --- | --- | --- |
| Returning inconsistent status codes across domains | Creates a difficult API contract for consumers. | Follow common API response conventions. |
| --- | --- | --- |

# 17\. Developer Checklist for a New Controller

☐ The controller represents one clear domain.

☐ The matching domain repository already exists in RepositoryWrapper.

☐ The class inherits from ControllerBase.

☐ The class has \[ApiController\] and the standard route attribute.

☐ The controller has RepositoryWrapper and typed ILogger dependencies.

☐ IConfiguration is included only if the controller truly uses non-database configuration.

☐ No connection string or OracleDAL code exists in the controller.

☐ Each endpoint accepts an intentional request DTO/model.

☐ Each endpoint delegates to repository.&lt;MatchingDomain&gt;.

☐ The controller contains no domain-specific database implementation.

☐ The HTTP response is based on the repository result.

☐ Async repository operations are awaited by async controller methods.

☐ Routes are unique, clear, and consistent with the existing API.

☐ The controller follows the application's standard response/error conventions.

# 18\. Reference Architecture

Client / Consumer  
|  
| HTTP  
v  
Domain-Specific Controller  
|  
| request DTO  
v  
RepositoryWrapper  
|  
v  
Matching Domain Repository  
|  
| domain-specific BL / data operations  
v  
OracleDAL  
|  
v  
Oracle Database / PL/SQL  
<br/>Environment configuration:  
appsettings / approved secret provider  
|  
v  
RepositoryWrapper / application infrastructure  
<br/>The controller does NOT receive the Oracle connection string.

# 19\. Summary

**Developer contract  
**A domain controller is a thin HTTP boundary. Create one controller per domain, use RepositoryWrapper to reach the matching repository, pass request DTOs to repository methods, evaluate repository results, and return consistent HTTP responses. Keep business logic, OracleDAL usage, connection strings, and environment-specific database configuration out of controllers.