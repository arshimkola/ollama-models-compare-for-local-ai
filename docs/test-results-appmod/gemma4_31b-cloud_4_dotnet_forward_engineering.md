To modernize this legacy WCF service, I recommend using **Minimal APIs**. For a service of this scale, Minimal APIs provide lower overhead, better performance, and a cleaner mapping between HTTP verbs and business logic compared to traditional Controllers.

### Architecture Improvements:
1.  **Result Pattern**: Replaced the `Success/ErrorMessage` properties in every DTO with a generic `Result<T>` type to decouple business outcomes from data models.
2.  **Repository Pattern**: Decoupled the data access (EF Core) from the API endpoints.
3.  **Async/Await**: Converted all synchronous ADO.NET calls to asynchronous EF Core calls to prevent thread pool starvation.
4.  **FluentValidation**: Moved validation logic out of the business flow into dedicated validator classes.
5.  **Dependency Injection**: Replaced `ConfigurationManager` with `IConfiguration` and registered services via the .NET 8 DI container.

---

### 1. Core Models & Result Pattern

```csharp
using System.ComponentModel.DataAnnotations;

namespace CustomerApi.Models;

public record Customer {
    public int Id { get; init; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? Phone { get; set; }
    public string Tier { get; set; } = "BASIC";
    public DateTime CreatedDate { get; set; }
    public bool IsActive { get; set; }
}

public record CustomerSearchCriteria(string? Name, string? Email, string? Tier, bool? IsActive, int Page = 1, int PageSize = 20);
public record CreateCustomerRequest(string FirstName, string LastName, string Email, string? Phone, string? Tier);

// Result Pattern Implementation
public class Result<T> {
    public bool IsSuccess { get; private set; }
    public T? Value { get; private set; }
    public string? Error { get; private set; }
    public string? ErrorCode { get; private set; }

    protected Result(T value) { IsSuccess = true; Value = value; }
    protected Result(string error, string? errorCode = null) { IsSuccess = false; Error = error; ErrorCode = errorCode; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error, string? errorCode = null) => new(error, errorCode);
}
```

### 2. Data Layer (EF Core)

```csharp
using Microsoft.EntityFrameworkCore;

namespace CustomerApi.Data;

public class CustomerDbContext : DbContext {
    public CustomerDbContext(DbContextOptions<CustomerDbContext> options) : base(options) { }
    public DbSet<Models.Customer> Customers => Set<Models.Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder) {
        modelBuilder.Entity<Models.Customer>().HasIndex(c => c.Email).IsUnique();
    }
}

public interface ICustomerRepository {
    Task<Models.Customer?> GetByIdAsync(int id);
    Task<(List<Models.Customer> Items, int Total)> SearchAsync(CustomerSearchCriteria criteria);
    Task<int> CreateAsync(Models.Customer customer);
    Task<bool> EmailExistsAsync(string email);
}

public class CustomerRepository : ICustomerRepository {
    private readonly CustomerDbContext _context;

    public CustomerRepository(CustomerDbContext context) => _context = context;

    public async Task<Models.Customer?> GetByIdAsync(int id) => await _context.Customers.FindAsync(id);

    public async Task<(List<Models.Customer> Items, int Total)> SearchAsync(CustomerSearchCriteria criteria) {
        var query = _context.Customers.AsQueryable();

        if (!string.IsNullOrEmpty(criteria.Name))
            query = query.Where(c => c.FirstName.Contains(criteria.Name) || c.LastName.Contains(criteria.Name));
        if (!string.IsNullOrEmpty(criteria.Email))
            query = query.Where(c => c.Email == criteria.Email);
        if (!string.IsNullOrEmpty(criteria.Tier))
            query = query.Where(c => c.Tier == criteria.Tier);
        if (criteria.IsActive.HasValue)
            query = query.Where(c => c.IsActive == criteria.IsActive);

        var total = await query.CountAsync();
        var items = await query
            .OrderBy(c => c.LastName).ThenBy(c => c.FirstName)
            .Skip((criteria.Page - 1) * criteria.PageSize)
            .Take(criteria.PageSize)
            .ToListAsync();

        return (items, total);
    }

    public async Task<int> CreateAsync(Models.Customer customer) {
        _context.Customers.Add(customer);
        await _context.SaveChangesAsync();
        return customer.Id;
    }

    public async Task<bool> EmailExistsAsync(string email) => await _context.Customers.AnyAsync(c => c.Email == email);
}
```

### 3. Validation Layer

```csharp
using FluentValidation;

namespace CustomerApi.Validation;

public class CreateCustomerValidator : AbstractValidator<CreateCustomerRequest> {
    public CreateCustomerValidator() {
        RuleFor(x => x.Email).NotEmpty().EmailAddress().WithMessage("Valid email is required");
        RuleFor(x => x.FirstName).NotEmpty().WithMessage("First name is required");
        RuleFor(x => x.LastName).NotEmpty().WithMessage("Last name is required");
    }
}
```

