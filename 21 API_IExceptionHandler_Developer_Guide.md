**API Exception Handling Architecture**

**IExceptionHandler Developer & Architect Guide**

Centralized exception handling for ASP.NET Core APIs

**Purpose  
**This document defines the application standard for API exception handling using ASP.NET Core IExceptionHandler. Developers throw meaningful application exceptions from controllers, repositories, or business logic and allow the ASP.NET Core exception-handling pipeline to convert them into consistent HTTP responses. Architects own exception-handler implementation, ordering, registration, middleware configuration, response standards, and cross-cutting logging behavior.

# 1\. Exception Handling Strategy

The API will use centralized exception handling instead of placing repetitive try/catch blocks around controller and repository methods. ASP.NET Core exception-handling middleware receives unhandled exceptions that propagate through the request pipeline and delegates them to registered IExceptionHandler implementations.

**Standard exception flow  
**Controller / Repository / BL Code → throw application exception → ASP.NET Core exception middleware → registered IExceptionHandler → standardized HTTP response

The initial application exception model contains three exception categories: NotFoundException, ValidationException, and GlobalFallbackException. Each exception represents a distinct API failure category and maps to a standardized HTTP response.

|     |     |     |     |
| --- | --- | --- | --- |
| **Exception** | **Meaning** | **HTTP Status** | **Typical Use** |
| NotFoundException | A requested business resource does not exist. | 404 Not Found | Requested estimate, invoice, customer, product, etc. cannot be found. |
| ValidationException | The request or business operation fails application validation. | 400 Bad Request | Required value missing, invalid state transition, invalid business rule, malformed domain input. |
| GlobalFallbackException | An unexpected application-level failure is intentionally surfaced through the global error path. | 500 Internal Server Error | Unexpected failure that cannot be represented as a known validation or not-found condition. |

# 2\. Why IExceptionHandler Is Used

- Creates one centralized exception-to-HTTP-response policy for the API.
- Removes repetitive try/catch blocks from controllers and repositories.
- Keeps controller methods focused on HTTP request/response behavior.
- Keeps repositories and business logic focused on domain behavior.
- Produces consistent error responses across every API controller.
- Centralizes exception logging and prevents each developer from implementing a different logging pattern.
- Makes it easier for architects to add or change exception policies without editing every endpoint.

**Important  
**IExceptionHandler does not mean try/catch is forbidden everywhere. Local try/catch is still appropriate when code can genuinely recover, retry, translate a third-party exception, clean up a non-managed resource, or add necessary context. It should not be used simply to catch an exception and return an HTTP response that the global handler already owns.

# 3\. Application Exception Classes

The application exceptions should be small, purpose-specific exception types. They carry the semantic meaning needed by the global exception handlers.

## 3.1 NotFoundException

public sealed class NotFoundException : Exception  
{  
public NotFoundException(string message)  
: base(message)  
{  
}  
}

## 3.2 ValidationException

public sealed class ValidationException : Exception  
{  
public ValidationException(string message)  
: base(message)  
{  
}  
}

## 3.3 GlobalFallbackException

public sealed class GlobalFallbackException : Exception  
{  
public GlobalFallbackException(string message)  
: base(message)  
{  
}  
<br/>public GlobalFallbackException(  
string message,  
Exception innerException)  
: base(message, innerException)  
{  
}  
}

**Naming caution  
**Do not confuse GlobalFallbackException with the fallback handler. GlobalFallbackException is an exception type that application code may throw. The GlobalFallbackExceptionHandler is the final registered handler and also acts as the catch-all safety net for any exception type not handled earlier.

# 4\. IExceptionHandler Implementations

Each handler implements IExceptionHandler.TryHandleAsync(HttpContext, Exception, CancellationToken). A handler returns true when it has produced the response and the exception is considered handled. It returns false when the exception is not its responsibility, allowing the next registered handler to evaluate it.

## 4.1 NotFoundExceptionHandler

