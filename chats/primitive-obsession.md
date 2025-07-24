---
title: Primitive Obsession in Domain Models
description: Understanding Primitive Obsession and how to refactor domain models using value objects for better type safety and clarity.
model: Claude Sonnet 4
mode: Expert .NET software engineer mode instructions
---
> explain me the "Primitive Obsession Anti-Pattern" you mentioned about TractorId


Great question! **Primitive Obsession** is a code smell where you use primitive data types (like `string`, `int`, `double`) to represent domain concepts that would be better expressed as value objects or custom types.

## What's the Problem with `TractorId` as `string`?

In your `ExpenseAuthorizationRequest` class, you have:

```csharp
/// <summary>
/// The identifier of the tractor involved in the expense.
/// </summary>
[BsonElement( "tractorId" )]
public required string TractorId { get; set; }
```

### Issues with Using `string` for `TractorId`:

1. **No Type Safety**: You can accidentally pass any string where a `TractorId` is expected
2. **No Validation**: There's no built-in validation for tractor ID format
3. **No Domain Logic**: Can't encapsulate tractor-specific business rules
4. **Easy to Mix Up**: Could accidentally swap `TractorId` with `DriverCode` since both are strings
5. **No Intellisense Benefits**: IDE can't help distinguish between different string concepts

### Example of the Problem:

```csharp
// This compiles but is logically wrong
var request = new ExpenseAuthorizationRequest 
{
    TractorId = "DRIVER123",    // Oops! Used driver code instead
    DriverCode = "TRACTOR456",  // Oops! Used tractor ID instead
    // ... other properties
};

// Method expects TractorId but gets any string
ProcessTractorExpense(request.DriverCode); // Runtime bug!
```

## The Solution: Value Objects

Create strongly-typed value objects for your domain concepts:

```csharp
/// <summary>
/// Represents a tractor identifier in the system.
/// </summary>
public readonly record struct TractorId
{
    private readonly string _value;
    
    public TractorId(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Tractor ID cannot be null or empty", nameof(value));
            
        if (value.Length > 10)
            throw new ArgumentException("Tractor ID cannot exceed 10 characters", nameof(value));
            
        if (!IsValidFormat(value))
            throw new ArgumentException("Invalid tractor ID format", nameof(value));
            
        _value = value.ToUpperInvariant();
    }
    
    public string Value => _value;
    
    private static bool IsValidFormat(string value)
    {
        // Example: Tractor IDs must be numeric or alphanumeric
        return value.All(c => char.IsLetterOrDigit(c));
    }
    
    public override string ToString() => _value;
    
    // Implicit conversion for convenience
    public static implicit operator string(TractorId tractorId) => tractorId._value;
    public static implicit operator TractorId(string value) => new(value);
}

/// <summary>
/// Represents a driver code in the system.
/// </summary>
public readonly record struct DriverCode
{
    private readonly string _value;
    
    public DriverCode(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Driver code cannot be null or empty", nameof(value));
            
        if (value.Length > 8)
            throw new ArgumentException("Driver code cannot exceed 8 characters", nameof(value));
            
        _value = value.ToUpperInvariant();
    }
    
    public string Value => _value;
    
    public override string ToString() => _value;
    
    public static implicit operator string(DriverCode driverCode) => driverCode._value;
    public static implicit operator DriverCode(string value) => new(value);
}
```

## Updated Domain Model

```csharp
public record ExpenseAuthorizationRequest
{
    // ... other properties

    /// <summary>
    /// The identifier of the tractor involved in the expense.
    /// </summary>
    [BsonElement( "tractorId" )]
    public required TractorId TractorId { get; set; }

    /// <summary>
    /// The code of the driver submitting the expense.
    /// </summary>
    [BsonElement( "driverCode" )]
    public required DriverCode DriverCode { get; set; }
    
    // ... other properties
}
```

## Benefits of This Approach

### 1. **Type Safety**
```csharp
// This won't compile - prevents bugs at compile time!
var request = new ExpenseAuthorizationRequest 
{
    TractorId = driverCode,    // Compiler error!
    DriverCode = tractorId,    // Compiler error!
};
```

### 2. **Built-in Validation**
```csharp
// Validation happens automatically
TractorId tractorId = "INVALID@ID";  // Throws ArgumentException
```

### 3. **Self-Documenting Code**
```csharp
public void ProcessTractorExpense(TractorId tractorId)
{
    // Method signature clearly shows what type of ID is expected
}
```

### 4. **Encapsulated Business Logic**
```csharp
public readonly record struct TractorId
{
    // Business logic for tractor IDs
    public bool IsMaintenanceUnit() => _value.StartsWith("MAINT");
    public bool IsLeased() => _value.StartsWith("LEASE");
    public int GetFleetNumber() => int.Parse(_value.Substring(5));
}
```

### 5. **Easy Serialization**
```csharp
// MongoDB serialization works seamlessly with implicit operators
[BsonElement( "tractorId" )]
public required TractorId TractorId { get; set; }  // Serializes as string
```

## When to Use Value Objects

Consider creating value objects for:

- **Identifiers**: `TractorId`, `DriverCode`, `TripNumber`
- **Formatted Strings**: `PhoneNumber`, `EmailAddress`, `ZipCode`
- **Constrained Values**: `Amount` (with currency), `Percentage`, `Temperature`
- **Domain Concepts**: `Money`, `Address`, `DateRange`

## Performance Considerations

- **`readonly record struct`**: Lightweight, stack-allocated, immutable
- **Implicit operators**: Make them feel like primitives when needed
- **Validation once**: At construction time, not on every use

## Conclusion

Primitive Obsession makes your code more error-prone and less expressive. Value objects provide:

✅ **Type safety** - Catch errors at compile time  
✅ **Validation** - Ensure data integrity  
✅ **Self-documenting** - Code intent is clear  
✅ **Business logic** - Encapsulate domain rules  
✅ **Better testing** - Isolated, testable units  

In your case, `TractorId`, `DriverCode`, and potentially `TripNumber` would all benefit from being value objects rather than raw primitives. This transforms your domain model from a collection of loosely-typed strings and numbers into a rich, type-safe representation of your business domain.