### 4. Business Service Layer

```csharp
using CustomerApi.Data;
using CustomerApi.Models;
using Serilog;

namespace CustomerApi.Services;

public interface ICustomerService {
    Task<Result<Customer>> GetCustomerAsync(int id);
    Task<Result<List<Customer>>> SearchCustomersAsync(CustomerSearchCriteria criteria);
    Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request);
}

public class CustomerService : ICustomerService {
    private readonly ICustomerRepository _repo;
    private readonly ILogger<CustomerService> _logger;

    public CustomerService(ICustomerRepository repo, ILogger<CustomerService> logger) {
        _repo = repo;
        _logger = logger;
    }

    public async Task<Result<Customer>> GetCustomerAsync(int id) {
        try {
            var customer = await _repo.GetByIdAsync(id);
            return customer != null 
                ? Result<Customer>.Success(customer) 
                : Result<Customer>.Failure("Customer not found", "CUST_NOT_FOUND");
        }
        catch (Exception ex) {
            _logger.LogError(ex, "Error retrieving customer {Id}", id);
            return Result<Customer>.Failure("Database error occurred", "DB_ERROR");
        }
    }

    public async Task<Result<List<Customer>>> SearchCustomersAsync(CustomerSearchCriteria criteria) {
        try {
            var (items, _) = await _repo.SearchAsync(criteria);
            return Result<List<Customer>>.Success(items);
        }
        catch (Exception ex) {
            _logger.LogError(ex, "Error searching customers");
            return Result<List<Customer>>.Failure("Search failed");
        }
    }

    public async Task<Result<int>> CreateCustomerAsync(CreateCustomerRequest request) {
        try {
            if (await _repo.EmailExistsAsync(request.Email))
                return Result<int>.Failure("Email already exists", "DUPLICATE_EMAIL");

            var customer = new Customer {
                FirstName = request.FirstName,
                LastName = request.LastName,
                Email = request.Email,
                Phone = request.Phone,
                Tier = request.Tier ?? "BASIC",
                CreatedDate = DateTime.UtcNow,
                IsActive = true
            };

            var id = await _repo.CreateAsync(customer);
            return Result<int>.Success(id);
        }
        catch (Exception ex) {
            _logger.LogError(ex, "Error creating customer {Email}", request.Email);
            return Result<int>.Failure("Failed to create customer");
        }
    }
}
```

### 5. Program.cs & Endpoints

```csharp
using CustomerApi.Data;
using CustomerApi.Models;
using CustomerApi.Services;
using CustomerApi.Validation;
using FluentValidation;
using Microsoft.EntityFrameworkCore;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// 1. Serilog Setup
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("logs/customer-api.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();

// 2. Dependency Injection
builder.Services.AddDbContext<CustomerDbContext>(options => 
    options.UseSqlServer(builder.Configuration.GetConnectionString("CustomerDB")));

builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddScoped<ICustomerService, CustomerService>();
builder.Services.AddValidatorsFromAssemblyContaining<CreateCustomerValidator>();

var app = builder.Build();

// 3. Endpoints
var customerGroup = app.MapGroup("/api/customers");

customerGroup.MapGet("/{id}", async (int id, ICustomerService service) => {
    var result = await service.GetCustomerAsync(id);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound(result);
});

customerGroup.MapGet("/search", async (CustomerSearchCriteria criteria, ICustomerService service) => {
    var result = await service.SearchCustomersAsync(criteria);
    return result.IsSuccess ? Results.Ok(result.Value) : Results.BadRequest(result);
});

customerGroup.MapPost("/", async (CreateCustomerRequest request, IValidator<CreateCustomerRequest> validator, ICustomerService service) => {
    var validationResult = await validator.ValidateAsync(request);
    if (!validationResult.IsValid) 
        return Results.ValidationProblem(validationResult.ToDictionary());

    var result = await service.CreateCustomerAsync(request);
    return result.IsSuccess ? Results.Created($"/api/customers/{result.Value}", result.Value) : Results.Conflict(result);
});

app.Run();
```

### Key Mapping Summary:
| Legacy WCF/ADO.NET | Modern .NET 8 Implementation |
| :--- | :--- |
| `SqlConnection`/`SqlCommand` | `DbContext` / `IQueryable` |
| `ConfigurationManager` | `IConfiguration` (appsettings.json) |
| `EventLog.WriteEntry` | `Serilog` (Structured Logging) |
| `ServiceContract` $\to$ `OperationContract` | `MapGroup` $\to$ `MapGet/Post` |
| `Success`/`ErrorMessage` properties | `Result<T>` pattern |
| Manual `if (string.IsNullOrEmpty)` | `FluentValidation` |
| Synchronous methods | `async Task<T>` |
