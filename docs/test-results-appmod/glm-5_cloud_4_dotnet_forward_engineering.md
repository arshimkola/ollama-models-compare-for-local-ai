### Architectural Recommendation: Minimal API

I recommend using **Minimal API** over Controller-based API for this modernization.

**Rationale:**
1.  **Conciseness:** The legacy WCF service is relatively CRUD-oriented with distinct operations. Minimal APIs allow defining endpoints and their specific middleware (like validation) in a single, streamlined file, reducing boilerplate.
2.  **Performance:** Minimal APIs have less overhead than the traditional MVC/controller pattern, which aligns with the performance goals of .NET 8.
3.  **Modern Patterns:** .NET 8 trends heavily toward Minimal APIs with vertical slice architecture, making the code easier to maintain and test.

---

### Modernized Code Structure

Here is the complete solution organized into logical sections.

#### 1. Models and DTOs
We replace the WCF Message contracts with modern C# records and classes.

```csharp
// Models/Customer.cs
namespace CustomerService.Models;

public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? Phone { get; set; }
    public string Tier { get; set; } = "BASIC";
    public DateTime CreatedDate { get; set; }
    public bool IsActive { get; set; }
}

// DTOs/CustomerDtos.cs
namespace CustomerService.DTOs;

// Using records for immutability and structural equality
public record CustomerDto(
    int Id, 
    string FirstName, 
    string LastName, 
    string Email, 
    string? Phone, 
    string Tier, 
    DateTime CreatedDate, 
    bool IsActive
);

public record CustomerSearchCriteria(
    string? Name,
    string? Email,
    string? Tier,
    bool? IsActive,
    int Page = 1,
    int PageSize = 20
);

public record CreateCustomerRequest(
    string FirstName,
    string LastName,
    string Email,
    string? Phone,
    string? Tier
);
```

#### 2. Result Pattern
Instead of the legacy `Success` boolean and `ErrorMessage` properties on response objects, we use a functional Result pattern to enforce explicit error handling.

```csharp
// Common/Result.cs
namespace CustomerService.Common;

public abstract class Result
{
    public bool IsSuccess { get; protected set; }
    public string? Error { get; protected set; }
    public string? ErrorCode { get; protected set; }

    public static Result Success() => new SuccessResult();
    public static Result Failure(string error, string? code = null) => new ErrorResult(error, code);
}

public abstract class Result<T> : Result
{
    public T? Value { get; protected set; }

    public static Result<T> Success(T value) => new SuccessResult<T>(value);
    public new static Result<T> Failure(string error, string? code = null) => new ErrorResult<T>(error, code);
}

// Internal implementations
file record SuccessResult : Result { public SuccessResult() => IsSuccess = true; }
file record SuccessResult<T>(T Value) : Result { public SuccessResult(T value) { Value = value; IsSuccess = true; } }
file record ErrorResult(string Error, string? Code) : Result { public ErrorResult(string e, string? c) { Error = e; ErrorCode = c; IsSuccess = false; } }
file record ErrorResult<T>(string Error, string? Code) : Result { public ErrorResult(string e, string? c) { Error = e; ErrorCode = c; IsSuccess = false; } }
```

#### 3. FluentValidation
We replace the manual `if (string.IsNullOrEmpty)` checks in the legacy service with robust validators.

```csharp
// Validators/CreateCustomerRequestValidator.cs
using CustomerService.DTOs;
using FluentValidation;

namespace CustomerService.Validators;

public class CreateCustomerRequestValidator : AbstractValidator<CreateCustomerRequest>
{
    public CreateCustomerRequestValidator()
    {
        RuleFor(x => x.FirstName).NotEmpty().WithMessage("First name is required.");
        RuleFor(x => x.LastName).NotEmpty().WithMessage("Last name is required.");
        RuleFor(x => x.Email).NotEmpty().EmailAddress().WithMessage("A valid email is required.");
        RuleFor(x => x.Tier).Must(t => t == null || new[] { "BASIC", "PREMIUM", "GOLD" }.Contains(t))
            .WithMessage("Invalid customer tier.");
    }
}
```

#### 4. Data Access (EF Core Repository)
We replace `SqlConnection` and `SqlCommand` with Entity Framework Core.

