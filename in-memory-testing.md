# In-Memory Testing s Effort pro Entity Framework 6

Kompletní dokumentace pro in-memory testování v projektu Doklad s použitím Effort knihovny.

## Obsah

1. [Úvod a Přehled](#1-úvod-a-přehled)
2. [Proč Effort knihovna?](#2-proč-effort-knihovna)
3. [Architektura řešení](#3-architektura-řešení)
4. [Praktický návod - Jak přidat testovací data](#4-praktický-návod---jak-přidat-testovací-data)
5. [InMemoryContext - Technické detaily](#5-inmemorycontext---technické-detaily)
6. [Auto-increment ID mechanismus SaveChanges()](#6-auto-increment-id-mechanismus-savechanges)
7. [Známé problémy a workarounds](#7-známé-problémy-a-workarounds)
8. [Doporučení pro budoucnost - Migrace na EF Core](#8-doporučení-pro-budoucnost---migrace-na-ef-core)
9. [Použití s AI Coding Assistants](#9-použití-s-ai-coding-assistants)
10. [Reference a odkazy](#10-reference-a-odkazy)

---

## 1. Úvod a Přehled

### In-memory testing

In-memory testing je technika testování aplikací, při které se místo reálné databáze (SQL Server, PostgreSQL, atd.) používá databáze běžící kompletně v paměti (RAM).

---

## 2. Proč Effort knihovna?

### 2.1 Omezení Entity Framework 6

Entity Framework 6 je legacy framework, který:
- Nemá vestavěnou in-memory databázi (na rozdíl od EF Core)
- Již není aktivně vyvíjen Microsoftem (pouze bugfixy)
- Vyžaduje externí knihovnu pro in-memory testing

### 2.2 Proč ne SQLite?

Při výběru in-memory řešení pro EF6 se nabízí SQLite, ale má zásadní omezení:

#### **SQLite nepodporuje EF6 Code First přístup**

- Existuje neoficiální knihovna:
  - **Není oficiálně podporována** ani Microsoftem ani SQLite.org
  - **Neřeší všechny problémy s EF6:**
    - Code First Migrations často selhávají
    - Některé data annotations nejsou podporovány
    - Problémy s convention-based mapping
  - **Vyžaduje kompletní migraci syntaxe:**
    - SQL Server a SQLite mají různou SQL syntaxi
    - Například: `NVARCHAR(MAX)` (SQL Server) vs `TEXT` (SQLite)
    - Auto-increment: `IDENTITY` (SQL Server) vs `AUTOINCREMENT` (SQLite)
    - DateTime formáty jsou rozdílné

#### **SQLite je optimalizována pro EF Core, ne pro EF6**

- Microsoft oficiálně doporučuje SQLite in-memory **pouze pro EF Core** projekty
- SQLite provider pro EF Core je actively maintained a plně podporovaný
- Pro EF6 není SQLite vhodná volba

---

## 3. Architektura řešení

### 3.1 Přehled komponent

Naše in-memory testing architektura se skládá z několika vrstev:

```
┌─────────────────────────────────────────────────┐
│          Test Class (např. AccountTest)         │
│  - Volá InitializeInMemoryIoCContainer()        │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│         WebInMemoryContainerInitializer         │
│  - Registruje in-memory services                │
│  - Fake implementace (Redis, Mailing, ...)      │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│      WebAppInMemoryUnitOfWorkProvider           │
│  - Golden Database Pattern                      │
│  - CreateGoldenDB → Seed → Snapshot             │
│  - Pro každý test: Rollback to snapshot         │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│             InMemoryContext                     │
│  - Effort DbConnection                          │
│  - Custom SaveChanges (auto-increment)          │
│  - OnModelCreating (keys, circular FKs)         │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│          Seed Data (Factories)                  │
│  - CurrencyFactory, UserFactory, ...            │
│  - BaseFactory<T> caching pattern               │
│  - TestConstants (SqlMinDate, ...)              │
└─────────────────────────────────────────────────┘
```

**Flow při spuštění testu:**

1. Test volá `InitializeInMemoryIoCContainer()`
2. container registruje in-memory services
3. `WebAppInMemoryUnitOfWorkProvider` vytvoří Golden DB (první test):
   - Vytvoří Effort persistent connection
   - Vytvoří schema (`context.Database.CreateIfNotExists()`)
   - Seedne data z factories (`context.SeedData()`)
   - Uloží restore point (`CreateRestorePoint()`)
4. Každý test dostane fresh copy:
   - `RollbackToRestorePoint()` obnoví Golden DB stav
   - Test běží s čistými daty
5. Test ukončí, rollback se provede automaticky

### 3.2 Popis souborů a jejich role

#### **BaseFactory<T>**
`Doklad.Tests.Base/Factories/BaseFactory.cs`

Abstraktní base třída pro všechny factories poskytující unified caching pattern.

**Zodpovědnosti:**
- Lazy initialization test dat pomocí `??=` operátoru
- Caching entit napříč testy (vytvoří se pouze jednou)
- `Reset()` metoda pro smazání cache mezi test runs

```csharp
public abstract class BaseFactory<TEntity> where TEntity : class
{
    private static List<TEntity>? _entities;

    protected static List<TEntity> GetOrCreateEntities(Func<List<TEntity>> createFunc)
    {
        _entities ??= createFunc();
        return _entities;
    }

    public static void Reset() => _entities = null;
}
```

---

#### **TestConstants**
`Doklad.Tests.Base/Factories/TestConstants.cs`

Centralizované konstanty používané napříč test data factories.


```csharp
public static class TestConstants
{
    /// <summary>
    /// SQL Server minimum date value (1753-01-01).
    /// Používá se pro DateTime pole která musí mít hodnotu ale logicky jsou "prázdná".
    /// </summary>
    public static readonly DateTime SqlMinDate = new(1753, 1, 1);
}
```

---

#### **Konkrétní Factories**
`Doklad.Tests.Base/Factories/Data/`

Factories pro jednotlivé entity (např. `CurrencyFactory`, `UserFactory`, `AgendaFactory`).

**Všechny factories používají jednotný pattern:**
- Dědí z `BaseFactory<T>`
- Jedna řádka: `public static List<T> Entities => GetOrCreateEntities(CreateEntities);`
- Private `CreateEntities()` metoda vytváří test data
- Helper metoda `CreateXyz(...)` s named parameters a default values
- Konstanty pro entity IDs (např. `public const int FirstAgendaId = 132747;`)

---

#### **DbContextExtensions**
`Doklad.Tests.Base/Extensions/DbContextExtensions.cs`

Extension metody pro DbContext poskytující seed logiku.

**Zodpovědnosti:**
- `SeedData()` - orchestruje seedování všech factories ve správném pořadí (FK dependencies)
- `SeedEntity<T>()` - generická metoda pro seedování jedné entity type
  - Workaround pro User.Validate() (vypíná validaci)
  - AddRange + SaveChanges + Detach entity
  - Fluent API (vrací context pro chaining)

---

#### **InMemoryContext**
`Doklad.Tests.Base/Infrastructure/InMemory/InMemoryContext.cs`

Custom DbContext pro Effort in-memory databázi s custom logikou.

**Zodpovědnosti:**
- Dědí z `DokladEntitiesContext`
- **Custom SaveChanges():** Auto-increment ID pro nové entity s `Id == 0`
- **OnModelCreating():**
  - Konfigurace primary keys (simple + composite)
  - Ignorování nepodporovaných entit
  - Řešení circular foreign key relationships
  - Volá `base.OnModelCreating()` pro načtení standard mappings

---

#### **WebAppInMemoryUnitOfWorkProvider**
`Doklad.Tests.Base/Infrastructure/InMemory/WebAppInMemoryUnitOfWorkProvider.cs`

UnitOfWorkProvider implementující **Golden Database Pattern** pro maximální performance.

**Zodpovědnosti:**
- **Golden DB inicializace** (jednou pro všechny testy):
  - Vytvoří persistent Effort connection
  - Vytvoří schema
  - Seedne test data
  - Uloží restore point (snapshot)
- **Per-test isolation:**
  - Každý test dostane fresh connection
  - `RollbackToRestorePoint()` obnoví Golden DB stav
  - Test běží s čistými daty bez re-seedování
- **Thread-safety:** Lock mechanismus zajišťuje že Golden DB se inicializuje pouze jednou

**Performance benefit:** ~500-1000ms saved per test (seed operace jednou místo N-krát).

---

#### **WebInMemoryContainerInitializer**
`Doklad.Tests.Base/Infrastructure/InMemory/WebInMemoryContainerInitializer.cs`

Windsor Castle container initializer pro in-memory testy.

**Zodpovědnosti:**
- Registruje `WebInMemoryDataAccessInstaller` → používá in-memory DB místo Azure SQL
- Registruje fake implementace services:
  - `FakeRedisCache` místo skutečného Redis
  - `FakeHttpContextWrapper` místo ASP.NET HttpContext
  - `ConsoleLogger` místo Azure Application Insights
  - `FakeDokladImportLogger`
- Metoda `RegisterPrincipalProvider<T>()` pro injektování různých user identity providers

---

#### **DbTestsBase**
`Doklad.Integration.Tests/Base/DbTestsBase.cs`

Base třída pro všechny integrační testy.

**Zodpovědnosti:**
- Setup/teardown test lifecycle
- Metody pro inicializaci IoC containeru:
  - `InitializeIoCContainer()` - pro testy s reálnou Azure SQL DB
  - `InitializeInMemoryIoCContainer()` - pro testy s Effort in-memory DB
  - Obě metody sdílí unified implementation (eliminace duplicity)
- Helper metody pro testy (SetUserIdentity, FakeTransactionScope, atd.)

---

### 3.3 Golden Database Pattern

Golden Database Pattern je optimalizace při které se test data vytvoří **pouze jednou** a pak se pro každý test použije snapshot (restore point) místo opakovaného seedování.


```
┌────────────────────────────────────────────────────────┐
│ 1. Inicializace Golden DB (první test)                │
│    - CreatePersistentConnection("golden-db")           │
│    - context.Database.CreateIfNotExists()              │
│    - context.SeedData() ← všechny factories            │
│    - goldenConnection.CreateRestorePoint() ← snapshot  │
│    Čas: ~500-1000ms                                    │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────┐
│ 2. Test #1 běží                                        │
│    - new InMemoryContext(goldenConnection)             │
│    - goldenConnection.RollbackToRestorePoint()         │
│    - Test má fresh data bez re-seedování               │
│    Čas: ~5-10ms (rollback je extrémně rychlý)         │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────┐
│ 3. Test #2 běží                                        │
│    - goldenConnection.RollbackToRestorePoint()         │
│    - Opět fresh data                                   │
│    Čas: ~5-10ms                                        │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
                        ...atd
```

---

## 4. Praktický návod - Jak přidat testovací data

### 4.1 Vytvoření nové factory


#### Krok 1: Vytvoř nový soubor

`Doklad.Tests.Base/Factories/Data/CategoryFactory.cs`

#### Krok 2: Použij tento skeleton template

```csharp
namespace Doklad.Tests.Base.Factories.Data
{
    using System;
    using System.Collections.Generic;
    using Doklad.DAL;

    public class CategoryFactory : BaseFactory<Category>
    {
        // ──────────────────────────────────────────────────
        // Konstanty pro entity IDs
        // ──────────────────────────────────────────────────
        public const int FirstCategoryId = 1000;
        public const int SecondCategoryId = 1001;

        // ──────────────────────────────────────────────────
        // Public API - unified pattern (1 řádek)
        // ──────────────────────────────────────────────────
        public static List<Category> Entities => GetOrCreateEntities(CreateEntities);

        // ──────────────────────────────────────────────────
        // Private: Vytvoření test dat
        // ──────────────────────────────────────────────────
        private static List<Category> CreateEntities()
        {
            return
            [
                CreateCategory(
                    id: FirstCategoryId,
                    name: "Elektronika",
                    agendaId: AgendaFactory.FirstAgendaId,
                    userCreatedId: UserFactory.MultipleAgendasUserId,
                    dateCreated: new DateTime(2024, 1, 15, 10, 0, 0)
                ),
                CreateCategory(
                    id: SecondCategoryId,
                    name: "Potraviny",
                    agendaId: AgendaFactory.FirstAgendaId,
                    userCreatedId: UserFactory.MultipleAgendasUserId,
                    dateCreated: new DateTime(2024, 1, 16, 11, 30, 0)
                ),
            ];
        }

        // ──────────────────────────────────────────────────
        // Private: Helper metoda pro vytvoření entity
        // ──────────────────────────────────────────────────
        private static Category CreateCategory(
            int id,
            string name,
            int agendaId,
            int userCreatedId,
            DateTime dateCreated,
            string description = "",
            bool isDeleted = false)
        {
            return new Category
            {
                Id = id,
                Name = name,
                Description = description,
                AgendaId = agendaId,
                UserCreatedId = userCreatedId,
                DateCreated = dateCreated,
                DateLastChange = dateCreated,
                IsDeleted = isDeleted
            };
        }
    }
}
```

#### Krok 3: Naming conventions

**Factory název:** `[Entity]Factory` (např. `CategoryFactory`, `ProductFactory`)

**Konstanty pro IDs:** `public const int [Popis]Id = [Hodnota];`
   - Např: `FirstCategoryId`, `SecondCategoryId`, `ElektronikaId`
   - Používej deskriptivní názvy

**Helper metoda:** `Create[Entity](...)`
   - Named parameters
   - Default values pro optional parametry

#### Krok 4: Best practices

**ID assignment:**
```csharp
// SPRÁVNĚ - použij konstantu
public const int FirstCategoryId = 1000;
CreateCategory(id: FirstCategoryId, ...)

// ŠPATNĚ - magic number
CreateCategory(id: 1000, ...)
```

**Foreign keys:**
```csharp
// SPRÁVNĚ - použij konstantu z jiné factory
agendaId: AgendaFactory.FirstAgendaId,
countryId: CountryFactory.CountryIdCz,

// ŠPATNĚ - magic number
agendaId: 132747,
```

**DateTime hodnoty:**
```csharp
// SPRÁVNĚ - konkrétní datum nebo TestConstants
dateCreated: new DateTime(2024, 1, 15, 10, 0, 0),
dateLastLogin: TestConstants.SqlMinDate,

// ŠPATNĚ - DateTime.Now (nestabilní v testech)
dateCreated: DateTime.Now,
```

---

### 4.2 Přidání testovacích dat do existující factory

Pokud již factory existuje a chceš přidat další testovací entity.

**Příklad: Přidání třetí agendy do AgendaFactory**

#### Krok 1: Přidej konstantu pro ID

```csharp
public class AgendaFactory : BaseFactory<Agenda>
{
    public const int FirstAgendaId = 132747;
    public const int SecondAgendaId = 132748;
    public const int TwoFactorAuthAgendaId = 3042782;
    public const int ThirdAgendaId = 3042783;  // ← nová konstanta
```

#### Krok 2: Přidej entitu do CreateEntities()

```csharp
private static List<Agenda> CreateEntities()
{
    return
    [
        CreateAgenda(
            id: FirstAgendaId,
            name: "yinaxan@acucre.com",
            contactId: ContactFactory.FirstContactId
        ),
        CreateAgenda(
            id: SecondAgendaId,
            name: "yinaxan@acucre.com",
            contactId: ContactFactory.SecondContactId
        ),
        CreateAgenda(
            id: TwoFactorAuthAgendaId,
            name: "H TEST a.s.",
            contactId: ContactFactory.TwoFactorAuthContactId
        ),
        // ────────────────────────────────────────
        // ← NOVÁ ENTITA
        // ────────────────────────────────────────
        CreateAgenda(
            id: ThirdAgendaId,
            name: "TEST Company s.r.o.",
            contactId: ContactFactory.ThirdContactId,
            accountingType: AccountingType.DoubleEntry,
            vatRegistrationType: VatRegistrationType.Payer
        ),
    ];
}
```

---

### 4.3 Registrace factory v seed procesu

Po vytvoření factory je **nutné** ji zaregistrovat do seed pipeline.

#### Krok 1: Otevři DbContextExtensions

`Doklad.Tests.Base/Extensions/DbContextExtensions.cs`

#### Krok 2: Přidej factory do SeedData()

```csharp
public static void SeedData(this InMemoryContext context)
{
    context
        .SeedEntity(CurrencyFactory.Entities)
        .SeedEntity(CountryFactory.Entities)
        .SeedEntity(LanguageFactory.Entities)
        .SeedEntity(UserFactory.Entities)
        .SeedEntity(AgendaFactory.Entities)
        .SeedEntity(ContactFactory.Entities)
        .SeedEntity(BillingAgendaFactory.Entities)
        .SeedEntity(UserAgendaFactory.Entities)
        .SeedEntity(User2faAuthenticatorFactory.Entities)
        .SeedEntity(User2faBackupCodeFactory.Entities)
        .SeedEntity(CategoryFactory.Entities);  // ← nová factory
}
```

#### Krok 3: DŮLEŽITÉ - Pořadí seedování (FK dependencies)

**Pravidlo:** Entita musí být seedována **PO všech entitách na které má FK**.

**Příklad analýzy dependencies:**

Máme novou entitu `Product`:

```csharp
public class Product
{
    public int Id { get; set; }
    public int CategoryId { get; set; }      // FK → Category
    public int AgendaId { get; set; }        // FK → Agenda
    public int CurrencyId { get; set; }      // FK → Currency

    // navigation properties
    public Category Category { get; set; }
    public Agenda Agenda { get; set; }
    public Currency Currency { get; set; }
}
```

**FK dependencies:**
- `Product.CategoryId` → `Category.Id`
- `Product.AgendaId` → `Agenda.Id`
- `Product.CurrencyId` → `Currency.Id`

**Správné pořadí:**

```csharp
public static void SeedData(this InMemoryContext context)
{
    context
        .SeedEntity(CurrencyFactory.Entities)      // 1. Currency (žádné FKs)
        .SeedEntity(CountryFactory.Entities)       // 2. Country (žádné FKs)
        .SeedEntity(LanguageFactory.Entities)      // 3. Language (žádné FKs)
        .SeedEntity(UserFactory.Entities)          // 4. User (žádné FKs)
        .SeedEntity(ContactFactory.Entities)       // 5. Contact (FK: Country)
        .SeedEntity(AgendaFactory.Entities)        // 6. Agenda (FK: Contact, Country, Currency)
        .SeedEntity(CategoryFactory.Entities)      // 7. Category (FK: Agenda)
        .SeedEntity(ProductFactory.Entities);      // 8. ← Product (FK: Category, Agenda, Currency)
        //                                            ↑ Až PO Category, Agenda, Currency!
}
```

---

## 5. InMemoryContext - Technické detaily

### 5.1 Proč vlastní DbContext?

`InMemoryContext` je custom DbContext který řeší několik Effort-specifických požadavků:

1. **Auto-increment ID mechanismus** - přepsaný `SaveChanges()` automaticky přiřazuje IDs
2. **Explicitní konfigurace keys** - Effort vyžaduje explicitní `HasKey()` pro všechny entity
3. **Ignorování nepodporovaných entit** - některé view classes nebo helper entity nefungují v Effort
4. **Řešení circular FK relationships** - konfigurace principal/dependent sides

**Bez InMemoryContext by Effort vyhodil runtime errory o chybějících keys nebo ambiguous relationships.**

---

### 5.2 Konfigurace entity keys

Effort vyžaduje **explicitní primary key konfiguraci** pro všechny entity. EF6 konvence (`Id` property = primary key) nefungují spolehlivě v Effort.

#### Simple primary keys (standardní entity)

Pro většinu entit s `int Id` property EF6 konvence fungují, ale pro jistotu máme global rule:

```csharp
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    // Automaticky konfiguruje DatabaseGeneratedOption.None pro VŠECHNY entity s int Id
    // Důvod: umožňuje manuální ID assignment v seed + náš custom SaveChanges dělá auto-increment
    modelBuilder.Properties<int>()
        .Where(p => p.Name == "Id")
        .Configure(p => p.HasDatabaseGeneratedOption(DatabaseGeneratedOption.None));

    // ...
}
```

**Co to dělá?**
- `DatabaseGeneratedOption.None` = EF negeneruje ID automaticky
- V seed datech můžeme použít konkrétní IDs (např. `AgendaId = 132747`)
- Náš custom `SaveChanges()` pak dělá auto-increment pro nové entity s `Id == 0`

#### Composite keys (složené primární klíče)

Některé entity mají composite primary key - kombinaci více properties.

**Příklady:**

```csharp
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    // UserAgenda: User ↔ Agenda many-to-many join table
    modelBuilder.Entity<UserAgenda>()
        .HasKey(x => new { x.UserId, x.AgendaId });

    // ContactDeliveryAddress: Agenda-scoped delivery addresses
    modelBuilder.Entity<ContactDeliveryAddress>()
        .HasKey(x => new { x.AgendaId, x.Id });

    // Oauth_Tokens: OAuth token storage
    modelBuilder.Entity<Oauth_Tokens>()
        .HasKey(x => new { x.Key, x.TokenType });

    // MailHistoryGrid: Read model s velkým composite key
    modelBuilder.Entity<MailHistoryGrid>()
        .HasKey(x => new {
            x.DateOfSent,
            x.AgendaId,
            x.MailType,
            x.CompanyName,
            x.Id,
            x.Origin,
            x.MailTo
        });
}
```

**Kdy použít composite key?**
- Many-to-many join tables (např. `UserAgenda`)
- Agenda-scoped entities (např. `ContactDeliveryAddress`)
- Read models / grid view models (např. `MailHistoryGrid`)
- Lookup tables s natural composite key

---

### 5.3 Ignorování nepodporovaných entit

Některé entity nepotřebujeme použít v testování a můžou být ignorovány pomocí `modelBuilder.Ignore<T>()`.

```csharp
protected override void OnModelCreating(DbModelBuilder modelBuilder)
{
    modelBuilder.Ignore<DeveloperUser>();
    // ...
}
```

---

### 5.4 Řešení circular foreign key relationships

**Problém:** "Unable to determine the principal end of an association"

Některé entity mají **circular relationships** (A → B a zároveň B → A). EF6 nedokáže automaticky určit která strana je principal a která dependent.

**Naše strategie:** Konfigurovat relationship z **DEPENDENT strany** (entita s required FK).

#### Příklad 1: Agenda ↔ BillingAgenda (1:1, sdílené ID)

**Entities:**

```csharp
public class Agenda
{
    public int Id { get; set; }
    public int? BillingAgendaId { get; set; }  // ← nullable FK

    public BillingAgenda BillingAgenda { get; set; }
}

public class BillingAgenda
{
    public int Id { get; set; }
    public int AgendaId { get; set; }  // ← required FK

    public Agenda Agendum { get; set; }
}
```
```

**Konfigurace (z DEPENDENT strany):**

```csharp
// Agenda ↔ BillingAgenda
// BillingAgenda.AgendaId (required) → Agenda.Id
// Agenda.BillingAgendaId (nullable) → BillingAgenda.Id
modelBuilder.Entity<BillingAgenda>()  // ← začni od dependent (BillingAgenda)
    .HasRequired(ba => ba.Agendum)    // ← BillingAgenda má required FK na Agenda
    .WithOptional(a => a.BillingAgenda);  // ← Agenda má optional navigation property zpět
```

**Důležité:** ID musí být stejné! `BillingAgenda.Id == Agenda.Id`

```csharp
// SPRÁVNĚ - BillingAgenda sdílí ID s Agenda (1:1 relationship)
return new BillingAgenda
{
    Id = agendaId,          // ← stejné jako Agenda.Id
    AgendaId = agendaId,    // ← FK na Agenda
    // ...
};
```

---

#### Shrnutí - Jak řešit circular FKs:

1. **Identifikuj circular relationship:** A → B a B → A
2. **Najdi dependent side:** Která entita má **required** FK? (ne-nullable)
3. **Konfiguruj z dependent side:**
   ```csharp
   modelBuilder.Entity<Dependent>()
       .HasRequired(d => d.Principal)
       .WithOptional(p => p.Dependent);
   ```
4. **Pro 1:1 se sdíleným ID:** Ujisti se že `Dependent.Id == Principal.Id`

---

## 6. Auto-increment ID mechanismus SaveChanges()

Máme dva konfliktní požadavky:

**Požadavek 1: Manuální ID assignment v seed datech**
```csharp
// Chceme použít konkrétní IDs pro předvídatelnost v testech
CreateAgenda(id: 132747, ...);  // ← manuální ID
```

**Požadavek 2: Auto-increment pro nové entity v testech**
```csharp
// V testech chceme vytvářet nové entity bez explicitního ID
var newAgenda = new Agenda
{
    Id = 0,  // ← auto-increment by měl přiřadit další ID
    Name = "Test"
};
context.Agendas.Add(newAgenda);
context.SaveChanges();  // ← chceme Id = 132750 (například)
```

**Problém:** `DatabaseGeneratedOption.None` (potřebné pro Požadavek 1) vypíná auto-increment!

**Řešení:** Custom `SaveChanges()` override který dělá auto-increment manuálně.

---

### 6.2 Jak funguje custom SaveChanges()

Náš `InMemoryContext.SaveChanges()` implementuje auto-increment logiku:

```csharp
public override int SaveChanges()
{
    // ─────────────────────────────────────────────────────
    // 1. Najdi všechny nově přidávané entity
    // ─────────────────────────────────────────────────────
    var entitiesByType = ChangeTracker.Entries()
        .Where(e => e.State == EntityState.Added)  // ← jen Added (nové)
        .Select(e => new
        {
            Entry = e,
            EntityType = e.Entity.GetType(),
            IdProperty = e.Entity.GetType().GetProperty("Id")  // ← reflexe pro Id property
        })
        .Where(x => x.IdProperty != null && x.IdProperty.PropertyType == typeof(int))  // ← jen int Id
        .GroupBy(x => x.EntityType);  // ← group po typu entity

    // ─────────────────────────────────────────────────────
    // 2. Pro každý typ entity zvlášť
    // ─────────────────────────────────────────────────────
    foreach (var group in entitiesByType)
    {
        var entityType = group.Key;
        var idProperty = group.First().IdProperty;

        // Najdi entity které potřebují auto-increment (Id == 0)
        var entitiesToAssignId = group
            .Where(x => (int)idProperty.GetValue(x.Entry.Entity) == 0)  // ← Id == 0
            .ToList();

        if (!entitiesToAssignId.Any())
            continue;  // žádné entity s Id == 0, pokračuj dalším typem

        // ─────────────────────────────────────────────────────
        // 3. Vypočítej MAX(Id) pro tento entity type
        // ─────────────────────────────────────────────────────
        var dbSet = Set(entityType);  // DbSet<T> pro tento entity type
        var maxId = ((System.Collections.IEnumerable)dbSet)
            .Cast<object>()
            .Select(e => (int)idProperty.GetValue(e))
            .DefaultIfEmpty(0)  // pokud je DbSet prázdný, použij 0
            .Max();

        // ─────────────────────────────────────────────────────
        // 4. Přiřaď sequential IDs počínaje maxId + 1
        // ─────────────────────────────────────────────────────
        foreach (var item in entitiesToAssignId)
        {
            idProperty.SetValue(item.Entry.Entity, ++maxId);  // reflexe: entity.Id = ++maxId
        }
    }

    // ─────────────────────────────────────────────────────
    // 5. Zavolej base.SaveChanges() - standardní EF logika
    // ─────────────────────────────────────────────────────
    return base.SaveChanges();
}
```

**Krok za krokem co se děje:**

**Příklad:** Máme v DB:
- `Agenda` s IDs: 132747, 132748, 3042782
- Vytváříme 3 nové agendy s `Id = 0`

```csharp
context.Agendas.Add(new Agenda { Id = 0, Name = "Test1" });
context.Agendas.Add(new Agenda { Id = 0, Name = "Test2" });
context.Agendas.Add(new Agenda { Id = 0, Name = "Test3" });
context.SaveChanges();  // ← Co se stane?
```

1. **ChangeTracker.Entries()** najde 3 Agenda entity s `State == Added`
2. **GroupBy EntityType** - všechny 3 jsou typ `Agenda`, takže jeden group
3. **Filter Id == 0** - všechny 3 mají `Id == 0`, takže všechny potřebují auto-increment
4. **Max(Id)** v `Agendas` DbSet = `3042782`
5. **Assign IDs:**
   - `Test1`: Id = 3042783 (maxId + 1)
   - `Test2`: Id = 3042784 (maxId + 2)
   - `Test3`: Id = 3042785 (maxId + 3)
6. **base.SaveChanges()** uloží entity s přiřazenými IDs

**Performance:**
- Reflexe je použita jen pro nově vytvářené entity (typicky 1-10 entit per test)
- `Max(Id)` query běží v paměti (Effort) = extrémně rychlý
- Celková režie: ~1-5ms per SaveChanges

---

## 7. Známé problémy a workarounds

### 7.1 User.Validate() Azure DB connection

#### Popis problému

`User` entita implementuje `IValidatableObject` s custom validací:

```csharp
// Doklad.DAL/Entities/User.cs
public partial class User : IValidatableObject
{
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        // PROBLÉM: Vytváří nový DbContext který se připojuje k Azure SQL!
        using (var context = new DokladEntitiesContext(new DokladConfiguration()))
        {
            var emailExist = context.Users.Count(u => u.UserName == this.UserName && u.Id != this.Id) > 0;

            if (emailExist)
            {
                yield return new ValidationResult(ApplicationStrings.EmailIsExist, new[] { EmailFieldName });
            }
        }
    }
}
```

**Proč je to problém v in-memory testech?**

1. `SeedEntity<User>()` volá `context.SaveChanges()`
2. EF6 během `SaveChanges()` volá `User.Validate()` pro všechny User entity
3. `User.Validate()` vytvoří nový `DokladEntitiesContext(new DokladConfiguration())`
4. `DokladConfiguration` vrací connection string na **Azure SQL Database**
5. V testech:
   - **Connection error** - build server nemá přístup k Azure
   - **False "email already exists"** - user existuje v Azure DB ale ne v in-memory

**Výsledek:** Test selže s `SqlException` nebo `ValidationException`.

---

#### Současný workaround v SeedEntity

```csharp
// Doklad.Tests.Base/Extensions/DbContextExtensions.cs
public static DbContext SeedEntity<TEntity>(this DbContext context, IEnumerable<TEntity> data)
    where TEntity : class
{
    // ──────────────────────────────────────────────────────────────
    // WORKAROUND: Vypni validaci pro User entities
    // ──────────────────────────────────────────────────────────────
    var isUserEntity = typeof(TEntity) == typeof(User);
    var originalValidation = context.Configuration.ValidateOnSaveEnabled;

    if (isUserEntity)
    {
        context.Configuration.ValidateOnSaveEnabled = false;  // ← vypni IValidatableObject
    }

    var entitiesToAdd = data.ToList();
    context.Set<TEntity>().AddRange(entitiesToAdd);
    context.SaveChanges();  // ← Validate() se NEvolá pro User

    // TODO: delete after User.Validate() fix
    context.Configuration.ValidateOnSaveEnabled = originalValidation;  // ← restore

    foreach (var entity in entitiesToAdd)
    {
        context.Entry(entity).State = EntityState.Detached;
    }

    return context;
}
```


- `ValidateOnSaveEnabled = false` vypíná `IValidatableObject.Validate()`
- Pro User entities se validace přeskočí
- Pro ostatní entity běží normálně

---

### 7.2 Další známá omezení Effort

#### Co Effort nepodporuje

**1. Stored Procedures**
- Effort nemá SQL engine, nemůže spustit T-SQL stored procedures

**2. Raw SQL Queries (částečně)**
- `context.Database.SqlQuery<T>("SELECT ...")` může selhat na komplexních queries

**3. SQL Server specifické funkce**
- `DATEADD`, `DATEDIFF`, `NEWID()`, atd. nejsou v Effort implementovány

**4. Database triggers**
- Effort nemá trigger support

**5. Transactions s `TransactionScope`**
- Effort má omezenou podporu pro distributed transactions

---

## 8. Doporučení pro budoucnost - Migrace na EF Core

### 8.1 SQLite pro EF Core

Microsoft **oficiálně doporučuje SQLite in-memory** pro testování EF Core aplikací.

📚 **Odkaz:** https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy

#### EF Core In-Memory provider má reliability problémy

EF Core má vestavěný `UseInMemoryDatabase()` provider, ale Microsoft ho **nedoporučuje pro většinu testů**:

**Není relační databáze**
- In-Memory je jen glorified Dictionary<string, List<object>>
- Žádné SQL queries, žádný query optimizer
- Různé chování od reálné DB

**False positives/negatives**
- Testy projdou v in-memory ale failnou v produkci
- Nebo testy failnou v in-memory ale fungují v produkci
- = **nespolehlivé testy**

#### SQLite in-memory výhody

**1. Skutečná relační databáze**
- Plná SQL podpora
- Query planner & optimizer
- Transactions, constraints, indexes

**2. Validuje všechny constraints**

**3. Chování velmi podobné SQL Server**
- Oba jsou relační databáze
- SQL syntax 95% kompatibilní
- Stejné constraints, triggers, atd.

**4. Odhalí problémy které in-memory přehlédne**
- FK violations
- Unique constraint violations
- Check constraint violations
- Incorrect query joins

**5. Microsoft oficiálně podporovaný provider**
- Aktivně vyvíjen a maintainován
- Plně testovaný s EF Core

**6. Stále velmi rychlý**
- SQLite in-memory běží v RAM (žádný disk I/O)

#### Užitečné odkazy pro migraci

- **EF6 → EF Core migrace guide:** https://learn.microsoft.com/en-us/ef/efcore-and-ef6/porting/
- **Breaking changes:** https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/breaking-changes
- **Testing s SQLite:** https://learn.microsoft.com/en-us/ef/core/testing/sqlite

---

## 9. Použití s AI Coding Assistants

### 9.1 AI Coding Assistant Prompt - Auto-trigger Setup

Pro vytváření testů je připravený univerzální prompt, který lze použít s jakýmkoliv AI kódovacím asistentem (Claude Code, GitHub Copilot, Cursor, atd.).

**Soubor**: `Doklad.Tests.Base/docs/init-inmemory-testing.md`

#### Jak nastavit

**Pro Claude Code:**
1. Zkopíruj obsah souboru `init-inmemory-testing.md`
2. Vlož ho do svého lokálního `.claude/init.md` souboru v root adresáři projektu
3. Claude Code ho načte při startu

**Pro GitHub Copilot / Cursor:**
1. Zkopíruj obsah souboru `init-inmemory-testing.md`
2. Vlož ho do `.github/copilot-instructions.md` nebo do workspace instructions
3. Copilot/Cursor ho načte automaticky

**Pro ChatGPT / jiné asistenty:**
1. Zkopíruj obsah souboru `init-inmemory-testing.md`
2. Vlož ho jako první zprávu do konverzace před začátkem práce

#### Jak používat

Po nastavení stačí napsat jednoduchý požadavek:

```
Create a test for User class where test data are:
- admin with email admin@test.cz
- regular user with email user@test.cz
```

AI asistent automaticky:
- Rozpozná že potřebuješ pracovat s factories
- Přečte si relevantní soubory (BaseFactory.cs, referenční factories)
- Vytvoří nebo upraví UserFactory podle unified patternu
- Zaregistruje factory do seedování pokud je to potřeba
- Respektuje FK dependencies a všechna pravidla

#### Výhody automatické aktivace

- **Bez opakování**: Nemusíš pokaždé kopírovat dlouhý prompt
- **Aktuální kontext**: AI si načte aktuální verze souborů z projektu
- **Konzistence**: Všichni vývojáři používají stejný pattern
- **Rychlost**: Napíšeš jen co potřebuješ, zbytek je automatický
- **Univerzální**: Funguje s jakýmkoliv AI asistentem

---

### 9.2 Full Prompt for Manual Use

Pokud nechceš používat auto-trigger setup, můžeš použít tento plný prompt přímo v konverzaci:

<details>
<summary>Klikni pro zobrazení plného promptu (260 řádků)</summary>

```markdown
# ÚKOL: Práce s Test Data Factories pro In-Memory Testing (Effort + EF6)

## Kontext projektu

- .NET Framework 4.7.1, Entity Framework 6
- In-memory testing s Effort library
- Namespace: `Doklad.Tests.Base.Factories.Data`

## Důležité soubory

Factories:
- C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\BaseFactory.cs
- C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\TestConstants.cs
- C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\Data\ (složka - najdi si seznam factory zde)

Seed & Context:
- C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Extensions\DbContextExtensions.cs
- C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Infrastructure\InMemory\InMemoryContext.cs

## Referenční příklad: CurrencyFactory.cs

namespace Doklad.Tests.Base.Factories.Data
{
    using System;
    using System.Collections.Generic;
    using Doklad.DAL;

    public class CurrencyFactory : BaseFactory<Currency>
    {
        public const int CurrencyIdCzk = 1;
        public const int CurrencyIdEur = 2;

        public static List<Currency> Entities => GetOrCreateEntities(CreateEntities);

        private static List<Currency> CreateEntities()
        {
            return
            [
                CreateCurrency(
                    id: CurrencyIdCzk,
                    code: "CZK",
                    symbol: "Kč",
                    priority: 10,
                    dateLastChange: new DateTime(2013, 4, 18, 8, 37, 11, 803)
                ),
                CreateCurrency(
                    id: CurrencyIdEur,
                    code: "EUR",
                    symbol: "EUR",
                    priority: 8,
                    dateLastChange: TestConstants.SqlMinDate
                ),
            ];
        }

        private static Currency CreateCurrency(
            int id,
            string code,
            string symbol,
            int priority,
            DateTime dateLastChange)
        {
            return new Currency
            {
                Id = id,
                Code = code,
                Symbol = symbol,
                Priority = priority,
                DateLastChange = dateLastChange
            };
        }
    }
}

---

## MOŽNOST 1: Vytvoření nové factory

Vytvoř novou factory třídu pro entitu [NÁZEV_ENTITY]:

### Požadavky

1. Factory musí dědit z BaseFactory<[NÁZEV_ENTITY]>

2. Implementuj unified pattern:
   - public static List<[NÁZEV_ENTITY]> Entities => GetOrCreateEntities(CreateEntities);
   - private static List<[NÁZEV_ENTITY]> CreateEntities() { ... }
   - private static [NÁZEV_ENTITY] Create[NÁZEV](...params...) { ... }

3. Přidej konstanty pro entity IDs:
   - public const int First[NÁZEV]Id = [HODNOTA];
   - public const int Second[NÁZEV]Id = [HODNOTA];
   - Použij vhodné, unikátní ID hodnoty

4. V CreateEntities() vytvoř alespoň 2 testovací entity pomocí collection expression [...]

5. V Create[NÁZEV]() helper metodě:
   - Všechny parametry jako named parameters
   - Použij default values pro optional parametry
   - Pro DateTime použij TestConstants.SqlMinDate kde je to vhodné
   - Pro FK použij konstanty z jiných factories (např. CountryFactory.CountryIdCz)

6. Soubor ulož do:
   C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\Data\[NÁZEV]Factory.cs

### Checklist

- [ ] Factory dědí z BaseFactory<T>
- [ ] Implementován pattern Entities => GetOrCreateEntities(CreateEntities)
- [ ] Konstanty pro IDs jsou public const int
- [ ] Helper metoda CreateXyz() má named parameters a default values
- [ ] Použity konstanty z TestConstants (SqlMinDate)
- [ ] FK reference používají konstanty z jiných factories
- [ ] Správný namespace Doklad.Tests.Base.Factories.Data
- [ ] Soubor na správné cestě v Factories/Data/ složce
- [ ] Kód kompiluje bez chyb

---

## MOŽNOST 2: Přidání testovacích dat do existující factory

Přidej novou testovací entitu do existující factory [NÁZEV]Factory:

### Kroky

1. Otevři soubor:
   C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Factories\Data\[NÁZEV]Factory.cs

2. Přidej konstantu pro ID:
   - Na začátek třídy přidej: public const int [POPIS]Id = [HODNOTA];
   - Ujisti se že ID je unikátní (nekonflikuje s existujícími konstantami)

3. Přidej entitu do CreateEntities():
   - V metodě CreateEntities() přidej novou entitu pomocí Create[NÁZEV]() helper metody
   - Použij collection expression syntax [...]
   - Nezapomeň trailing comma za poslední entitou

4. Ujisti se že:
   - ID je unikátní
   - Všechny FK používají konstanty z jiných factories
   - DateTime hodnoty jsou buď konkrétní nebo TestConstants.SqlMinDate
   - Používáš named parameters
   - Formátování je konzistentní s existujícími entitami

### Checklist

- [ ] Nová konstanta pro ID přidána na začátek třídy
- [ ] ID je unikátní
- [ ] Entita přidána do CreateEntities() collection
- [ ] Používá helper metodu CreateXyz()
- [ ] Všechny FK používají konstanty z jiných factories
- [ ] Named parameters pro všechny argumenty
- [ ] DateTime použity správně
- [ ] Formátování konzistentní
- [ ] Kód kompiluje bez chyb

---

## MOŽNOST 3: Registrace factory v seedování

Zaregistruj novou factory [NÁZEV]Factory do seed procesu:

### Kontext

- Seed metoda: C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Extensions\DbContextExtensions.cs
- Metoda: SeedData(this InMemoryContext context)

### Kroky

1. Otevři soubor:
   C:\Users\Jaros\source\repos\Idoklad\src\server\Doklad.Tests.Base\Extensions\DbContextExtensions.cs

2. Přidej factory do SeedData():
   - V metodě SeedData() přidej řádek: .SeedEntity([NÁZEV]Factory.Entities)

3. DŮLEŽITÉ - Pořadí je klíčové kvůli FK dependencies:
   - Pravidlo: Entita musí být seedována PO všech entitách na které má FK

4. Analýza FK dependencies:
   a) Podívej se na [NÁZEV] entitu - jaké má FK properties?
   b) Pro každý FK: identifikuj na jakou entitu ukazuje a najdi odpovídající factory
   c) Umísti .SeedEntity([NÁZEV]Factory.Entities) AŽ PO všech FK factories

### Příklad validace pořadí

ŠPATNĚ:
public static void SeedData(this InMemoryContext context)
{
    context
        .SeedEntity(CurrencyFactory.Entities)
        .SeedEntity(CountryFactory.Entities)
        .SeedEntity(AgendaFactory.Entities)   // FK: ContactId → Contact
        .SeedEntity(ContactFactory.Entities);  // PROBLÉM! Contact je až PO Agenda
}

SPRÁVNĚ:
public static void SeedData(this InMemoryContext context)
{
    context
        .SeedEntity(CurrencyFactory.Entities)
        .SeedEntity(CountryFactory.Entities)
        .SeedEntity(ContactFactory.Entities)  // Contact PŘED Agenda
        .SeedEntity(AgendaFactory.Entities);  // Agenda až PO Contact
}

### Aktuální pořadí v projektu

public static void SeedData(this InMemoryContext context)
{
    context
        .SeedEntity(CurrencyFactory.Entities)     // Žádné FKs
        .SeedEntity(CountryFactory.Entities)      // Žádné FKs
        .SeedEntity(LanguageFactory.Entities)     // Žádné FKs
        .SeedEntity(UserFactory.Entities)         // Žádné FKs
        .SeedEntity(ContactFactory.Entities)      // FK: CountryId → Country
        .SeedEntity(AgendaFactory.Entities)       // FK: ContactId → Contact, CountryId → Country, CurrencyId → Currency
        .SeedEntity(BillingAgendaFactory.Entities)        // FK: AgendaId → Agenda
        .SeedEntity(UserAgendaFactory.Entities)           // FK: UserId → User, AgendaId → Agenda
        .SeedEntity(User2faAuthenticatorFactory.Entities) // FK: UserId → User
        .SeedEntity(User2faBackupCodeFactory.Entities);   // FK: UserId → User
}

Kam patří tvoje nová factory v tomto pořadí?

### Checklist

- [ ] Factory přidána do SeedData() metody
- [ ] Zachována fluent API struktura (.SeedEntity() chaining)
- [ ] Zkontrolovány všechny FK dependencies v entitě
- [ ] Factory je v seed pořadí PO všech FK dependencies
- [ ] Zkontrolováno že žádná jiná entita nemá FK na tuto novou entitu
- [ ] Kód kompiluje bez chyb
- [ ] Spuštěn nějaký existující test pro ověření že seed funguje

### Pomoc s validací FK dependencies

Pokud nejsi si jistý FK dependencies:

1. Otevři entity class pro [NÁZEV]
2. Najdi všechny properties končící na Id (např. AgendaId, UserId, CountryId)
3. Pro každé XyzId property:
   - Je to FK? (obvykle ano pokud má odpovídající Xyz navigation property)
   - Existuje XyzFactory?
   - Je XyzFactory seedovaná PŘED touto factory?
4. Pokud ne, přesuň factory níže v pořadí

---

## Závěr

Vyber si jednu z MOŽNOSTÍ 1-3 podle toho co potřebuješ udělat a postupuj podle instrukcí.
Nahraď všechny [PLACEHOLDERS] konkrétními hodnotami před použitím.
```

</details>

---

### 9.3 Srovnání přístupů

| Přístup | Výhody | Nevýhody | Použití |
|---------|--------|----------|---------|
| **Claude Code .init** | Automatické, rychlé, konzistentní | Jen pro Claude Code | Doporučeno pro běžnou práci |
| **Plný prompt** | Univerzální, funguje všude | Musíš kopírovat pokaždé | Pro jiné AI asistenty |

---
## 10. Reference a odkazy

### Microsoft dokumentace

**Entity Framework Core:**
- **Testing strategies:** https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy
- **In-Memory as database fake (problémy):** https://learn.microsoft.com/en-us/ef/core/testing/choosing-a-testing-strategy#inmemory-as-a-database-fake
- **Testing with SQLite:** https://learn.microsoft.com/en-us/ef/core/testing/sqlite

**Entity Framework 6:**
- **Testing with EF6:** https://learn.microsoft.com/en-us/ef/ef6/fundamentals/testing/
- **EF6 → EF Core migrace guide:** https://learn.microsoft.com/en-us/ef/efcore-and-ef6/porting/

**Breaking changes:**
- **EF Core 8.0 breaking changes:** https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/breaking-changes

---

### Effort knihovna

**GitHub repository:**
- https://github.com/tamasflamich/effort

**NuGet package:**
- https://www.nuget.org/packages/Effort.EF6/

**Dokumentace:**
- Wiki: https://github.com/tamasflamich/effort/wiki

---