public sealed class NotFoundExceptionHandler(  
ILogger&lt;NotFoundExceptionHandler&gt; logger)  
: IExceptionHandler  
{  
public async ValueTask&lt;bool&gt; TryHandleAsync(  
HttpContext httpContext,  
Exception exception,  
CancellationToken cancellationToken)  
{  
if (exception is not NotFoundException notFoundException)  
{  
return false;  
}  
<br/>logger.LogWarning(  
notFoundException,  
"Requested resource was not found.");  
<br/>ProblemDetails problem = new()  
{  
Status = StatusCodes.Status404NotFound,  
Title = "Resource Not Found",  
Detail = notFoundException.Message,  
Instance = httpContext.Request.Path  
};  
<br/>httpContext.Response.StatusCode = problem.Status.Value;  
<br/>await httpContext.Response.WriteAsJsonAsync(  
problem,  
cancellationToken);  
<br/>return true;  
}  
}

## 4.2 ValidationExceptionHandler

public sealed class ValidationExceptionHandler(  
ILogger&lt;ValidationExceptionHandler&gt; logger)  
: IExceptionHandler  
{  
public async ValueTask&lt;bool&gt; TryHandleAsync(  
HttpContext httpContext,  
Exception exception,  
CancellationToken cancellationToken)  
{  
if (exception is not ValidationException validationException)  
{  
return false;  
}  
<br/>logger.LogWarning(  
validationException,  
"Request or business validation failed.");  
<br/>ProblemDetails problem = new()  
{  
Status = StatusCodes.Status400BadRequest,  
Title = "Validation Failed",  
Detail = validationException.Message,  
Instance = httpContext.Request.Path  
};  
<br/>httpContext.Response.StatusCode = problem.Status.Value;  
<br/>await httpContext.Response.WriteAsJsonAsync(  
problem,  
cancellationToken);  
<br/>return true;  
}  
}

## 4.3 GlobalFallbackExceptionHandler

public sealed class GlobalFallbackExceptionHandler(  
ILogger&lt;GlobalFallbackExceptionHandler&gt; logger)  
: IExceptionHandler  
{  
public async ValueTask&lt;bool&gt; TryHandleAsync(  
HttpContext httpContext,  
Exception exception,  
CancellationToken cancellationToken)  
{  
// This is the final catch-all handler.  
// Any exception reaching this handler is treated as a 500.  
<br/>logger.LogError(  
exception,  
"Unhandled API exception. TraceId: {TraceId}",  
httpContext.TraceIdentifier);  
<br/>ProblemDetails problem = new()  
{  
Status = StatusCodes.Status500InternalServerError,  
Title = "An unexpected error occurred",  
Detail = "The request could not be completed.",  
Instance = httpContext.Request.Path  
};  
<br/>problem.Extensions\["traceId"\] =  
httpContext.TraceIdentifier;  
<br/>httpContext.Response.StatusCode = problem.Status.Value;  
<br/>await httpContext.Response.WriteAsJsonAsync(  
problem,  
cancellationToken);  
<br/>return true;  
}  
}

**Security rule  
**The global fallback response must not expose stack traces, database details, connection strings, SQL/PLSQL text, credentials, or internal exception messages to API consumers. Log diagnostic detail internally and return a safe public message plus a trace/correlation identifier.

# 5\. Architect-Owned Registration and Pipeline Configuration

Exception-handler registration is an application architecture concern and should be handled centrally by the architect or platform owner. Feature developers should not add or reorder global exception handlers from individual controllers or repositories.

var builder = WebApplication.CreateBuilder(args);  
<br/>builder.Services.AddControllers();  
builder.Services.AddProblemDetails();  
<br/>// Register most-specific handlers first.  
builder.Services.AddExceptionHandler&lt;NotFoundExceptionHandler&gt;();  
builder.Services.AddExceptionHandler&lt;ValidationExceptionHandler&gt;();  
<br/>// Catch-all MUST remain last.  
builder.Services.AddExceptionHandler&lt;GlobalFallbackExceptionHandler&gt;();  
<br/>var app = builder.Build();  
<br/>// Enable centralized exception handling before controller execution.  
app.UseExceptionHandler();  
<br/>app.MapControllers();  
<br/>app.Run();

