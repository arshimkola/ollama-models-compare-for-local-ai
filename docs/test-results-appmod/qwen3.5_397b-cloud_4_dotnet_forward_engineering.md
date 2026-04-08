### Modernization Strategy & Rationale

**Architecture Choice: Minimal APIs**
I recommend **Minimal APIs** over Controllers for this scenario.
*   **Rationale:** The legacy service is a straightforward CRUD-style service. Minimal APIs in .NET 8 offer reduced boilerplate, better performance, and a clearer linear flow for small-to-medium services.
*   **Separation of Concerns:** To prevent "spaghetti code" in `Program.cs`, business logic is moved to a **Service Layer**, and routing is organized into **Endpoint Groups**.

**Key Modernizations:**
1.  **ADO.NET → EF Core:** Replaces manual SQL/Connection management with type-safe LINQ and automatic connection pooling.
2.  **WCF → REST:** `OperationContract` becomes standard HTTP verbs (GET, POST).
3.  **Error Handling:** Replaces `try/catch` returning error codes with the **Result Pattern** and standard HTTP Status Codes (200, 400, 404, 500).
4.  **Validation:** Replaces manual `if` checks with **FluentValidation** pipelines.
5.  **Logging:** Replaces Windows Event Log with **Serilog** (structured JSON logging).
6.  **Async:** All I/O operations are now non-blocking (`async/await`).

---

### 1. Project Setup & Dependencies

**NuGet Packages:**
```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />
<PackageReference Include="FluentValidation" Version="11.9.0" />
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.9.0" />
<PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
```

---

### 2. Domain & DTOs

**Domain/Entities/Customer.cs**
```csharp
namespace ModernCustomerService.Domain.Entities;

public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? Phone { get; set; }
    public string Tier { get; set; } = "BASIC";
    public DateTime CreatedDate { get; set; }
    public bool IsActive { get; set; } = true;
}
```

**DTOs/Requests.cs**
```csharp
namespace ModernCustomerService.DTOs;

public record CreateCustomerRequest(string FirstName, string LastName, string Email, string? Phone, string? Tier);
public record SearchCriteria(string? Name, string? Email, string? Tier, bool? IsActive, int Page, int PageSize);
```

**DTOs/Responses.cs**
```csharp
namespace ModernCustomerService.DTOs;

public record CustomerDto(int Id, string FirstName, string LastName, string Email, string? Phone, string Tier, DateTime CreatedDate, bool IsActive);
public record CustomerListResponse(IEnumerable<CustomerDto> Customers, int TotalCount);
// We use the Result pattern for operation outcomes instead of embedding Success/ErrorCode in every DTO
```

---

### 3. Common: Result Pattern

**Common/Result.cs**
```csharp
namespace ModernCustomerService.Common;

public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? ErrorMessage { get; }
    public string? ErrorCode { get; }

    private Result(bool isSuccess, T? value, string? errorMessage, string? errorCode)
    {
        IsSuccess = isSuccess;
        Value = value;
        ErrorMessage = errorMessage;
        ErrorCode = errorCode;
    }

    public static Result<T> Success(T value) => new(true, value, null, null);
    public static Result<T> Failure(string errorMessage, string errorCode) => new(false, default, errorMessage, errorCode);
}

// Non-generic helper for void returns
public class Result
{
    public bool IsSuccess { get; }
    public string? ErrorMessage { get; }
    public string? ErrorCode { get; }

    private Result(bool isSuccess, string? errorMessage, string? errorCode)
    {
        IsSuccess = isSuccess;
        ErrorMessage = errorMessage;
        ErrorCode = errorCode;
    }

    public static Result Success() => new(true, null, null);
    public static Result Failure(string errorMessage, string errorCode) => new(false, errorMessage, errorCode);
}
```

---

### 4. Data Layer (EF Core)

**Data/CustomerDbContext.cs**
```csharp
using Microsoft.EntityFrameworkCore;
using ModernCustomerService.Domain.Entities;

namespace ModernCustomerService.Data;

public class CustomerDbContext : DbContext
{
    public CustomerDbContext(DbContextOptions<CustomerDbContext> options) : base(options) { }

    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Customer>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.HasIndex(e => e.Email).IsUnique(); // Enforce duplicate check at DB level
            entity.Property(e => e.Tier).HasDefaultValue("BASIC");
        });
    }
}
```

---

### 5. Validation (FluentValidation)

**Validators/CreateCustomerRequestValidator.cs**
```csharp
using FluentValidation;
using ModernCustomerService.DTOs;

namespace ModernCustomerService.Validators;

public class CreateCustomerRequestValidator : AbstractValidator<CreateCustomerRequest>
{
    public CreateCustomerRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format");

        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First Name is required");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last Name is required");
    }
}
```