```csharp
// Data/CustomerDbContext.cs
using CustomerService.Models;
using Microsoft.EntityFrameworkCore;

namespace CustomerService.Data;

public class CustomerDbContext : DbContext
{
    public CustomerDbContext(DbContextOptions<CustomerDbContext> options) : base(options) { }
    public DbSet<Customer> Customers { get; set; }
}

// Repositories/ICustomerRepository.cs
using CustomerService.Common;
using CustomerService.DTOs;
using CustomerService.Models;

namespace CustomerService.Repositories;

public interface ICustomerRepository
{
    Task<Result<CustomerDto>> GetByIdAsync(int id);
    Task<(List<CustomerDto> Customers, int TotalCount)> SearchAsync(CustomerSearchCriteria criteria);
    Task<Result<int>> CreateAsync(Customer customer);
    Task<bool> EmailExistsAsync(string email);
}
```

#### 5. Business Logic (Service Layer)
The `CustomerService` class now orchestrates validation and repository calls, replacing the original WCF class.

```csharp
// Services/CustomerService.cs
using CustomerService.Common;
using CustomerService.Data;
using CustomerService.DTOs;
using CustomerService.Models;
using CustomerService.Repositories;
using Microsoft.EntityFrameworkCore;

namespace CustomerService.Services;

public class CustomerService : ICustomerService
{
    private readonly ICustomerRepository _repository;
    private readonly ILogger<CustomerService> _logger;

    public CustomerService(ICustomerRepository repository, ILogger<CustomerService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Result<CustomerDto>> GetCustomerAsync(int id)
    {
        try
        {
            var result = await _repository.GetByIdAsync(id);
            if (!result.IsSuccess)
            {
                _logger.LogWarning("Customer lookup failed for ID {CustomerId}: {Error}", id, result.Error);
            }
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database error occurred while retrieving customer {CustomerId}", id);
            return Result<CustomerDto>.Failure("An internal error occurred.", "DB_ERROR");
        }
    }

    public async Task<Result<List<CustomerDto>>> SearchCustomersAsync(CustomerSearchCriteria criteria)
    {
        try
        {
            var (customers, _) = await _repository.SearchAsync(criteria);
            return Result<List<CustomerDto>>.Success(customers);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database error occurred during customer search");
            return Result<List<CustomerDto>>.Failure("Search failed due to an internal error.", "DB_ERROR");
        }
    }

    public async Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request)
    {
        // Business Logic: Check duplicates (handled in service layer, not DB constraints necessarily)
        if (await _repository.EmailExistsAsync(request.Email))
        {
            return Result<int>.Failure("Email already exists.", "DUPLICATE_EMAIL");
        }

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

        try
        {
            var newId = await _repository.CreateAsync(customer);
            return Result<int>.Success(newId.Value);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create customer for email {Email}", request.Email);
            return Result<int>.Failure("Failed to create customer.", "CREATE_FAILED");
        }
    }
}

public interface ICustomerService
{
    Task<Result<CustomerDto>> GetCustomerAsync(int id);
    Task<Result<List<CustomerDto>>> SearchCustomersAsync(CustomerSearchCriteria criteria);
    Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request);
}
```

#### 6. Infrastructure (Repository Implementation)
Encapsulating the EF Core logic.