**Registration order is part of the architecture  
**ASP.NET Core calls multiple IExceptionHandler implementations in the order they are registered. Specific handlers must therefore be registered before the catch-all handler. Because the fallback handler returns true for every exception it receives, any handler registered after it would never be reached.

## 5.1 Architect Responsibilities

- Define the approved application exception types and their HTTP status mappings.
- Implement the IExceptionHandler classes.
- Register handlers centrally in Program.cs or an architect-owned service-registration extension.
- Preserve handler order: specific handlers first, fallback last.
- Configure UseExceptionHandler in the API middleware pipeline.
- Define the standard error response contract, preferably ProblemDetails or an approved equivalent.
- Define logging severity and telemetry policy for handled versus unexpected exceptions.
- Ensure sensitive exception detail is not returned to production clients.
- Review requests for new exception types before they become part of the API contract.
- Own changes to global handler registration rather than allowing per-feature registration.

## 5.2 Handler Lifetime Consideration

ASP.NET Core registers IExceptionHandler implementations with singleton lifetime. Handler implementations should therefore avoid directly depending on request-scoped services. ILogger&lt;T&gt; and other singleton-safe dependencies are appropriate. If a handler ever needs scoped data, the architecture should explicitly resolve that requirement rather than casually injecting a scoped dependency into the handler.

# 6\. How Exceptions Are Caught Automatically

No controller-level try/catch is required for exceptions that should be handled by the global API policy. The exception simply propagates up the request pipeline.

\[HttpPost("GetEstimateById")\]  
public IActionResult GetEstimateById(  
\[FromBody\] RequestHeaderDTO requestHeader)  
{  
var estimate =  
repository.Estimate.Get(requestHeader);  
<br/>if (estimate == null)  
{  
throw new NotFoundException(  
$"Estimate {requestHeader.ID} was not found.");  
}  
<br/>return Ok(estimate);  
}

Execution flow for the example above:

- EstimateController calls repository.Estimate.Get(...).
- The estimate is not found.
- The controller throws NotFoundException.
- The exception leaves the controller method; no local catch handles it.
- UseExceptionHandler intercepts the unhandled request exception.
- NotFoundExceptionHandler recognizes the exception and writes a 404 response.
- TryHandleAsync returns true, so no later exception handler is invoked.

# 7\. Throwing Exceptions in Application Code

Developers select the exception type based on the business meaning of the failure, not based on the layer where it occurs.

## 7.1 Throw NotFoundException

EstimateDTO? estimate = repository.Estimate.Get(requestHeader);  
<br/>if (estimate == null)  
{  
throw new NotFoundException(  
$"Estimate {requestHeader.ID} was not found.");  
}

## 7.2 Throw ValidationException

if (estimate.ExpirationDate < estimate.IssueDate)  
{  
throw new ValidationException(  
"Expiration date cannot be earlier than issue date.");  
}

## 7.3 Throw GlobalFallbackException

if (!conversionResult.Success)  
{  
throw new GlobalFallbackException(  
"The estimate could not be converted to an invoice.");  
}

## 7.4 Wrap an Unexpected Lower-Level Exception When Context Is Needed

try  
{  
return externalService.Execute(request);  
}  
catch (ThirdPartyServiceException ex)  
{  
throw new GlobalFallbackException(  
"The external service operation failed.",  
ex);  
}

**Why this try/catch is acceptable  
**This catch is not being used to create an HTTP response. It translates a specific lower-level exception into an application exception and preserves the original exception as InnerException. The global IExceptionHandler still owns the HTTP response.

# 8\. Why Controller Try/Catch Blocks Are Normally Not Needed

