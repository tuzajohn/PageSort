# PageSort.Core

[![NuGet](https://img.shields.io/nuget/v/PageSort.Core.svg)](https://www.nuget.org/packages/PageSort.Core)
[![CI/CD](https://github.com/PageSort/PageSort/actions/workflows/nuget.yml/badge.svg)](https://github.com/PageSort/PageSort/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE.txt)

A lightweight, zero-dependency .NET library for **paging**, **sorting**, **filtering**, and **dynamic field selection** on `IQueryable<T>` collections. Built for APIs that need runtime-controlled query shaping.

## Target Frameworks

| Framework | Supported |
|-----------|-----------|
| .NET 9 | Yes |
| .NET 8 | Yes |
| .NET 7 | Yes |
| .NET 6 | Yes |

---

## Installation

```shell
dotnet add package PageSort.Core
```

Or via the Package Manager Console:

```
Install-Package PageSort.Core
```

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Paging](#paging)
- [Sorting](#sorting)
- [Filtering](#filtering)
- [Dynamic Field Selection](#dynamic-field-selection)
- [Sensitive Data Protection](#sensitive-data-protection)
- [Mapping to DTOs](#mapping-to-dtos)
- [Async Support](#async-support)
- [API Reference](#api-reference)
- [ASP.NET Core Integration](#aspnet-core-integration)
- [License](#license)

---

## Core Concepts

### PageQuery

The base request object for paging and sorting:

```csharp
public class PageQuery
{
    public int PageNumber { get; set; }      // Auto-corrects to 1 if <= 0
    public int PageSize { get; set; }
    public string? SortProperty { get; set; }
    public ListSortDirection? SortDirection { get; set; } // Default: Ascending
}
```

### AdvancedPageQuery

Extends `PageQuery` with filtering and field selection:

```csharp
public sealed class AdvancedPageQuery : PageQuery
{
    public List<Filter> Filters { get; set; } = [];
    public string[]? Fields { get; set; } = null;
}
```

### PagedResult<T>

The response envelope returned by all paging operations:

| Property | Description |
|----------|-------------|
| `CurrentPage` | The 1-based page number being returned |
| `PageSize` | Number of items per page |
| `TotalCount` | Total items in the source (post-filter) |
| `TotalPages` | Calculated ceiling of `TotalCount / PageSize` |
| `Collection` | The materialized items for the current page |
| `PreviousPage` | `true` if `CurrentPage > 1` |
| `NextPage` | `true` if `CurrentPage < TotalPages` |

---

## Paging

### Extension Method

```csharp
using PageSort.Core.Extensions;

// Returns items 21-40 (page 2, 20 per page)
IQueryable<Student> page = students.Page(pageNumber: 2, pageSize: 20);
```

**Validation:**
- `pageNumber <= 0` throws `ArgumentException`
- `pageSize < 0` throws `ArgumentException`

### Using Page<T> Helper

```csharp
using PageSort.Core;

var query = new PageQuery { PageNumber = 1, PageSize = 25 };

PagedResult<Student> result = Page<Student>.GeneratePaging(dbContext.Students, query);
// result.TotalCount, result.TotalPages, result.Collection, etc.
```

---

## Sorting

### OrderByProperty Extension

Sort any `IQueryable<T>` by a property name string at runtime:

```csharp
using PageSort.Core.Extensions;
using System.ComponentModel;

// Ascending (default)
var sorted = students.OrderByProperty("LastName");

// Descending
var sorted = students.OrderByProperty("DateOfBirth", ListSortDirection.Descending);
```

### Combined Paging + Sorting

```csharp
var query = new PageQuery
{
    PageNumber = 1,
    PageSize = 20,
    SortProperty = "GPA",
    SortDirection = ListSortDirection.Descending
};

PagedResult<Student> result = Page<Student>.GeneratePaging(students, query);
// Returns: top 20 students by GPA, with full pagination metadata
```

---

## Filtering

### Creating Filters

Filters are created through the `Filter.Create` factory which validates operator-value compatibility:

```csharp
using PageSort.Core;

// Comparison filters (numeric/date/IComparable)
var ageFilter = Filter.Create("Age", "GreaterThan", 18);
var dateFilter = Filter.Create("EnrollmentDate", "LessThan", DateTime.Parse("2024-01-01"));

// String filters
var nameFilter = Filter.Create("FirstName", "Contains", "John");
var emailFilter = Filter.Create("Email", "EndsWith", "gmail");

// Equality (all types)
var statusFilter = Filter.Create("IsActive", "Equals", true);
```

### Supported Operators

| Operator | Valid For | Example |
|----------|-----------|---------|
| `Equals` | All types | `Filter.Create("Status", "Equals", true)` |
| `NotEquals` | All types | `Filter.Create("Role", "NotEquals", "Admin")` |
| `GreaterThan` | IComparable | `Filter.Create("Age", "GreaterThan", 21)` |
| `GreaterThanOrEqual` | IComparable | `Filter.Create("Score", "GreaterThanOrEqual", 90)` |
| `LessThan` | IComparable | `Filter.Create("Price", "LessThan", 50)` |
| `LessThanOrEqual` | IComparable | `Filter.Create("Rank", "LessThanOrEqual", 10)` |
| `Contains` | Strings only | `Filter.Create("Name", "Contains", "son")` |
| `StartsWith` | Strings only | `Filter.Create("Name", "StartsWith", "Jo")` |
| `EndsWith` | Strings only | `Filter.Create("Email", "EndsWith", "com")` |

**Validation rules:**
- Comparison operators require `IComparable` values — throws `InvalidOperationException` otherwise
- String operators require pure string values — throws `InvalidOperationException` otherwise
- Invalid operator names throw `ArgumentException`

### Applying Filters

#### Via Extension Method

```csharp
using PageSort.Core.Extensions;

var filters = new List<Filter>
{
    Filter.Create("Age", "GreaterThan", 18),
    Filter.Create("IsActive", "Equals", true)
};

// Filters are combined with AND logic
IQueryable<Student> filtered = students.ApplyFilters(filters);
```

#### Via AdvancedPageQuery (Filter + Sort + Page + Select)

```csharp
var query = new AdvancedPageQuery
{
    PageNumber = 1,
    PageSize = 10,
    SortProperty = "LastName",
    SortDirection = ListSortDirection.Ascending,
    Filters = new List<Filter>
    {
        Filter.Create("Age", "GreaterThanOrEqual", 18),
        Filter.Create("Department", "Equals", "Engineering")
    },
    Fields = new[] { "Id", "FirstName", "LastName", "Age" }
};

var result = Page<Student>.GeneratePaging<Student>(students, query);
```

---

## Dynamic Field Selection

Select only specific properties at runtime — ideal for APIs where clients request partial responses.

### SelectDynamic Extension

```csharp
using PageSort.Core.Extensions;

var fields = new[] { "Id", "FirstName", "Email" };

IQueryable<Dictionary<string, object>> projected = students.SelectDynamic(fields);
// Each item is a dictionary: { "Id": 1, "FirstName": "John", "Email": "john@test.com" }
```

**Behavior:**
- Fields are matched **case-insensitively**
- Unknown field names throw `ArgumentException`
- Properties marked `[MarkAsSensitive]` are automatically excluded (silently)

---

## Sensitive Data Protection

Mark properties that should never appear in dynamic field selection results:

```csharp
using PageSort.Core.Attributes;

public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }

    [MarkAsSensitive]
    public string PasswordHash { get; set; }

    [MarkAsSensitive]
    public string SSN { get; set; }
}
```

When a client requests `Fields = ["Id", "Name", "PasswordHash"]`, the `PasswordHash` field is **silently excluded** — no exception is thrown.

---

## Mapping to DTOs

Map dynamic field results directly to a strongly-typed DTO:

```csharp
// Destination DTO
public class StudentSummary
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

// Usage
var query = new AdvancedPageQuery
{
    PageNumber = 1,
    PageSize = 10,
    Fields = new[] { "Id", "FirstName", "LastName" }
};

PagedResult<StudentSummary> result =
    Page<Student>.GeneratePaging<Student, StudentSummary>(students, query);
// result.Collection is IEnumerable<StudentSummary>
```

**Requirements:**
- At least one field must match a writable property on `TDestination`
- Unmatched properties remain at their default value
- Type conversion is handled via `Convert.ChangeType`

### ToDynamicObjects Extension

For scenarios where you want `ExpandoObject` results:

```csharp
using PageSort.Core.Extensions;

IEnumerable<ExpandoObject> dynamics = students
    .SelectDynamic(new[] { "Id", "Name" })
    .ToDynamicObjects();
```

---

## Async Support

All `Page<T>` methods have async counterparts:

```csharp
// Simple paging
PagedResult<Student> result =
    await Page<Student>.GeneratePagingAsync(students, query);

// Advanced with field selection (KeyValuePair result)
PagedResult<KeyValuePair<string, object?>> result =
    await Page<Student>.GeneratePagingAsync<Student>(students, advancedQuery);

// Advanced with DTO mapping
PagedResult<StudentSummary> result =
    await Page<Student>.GeneratePagingAsync<Student, StudentSummary>(students, advancedQuery);
```

---

## API Reference

### IQueryable Extensions (`PageSort.Core.Extensions`)

| Method | Description |
|--------|-------------|
| `.Page(pageNumber, pageSize)` | Paginate a query |
| `.OrderByProperty(name, direction?)` | Sort by runtime property name |
| `.SelectDynamic(fields)` | Project to `Dictionary<string, object>` |
| `.ApplyFilters(filters)` | Apply dynamic WHERE clauses (AND logic) |

### Dictionary Extensions (`PageSort.Core.Extensions`)

| Method | Description |
|--------|-------------|
| `.ToEnumerable()` | Flatten dictionaries to `KeyValuePair` stream |
| `.MapTo<T>()` | Map dictionaries to strongly-typed objects |
| `.ToDynamicObjects()` | Convert to `ExpandoObject` collection |

### Page<T> Static Helper (`PageSort.Core`)

| Method | Returns |
|--------|---------|
| `GeneratePaging(collection, PageQuery)` | `PagedResult<T>` |
| `GeneratePaging<TSource>(collection, AdvancedPageQuery)` | `PagedResult<KeyValuePair<string, object?>>` |
| `GeneratePaging<TSource, TDest>(collection, AdvancedPageQuery)` | `PagedResult<TDest>` |
| `GeneratePagingAsync(...)` | Async variants of all above |

---

## Async Support

All `Page<T>` methods have async counterparts:

```csharp
// Simple paging
PagedResult<Student> result =
    await Page<Student>.GeneratePagingAsync(students, query);

// Advanced with field selection (KeyValuePair result)
PagedResult<KeyValuePair<string, object?>> result =
    await Page<Student>.GeneratePagingAsync<Student>(students, advancedQuery);

// Advanced with DTO mapping
PagedResult<StudentSummary> result =
    await Page<Student>.GeneratePagingAsync<Student, StudentSummary>(students, advancedQuery);
```

---

## API Reference

### IQueryable Extensions (`PageSort.Core.Extensions`)

| Method | Description |
|--------|-------------|
| `.Page(pageNumber, pageSize)` | Paginate a query |
| `.OrderByProperty(name, direction?)` | Sort by runtime property name |
| `.SelectDynamic(fields)` | Project to `Dictionary<string, object>` |
| `.ApplyFilters(filters)` | Apply dynamic WHERE clauses (AND) |

### Dictionary Extensions (`PageSort.Core.Extensions`)

| Method | Description |
|--------|-------------|
| `.ToEnumerable()` | Flatten dictionaries to KeyValuePair stream |
| `.MapTo<T>()` | Map dictionaries to strongly-typed objects |
| `.ToDynamicObjects()` | Convert to ExpandoObject collection |

### Page<T> Static Helper (`PageSort.Core`)

| Method | Returns |
|--------|---------|
| `GeneratePaging(collection, PageQuery)` | `PagedResult<T>` |
| `GeneratePaging<TSource>(collection, AdvancedPageQuery)` | `PagedResult<KeyValuePair<string, object?>>` |
| `GeneratePaging<TSource, TDest>(collection, AdvancedPageQuery)` | `PagedResult<TDest>` |
| `GeneratePagingAsync(...)` | Async variants of all above |

---

## ASP.NET Core Integration

A typical controller using PageSort.Core:

```csharp
[ApiController]
[Route("api/[controller]")]
public class StudentsController : ControllerBase
{
    private readonly AppDbContext _db;
    public StudentsController(AppDbContext db) => _db = db;

    [HttpGet]
    public IActionResult Get([FromQuery] PageQuery query)
    {
        var result = Page<Student>.GeneratePaging(_db.Students, query);
        return Ok(result);
    }

    [HttpPost("search")]
    public IActionResult Search([FromBody] AdvancedPageQuery query)
    {
        var result = Page<Student>.GeneratePaging<Student, StudentDto>(_db.Students, query);
        return Ok(result);
    }
}
```

**Example GET request:**

```
GET /api/students?PageNumber=1&PageSize=10&SortProperty=LastName&SortDirection=0
```

**Example POST body:**

```json
{
  "pageNumber": 1,
  "pageSize": 10,
  "sortProperty": "GPA",
  "sortDirection": 1,
  "filters": [
    { "field": "Age", "operator": "GreaterThan", "value": 18 },
    { "field": "Department", "operator": "Equals", "value": "CS" }
  ],
  "fields": ["Id", "FirstName", "LastName", "GPA"]
}
```

---

## License

[MIT](LICENSE.txt)
