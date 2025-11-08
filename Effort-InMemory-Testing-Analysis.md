# Effort.EF6 In-Memory Testing - Technická Analýza a Důvody Selhání

---

## Shrnutí

**Cíl:** Zrychlit integration testy pomocí in-memory databáze a odpojit od DEV Azure DB s lepší 
spolehlivostí a udržitelností.

**Výsledek:** Effort není použitelný pro tento projekt nebo jiné enterprise projekty.
- PoC s Effort ukončit. Investovat čas do SQLite + unit testů místo dalších 20+ hodin ladění zastaralé knihovny/metody.

**Hlavní důvody:**
- Projekt bez aktivního vývoje/Zastaralá knihovna bez podpory EF Core
- Minimální dokumentace
- Téměr žádná podpora od komunity
- Žádná podpora pro EF Core (nemožnost migrace na EF core v budoucnu)
- IDENTITY auto-increment se neresetuje → FK violations
- `SetIdentityFields()` API nefunguje s Code First + komplexním EF modelem
- Vyžaduje masivní refactoring 150+ factories včetně samostatných linkerů dat
- Microsoft a další zdroje (včetně Jimmy Bogard) uvádí, že in-memory testy jsou nevyhovující
- Effort knihovna je použitelná pro malé projekty se starším .NET frameworkem
- Effort je zastaralý projekt bez podpory EF Core, má dokumentované výkonnostní problémy už při stovkách testů (https://github.com/zzzprojects/EntityFramework-Effort/issues/93), a Microsoft sám doporučuje nepoužívat in-memory providery.Pro projekt s 150 tabulkami doporučuji Testcontainers s reálným SQL Serverem - industry standard.
- Minimální zrychlení testů. U testované třídy "AccountControllerTests" byl Effort o 2-3 sekundy rychlejší. To může
být ale způsobeno pomalejším internetem (připojování se do Azure DB). 

Zdroje:
- https://learn.microsoft.com/en-us/ef/core/testing/testing-without-the-database
- https://www.jimmybogard.com/avoid-in-memory-databases-for-tests/
- https://goatreview.com/dotnet-testing-strategy-inmemory-sqlite-repository-pattern/


**Doporučení:** Použít **SQLite in-memory** nebo **LocalDB** místo Effort případně kombinace s Docker podle doporučeno (https://learn.microsoft.com/en-us/ef/core/testing/testing-without-the-database). 
Případně rezervovat čas a najít moderní, nejvíce používané řešení. 

**Další doporučení s Testcontainers se SQL serverem:**

Testcontainers je knihovna, která umožňuje spouštět Docker kontejnery přímo z testů. Pro SQL Server to znamená, že každý test (nebo sada testů) může mít svoji vlastní, izolovanou, reálnou SQL Server databázi, která běží v Dockeru a automaticky se po testech smaže.
Hlavní výhody:

- Reálná databáze
- Izolace - každý test má čistou databázi
- Automatický cleanup - kontejner se sám smaže po testu
- Rychlost - kontejnery startují za pár sekund
- CI/CD ready - funguje v pipeline

**Finální doporučení**

- xUnit + NSubstitute + Bogus + FluentAssertions (unit)
- SQLite In-Memory (integration, kritické flow)
- GitHub Copilot (AI generování testů)
- Testcontainers

**V případě, že se bude chtít zachovat Effort**
- přepsání testů, aby se kontrola nedělala podle ID např. agendy, uživatele atd. -> refactoring mnoha PrincipalProvider
- Zbavit se závislostí na ID v testech
  - Refactoring všech Factory tříd (150+ souborů):
      - Změnit všechny FK reference, aby používaly relativní vazby místo konkrétních ID
      - Příklad: User.LanguageId = 1 → User.Language = LanguageFactory.Entities.First()
- Pro zrychlení příkazu  ``` schemaContext.Database.CreateIfNotExists();``` vytvořit samostatné EntitiesContext. Tímto
by se vytvořilo schéma, které by mělo pouze potřebné tabulky a ne celá DB (150+ tabulek). Např. třída AccountControllerTests
vyžaduje pouze 10 tabulek. Jednalo by se ale o velmi časově náročnou operaci.

**Rizika**
- Objevení dalších Effort problémů během implementace
- Možné problémy s komplexními EF dotazy (LINQ → Effort může mít odlišné chování)
- Žádná community podpora při problémech

---

## 1. Původní Cíl Projektu

### Motivace
- **Současný stav:** Integration testy proti Azure Database
- **Stabilita:** <99.9% kvůli občasným Azure DB timeoutům
- **Cíl:** Zrychlit na 10-30 sekund pomocí in-memory DB

### Očekávaný Výkon
```
První test:    900ms (schema creation)
Další testy:   10-20ms (data re-seed)
────────────────────────────────────
100 testů:     ~2 sekundy
500 testů:     ~6 sekund

vs. Azure DB:  120-300 sekund
Zrychlení:     45-75×
```

---

## 2. Zvolená Strategie Optimalizace

### Persistent Connection Pattern
```csharp
// Idea: Vytvoř schema jednou, reuse napříč testy
var connection = Effort.DbConnectionFactory.CreatePersistent("DokladTestDb");

// První test (900ms)
context.Database.CreateIfNotExists();
context.SeedData();

// Další testy (12ms)
context.SeedData(); // Pouze re-seed dat
```

**Očekávaný benefit:** Schema creation (900ms) jen jednou místo při každém testu.

---

## 3. Kritický Problém: IDENTITY Auto-Increment Neresetuje

### Popis Problému

#### První Test
```sql
INSERT INTO Currency (Id, Code) VALUES (5, 'CZK')
-- Effort ignoruje Id=5 a vygeneruje Id=1

INSERT INTO Currency (Id, Code) VALUES (6, 'EUR')
-- Effort vygeneruje Id=2
```

**Factory očekává:**
```csharp
public const int CurrencyIdCzk = 5; // Z původní Azure DB
new Country { CurrencyId = 5 }      // FK reference
```

**Realita v Effort:**
```
Currency.Id = 1 (ne 5!)
Country.CurrencyId = 5 → FK violation!
```

#### Druhý Test (po DELETE všech dat)
```sql
DELETE FROM Currency; -- Tabulka prázdná

INSERT INTO Currency (Id, Code) VALUES (5, 'CZK')
-- Effort pokračuje od 2 → vygeneruje Id=3

INSERT INTO Currency (Id, Code) VALUES (6, 'EUR')
-- Effort vygeneruje Id=4
```

**IDENTITY seed není resetován!**

```
Očekáváno:  IDs 5, 6
Skutečnost: IDs 3, 4
Výsledek:   FK violation znovu!
```

### Root Cause
Effort interně používá `Dictionary<Type, int>` pro tracking posledního ID. DELETE dat tuto dictionary neresetuje.

---

## 4. Pokus 1: Transaction Rollback

### Strategie
```csharp
[SetUp]
public void Setup()
{
    BeginTestTransaction(); // Izolace pomocí transakce
}

[TearDown]
public void TearDown()
{
    RollbackTransaction(); // Vrať změny zpět
}
```

**Očekávaný benefit:** Žádné DELETE potřeba, data se automaticky vrátí.

### Proč Selhalo

**Production kód vytváří vlastní transakce:**
```csharp
// IssuedInvoiceDeleteCommands.cs
using (var transactionScope = this.TransactionScopeProxy.CreateTransactionScope())
{
    // Delete operations
    transactionScope.Complete();
}
```

**Chyba:**
```
System.InvalidOperationException:
  A transaction is already in progress.
  Nested transactions are not supported.
```

**Závěr:** Vnořené transakce v Effort nejsou spolehlivé.

---

## 5. Pokus 2: Effort SetIdentityFields() API

### Strategie Podle GitHub Příkladu

Effort poskytuje API pro vypnutí IDENTITY:
```csharp
connection.Open();
connection.DbManager.SetIdentityFields(false); // Vypni auto-increment
connection.Close();

context.Add(new Currency { Id = 5 }); // Explicit ID
context.SaveChanges();

Assert.AreEqual(5, currency.Id); // Mělo by fungovat
```

**Zdroj:** [GitHub Issue #71](https://github.com/tamasflamich/effort/issues/71)

### Problém 1: ID Se Mění na 0 Po SaveChanges

```csharp
var currency = new Currency { Id = 5, Code = "CZK" };
context.Currencies.Add(currency);

Console.WriteLine($"PŘED: {currency.Id}"); // Výstup: 5

context.SaveChanges();

Console.WriteLine($"PO: {currency.Id}");   // Výstup: 0
```

**Chování:** `SetIdentityFields(false)` s normálním EF contextem nefunguje - Effort vygeneruje 0 místo zachování našeho ID.

### Problém 2: Code First vs Database First

**GitHub příklad předpokládá Database First:**
```csharp
// Database First - schema už existuje
connection.SetIdentityFields(false);
var context = new MyContext(connection); // Schema ready
```

**Náš Code First:**
```csharp
connection.SetIdentityFields(false);
var context = new DokladEntitiesContext(connection);

// Chyba:
// Effort.Exceptions.EffortException:
//   'Database has not been initialized.
//   If using CodeFirst try to add the following line:
//   context.Database.CreateIfNotExists()'
```

**Problém:** Musíme volat `CreateIfNotExists()` PŘED `SetIdentityFields()`, ale context už je vytvořený.

### Problém 3: Context Cachuje Identity Setting

```csharp
// NEFUNGUJE:
var context = new DokladEntitiesContext(connection);
connection.SetIdentityFields(false); // Context si pamatuje "enabled"

// SPRÁVNĚ (podle příkladu):
connection.SetIdentityFields(false);
var context = new DokladEntitiesContext(connection);
```

Ale s Code First nemůžeme vytvořit context druhý protože schema neexistuje!

### Pokus o 4-Step proces

```csharp
// STEP 1: Create schema
using (var schemaContext = new DokladEntitiesContext(connection))
{
    schemaContext.Database.CreateIfNotExists();
}

// STEP 2: Disable identity
connection.Open();
connection.DbManager.SetIdentityFields(false);
connection.Close();

// STEP 3: Seed with NEW context
using (var seedContext = new DokladEntitiesContext(connection))
{
    seedContext.SeedData();
}

// STEP 4: Check result
var currencies = context.Set<Currency>().ToList();
// Currency.Id = 0 Stále nefunguje!
```

**Závěr:** SetIdentityFields() nefunguje s Code First + komplexním modelem.

---

## 6. Pokus 3: DokladEntitiesContextNoIdentity

### Strategie

Vytvořit speciální context který vypne IDENTITY v EF mappingu:

```csharp
public class DokladEntitiesContextNoIdentity : DokladEntitiesContext
{
    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder); // Load 150+ mappings

        // Override jen Currency
        modelBuilder.Entity<Currency>()
            .Property(x => x.Id)
            .HasDatabaseGeneratedOption(DatabaseGeneratedOption.None);
    }
}
```

**Očekávaný benefit:** EF bude respektovat naše explicit IDs (5, 6) místo auto-generated.

### Proč Selhalo: Pokazí Mappings Pro Composite Keys

**Chyba:**
```
System.Data.Entity.ModelConfiguration.ModelValidationException:
One or more validation errors were detected during model generation:

- DeveloperUsers: EntitySet 'DeveloperUsers' is based on type
  'DeveloperUser' that has no keys defined.

- UserAgendas: EntitySet 'UserAgendas' is based on type
  'UserAgenda' that has no keys defined.

- UserMemberships: EntitySet 'UserMemberships' is based on type
  'UserMembership' that has no keys defined.

... (9 entit celkem)
```

**Analýza:**

Všechny entity které selhaly mají **non-standard keys**:

```csharp
// DeveloperUser - key property není "Id"
public class DeveloperUserMap : EntityTypeConfiguration<DeveloperUser>
{
    this.HasKey(t => t.UserId); // Ne "Id"!
}

// UserAgenda - composite key
public class UserAgendaMap : EntityTypeConfiguration<UserAgenda>
{
    this.HasKey(t => new { t.UserId, t.AgendaId }); // Composite!
}
```

**Root cause:** Volání `modelBuilder.Entity<Currency>()` PO `base.OnModelCreating()` způsobuje rebuild EF modelu, který ztratí některé konfigurace z:

```csharp
// DokladEntitiesContext.cs
modelBuilder.Configurations.AddFromAssembly(this.GetType().Assembly);
// ↑ Load 195+ EntityTypeConfiguration souborů
```

**Závěr:** Nelze použít - pokazí to celý EF model.

---

## 7. Technické Limity Effort

### Limit 1: ExecuteSqlCommand() Není Podporován

```csharp
context.Database.ExecuteSqlCommand(
    "UPDATE Agendas SET ContactId = NULL WHERE ContactId IS NOT NULL"
);

// System.NotSupportedException:
//   Určená metoda není podporována.
```

**Workaround:** Musíme použít EF LINQ API:
```csharp
var agendas = context.Set<Agenda>().ToList(); // Load do paměti
foreach (var agenda in agendas)
{
    agenda.ContactId = null;
}
context.SaveChanges();
```

**Problém:** S 150+ tabulkami a x desítky řádky je to extrémně pomalé.

### Limit 2: Persistent Storage Není Thread-Safe (vyžaduje ještě další ověření a testing)

**Debug log:**
```
[GetConnection BEFORE] Thread: 18, Users: 0, Agendas: 0, SchemaCreated: True
```

Druhý test vidí:
- `SchemaCreated = True` (schema existuje)
- `Users: 0, Agendas: 0` (data zmizela!)

**Možné příčiny:**
- Thread-local storage
- Race condition při paralelním běhu testů
- Effort bug s persistent connections

### Limit 3: Entity Caching Konflikty

```csharp
// UserFactory.cs - původní verze
private static List<User> _entities;
public static List<User> Entities => _entities ??= GetUsers();
```

**Problém:** Static cache způsoboval "email already exists" errory kvůli EF ChangeTracker.

**Workaround:**
```csharp
public static List<User> Entities => GetUsers(); // Always new
```

### Limit 4: User.Validate() Připojuje K Azure DB!

```csharp
// User.cs lines 42-55
public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
{
    // TODO rework - don't use DBContext directly
    using (var context = new DokladEntitiesContext(new DokladConfiguration()))
    {
        // ↑ Vytvoří NOVÝ context s Azure DB connection!
        var emailExist = context.Users.Count(u => u.UserName == this.UserName) > 0;
    }
}
```

**Problém:** I při seedování in-memory DB, validace kontroluje Azure DB!

**Workaround:**
```csharp
context.Configuration.ValidateOnSaveEnabled = false; // Disable při seedu
```

---

## 8. Proč GitHub Příklad Nefunguje V Našem Případě

### GitHub Příklad (Funguje)

```csharp
// Jednoduchý model, Database First nebo POCOs
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}

var connection = Effort.DbConnectionFactory.CreateTransient() as EffortConnection;
connection.DbManager.SetIdentityFields(false);

var context = new SimpleContext(connection);
context.People.Add(new Person { Id = 5, Name = "John" });
context.SaveChanges();

Assert.AreEqual(5, person.Id); // Funguje!
```

### iDoklad Projekt (Nefunguje)

**Rozdíly:**

| Aspekt | GitHub Příklad | iDoklad Projekt |
|--------|---------------|-----------------|
| **EF Approach** | Database First nebo simple Code First | Code First s komplexním modelem |
| **Mappings** | 0-5 jednoduchých entit | 195+ EntityTypeConfiguration souborů |
| **Model Complexity** | Jednoduché POCOs | Circular dependencies, composite keys, validations |
| **Conventions** | Default EF conventions | Custom DecimalPropertyConvention, FunctionsConvention |
| **Dependencies** | Žádné nebo jednoduché | Agenda ↔ Contact, Agenda ↔ BillingAgenda circular |
| **Key Types** | Vždy `int Id` | Composite keys (UserAgenda), custom key properties (DeveloperUser.UserId) |
| **Validations** | Žádné | User.Validate() vytváří vlastní DbContext |
| **Table Count** | 1-10 tabulek | 150+ tabulek |

**SetIdentityFields() API je navrženo pro jednoduché use cases. S komplexním modelem prostě nefunguje.**

---

## 9. Časová Analýza Debugging Effort

### Co se testovalo

| # | Pokus | | Výsledek |
|---|-------|-|----------|
| 1 | CreatePersistent základní verze | | ️ Částečně - IDENTITY problém |
| 2 | Transaction rollback pattern | |  Vnořené transakce selhaly |
| 3 | SetIdentityFields() s normálním contextem | |  ID=0 po SaveChanges |
| 4 | DokladEntitiesContextNoIdentity | |  Pokazí EF mappings |
| 5 | SetIdentityFields() s CreateTransient | |  Stále ID=0 |
| 6 | 4-step process (schema → disable → seed → enable) | |  Stále nefunguje |
| 7 | Debug thread safety issues |  |  Persistent storage unreliable |


**Výsledek:** **Effort nelze použít pro tento use case**

---

## 10. Co Částečně Fungovalo

### CreatePersistent() + Manuální DELETE

```csharp
protected override DokladEntitiesContext GetConnection()
{
    if (currentConnection == null)
    {
        currentConnection = CreatePersistent("DokladTestDb");
        context.Database.CreateIfNotExists(); // 900ms
        context.SeedData();
    }
    else
    {
        // DELETE všech dat
        context.Set<User>().RemoveRange(context.Set<User>());
        context.Set<Agenda>().RemoveRange(context.Set<Agenda>());
        // ... 150+ tabulek
        context.SaveChanges(); // ~10ms

        context.SeedData(); // 12ms
    }

    return context;
}
```

**Performance:**
```
První test:    900ms (schema) + 12ms (seed) = 912ms
Další testy:   10ms (DELETE) + 12ms (seed) = 22ms
────────────────────────────────────────────────────
100 testů:     912ms + 99×22ms = ~2.9 sekundy
Zrychlení:     30× rychlejší než Azure DB
```

**Problémy:**
- IDENTITY se stále neresetuje → FK violations pokud factories mají explicitní IDs
- Musíme DELETE 150+ tabulek (s více daty bude pomalejší)
- Thread safety issues

### CreateTransient() - Vždy Funguje

```csharp
protected override DokladEntitiesContext GetConnection()
{
    var connection = CreateTransient();
    var context = new DokladEntitiesContext(connection);
    context.Database.CreateIfNotExists(); // 900ms
    context.SeedData();
    return context;
}
```

**Performance:**
```
Každý test:    900ms + 12ms = 912ms
────────────────────────────────────
100 testů:     100 × 912ms = ~91 sekund
Zrychlení:     2× rychlejší než Azure DB (ne 45×)
```

**Problémy:**
- Žádná reálná optimalizace
- S 8000+ testy bude horší než Azure DB

---


### Doporučený Postup

#### Fáze 1: SQLite PoC
```
□ Install System.Data.SQLite.EF6 NuGet package
□ Upravit connection string logic v test base
□ Otestovat 5-10 existujících testů
□ Změřit performance
□ Identifikovat SQL dialect issues (pokud nějaké)
```

#### Fáze 2: SQLite Dialects Fix
```
□ Fix GETDATE() → datetime('now')
□ Fix TOP N → LIMIT N
□ Fix [dbo].[Table] → Table
□ Otestovat všechny testy
```

#### Fáze 3: Rollout 
```
□ Migrovat všech 100+ testů na SQLite
□ Update CI/CD pipeline
□ Dokumentace pro tým
```



## Přílohy

### Relevantní Soubory V Projektu

```
src/
├── Doklad.Tests.Base/
│   ├── Infrastructure/
│   │   ├── WebAppInMemoryUnitOfWorkProvider.cs
│   │   └── DokladEntitiesContextNoIdentity.cs (failed attempt)
│   ├── Factories/
│   │   ├── Data/
│   │   │   ├── CurrencyFactory.cs
│   │   │   ├── CountryFactory.cs
│   │   │   ├── UserFactory.cs
│   │   │   └── AgendaFactory.cs
│   │   └── Linkers/
│   │       ├── AgendaContactLinker.cs
│   │       └── AgendaBillingLinker.cs
│   └── Extensions/
│       └── DbContextExtensions.cs
└── Doklad.Integration.Tests/
    └── Doklad.Web.React/
        └── Controllers/
            └── Account/
                └── AccountControllerTests_InMemory.cs
```

###  References

- [Effort GitHub Repository](https://github.com/tamasflamich/effort)
- [Issue #71: SetIdentityFields](https://github.com/tamasflamich/effort/issues/71)
- [EF6 Documentation](https://docs.microsoft.com/en-us/ef/ef6/)
- [SQLite EF6 Provider](https://system.data.sqlite.org/)