Under this architecture, the following pattern should generally be avoided:

\[HttpPost("CreateEstimate")\]  
public IActionResult CreateEstimate(  
\[FromBody\] EstimateDTO estimate)  
{  
try  
{  
var result = repository.Estimate.Create(estimate);  
return Ok(result);  
}  
catch (ValidationException ex)  
{  
return BadRequest(ex.Message);  
}  
catch (NotFoundException ex)  
{  
return NotFound(ex.Message);  
}  
catch (Exception ex)  
{  
\_logger.LogError(ex, "CreateEstimate failed");  
return StatusCode(500);  
}  
}

Replace it with the much smaller controller method:

\[HttpPost("CreateEstimate")\]  
public IActionResult CreateEstimate(  
\[FromBody\] EstimateDTO estimate)  
{  
var result = repository.Estimate.Create(estimate);  
<br/>return Ok(result);  
}

If repository.Estimate.Create throws ValidationException, NotFoundException, GlobalFallbackException, or another unexpected exception, the exception pipeline handles it centrally. The controller does not need to know the HTTP mapping for every exception type.

# 9\. Repository / Business Logic Example

In many cases, the repository or business-logic layer is the best location to identify domain failures because it has the context required to classify the condition correctly.

public EstimateDTO Get(RequestHeaderDTO requestHeader)  
{  
EstimateDTO? estimate = GetEstimateFromDatabase(requestHeader.ID);  
<br/>if (estimate == null)  
{  
throw new NotFoundException(  
$"Estimate {requestHeader.ID} was not found.");  
}  
<br/>return estimate;  
}  
<br/>public EstimateDTO Create(EstimateDTO estimate)  
{  
if (string.IsNullOrWhiteSpace(estimate.CustomerName))  
{  
throw new ValidationException(  
"Customer name is required.");  
}  
<br/>return InsertEstimate(estimate);  
}

**Thin controller result  
**When the repository/business layer throws meaningful exceptions, the controller can often remain a simple delegate-and-return method. This keeps business validation out of the HTTP layer.

# 10\. Standardized API Error Responses

The handler layer should return a consistent error contract. ProblemDetails is a strong default because it is a standard ASP.NET Core API error representation.

## 10.1 Not Found - 404

{  
"type": null,  
"title": "Resource Not Found",  
"status": 404,  
"detail": "Estimate 1254 was not found.",  
"instance": "/api/Estimate/GetEstimateById"  
}

## 10.2 Validation - 400

{  
"type": null,  
"title": "Validation Failed",  
"status": 400,  
"detail": "Expiration date cannot be earlier than issue date.",  
"instance": "/api/Estimate/CreateEstimate"  
}

## 10.3 Global Fallback - 500

{  
"type": null,  
"title": "An unexpected error occurred",  
"status": 500,  
"detail": "The request could not be completed.",  
"instance": "/api/Estimate/ConvertEstimateToInvoice",  
"traceId": "00-..."  
}

# 11\. Handler Chain and Registration Order

Unhandled exception from request pipeline  
|  
v  
NotFoundExceptionHandler  
| true | false  
v v  
HTTP 404 ValidationExceptionHandler  
| true | false  
v v  
HTTP 400 GlobalFallbackExceptionHandler  
|  
v  
HTTP 500

The fallback handler must be the final handler because it deliberately handles anything that reaches it. Specific handlers return false when the exception type does not match, allowing the chain to continue.

# 12\. Adding a New Exception Type

Adding a new globally handled exception is an architect-governed change because it changes the API error contract. The recommended process is:

- Define the business meaning and determine whether an existing exception already covers the condition.
- Define the new exception class if a distinct category is warranted.
- Define its HTTP status and public ProblemDetails contract.
- Create a dedicated IExceptionHandler that returns false for all other exception types.
- Register the handler before GlobalFallbackExceptionHandler.
- Add automated tests for the exception-to-response mapping.
- Update developer documentation so feature teams know when to throw the new exception.

