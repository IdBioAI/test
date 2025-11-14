# Test Data Factories - System Instructions

## Activation Trigger

When the user requests:
- Creating a test for an entity/class
- Working with test data
- Creating or modifying a factory
- Adding new test data

**Automatically apply these instructions for working with Test Data Factories**

---

## Project Context

- **Framework**: .NET Framework 4.7.1, Entity Framework 6
- **In-memory testing**: Effort library
- **Namespace**: `Doklad.Tests.Base.Factories.Data`
- **Pattern**: BaseFactory<T> with unified caching pattern

---

## Important Files

### Factories (ALWAYS read them as needed):
```
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\BaseFactory.cs
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\TestConstants.cs
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\Data\
```

### Seed & Context:
```
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Extensions\DbContextExtensions.cs
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Infrastructure\InMemory\InMemoryContext.cs
```

### Reference Example:
```
C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\Data\CurrencyFactory.cs
```
→ **Read CurrencyFactory.cs as a reference implementation example before creating a new factory**

---

## Factory Pattern - Unified Approach

Every factory MUST follow this pattern:

```csharp
public class XyzFactory : BaseFactory<Xyz>
{
    // 1. Constants for IDs
    public const int FirstXyzId = 1;
    public const int SecondXyzId = 2;

    // 2. Unified property (ALWAYS the same pattern)
    public static List<Xyz> Entities => GetOrCreateEntities(CreateEntities);

    // 3. CreateEntities() - creates test data
    private static List<Xyz> CreateEntities()
    {
        return
        [
            CreateXyz(id: FirstXyzId, ...),
            CreateXyz(id: SecondXyzId, ...),
        ];
    }

    // 4. Helper method with named parameters
    private static Xyz CreateXyz(int id, string name, ...)
    {
        return new Xyz { Id = id, Name = name, ... };
    }
}
```

---

## 3 Options for Working with Factories

### Creating a New Factory
- Create a new file in `Factories/Data/[Name]Factory.cs`
- Implement the pattern above (read CurrencyFactory as reference)
- Constants for IDs (unique values)
- At least 2 test entities

### Adding Data to Existing Factory
- Open existing factory in `Factories/Data/`
- Add constant for new ID
- Add entity to `CreateEntities()` collection
- Preserve formatting and pattern

### Registering Factory in Seeding
- Open `DbContextExtensions.cs`
- Add `.SeedEntity(XyzFactory.Entities)` to `SeedData()`
- **CRITICAL**: Respect order due to FK dependencies
  - Entity must be seeded AFTER all entities it has FK references to
  - Example: If `Agenda` has FK to `Contact`, then `ContactFactory` must be BEFORE `AgendaFactory`

---

## Key Rules (MUST Follow)

1. **Named parameters**: Always use named parameters in helper methods
   ```csharp
   CreateUser(id: 1, name: "Test", email: "test@test.cz")
   ```

2. **TestConstants**: For DateTime use `TestConstants.SqlMinDate` where appropriate

3. **FK constants**: For FK use constants from other factories
   ```csharp
   CreateAgenda(countryId: CountryFactory.CountryIdCz, currencyId: CurrencyFactory.CurrencyIdCzk)
   ```

4. **Collection expression**: Use `[...]` syntax for collections

5. **FK dependencies in seeding**: Check all FKs and seed in correct order

---

## Work Process

1. **Read reference files** as needed (BaseFactory.cs, CurrencyFactory.cs, etc.)
2. **Identify what user needs**: New factory? Adding data? Seed registration?
3. **Implement according to pattern** above
4. **Check**: Compilation, FK dependencies, formatting
5. **For seeding**: Verify order in `DbContextExtensions.cs`

---

## Notes

- If unsure about implementation, read `CurrencyFactory.cs` as example
- For list of all existing factories, explore `Factories/Data/` folder
- For seed order, check `DbContextExtensions.cs` method `SeedData()`