```csharp
// Repositories/CustomerRepository.cs
using CustomerService.Common;
using CustomerService.Data;
using CustomerService.DTOs;
using CustomerService.Models;
using Microsoft.EntityFrameworkCore;

namespace CustomerService.Repositories;

public class CustomerRepository : ICustomerRepository
{
    private readonly CustomerDbContext _context;

    public CustomerRepository(CustomerDbContext context) => _context = context;

    public async Task<Result<CustomerDto>> GetByIdAsync(int id)
    {
        var customer = await _context.Customers.FindAsync(id);
        if (customer == null) return Result<CustomerDto>.Failure("Customer not found", "CUST_NOT_FOUND");

        return Result<CustomerDto>.Success(MapToDto(customer));
    }

    public async Task<(List<CustomerDto>, int)> SearchAsync(CustomerSearchCriteria criteria)
    {
        var query = _context.Customers.AsQueryable();

        if (!string.IsNullOrEmpty(criteria.Name))
            query = query.Where(c => c.FirstName.Contains(criteria.Name) || c.LastName.Contains(criteria.Name));

        if (!string.IsNullOrEmpty(criteria.Email))
            query = query.Where(c => c.Email == criteria.Email);

        if (!string.IsNullOrEmpty(criteria.Tier))
            query = query.Where(c => c.Tier == criteria.Tier);

        if (criteria.IsActive.HasValue)
            query = query.Where(c => c.IsActive == criteria.IsActive);

        // Pagination
        var total = await query.CountAsync();
        var results = await query
            .OrderBy(c => c.LastName).ThenBy(c => c.FirstName)
            .Skip((criteria.Page - 1) * criteria.PageSize)
            .Take(criteria.PageSize)
            .Select(c => MapToDto(c))
            .ToListAsync();

        return (results, total);
    }

    public async Task<Result<int>> CreateAsync(Customer customer)
    {
        _context.Customers.Add(customer);
        await _context.SaveChangesAsync();
        return Result<int>.Success(customer.Id);
    }

    public async Task<bool> EmailExistsAsync(string email)
    {
        return await _context.Customers.AnyAsync(c => c.Email == email);
    }

    private static CustomerDto MapToDto(Customer c) => 
        new CustomerDto(c.Id, c.FirstName, c.LastName, c.Email, c.Phone, c.Tier, c.CreatedDate, c.IsActive);
}
```

#### 7. Program.cs (Composition Root)
Wiring everything together using Minimal API syntax.

```csharp
using CustomerService.Common;
using CustomerService.Data;
using CustomerService.DTOs;
using CustomerService.Repositories;
using CustomerService.Services;
using CustomerService.Validators;
using FluentValidation;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// 1. Configure Serilog
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext());

// 2. Register Services
builder.Services.AddDbContext<CustomerDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("CustomerDB")));

builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddScoped<ICustomerService, CustomerService>();

// Register Validators
builder.Services.AddValidatorsFromAssemblyContaining<CreateCustomerRequestValidator>();

var app = builder.Build();

// 3. Define Endpoints (Minimal API)

// GET /customers/{id}
app.MapGet("/customers/{id:int}", async (int id, ICustomerService service) =>
{
    var result = await service.GetCustomerAsync(id);
    
    return result.IsSuccess 
        ? Results.Ok(result.Value) 
        : Results.NotFound(new { result.ErrorCode, result.Error });
});

// POST /customers/search
app.MapPost("/customers/search", async (CustomerSearchCriteria criteria, ICustomerService service) =>
{
    var result = await service.SearchCustomersAsync(criteria);
    return Results.Ok(result.Value);
});

// POST /customers
app.MapPost("/customers", async (CreateCustomerRequest request, IValidator<CreateCustomerRequest> validator, ICustomerService service) =>
{
    // Validation
    var validation = await validator.ValidateAsync(request);
    if (!validation.IsValid)
        return Results.BadRequest(validation.Errors);

    // Logic
    var result = await service.CreateCustomerAsync(request);
    
    return result.IsSuccess 
        ? Results.Created($"/customers/{result.Value}", result.Value) 
        : Results.Conflict(new { result.ErrorCode, result.Error });
});

app.Run();
```

### Key Modernization Highlights

1.  **Structured Logging**: Replaced `EventLog.WriteEntry` with `Serilog`. The logger is injected via DI, allowing for structured logging (e.g., `LogError(ex, "Message {Id}", id)`) which is queryable in modern logging stacks (ELK, Seq, Application Insights).
2.  **Async/Await**: The legacy code used blocking ADO.NET calls (`ExecuteScalar`, `Read`). The new code uses `await` all the way down to the database context (`FindAsync`, `ToListAsync`), ensuring thread safety and scalability.
3.  **Result Pattern**: Removed `try/catch` blocks for business logic flow (like "Customer Not Found") from the endpoint middleware. The service layer returns a `Result<T>` object, allowing the API layer to simply map success/failure to appropriate HTTP Status Codes (200, 404, 409).
4.  **Separation of Concerns**:
    *   **Validators**: Handle input sanity checks.
    *   **Service**: Handles business rules (e.g., "Email is required", "Duplicate check").
    *   **Repository**: Handles SQL generation/EF Core logic.
5.  **Minimal API**: Eliminates the boilerplate of Controllers and WCF ServiceContracts/ServiceBehaviors, keeping the codebase concise.