**Do not self-register from feature code  
**A feature developer should not add AddExceptionHandler&lt;T&gt;() calls from a controller, repository, or feature-specific startup path. Global exception policy belongs to central application composition.

# 13\. Testing Expectations

The exception architecture should be covered by both handler-level and API integration tests.

|     |     |
| --- | --- |
| **Test Scenario** | **Expected Result** |
| NotFoundException | Handler returns true; status 404; public detail matches approved message. |
| ValidationException | Handler returns true; status 400; validation detail is returned. |
| Unknown Exception | Specific handlers return false; fallback handles it; status 500; internal message is not leaked. |
| Registration Order | Specific exception is handled by its specific handler before fallback. |
| Trace ID | Fallback response includes the request trace identifier when configured. |

# 14\. Developer Rules

- Throw NotFoundException when a requested domain resource does not exist.
- Throw ValidationException when application/business validation fails.
- Throw GlobalFallbackException only when an operation needs to explicitly surface an unexpected application failure; otherwise truly unexpected exceptions may simply propagate to the fallback handler.
- Do not write repetitive controller try/catch blocks that translate these exceptions to HTTP responses.
- Do not expose stack traces, SQL, connection strings, credentials, or internal exception details to callers.
- Do not register IExceptionHandler implementations from feature code.
- Do not reorder the global handlers.
- Use a local try/catch only when there is a real local responsibility such as recovery, retry, cleanup, or exception translation.
- Preserve InnerException when translating lower-level failures.
- Let the global handler own the final HTTP error response.

# 15\. Developer Checklist

☐ I selected the exception based on the business meaning of the failure.

☐ I throw the exception rather than manually constructing the global HTTP error response.

☐ I did not add a try/catch solely to return BadRequest, NotFound, or 500.

☐ I did not add global exception-handler registration from my feature.

☐ I did not expose internal exception details to the API consumer.

☐ If I translated a lower-level exception, I preserved it as InnerException.

☐ My controller remains focused on request/response concerns.

☐ My repository/business layer throws meaningful domain exceptions where appropriate.

# 16\. Architect Checklist

☐ IExceptionHandler implementations are registered centrally.

☐ NotFoundExceptionHandler is registered before fallback.

☐ ValidationExceptionHandler is registered before fallback.

☐ GlobalFallbackExceptionHandler is last.

☐ app.UseExceptionHandler() is enabled before mapped controller execution.

☐ ProblemDetails/error response structure is consistent.

☐ Unexpected exception details are logged internally but sanitized externally.

☐ Handler dependencies are compatible with singleton handler lifetime.

☐ New exception categories are reviewed as API-contract changes.

☐ Integration tests verify 400, 404, and 500 behavior.

# 17\. ASP.NET Core Technical Notes

The architecture in this guide follows the ASP.NET Core IExceptionHandler model: handlers implement TryHandleAsync, multiple handlers can be registered with AddExceptionHandler&lt;T&gt;, handlers are invoked in registration order, and a handler indicates completion by returning true. IExceptionHandler registrations use singleton lifetime. UseExceptionHandler enables centralized exception handling in the HTTP pipeline.

Architects should verify framework-version-specific diagnostics behavior when upgrading ASP.NET Core. In newer framework versions, telemetry/logging behavior for exceptions marked handled by IExceptionHandler may differ, so explicit logging inside handlers should be treated as part of the application observability design.

# 18\. Summary

**Developer contract  
**Throw meaningful application exceptions and let them propagate. Do not duplicate global exception-to-HTTP translation in controllers or repositories. NotFoundException maps to 404, ValidationException maps to 400, and the global fallback path maps unexpected failures to a safe 500 response.

**Architect contract  
**Own the exception types, IExceptionHandler implementations, handler registration order, UseExceptionHandler pipeline configuration, logging policy, security of public error responses, and approval of future global exception categories.