---

### 6. Service Layer (Business Logic)

**Services/ICustomerService.cs & CustomerService.cs**
*Encapsulates logic previously in the WCF class.*

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using ModernCustomerService.Common;
using ModernCustomerService.Data;
using ModernCustomerService.Domain.Entities;
using ModernCustomerService.DTOs;

namespace ModernCustomerService.Services;

public interface ICustomerService
{
    Task<Result<CustomerDto>> GetCustomerByIdAsync(int customerId);
    Task<Result<CustomerListResponse>> SearchCustomersAsync(SearchCriteria criteria);
    Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request);
}

public class CustomerService : ICustomerService
{
    private readonly CustomerDbContext _dbContext;
    private readonly ILogger<CustomerService> _logger;

    public CustomerService(CustomerDbContext dbContext, ILogger<CustomerService> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task<Result<CustomerDto>> GetCustomerByIdAsync(int customerId)
    {
        try
        {
            var customer = await _dbContext.Customers
                .AsNoTracking()
                .FirstOrDefaultAsync(c => c.Id == customerId);

            if (customer == null)
            {
                _logger.LogWarning("Customer {CustomerId} not found", customerId);
                return Result<CustomerDto>.Failure("Customer not found", "CUST_NOT_FOUND");
            }

            return Result<CustomerDto>.Success(MapToDto(customer));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database error occurred while fetching customer {CustomerId}", customerId);
            return Result<CustomerDto>.Failure("Database error occurred", "DB_ERROR");
        }
    }

    public async Task<Result<CustomerListResponse>> SearchCustomersAsync(SearchCriteria criteria)
    {
        try
        {
            var query = _dbContext.Customers.AsNoTracking();

            if (!string.IsNullOrEmpty(criteria.Name))
            {
                query = query.Where(c => c.FirstName.Contains(criteria.Name) || c.LastName.Contains(criteria.Name));
            }
            if (!string.IsNullOrEmpty(criteria.Email))
            {
                query = query.Where(c => c.Email == criteria.Email);
            }
            if (!string.IsNullOrEmpty(criteria.Tier))
            {
                query = query.Where(c => c.Tier == criteria.Tier);
            }
            if (criteria.IsActive.HasValue)
            {
                query = query.Where(c => c.IsActive == criteria.IsActive.Value);
            }

            // Ordering and Pagination
            query = query.OrderBy(c => c.LastName).ThenBy(c => c.FirstName);
            
            var totalCount = await query.CountAsync();
            var customers = await query
                .Skip((criteria.Page - 1) * criteria.PageSize)
                .Take(criteria.PageSize)
                .ToListAsync();

            var dtos = customers.Select(MapToDto);

            return Result<CustomerListResponse>.Success(new CustomerListResponse(dtos, totalCount));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Search failed for criteria {@Criteria}", criteria);
            return Result<CustomerListResponse>.Failure("Search failed", "SEARCH_ERROR");
        }
    }

    public async Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request)
    {
        // Duplicate check (EF Core Unique Index will also catch this, but we return specific error)
        var exists = await _dbContext.Customers.AnyAsync(c => c.Email == request.Email);
        if (exists)
        {
            return Result<int>.Failure("Email already exists", "DUPLICATE_EMAIL");
        }

        try
        {
            var customer = new Customer
            {
                FirstName = request.FirstName,
                LastName = request.LastName,
                Email = request.Email,
                Phone = request.Phone,
                Tier = request.Tier ?? "BASIC",
                CreatedDate = DateTime.UtcNow,
                IsActive = true
            };

            _dbContext.Customers.Add(customer);
            await _dbContext.SaveChangesAsync();

            _logger.LogInformation("Customer created with Id {CustomerId}", customer.Id);
            return Result<int>.Success(customer.Id);
        }
        catch (DbUpdateException ex)
        {
            // Handle unique constraint violation specifically if needed
            _logger.LogError(ex, "Failed to create customer");
            return Result<int>.Failure("Failed to create customer", "DB_ERROR");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error creating customer");
            return Result<int>.Failure("Failed to create customer", "UNKNOWN_ERROR");
        }
    }

    private static CustomerDto MapToDto(Customer c) => new(
        c.Id, c.FirstName, c.LastName, c.Email, c.Phone, c.Tier, c.CreatedDate, c.IsActive
    );
}
```

---

### 7. Endpoints (Minimal API)

**Endpoints/CustomerEndpoints.cs**
*Maps Service Results to HTTP Status Codes.*

```csharp
using ModernCustomerService.Common;
using ModernCustomerService.DTOs;
using ModernCustomerService.Services;
using ModernCustomerService.Validators;

namespace ModernCustomerService.Endpoints;

public static class CustomerEndpoints
{
    public static void MapCustomerEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/customers").WithTags("Customers");

        // GET /api/customers/{id}
        group.MapGet("/{id:int}", async (int id, ICustomerService service) =>
        {
            var result = await service.GetCustomerByIdAsync(id);
            return result.IsSuccess 
                ? Results.Ok(result.Value) 
                : Results.NotFound(new { result.ErrorMessage, result.ErrorCode });
        })
        .WithName("GetCustomer")
        .WithOpenApi();

        // POST /api/customers/search
        group.MapPost("/search", async (SearchCriteria criteria, ICustomerService service) =>
        {
            // Default pagination if not provided
            if (criteria.Page < 1) criteria = criteria with { Page = 1 };
            if (criteria.PageSize < 1 || criteria.PageSize > 100) criteria = criteria with { PageSize = 10 };

            var result = await service.SearchCustomersAsync(criteria);
            return result.IsSuccess 
                ? Results.Ok(result.Value) 
                : Results.StatusCode(500, new { result.ErrorMessage, result.ErrorCode });
        })
        .WithName("SearchCustomers")
        .WithOpenApi();

        // POST /api/customers
        group.MapPost("/", async (CreateCustomerRequest request, ICustomerService service, FluentValidation.IValidator<CreateCustomerRequest> validator) =>
        {
            var validationResult = await validator.ValidateAsync(request);
            if (!validationResult.IsValid)
            {
                return Results.BadRequest(validationResult.Errors);
            }

            var result = await service.CreateCustomerAsync(request);
            return result.IsSuccess 
                ? Results.Created($"/api/customers/{result.Value}", new { result.Value }) 
                : Results.Conflict(new { result.ErrorMessage, result.ErrorCode });
        })
        .WithName("CreateCustomer")
        .WithOpenApi();
    }
}
```

---

### 8. Program.cs (Composition Root)

```csharp
using FluentValidation;
using Microsoft.EntityFrameworkCore;
using ModernCustomerService.Data;
using ModernCustomerService.Endpoints;
using ModernCustomerService.Services;
using ModernCustomerService.Validators;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// 1. Structured Logging (Serilog)
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateLogger();

builder.Host.UseSerilog();

// 2. Database Context
builder.Services.AddDbContext<CustomerDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("CustomerDB")));

// 3. Services
builder.Services.AddScoped<ICustomerService, CustomerService>();

// 4. Validation
builder.Services.AddValidatorsFromAssemblyContaining<CreateCustomerRequestValidator>();

// 5. API Configuration
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// 6. Middleware Pipeline
app.UseSerilogRequestLogging();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Global Exception Handling (Fallback for unexpected errors)
app.UseExceptionHandler("/error");

app.UseHttpsRedirection();

// 7. Endpoint Registration
app.MapCustomerEndpoints();

// Health Check
app.MapGet("/health", () => Results.Ok("OK"));

app.Run();
```

---

### 9. appsettings.json Example

```json
{
  "ConnectionStrings": {
    "CustomerDB": "Server=localhost;Database=CustomerDb;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Serilog": {
    "MinimumLevel": "Information",
    "WriteTo": [
      { "Name": "Console" }
    ]
  },
  "AllowedHosts": "*"
}
```

### Summary of Improvements

| Feature | Legacy WCF | Modern .NET 8 | Benefit |
| :--- | :--- | :--- | :--- |
| **Communication** | SOAP/XML (WCF) | JSON/REST (Minimal API) | Interoperable, lighter payload, web-friendly. |
| **Data Access** | ADO.NET (Manual SQL) | EF Core (LINQ) | Type safety, less boilerplate, easier refactoring. |
| **Validation** | Manual `if` checks | FluentValidation | Decoupled validation, reusable rules, cleaner logic. |
| **Error Handling** | Custom Error Codes | HTTP Status + Result Pattern | Standard web semantics, clear success/failure flow. |
| **Logging** | Windows Event Log | Serilog (Structured) | Queryable logs, cloud-ready, performance. |
| **Concurrency** | Synchronous (Blocking) | Asynchronous (`async/await`) | Better scalability under load. |
| **Config** | `web.config` | `appsettings.json` + DI | Environment-agnostic, testable configuration. |

This solution preserves the exact business rules (validation logic, search filtering, duplicate checking) while completely modernizing the infrastructure for maintainability and cloud readiness.
