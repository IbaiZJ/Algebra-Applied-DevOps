# Lab 7: Automated testing and the CI/CD pipeline

Use the results of **Lab 6** as the starting point for this lab.

In [Lab 6](Lab06.md), you described the build of an application as code: native build tools and a multi-stage Dockerfile. A successful build only proves that the code **compiles and packages**. It does not prove that the application **does what it should**. In this lab, you will write automated tests on four levels, and then connect them to a **CI/CD pipeline** (GitHub Actions), so that every push and every pull request is built, tested and, only if all tests pass, published as a Docker image.

You will cover:

1. **unit tests** (one class or function, in isolation),
2. **integration tests** (several components together, in-process),
3. **end-to-end (E2E) tests** (the whole running system, through its public interface),
4. **automated UI tests** (a real browser driving the web page),
5. **test automation in a CI/CD pipeline**, built on top of the Docker build from Lab 6.

Every section has examples in both **C#** (.NET, xUnit, NUnit) and **Java** (JUnit 5, Mockito). Do the track that matches your project (or both, if you want to compare them).

> **Prerequisites:** the skills from Labs 4 to 6, a GitHub account, Docker, and either the **.NET SDK 10** or a **JDK 21** with the Maven wrapper (or both). For the C# UI tests you also need **PowerShell (`pwsh`)** to install the browser (`dotnet tool install --global PowerShell` if you do not have it).

---

## Concepts

### Why test automatically?

- A change that breaks existing behavior (a *regression*) is found within minutes, not in production.
- Tests document the intended behavior, and give you the confidence to refactor.
- Automated tests are repeatable and cost nothing to run again, which is the precondition for **Continuous Integration**.

### The test pyramid

```text
                 /\           few       slow      expensive      closest to the user
                /UI\          (a browser drives the page)
               /----\
              / E2E  \        (the whole system is running, tested through its public API)
             /--------\
            /Integration\     (several components together, in one process)
           /------------\
          /     Unit      \   many      fast      cheap          closest to the code
         /------------------\
```

| Level | What is tested | Needs | Speed | Typical failure points | Flakiness risk |
|-------|----------------|-------|-------|------------------------|----------------|
| **Unit** | One class/function, collaborators replaced by test doubles | nothing | milliseconds | logic errors, boundary values | very low |
| **Integration** | Several real components together (controller + service + repository, or code + database) | the application in-process, sometimes a database | tens of ms to seconds | wiring, serialization, configuration, SQL | low |
| **E2E** | The deployed application as a black box, through HTTP | a running application (or container) | seconds | deployment, configuration, real environment | medium |
| **UI** | A real browser, clicking and typing like a user | running application + browser | seconds to minutes | front-end code, browser behavior, the whole stack | **highest** |

The rule: **many fast tests at the bottom, few slow tests at the top.** If something can be checked by a unit test, do not check it in the UI.

### Anatomy of a good test

Use the **Arrange – Act – Assert** structure:

```text
Arrange:  create the object under test and its inputs
Act:      call exactly ONE thing
Assert:   verify the result (or the interaction)
```

Properties of good tests (**FIRST**): **F**ast, **I**ndependent (any order, no shared state), **R**epeatable (same result every time, anywhere), **S**elf-validating (pass/fail, no manual checking), **T**imely (written with or just after the code).

Naming: a test name should say *what is verified and under which condition*, e.g. `Lend_throws_when_item_is_already_lent_out` (C#) or `lend_throwsWhenItemIsAlreadyLentOut` (Java).

### Test doubles

When a unit has collaborators (a repository, a clock, an e-mail sender), replace them in unit tests:

| Double | Description | Example |
|--------|-------------|---------|
| **Dummy** | Passed but never used | `null` or an empty object |
| **Stub** | Returns prepared answers | "`items.Get(1)` returns a Drill" |
| **Fake** | A simplified working implementation | an in-memory repository |
| **Mock** | Stub that also records calls so you can verify them | "`items.Save(...)` was called once" |
| **Spy** | A real object wrapped to record calls | |

Libraries used here: **NSubstitute** (C#) and **Mockito** (Java). Control **time** and **randomness** the same way: inject a clock (`TimeProvider` in C#, `java.time.Clock` in Java) instead of calling `DateTime.Now`/`LocalDate.now()` directly.

### Code coverage

Coverage tells you **which lines your tests executed**. It does *not* tell you whether they were checked. A test without assertions gives 100% coverage and no protection. Use coverage to find **untested code**, and never as the only quality goal. In this lab, you will use it as a **quality gate** (a minimum) in the pipeline.

### What is not automated

Not everything should be a coded test. In Lab 2, you defined which test tasks are *automated* and which are *manual*. Exploratory testing, usability and "does it look right" checks stay manual, and are written as short **manual test cases** (preconditions, steps, expected result).

---

## Task 0: Preparation: the sample application

You need something to test. In this task you will create a small, but not trivial application, **LendIt**, which lends items to borrowers, enforces business rules and has a minimal web page. **The application code is given below. Copy it. Your job in this lab is to write the tests.**

### The domain and its rules

- An **item** can be lent only if it exists and is available.
- A borrower may have at most **3 active loans**; the borrower name must not be blank.
- When an item is **returned**, it becomes available again.
- A **late fee** applies when an item is kept longer than **14 days**: **0.50 per extra day**, at most **10.00**.

### The HTTP API

| Method and path | Description | Success | Errors |
|-----------------|-------------|---------|--------|
| `GET /items` | List items | `200` | |
| `GET /items/{id}` | One item | `200` | `404` |
| `POST /items` `{"name": "Drill"}` | Create an item | `201` | `400` blank name |
| `POST /loans` `{"itemId": 1, "borrower": "ana"}` | Lend an item | `201` | `400` blank borrower, `404` unknown item, `409` rule violated |
| `POST /loans/{id}/return` | Return an item | `200` with `{loan, lateFee}` | `404`, `409` already returned |

Error responses have the body `{"error": "message"}`.

### Repository layout

Create a public repository `testing-demo`:

```text
testing-demo/
├── .github/workflows/          # Task 5
├── dotnet-app/                 # C# track
│   ├── src/LendIt.Api/
│   └── tests/
│       ├── LendIt.Tests/       # unit + integration tests
│       ├── LendIt.E2E.Tests/   # end-to-end tests
│       └── LendIt.UI.Tests/    # Playwright UI tests
└── java-app/                   # Java track
    ├── pom.xml
    └── src/{main,test}/java/hr/lendit/...
```

### The web page (the same for both tracks)

Save as `dotnet-app/src/LendIt.Api/wwwroot/index.html` (C#) and as `java-app/src/main/resources/static/index.html` (Java). Note the `data-testid` attributes: they are **stable hooks for UI tests** that do not break when the design changes.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>LendIt</title>
</head>
<body>
  <h1>LendIt</h1>

  <div>
    <input data-testid="new-item-name" placeholder="New item name">
    <button data-testid="add-item-button" onclick="addItem()">Add item</button>
  </div>

  <div>
    <label>Borrower <input data-testid="borrower-input"></label>
  </div>

  <p data-testid="message" role="status"></p>
  <ul id="items"></ul>

  <script>
    const message = () => document.querySelector('[data-testid=message]');

    async function load() {
      const items = await (await fetch('/items')).json();
      const ul = document.getElementById('items');
      ul.replaceChildren();
      for (const item of items) {
        const li = document.createElement('li');
        li.dataset.testid = 'item-' + item.id;

        const name = document.createElement('span');
        name.textContent = item.name + ' – ';

        const status = document.createElement('span');
        status.dataset.testid = 'item-' + item.id + '-status';
        status.textContent = item.available ? 'Available' : 'Lent out';

        const button = document.createElement('button');
        button.dataset.testid = 'lend-button-' + item.id;
        button.textContent = 'Lend';
        button.disabled = !item.available;
        button.onclick = () => lend(item.id);

        li.append(name, status, ' ', button);
        ul.append(li);
      }
    }

    async function addItem() {
      const input = document.querySelector('[data-testid=new-item-name]');
      const res = await fetch('/items', {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: input.value })
      });
      message().textContent = res.ok ? 'Added ' + input.value : 'Error: ' + (await res.json()).error;
      if (res.ok) input.value = '';
      await load();
    }

    async function lend(itemId) {
      const borrower = document.querySelector('[data-testid=borrower-input]').value;
      const res = await fetch('/loans', {
        method: 'POST', headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ itemId, borrower })
      });
      message().textContent = res.ok ? 'Lent to ' + borrower : 'Error: ' + (await res.json()).error;
      await load();
    }

    load();
  </script>
</body>
</html>
```

### C# track: the application

```bash
mkdir dotnet-app && cd dotnet-app
dotnet new web      -n LendIt.Api       -o src/LendIt.Api
dotnet new xunit    -n LendIt.Tests     -o tests/LendIt.Tests
dotnet new xunit    -n LendIt.E2E.Tests -o tests/LendIt.E2E.Tests
dotnet new nunit    -n LendIt.UI.Tests  -o tests/LendIt.UI.Tests
dotnet new sln      -n LendIt
dotnet sln add src/LendIt.Api tests/LendIt.Tests tests/LendIt.E2E.Tests tests/LendIt.UI.Tests
dotnet new gitignore

dotnet add tests/LendIt.Tests reference src/LendIt.Api
dotnet add tests/LendIt.Tests package NSubstitute
dotnet add tests/LendIt.Tests package Microsoft.Extensions.TimeProvider.Testing
dotnet add tests/LendIt.Tests package Microsoft.AspNetCore.Mvc.Testing
dotnet remove tests/LendIt.Tests package coverlet.collector
dotnet add tests/LendIt.Tests package coverlet.msbuild
dotnet add tests/LendIt.UI.Tests package Microsoft.Playwright.NUnit
```

> `dotnet new xunit` generates an **xUnit v2** project at the time of writing. (The `xunit3` template creates v3, where e.g. `IAsyncLifetime` returns `ValueTask`.) The examples assume v2.
>
> Why NUnit for the UI project? Playwright's official .NET test integration is available for NUnit and MSTest. It manages the browser and page for you (`PageTest`). Using different frameworks in different projects is perfectly normal.

Create the following files in `src/LendIt.Api/`.

**`Domain/Models.cs`**

```csharp
namespace LendIt.Api.Domain;

public record Item(int Id, string Name, bool Available);

public record Loan(int Id, int ItemId, string Borrower, DateOnly LentOn, DateOnly? ReturnedOn);

public record ReturnResult(Loan Loan, decimal LateFee);

public class NotFoundException(string message) : Exception(message);

public class RuleViolationException(string message) : Exception(message);
```

**`Domain/Repositories.cs`**

```csharp
namespace LendIt.Api.Domain;

public interface IItemRepository
{
    Item? Get(int id);
    IReadOnlyList<Item> All();
    Item Add(string name);
    void Save(Item item);
}

public interface ILoanRepository
{
    Loan? Get(int id);
    Loan Add(int itemId, string borrower, DateOnly lentOn);
    void Update(Loan loan);
    int CountActive(string borrower);
}
```

**`Domain/LateFeeCalculator.cs`**

```csharp
namespace LendIt.Api.Domain;

public static class LateFeeCalculator
{
    public const int FreeDays = 14;
    public const decimal FeePerDay = 0.50m;
    public const decimal MaxFee = 10.00m;

    public static decimal Calculate(DateOnly lentOn, DateOnly returnedOn)
    {
        if (returnedOn < lentOn)
            throw new ArgumentException("Return date cannot be before the lending date.");

        var daysLate = returnedOn.DayNumber - lentOn.DayNumber - FreeDays;
        return daysLate <= 0 ? 0m : Math.Min(daysLate * FeePerDay, MaxFee);
    }
}
```

**`Domain/LoanService.cs`**

```csharp
namespace LendIt.Api.Domain;

public class LoanService(IItemRepository items, ILoanRepository loans, TimeProvider clock)
{
    public const int MaxActiveLoans = 3;

    public Loan Lend(int itemId, string borrower)
    {
        if (string.IsNullOrWhiteSpace(borrower))
            throw new ArgumentException("Borrower is required.");

        var item = items.Get(itemId) ?? throw new NotFoundException($"Item {itemId} not found.");

        if (!item.Available)
            throw new RuleViolationException("Item is already lent out.");

        if (loans.CountActive(borrower) >= MaxActiveLoans)
            throw new RuleViolationException($"{borrower} already has {MaxActiveLoans} active loans.");

        items.Save(item with { Available = false });
        return loans.Add(itemId, borrower, Today());
    }

    public ReturnResult Return(int loanId)
    {
        var loan = loans.Get(loanId) ?? throw new NotFoundException($"Loan {loanId} not found.");

        if (loan.ReturnedOn is not null)
            throw new RuleViolationException("Loan is already returned.");

        var today = Today();
        var returned = loan with { ReturnedOn = today };
        loans.Update(returned);

        var item = items.Get(loan.ItemId) ?? throw new NotFoundException($"Item {loan.ItemId} not found.");
        items.Save(item with { Available = true });

        return new ReturnResult(returned, LateFeeCalculator.Calculate(loan.LentOn, today));
    }

    private DateOnly Today() => DateOnly.FromDateTime(clock.GetUtcNow().UtcDateTime);
}
```

**`Infrastructure/InMemoryRepositories.cs`**

```csharp
using System.Collections.Concurrent;
using LendIt.Api.Domain;

namespace LendIt.Api.Infrastructure;

public class InMemoryItemRepository : IItemRepository
{
    private readonly ConcurrentDictionary<int, Item> _items = new();
    private int _nextId;

    public Item? Get(int id) => _items.TryGetValue(id, out var item) ? item : null;

    public IReadOnlyList<Item> All() => _items.Values.OrderBy(i => i.Id).ToList();

    public Item Add(string name)
    {
        var item = new Item(Interlocked.Increment(ref _nextId), name, true);
        _items[item.Id] = item;
        return item;
    }

    public void Save(Item item) => _items[item.Id] = item;
}

public class InMemoryLoanRepository : ILoanRepository
{
    private readonly ConcurrentDictionary<int, Loan> _loans = new();
    private int _nextId;

    public Loan? Get(int id) => _loans.TryGetValue(id, out var loan) ? loan : null;

    public Loan Add(int itemId, string borrower, DateOnly lentOn)
    {
        var loan = new Loan(Interlocked.Increment(ref _nextId), itemId, borrower, lentOn, null);
        _loans[loan.Id] = loan;
        return loan;
    }

    public void Update(Loan loan) => _loans[loan.Id] = loan;

    public int CountActive(string borrower) =>
        _loans.Values.Count(l => l.Borrower == borrower && l.ReturnedOn is null);
}
```

**`Program.cs`** (replace the generated content)

```csharp
using LendIt.Api.Domain;
using LendIt.Api.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IItemRepository, InMemoryItemRepository>();
builder.Services.AddSingleton<ILoanRepository, InMemoryLoanRepository>();
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddSingleton<LoanService>();

var app = builder.Build();

app.UseDefaultFiles();
app.UseStaticFiles();

app.MapGet("/items", (IItemRepository items) => items.All());

app.MapGet("/items/{id:int}", (int id, IItemRepository items) =>
    items.Get(id) is { } item
        ? Results.Ok(item)
        : Results.NotFound(new { error = $"Item {id} not found." }));

app.MapPost("/items", (CreateItemRequest request, IItemRepository items) =>
{
    if (string.IsNullOrWhiteSpace(request.Name))
        return Results.BadRequest(new { error = "Name is required." });

    var item = items.Add(request.Name.Trim());
    return Results.Created($"/items/{item.Id}", item);
});

app.MapPost("/loans", (LendRequest request, LoanService service) =>
    Handle(() =>
    {
        var loan = service.Lend(request.ItemId, request.Borrower ?? string.Empty);
        return Results.Created($"/loans/{loan.Id}", loan);
    }));

app.MapPost("/loans/{id:int}/return", (int id, LoanService service) =>
    Handle(() => Results.Ok(service.Return(id))));

app.Run();

static IResult Handle(Func<IResult> action)
{
    try { return action(); }
    catch (NotFoundException e) { return Results.NotFound(new { error = e.Message }); }
    catch (RuleViolationException e) { return Results.Conflict(new { error = e.Message }); }
    catch (ArgumentException e) { return Results.BadRequest(new { error = e.Message }); }
}

public record CreateItemRequest(string? Name);
public record LendRequest(int ItemId, string? Borrower);

public partial class Program { }   // makes the entry point visible to the integration tests
```

Run it: `dotnet run --project src/LendIt.Api` and try the web page and the API (`curl -X POST localhost:<port>/items -H "Content-Type: application/json" -d '{"name":"Drill"}'`).

### Java track: the application

Generate a Spring Boot project:

```bash
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d groupId=hr.lendit -d artifactId=lendit -d name=lendit -d packageName=hr.lendit \
  -d dependencies=web -o lendit.zip
unzip lendit.zip -d java-app && rm lendit.zip
```

> **Spring Boot versions.** Spring Boot 4 renamed several starters and moved many test annotations to new packages (for example `@AutoConfigureMockMvc`). To work with **any** Boot version, the tests in this lab deliberately use only `@SpringBootTest`, the JDK `HttpClient` and JSON Path, which are stable across versions. Do not replace them with version-specific helpers unless you know which Boot version you have.

Add the following to `pom.xml`: the test dependencies inside `<dependencies>`, a property for the Playwright version, and the JaCoCo plugin inside `<build><plugins>`.

```xml
<properties>
  <java.version>21</java.version>
  <!-- check Maven Central for the latest Playwright version -->
  <playwright.version>1.55.0</playwright.version>
</properties>

<dependencies>
  <!-- ...the generated dependencies stay... -->
  <dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>com.microsoft.playwright</groupId>
    <artifactId>playwright</artifactId>
    <version>${playwright.version}</version>
    <scope>test</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <!-- ...the generated spring-boot-maven-plugin stays... -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.14</version>
      <executions>
        <execution>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals><goal>report</goal></goals>
          <configuration>
            <includes><include>hr/lendit/domain/*</include></includes>
          </configuration>
        </execution>
        <execution>
          <id>check</id>
          <phase>verify</phase>
          <goals><goal>check</goal></goals>
          <configuration>
            <includes><include>hr/lendit/domain/*</include></includes>
            <rules>
              <rule>
                <element>BUNDLE</element>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>
                  </limit>
                </limits>
              </rule>
            </rules>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

(The JaCoCo rule is a **quality gate**: `mvn verify` fails when less than 80% of the lines in the `domain` package are covered. You will use it in Task 5.)

Create these files under `src/main/java/hr/lendit/`. The generated `LenditApplication.java` only needs a `Clock` bean:

**`LenditApplication.java`**

```java
package hr.lendit;

import java.time.Clock;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class LenditApplication {

    public static void main(String[] args) {
        SpringApplication.run(LenditApplication.class, args);
    }

    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }
}
```

**`domain/Models.java`** (put each public type in its own file in your project; they are shown together here for brevity)

```java
package hr.lendit.domain;

import java.math.BigDecimal;
import java.time.LocalDate;

public record Item(long id, String name, boolean available) {}

public record Loan(long id, long itemId, String borrower, LocalDate lentOn, LocalDate returnedOn) {}

public record ReturnResult(Loan loan, BigDecimal lateFee) {}

public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) { super(message); }
}

public class RuleViolationException extends RuntimeException {
    public RuleViolationException(String message) { super(message); }
}
```

**`domain/ItemRepository.java`** and **`domain/LoanRepository.java`**

```java
package hr.lendit.domain;

import java.util.List;
import java.util.Optional;

public interface ItemRepository {
    Optional<Item> findById(long id);
    List<Item> findAll();
    Item add(String name);
    void update(Item item);
}
```

```java
package hr.lendit.domain;

import java.time.LocalDate;
import java.util.Optional;

public interface LoanRepository {
    Optional<Loan> findById(long id);
    Loan add(long itemId, String borrower, LocalDate lentOn);
    void update(Loan loan);
    long countActiveByBorrower(String borrower);
}
```

**`domain/LateFeeCalculator.java`**

```java
package hr.lendit.domain;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public final class LateFeeCalculator {

    public static final int FREE_DAYS = 14;
    public static final BigDecimal FEE_PER_DAY = new BigDecimal("0.50");
    public static final BigDecimal MAX_FEE = new BigDecimal("10.00");

    private LateFeeCalculator() {}

    public static BigDecimal calculate(LocalDate lentOn, LocalDate returnedOn) {
        if (returnedOn.isBefore(lentOn)) {
            throw new IllegalArgumentException("Return date cannot be before the lending date.");
        }
        long daysLate = ChronoUnit.DAYS.between(lentOn, returnedOn) - FREE_DAYS;
        if (daysLate <= 0) {
            return BigDecimal.ZERO;
        }
        return FEE_PER_DAY.multiply(BigDecimal.valueOf(daysLate)).min(MAX_FEE);
    }
}
```

**`domain/LoanService.java`**

```java
package hr.lendit.domain;

import java.time.Clock;
import java.time.LocalDate;
import org.springframework.stereotype.Service;

@Service
public class LoanService {

    public static final int MAX_ACTIVE_LOANS = 3;

    private final ItemRepository items;
    private final LoanRepository loans;
    private final Clock clock;

    public LoanService(ItemRepository items, LoanRepository loans, Clock clock) {
        this.items = items;
        this.loans = loans;
        this.clock = clock;
    }

    public Loan lend(long itemId, String borrower) {
        if (borrower == null || borrower.isBlank()) {
            throw new IllegalArgumentException("Borrower is required.");
        }
        Item item = items.findById(itemId)
                .orElseThrow(() -> new NotFoundException("Item " + itemId + " not found."));

        if (!item.available()) {
            throw new RuleViolationException("Item is already lent out.");
        }
        if (loans.countActiveByBorrower(borrower) >= MAX_ACTIVE_LOANS) {
            throw new RuleViolationException(borrower + " already has " + MAX_ACTIVE_LOANS + " active loans.");
        }

        items.update(new Item(item.id(), item.name(), false));
        return loans.add(itemId, borrower, LocalDate.now(clock));
    }

    public ReturnResult returnLoan(long loanId) {
        Loan loan = loans.findById(loanId)
                .orElseThrow(() -> new NotFoundException("Loan " + loanId + " not found."));

        if (loan.returnedOn() != null) {
            throw new RuleViolationException("Loan is already returned.");
        }

        LocalDate today = LocalDate.now(clock);
        Loan returned = new Loan(loan.id(), loan.itemId(), loan.borrower(), loan.lentOn(), today);
        loans.update(returned);

        Item item = items.findById(loan.itemId())
                .orElseThrow(() -> new NotFoundException("Item " + loan.itemId() + " not found."));
        items.update(new Item(item.id(), item.name(), true));

        return new ReturnResult(returned, LateFeeCalculator.calculate(loan.lentOn(), today));
    }
}
```

**`infrastructure/InMemoryItemRepository.java`** and **`infrastructure/InMemoryLoanRepository.java`**

```java
package hr.lendit.infrastructure;

import hr.lendit.domain.Item;
import hr.lendit.domain.ItemRepository;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import org.springframework.stereotype.Repository;

@Repository
public class InMemoryItemRepository implements ItemRepository {

    private final Map<Long, Item> items = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong();

    @Override public Optional<Item> findById(long id) { return Optional.ofNullable(items.get(id)); }

    @Override public List<Item> findAll() {
        return items.values().stream().sorted(Comparator.comparingLong(Item::id)).toList();
    }

    @Override public Item add(String name) {
        Item item = new Item(nextId.incrementAndGet(), name, true);
        items.put(item.id(), item);
        return item;
    }

    @Override public void update(Item item) { items.put(item.id(), item); }
}
```

```java
package hr.lendit.infrastructure;

import hr.lendit.domain.Loan;
import hr.lendit.domain.LoanRepository;
import java.time.LocalDate;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import org.springframework.stereotype.Repository;

@Repository
public class InMemoryLoanRepository implements LoanRepository {

    private final Map<Long, Loan> loans = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong();

    @Override public Optional<Loan> findById(long id) { return Optional.ofNullable(loans.get(id)); }

    @Override public Loan add(long itemId, String borrower, LocalDate lentOn) {
        Loan loan = new Loan(nextId.incrementAndGet(), itemId, borrower, lentOn, null);
        loans.put(loan.id(), loan);
        return loan;
    }

    @Override public void update(Loan loan) { loans.put(loan.id(), loan); }

    @Override public long countActiveByBorrower(String borrower) {
        return loans.values().stream()
                .filter(l -> l.borrower().equals(borrower) && l.returnedOn() == null)
                .count();
    }
}
```

**`web/LendItController.java`** and **`web/ApiExceptionHandler.java`**

```java
package hr.lendit.web;

import hr.lendit.domain.*;
import java.net.URI;
import java.util.List;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
public class LendItController {

    public record CreateItemRequest(String name) {}
    public record LendRequest(long itemId, String borrower) {}

    private final ItemRepository items;
    private final LoanService loanService;

    public LendItController(ItemRepository items, LoanService loanService) {
        this.items = items;
        this.loanService = loanService;
    }

    @GetMapping("/items")
    public List<Item> items() {
        return items.findAll();
    }

    @GetMapping("/items/{id}")
    public Item item(@PathVariable long id) {
        return items.findById(id).orElseThrow(() -> new NotFoundException("Item " + id + " not found."));
    }

    @PostMapping("/items")
    public ResponseEntity<Item> createItem(@RequestBody CreateItemRequest request) {
        if (request.name() == null || request.name().isBlank()) {
            throw new IllegalArgumentException("Name is required.");
        }
        Item item = items.add(request.name().trim());
        return ResponseEntity.created(URI.create("/items/" + item.id())).body(item);
    }

    @PostMapping("/loans")
    public ResponseEntity<Loan> lend(@RequestBody LendRequest request) {
        Loan loan = loanService.lend(request.itemId(), request.borrower());
        return ResponseEntity.created(URI.create("/loans/" + loan.id())).body(loan);
    }

    @PostMapping("/loans/{id}/return")
    public ReturnResult returnLoan(@PathVariable long id) {
        return loanService.returnLoan(id);
    }
}
```

```java
package hr.lendit.web;

import hr.lendit.domain.NotFoundException;
import hr.lendit.domain.RuleViolationException;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
class ApiExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    ResponseEntity<Map<String, String>> notFound(NotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler(RuleViolationException.class)
    ResponseEntity<Map<String, String>> conflict(RuleViolationException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of("error", e.getMessage()));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    ResponseEntity<Map<String, String>> badRequest(IllegalArgumentException e) {
        return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
    }
}
```

Run it: `./mvnw spring-boot:run` (the application listens on `localhost:8080`), and try the web page and the API.

### Expected result of Task 0

- The application runs in your track(s), and the page at `/` lets you add an item and lend it.
- The code is committed on a branch and merged by a pull request into `main`.

---

## Task 1: Unit tests

A unit test checks **one unit** (a class or a function) in isolation: no web server, no files, no network, no database. Its collaborators are replaced by test doubles, and the clock is controlled.

The two units worth testing here are `LateFeeCalculator` (pure logic, no collaborators) and `LoanService` (business rules, with collaborators).

### 1.1 Parameterized tests for pure logic

The same test with different data: ideal for **boundary values**. Here the interesting boundaries are 14 days (last free day) and 15 days (first fee), and the cap.

**C#: `tests/LendIt.Tests/Unit/LateFeeCalculatorTests.cs`**

```csharp
using LendIt.Api.Domain;

namespace LendIt.Tests.Unit;

[Trait("Category", "Unit")]
public class LateFeeCalculatorTests
{
    private static readonly DateOnly LentOn = new(2026, 1, 1);

    [Theory]
    [InlineData(0, 0.00)]
    [InlineData(14, 0.00)]     // last free day
    [InlineData(15, 0.50)]     // first day with a fee
    [InlineData(20, 3.00)]
    [InlineData(34, 10.00)]    // cap reached exactly
    [InlineData(100, 10.00)]   // cap holds
    public void Fee_depends_on_the_number_of_days_kept(int daysKept, double expectedFee)
    {
        var fee = LateFeeCalculator.Calculate(LentOn, LentOn.AddDays(daysKept));

        Assert.Equal((decimal)expectedFee, fee);
    }

    [Fact]
    public void Returning_before_lending_is_rejected()
    {
        Assert.Throws<ArgumentException>(
            () => LateFeeCalculator.Calculate(LentOn, LentOn.AddDays(-1)));
    }
}
```

**Java: `src/test/java/hr/lendit/domain/LateFeeCalculatorTest.java`**

```java
package hr.lendit.domain;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.math.BigDecimal;
import java.time.LocalDate;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@Tag("unit")
class LateFeeCalculatorTest {

    private static final LocalDate LENT_ON = LocalDate.of(2026, 1, 1);

    @ParameterizedTest(name = "{0} days kept -> fee {1}")
    @CsvSource({
            "0,   0.00",
            "14,  0.00",   // last free day
            "15,  0.50",   // first day with a fee
            "20,  3.00",
            "34,  10.00",  // cap reached exactly
            "100, 10.00"   // cap holds
    })
    void feeDependsOnTheNumberOfDaysKept(int daysKept, BigDecimal expectedFee) {
        BigDecimal fee = LateFeeCalculator.calculate(LENT_ON, LENT_ON.plusDays(daysKept));

        assertThat(fee).isEqualByComparingTo(expectedFee);
    }

    @Test
    void returningBeforeLendingIsRejected() {
        assertThatThrownBy(() -> LateFeeCalculator.calculate(LENT_ON, LENT_ON.minusDays(1)))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

Note `isEqualByComparingTo`: `new BigDecimal("0.0")` and `BigDecimal.ZERO` are *not* `equals()`, because their scale differs. Why does `Assert.Equal` on `decimal` in C# not have this problem?

### 1.2 Unit tests with test doubles and a controlled clock

**C#: `tests/LendIt.Tests/Unit/LoanServiceTests.cs`**

```csharp
using LendIt.Api.Domain;
using Microsoft.Extensions.Time.Testing;
using NSubstitute;

namespace LendIt.Tests.Unit;

[Trait("Category", "Unit")]
public class LoanServiceTests
{
    private static readonly DateOnly Today = new(2026, 1, 15);

    private readonly IItemRepository _items = Substitute.For<IItemRepository>();
    private readonly ILoanRepository _loans = Substitute.For<ILoanRepository>();
    private readonly FakeTimeProvider _clock = new(new DateTimeOffset(2026, 1, 15, 10, 0, 0, TimeSpan.Zero));
    private readonly LoanService _sut;

    public LoanServiceTests() => _sut = new LoanService(_items, _loans, _clock);

    [Fact]
    public void Lend_marks_the_item_as_unavailable_and_creates_a_loan()
    {
        // Arrange
        _items.Get(1).Returns(new Item(1, "Drill", true));
        _loans.CountActive("ana").Returns(0);
        _loans.Add(1, "ana", Today).Returns(new Loan(7, 1, "ana", Today, null));

        // Act
        var loan = _sut.Lend(1, "ana");

        // Assert
        Assert.Equal(7, loan.Id);
        _items.Received(1).Save(Arg.Is<Item>(i => i.Id == 1 && !i.Available));
    }

    [Fact]
    public void Lend_throws_when_the_item_does_not_exist()
    {
        _items.Get(99).Returns((Item?)null);

        Assert.Throws<NotFoundException>(() => _sut.Lend(99, "ana"));
    }

    [Fact]
    public void Lend_throws_when_the_item_is_already_lent_out()
    {
        _items.Get(1).Returns(new Item(1, "Drill", false));

        Assert.Throws<RuleViolationException>(() => _sut.Lend(1, "ana"));
        _loans.DidNotReceive().Add(Arg.Any<int>(), Arg.Any<string>(), Arg.Any<DateOnly>());
    }

    [Theory]
    [InlineData("")]
    [InlineData("   ")]
    public void Lend_rejects_a_blank_borrower(string borrower)
    {
        Assert.Throws<ArgumentException>(() => _sut.Lend(1, borrower));
    }
}
```

**Java: `src/test/java/hr/lendit/domain/LoanServiceTest.java`**

```java
package hr.lendit.domain;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.time.Clock;
import java.time.Instant;
import java.time.LocalDate;
import java.time.ZoneOffset;
import java.util.Optional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@Tag("unit")
@ExtendWith(MockitoExtension.class)
class LoanServiceTest {

    private static final LocalDate TODAY = LocalDate.of(2026, 1, 15);

    @Mock ItemRepository items;
    @Mock LoanRepository loans;

    private final Clock clock = Clock.fixed(Instant.parse("2026-01-15T10:00:00Z"), ZoneOffset.UTC);
    private LoanService service;

    @BeforeEach
    void setUp() {
        service = new LoanService(items, loans, clock);
    }

    @Test
    void lend_marksTheItemAsUnavailableAndCreatesALoan() {
        // Arrange
        when(items.findById(1L)).thenReturn(Optional.of(new Item(1, "Drill", true)));
        when(loans.countActiveByBorrower("ana")).thenReturn(0L);
        when(loans.add(1L, "ana", TODAY)).thenReturn(new Loan(7, 1, "ana", TODAY, null));

        // Act
        Loan loan = service.lend(1L, "ana");

        // Assert
        assertThat(loan.id()).isEqualTo(7);
        verify(items).update(new Item(1, "Drill", false));
    }

    @Test
    void lend_throwsWhenTheItemDoesNotExist() {
        when(items.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.lend(99L, "ana"))
                .isInstanceOf(NotFoundException.class)
                .hasMessageContaining("99");
    }

    @Test
    void lend_throwsWhenTheItemIsAlreadyLentOut() {
        when(items.findById(1L)).thenReturn(Optional.of(new Item(1, "Drill", false)));

        assertThatThrownBy(() -> service.lend(1L, "ana"))
                .isInstanceOf(RuleViolationException.class);
        verify(loans, never()).add(anyLong(), any(), any());
    }

    @ParameterizedTest
    @ValueSource(strings = {"", "   "})
    void lend_rejectsABlankBorrower(String borrower) {
        assertThatThrownBy(() -> service.lend(1L, borrower))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

Observe how time is controlled: the tests never depend on today's date, so they give the same result on every day and on every machine. Mockito's default **strict stubs** fail a test that stubs something the code never calls. Why is that useful?

### 1.3 Your unit tests (to write)

Write the remaining tests yourself. At least:

| Class | Test cases to write |
|-------|---------------------|
| `LateFeeCalculator` | at least 6 cases (the ones above are a start), plus one test of your own boundary |
| `LoanService.Lend` | borrower has **exactly 3** active loans (rejected), has **2** (allowed): the boundary of the rule; `null` borrower |
| `LoanService.Return` | returns the item to *available*; the late fee is 0 within 14 days; the late fee is calculated after 20 days (advance the clock: `_clock.Advance(TimeSpan.FromDays(20))`, or build a second fixed `Clock`); unknown loan → not found; already returned loan → rule violation |

Rules:

- one logical assertion per test, with the AAA structure and a name that describes the behavior;
- do not test the repositories' mocks. Test **your** code.

### 1.4 Run the tests and measure coverage

**C#**

```bash
dotnet test tests/LendIt.Tests --filter "Category=Unit"
dotnet test tests/LendIt.Tests --filter "Category=Unit" \
  -p:CollectCoverage=true -p:CoverletOutputFormat=cobertura \
  -p:CoverletOutput=../../TestResults/coverage/ \
  -p:Include="[LendIt.Api]LendIt.Api.Domain.*" \
  -p:Threshold=80 -p:ThresholdType=line -p:ThresholdStat=total
```

(Do not run a plain `dotnet test` at the solution level: it would also start the E2E and UI projects, which need a running application.)

**Java**

```bash
./mvnw -B test -Dgroups=unit                   # Windows: mvnw.cmd -B test -Dgroups=unit
./mvnw -B verify -Dgroups=unit                 # also runs the JaCoCo coverage gate (80% of the domain)
# report: target/site/jacoco/index.html
```

Open the report. Which lines or branches of the domain are not covered? Add tests until the gate passes.

### 1.5 Find the bug on purpose (mutation by hand)

Introduce a bug in the service: change `>= MaxActiveLoans` to `> MaxActiveLoans`. Run the unit tests. **Did a test fail?** If not, your boundary test is missing, and coverage alone did not tell you that, because the line was executed. Fix the test suite, then revert the bug. Write down what you learned in the report.

### Expected result of Task 1

- At least 15 unit tests for `LateFeeCalculator` and `LoanService` in your track, all passing.
- Coverage of the domain code of at least 80%, with a screenshot of the report.
- The result of the "bug on purpose" experiment.

---

## Task 2: Integration tests

An integration test checks that **real components work together**: routing, JSON serialization, validation, the service, the repositories and the error handling. It still runs **in one process**, with no deployment. It is slower than a unit test, but it finds problems that unit tests cannot: a wrong URL, a missing registration in the dependency injection container, a property with a different JSON name.

### 2.1 C#: `WebApplicationFactory`

`WebApplicationFactory<Program>` starts the real application **in memory** (a test server) and gives you an `HttpClient` for it. That is why `Program` was made `public partial`.

**`tests/LendIt.Tests/Integration/LoanApiTests.cs`**

```csharp
using System.Net;
using System.Net.Http.Json;
using LendIt.Api.Domain;
using Microsoft.AspNetCore.Mvc.Testing;

namespace LendIt.Tests.Integration;

[Trait("Category", "Integration")]
public class LoanApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client = factory.CreateClient();

    // test data helper: every test creates its own item, so tests stay independent
    private async Task<Item> CreateItem(string name)
    {
        var response = await _client.PostAsJsonAsync("/items", new { name });
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        return (await response.Content.ReadFromJsonAsync<Item>())!;
    }

    [Fact]
    public async Task Lending_an_item_returns_201_and_makes_the_item_unavailable()
    {
        var item = await CreateItem("Drill");

        var response = await _client.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower = "ana-1" });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var updated = await _client.GetFromJsonAsync<Item>($"/items/{item.Id}");
        Assert.False(updated!.Available);
    }

    [Fact]
    public async Task Lending_the_same_item_twice_returns_409()
    {
        var item = await CreateItem("Ladder");
        await _client.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower = "ana-2" });

        var second = await _client.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower = "marko-2" });

        Assert.Equal(HttpStatusCode.Conflict, second.StatusCode);
    }

    [Fact]
    public async Task Lending_an_unknown_item_returns_404_with_an_error_message()
    {
        var response = await _client.PostAsJsonAsync("/loans", new { itemId = 424242, borrower = "ana-3" });

        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
        Assert.Contains("not found", await response.Content.ReadAsStringAsync());
    }
}
```

**Controlling time in an integration test** (replace a service of the real application): the late fee needs a loan that is 20 days old, and we cannot wait 20 days.

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using Microsoft.Extensions.Time.Testing;

[Fact]
public async Task Returning_after_20_days_charges_a_late_fee()
{
    var clock = new FakeTimeProvider(new DateTimeOffset(2026, 1, 15, 10, 0, 0, TimeSpan.Zero));
    await using var timeTravelFactory = new WebApplicationFactory<Program>()
        .WithWebHostBuilder(builder => builder.ConfigureServices(services =>
        {
            services.RemoveAll<TimeProvider>();
            services.AddSingleton<TimeProvider>(clock);
        }));
    var client = timeTravelFactory.CreateClient();

    var item = (await (await client.PostAsJsonAsync("/items", new { name = "Drill" }))
        .Content.ReadFromJsonAsync<Item>())!;
    var loan = (await (await client.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower = "ana-4" }))
        .Content.ReadFromJsonAsync<Loan>())!;

    clock.Advance(TimeSpan.FromDays(20));
    var response = await client.PostAsync($"/loans/{loan.Id}/return", content: null);

    var result = await response.Content.ReadFromJsonAsync<ReturnResult>();
    Assert.Equal(3.00m, result!.LateFee);
}
```

Run: `dotnet test tests/LendIt.Tests --filter "Category=Integration"`.

### 2.2 Java: `@SpringBootTest` with a real server on a random port

`@SpringBootTest(webEnvironment = RANDOM_PORT)` starts the full Spring context and an embedded web server on a free port. The test then calls it over real HTTP. First, a small helper that you will reuse in Tasks 3 and 4.

**`src/test/java/hr/lendit/support/TestApi.java`**

```java
package hr.lendit.support;

import com.jayway.jsonpath.JsonPath;
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

/** A tiny HTTP client for tests: calls the application under test and prepares test data. */
public class TestApi {

    private final String baseUrl;
    private final HttpClient http = HttpClient.newHttpClient();

    public TestApi(String baseUrl) {
        this.baseUrl = baseUrl;
    }

    public HttpResponse<String> get(String path) throws IOException, InterruptedException {
        var request = HttpRequest.newBuilder(URI.create(baseUrl + path)).GET().build();
        return http.send(request, HttpResponse.BodyHandlers.ofString());
    }

    public HttpResponse<String> post(String path, String json) throws IOException, InterruptedException {
        var request = HttpRequest.newBuilder(URI.create(baseUrl + path))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(json))
                .build();
        return http.send(request, HttpResponse.BodyHandlers.ofString());
    }

    /** Creates an item and returns its id. */
    public long createItem(String name) throws IOException, InterruptedException {
        var response = post("/items", "{\"name\":\"" + name + "\"}");
        if (response.statusCode() != 201) {
            throw new IllegalStateException("Could not create item: " + response.statusCode() + " " + response.body());
        }
        return ((Number) JsonPath.read(response.body(), "$.id")).longValue();
    }

    public static String lendJson(long itemId, String borrower) {
        return "{\"itemId\":" + itemId + ",\"borrower\":\"" + borrower + "\"}";
    }

    public static boolean isAvailable(String itemJson) {
        return JsonPath.read(itemJson, "$.available");
    }
}
```

**`src/test/java/hr/lendit/web/LoanApiTest.java`**

```java
package hr.lendit.web;

import static org.assertj.core.api.Assertions.assertThat;

import hr.lendit.support.TestApi;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.context.SpringBootTest;

@Tag("integration")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class LoanApiTest {

    @Value("${local.server.port}")
    int port;

    private TestApi api() {
        return new TestApi("http://localhost:" + port);
    }

    @Test
    void lendingAnItem_returns201_andMakesTheItemUnavailable() throws Exception {
        long itemId = api().createItem("Drill");

        var response = api().post("/loans", TestApi.lendJson(itemId, "ana-1"));

        assertThat(response.statusCode()).isEqualTo(201);
        assertThat(TestApi.isAvailable(api().get("/items/" + itemId).body())).isFalse();
    }

    @Test
    void lendingTheSameItemTwice_returns409() throws Exception {
        long itemId = api().createItem("Ladder");
        api().post("/loans", TestApi.lendJson(itemId, "ana-2"));

        var second = api().post("/loans", TestApi.lendJson(itemId, "marko-2"));

        assertThat(second.statusCode()).isEqualTo(409);
    }

    @Test
    void lendingAnUnknownItem_returns404WithAnErrorMessage() throws Exception {
        var response = api().post("/loans", TestApi.lendJson(424242, "ana-3"));

        assertThat(response.statusCode()).isEqualTo(404);
        assertThat(response.body()).contains("not found");
    }
}
```

The `Clock` is a Spring bean, so a test can replace it with a `@TestConfiguration` that defines a different `@Primary` `Clock` bean (give it a different method name than the production `clock()` bean, because bean-name overriding is disabled by default) and imports it with `@Import`. Use this idea for a late-fee test with a mutable clock of your own.

Run: `./mvnw -B test -Dgroups=integration`.

### 2.3 Your integration tests (to write)

Write at least these, in your track:

1. `POST /items` with a blank name → `400` and an error message.
2. `POST /loans` with a blank borrower → `400`.
3. A borrower's **4th** loan is rejected with `409` (create 4 items, lend 3, then the 4th).
4. After `POST /loans/{id}/return`, `GET /items/{id}` shows the item as available again, and returning the same loan twice gives `409`.
5. `GET /items` contains the item you created.

### 2.4 Integration with a real database (when you have one)

LendIt keeps its data in memory, so a database is not needed here. In most projects, though, the integration tests that matter most are **the ones that talk to the real database**, because SQL dialects, constraints and migrations behave differently from a mock. The accepted approach is **Testcontainers**: a library that starts a throw-away Docker container (for example PostgreSQL) for the test run, so every developer and the CI server test against the *same* database engine as in production. It needs Docker, which you already have from Lab 6. See the Testcontainers documentation for [.NET](https://dotnet.testcontainers.org/) and [Java](https://java.testcontainers.org/). If your project has a database, use this approach for your own integration tests in Task 6.

### Questions for your report

1. Name two bugs that your integration tests can find and your unit tests cannot.
2. Why does every test create its own item and use its own borrower name? What would go wrong if tests shared data and ran in a different order?
3. Why is the integration test slower than a unit test, in which part of its work?

### Expected result of Task 2

- At least 8 integration tests in your track (3 given + 5 yours), all passing, and a screenshot of the test run.
- Answers to the questions.

---

## Task 3: End-to-end (E2E) tests

An E2E test treats the application as a **black box**, as deployed: a **real running process or container**, with real configuration, reached over the network. It does not reference the application code, and it does not replace any services. It verifies a whole **user journey**, not a single rule.

| | Integration test (Task 2) | E2E test (this task) |
|---|---|---|
| Application | started *inside* the test process | already running somewhere (container, server) |
| Address | in-memory or random port | `BASE_URL`, e.g. `http://localhost:8080` |
| Knows the code? | references the application project | **no**, only the HTTP interface |
| Failures it finds | wiring, serialization, rules | **also**: Dockerfile, ports, configuration, runtime |

### 3.1 Run the application as a container (from Lab 6)

E2E tests run against the **image** produced by your Docker build. That is the same artifact that will be deployed.

**C#**: create `dotnet-app/Dockerfile` (the multi-stage Dockerfile from Lab 6, adapted to the new project name) and `dotnet-app/.dockerignore`:

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY src/LendIt.Api/LendIt.Api.csproj src/LendIt.Api/
RUN dotnet restore src/LendIt.Api/LendIt.Api.csproj
COPY src/LendIt.Api/ src/LendIt.Api/
RUN dotnet publish src/LendIt.Api/LendIt.Api.csproj -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "LendIt.Api.dll"]
```

```text
# .dockerignore
**/bin/
**/obj/
**/TestResults/
tests/
.git/
```

**Java**: reuse the multi-stage `java-app/Dockerfile` and `.dockerignore` from Lab 6 unchanged (the `COPY --from=build /build/target/*.jar app.jar` line finds the new jar). Add `src/test/` to `.dockerignore` if you like, as the tests are not needed in the image.

Build and start the container:

```bash
cd dotnet-app          # or java-app
docker build -t lendit:local .
docker run -d --name lendit -p 8080:8080 lendit:local
curl http://localhost:8080/items         # []
```

### 3.2 The E2E journey tests

An E2E test covers a complete journey: *create an item → lend it → see that it is unavailable → try to lend it again (rejected) → return it → see that it is available.* The tests read the address from the `BASE_URL` environment variable.

**C#: `tests/LendIt.E2E.Tests/LoanJourneyTests.cs`**

```csharp
using System.Net;
using System.Net.Http.Json;

namespace LendIt.E2E.Tests;

// Black box: no reference to the application project. The DTOs are declared here.
public record ItemDto(int Id, string Name, bool Available);
public record LoanDto(int Id, int ItemId, string Borrower);

[Trait("Category", "E2E")]
public class LoanJourneyTests
{
    private readonly HttpClient _http = new()
    {
        BaseAddress = new Uri(Environment.GetEnvironmentVariable("BASE_URL") ?? "http://localhost:8080")
    };

    [Fact]
    public async Task A_borrower_can_lend_and_return_an_item()
    {
        // create the test data through the public API
        var created = await _http.PostAsJsonAsync("/items", new { name = "Projector" });
        var item = (await created.Content.ReadFromJsonAsync<ItemDto>())!;
        var borrower = $"ana-{Guid.NewGuid():N}";

        // lend
        var lend = await _http.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower });
        Assert.Equal(HttpStatusCode.Created, lend.StatusCode);
        var loan = (await lend.Content.ReadFromJsonAsync<LoanDto>())!;
        Assert.False((await _http.GetFromJsonAsync<ItemDto>($"/items/{item.Id}"))!.Available);

        // a second loan of the same item is rejected
        var second = await _http.PostAsJsonAsync("/loans", new { itemId = item.Id, borrower = "marko" });
        Assert.Equal(HttpStatusCode.Conflict, second.StatusCode);

        // return
        var returned = await _http.PostAsync($"/loans/{loan.Id}/return", content: null);
        Assert.Equal(HttpStatusCode.OK, returned.StatusCode);
        Assert.True((await _http.GetFromJsonAsync<ItemDto>($"/items/{item.Id}"))!.Available);
    }
}
```

Run: `dotnet test tests/LendIt.E2E.Tests`.

**Java: `src/test/java/hr/lendit/e2e/LoanJourneyTest.java`**

```java
package hr.lendit.e2e;

import static org.assertj.core.api.Assertions.assertThat;

import com.jayway.jsonpath.JsonPath;
import hr.lendit.support.TestApi;
import java.util.UUID;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;

@Tag("e2e")
@EnabledIfEnvironmentVariable(named = "BASE_URL", matches = ".+")   // skipped when no application is running
class LoanJourneyTest {

    private final TestApi api = new TestApi(System.getenv("BASE_URL"));

    @Test
    void aBorrowerCanLendAndReturnAnItem() throws Exception {
        long itemId = api.createItem("Projector");
        String borrower = "ana-" + UUID.randomUUID();

        // lend
        var lend = api.post("/loans", TestApi.lendJson(itemId, borrower));
        assertThat(lend.statusCode()).isEqualTo(201);
        int loanId = JsonPath.read(lend.body(), "$.id");
        assertThat(TestApi.isAvailable(api.get("/items/" + itemId).body())).isFalse();

        // a second loan of the same item is rejected
        var second = api.post("/loans", TestApi.lendJson(itemId, "marko"));
        assertThat(second.statusCode()).isEqualTo(409);

        // return
        var returned = api.post("/loans/" + loanId + "/return", "");
        assertThat(returned.statusCode()).isEqualTo(200);
        assertThat(TestApi.isAvailable(api.get("/items/" + itemId).body())).isTrue();
    }
}
```

Run (Linux/macOS): `BASE_URL=http://localhost:8080 ./mvnw -B test -Dgroups=e2e`. On Windows PowerShell: `$env:BASE_URL="http://localhost:8080"; ./mvnw.cmd -B test -Dgroups=e2e`.

> Without `BASE_URL`, a plain `./mvnw test` skips this test class instead of failing. That is what the `@EnabledIfEnvironmentVariable` annotation is for.

### 3.3 Your E2E tests (to write)

1. **The loan limit journey:** one borrower lends 3 items, the 4th is rejected with `409`, after returning one of them the 4th succeeds.
2. **Validation journey:** a blank item name and a blank borrower are rejected, and nothing is created (check `GET /items`).

### 3.4 Break it on purpose

Stop the container, change the port mapping so the app is not reachable (`-p 9090:8080`), and run the E2E tests. What do you see? Then fix the mapping and run them against a *deliberately broken image* (for example, an image whose `ENTRYPOINT` points to a wrong dll/jar name). Which test levels would catch each of these two problems? (This is the reason E2E tests exist.)

Clean up: `docker rm -f lendit`.

### Questions for your report

1. What can an E2E test detect that an integration test of Task 2 cannot?
2. Why do the E2E tests make up unique names (`Guid`, `UUID`)? What would happen if you run the suite twice against the same container?
3. Which parts of an E2E test make it slower and more fragile than an integration test?

### Expected result of Task 3

- A container started from your Docker image, and a passing E2E suite against it (3 journeys: 1 given + 2 yours).
- Answers and the "break it on purpose" findings.

---

## Task 4: Automated UI tests

A UI test drives a **real browser** as a user would: it opens the page, types, clicks and looks at what is displayed. It is the only level that verifies the **front-end code** (`index.html`, JavaScript) together with the back end. It is also the slowest and most fragile, so keep these tests **few** and focused on the most important user flows.

### 4.1 The tool: Playwright

**Playwright** (Microsoft) is a browser automation library for C#, Java, JavaScript/TypeScript and Python. It controls Chromium, Firefox and WebKit. A comparison with other tools:

| | Playwright | Selenium WebDriver | Cypress |
|---|---|---|---|
| Languages | C#, Java, JS/TS, Python | C#, Java, JS, Python, ... | JavaScript/TypeScript |
| Waiting for elements | **automatic** (auto-wait) | manual waits needed | automatic |
| Browsers | Chromium, Firefox, WebKit | all major (through drivers) | Chromium family, Firefox, WebKit (experimental) |
| Maturity | newer, fast-growing | oldest, huge ecosystem | popular in JS projects |

Everything in the lab can also be written with Selenium. The concepts (locators, waiting, page objects) are the same.

### 4.2 Four rules for stable UI tests

1. **Locate elements the way a user would:** by role, label, text, or by a dedicated test id (`data-testid`). Avoid long CSS/XPath paths, which break when the layout changes.

   | Locator | Example | Use when |
   |---------|---------|----------|
   | by role | `GetByRole(AriaRole.Button, new() { Name = "Add item" })` | the element has an accessible role and name (best) |
   | by label | `GetByLabel("Borrower")` | form fields with a label |
   | by text | `GetByText("Lent out")` | visible text |
   | by test id | `GetByTestId("add-item-button")` | no good role/label; stable hook added for tests |
   | CSS / XPath | `Locator("div > ul li:nth-child(3)")` | last resort |

2. **Never use `sleep`.** Playwright waits for elements to be ready, and its *web-first assertions* (`Expect(...)`/`assertThat(...)`) retry until the condition holds or a timeout expires. A fixed `Thread.Sleep(2000)` makes tests slow *and* still flaky.
3. **Prepare test data through the API, not through the UI.** The UI is used only for what is being tested. The setup is much faster and more reliable.
4. **Make every test independent:** unique data (`Guid`), no dependence on test order or on leftovers from other tests.

### 4.3 C#: Playwright with NUnit

The project `tests/LendIt.UI.Tests` was created with `Microsoft.Playwright.NUnit`. The base class `PageTest` gives every test a fresh browser context and a `Page`.

First build the project and install the browser once:

```bash
cd dotnet-app
dotnet build tests/LendIt.UI.Tests
pwsh tests/LendIt.UI.Tests/bin/Debug/net10.0/playwright.ps1 install chromium
```

**`tests/LendIt.UI.Tests/LendingPageTests.cs`** (delete the generated `UnitTest1.cs`/`Tests.cs`)

```csharp
using System.Net.Http.Json;
using System.Text.Json;
using Microsoft.Playwright;
using Microsoft.Playwright.NUnit;

namespace LendIt.UI.Tests;

[TestFixture]
public class LendingPageTests : PageTest
{
    private static readonly string BaseUrl =
        Environment.GetEnvironmentVariable("BASE_URL") ?? "http://localhost:8080";

    // test data is prepared through the API (fast, reliable), not through the UI
    private static async Task<int> CreateItemAsync(string name)
    {
        using var http = new HttpClient { BaseAddress = new Uri(BaseUrl) };
        var response = await http.PostAsJsonAsync("/items", new { name });
        response.EnsureSuccessStatusCode();
        using var json = JsonDocument.Parse(await response.Content.ReadAsStringAsync());
        return json.RootElement.GetProperty("id").GetInt32();
    }

    [Test]
    public async Task Adding_an_item_shows_it_as_available()
    {
        var name = $"Ladder-{Guid.NewGuid():N}";

        await Page.GotoAsync(BaseUrl);
        await Page.GetByTestId("new-item-name").FillAsync(name);
        await Page.GetByTestId("add-item-button").ClickAsync();

        await Expect(Page.GetByTestId("message")).ToHaveTextAsync($"Added {name}");
        await Expect(Page.GetByText(name)).ToBeVisibleAsync();
    }

    [Test]
    public async Task Lending_an_item_marks_it_as_lent_out()
    {
        var itemId = await CreateItemAsync($"Drill-{Guid.NewGuid():N}");
        var borrower = $"ana-{Guid.NewGuid():N}";

        await Page.GotoAsync(BaseUrl);
        await Page.GetByTestId("borrower-input").FillAsync(borrower);
        await Page.GetByTestId($"lend-button-{itemId}").ClickAsync();

        await Expect(Page.GetByTestId("message")).ToHaveTextAsync($"Lent to {borrower}");
        await Expect(Page.GetByTestId($"item-{itemId}-status")).ToHaveTextAsync("Lent out");
        await Expect(Page.GetByTestId($"lend-button-{itemId}")).ToBeDisabledAsync();
    }
}
```

Run (the application must be running, for example in the container from Task 3):

```bash
dotnet test tests/LendIt.UI.Tests
HEADED=1 dotnet test tests/LendIt.UI.Tests       # watch the browser (PowerShell: $env:HEADED=1)
```

### 4.4 Java: Playwright with JUnit 5

**`src/test/java/hr/lendit/ui/LoanUiTest.java`**

```java
package hr.lendit.ui;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

import com.microsoft.playwright.Browser;
import com.microsoft.playwright.BrowserContext;
import com.microsoft.playwright.Page;
import com.microsoft.playwright.Playwright;
import hr.lendit.support.TestApi;
import java.util.UUID;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;

@Tag("ui")
@EnabledIfEnvironmentVariable(named = "BASE_URL", matches = ".+")
class LoanUiTest {

    private static final String BASE_URL = System.getenv("BASE_URL");

    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll
    static void launchBrowser() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch();   // headless by default
    }

    @AfterAll
    static void closeBrowser() {
        browser.close();
        playwright.close();
    }

    @BeforeEach
    void createContext() {
        context = browser.newContext();             // a fresh, isolated "browser profile" for each test
        page = context.newPage();
    }

    @AfterEach
    void closeContext() {
        context.close();
    }

    @Test
    void addingAnItemShowsItAsAvailable() {
        String name = "Ladder-" + UUID.randomUUID();

        page.navigate(BASE_URL);
        page.getByTestId("new-item-name").fill(name);
        page.getByTestId("add-item-button").click();

        assertThat(page.getByTestId("message")).hasText("Added " + name);
        assertThat(page.getByText(name)).isVisible();
    }

    @Test
    void lendingAnItemMarksItAsLentOut() throws Exception {
        long itemId = new TestApi(BASE_URL).createItem("Drill-" + UUID.randomUUID());
        String borrower = "ana-" + UUID.randomUUID();

        page.navigate(BASE_URL);
        page.getByTestId("borrower-input").fill(borrower);
        page.getByTestId("lend-button-" + itemId).click();

        assertThat(page.getByTestId("message")).hasText("Lent to " + borrower);
        assertThat(page.getByTestId("item-" + itemId + "-status")).hasText("Lent out");
        assertThat(page.getByTestId("lend-button-" + itemId)).isDisabled();
    }
}
```

(In this file only the Playwright `assertThat` is imported. Do not also import AssertJ's `assertThat`, because both have the same name.)

Run (the application must be running):

```bash
BASE_URL=http://localhost:8080 ./mvnw -B test -Dgroups=ui
```

The first run downloads the browser (this can take a minute). To see the browser, launch it with `new BrowserType.LaunchOptions().setHeadless(false).setSlowMo(300)`. Use this **only while debugging**.

### 4.5 Your UI tests and the page object pattern

Add at least two more UI tests:

1. A blank borrower is rejected: the message starts with `Error:` and the item remains **Available**.
2. A lent item's **Lend** button is disabled, and another browser context cannot lend it (open a second `context`/page, load the page and check the button is disabled there too).

When several tests perform the same sequences of actions, put the page's structure into a **page object**. The tests then read as intentions, and if a selector changes you fix **one** place.

```csharp
// C#
public class LendingPage(IPage page, string baseUrl)
{
    public Task OpenAsync() => page.GotoAsync(baseUrl);
    public Task EnterBorrowerAsync(string name) => page.GetByTestId("borrower-input").FillAsync(name);
    public Task LendAsync(int itemId) => page.GetByTestId($"lend-button-{itemId}").ClickAsync();
    public ILocator Message => page.GetByTestId("message");
    public ILocator StatusOf(int itemId) => page.GetByTestId($"item-{itemId}-status");
}
```

```java
// Java
class LendingPage {
    private final Page page;
    private final String baseUrl;

    LendingPage(Page page, String baseUrl) { this.page = page; this.baseUrl = baseUrl; }

    void open() { page.navigate(baseUrl); }
    void enterBorrower(String name) { page.getByTestId("borrower-input").fill(name); }
    void lend(long itemId) { page.getByTestId("lend-button-" + itemId).click(); }
    Locator message() { return page.getByTestId("message"); }
    Locator statusOf(long itemId) { return page.getByTestId("item-" + itemId + "-status"); }
}
```

Refactor your tests to use it.

### 4.6 Debugging tools

- **Playwright codegen** records your clicks and generates test code and selectors: `pwsh bin/Debug/net10.0/playwright.ps1 codegen http://localhost:8080` (C#), or `mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="codegen http://localhost:8080" -Dexec.classpathScope=test` (Java).
- **Screenshots:** `await Page.ScreenshotAsync(new() { Path = "fail.png" })` / `page.screenshot(new Page.ScreenshotOptions().setPath(Paths.get("fail.png")))`. In CI, save them as artifacts when a test fails.
- **Traces:** Playwright can record a *trace* (a time-travel recording of a test) that can be opened with the Trace Viewer. See the Playwright documentation.

### Questions for your report

1. Why is the item created through the API and not by clicking in the UI in the test `Lending_an_item_marks_it_as_lent_out`?
2. What is wrong with `Thread.Sleep(3000)` before an assertion? What does Playwright do instead?
3. Which of your UI tests could be replaced by a lower-level test without losing anything? Which ones could not, and why?

### Expected result of Task 4

- At least 4 UI tests in your track, passing against the running container.
- The page object refactoring.
- Answers to the questions, and one screenshot taken by a test.

---

## Task 5: Test automation in the CI/CD pipeline

You now have four kinds of tests. Running them by hand does not scale, so this task connects them to a **pipeline** that runs on every push and pull request and **blocks a bad change from being merged or published**. It builds on the Docker build from Lab 6: the same Dockerfile now produces the image that the E2E and UI tests run against, and the image is published only after all tests pass.

### 5.1 Design the pipeline

```text
 push / pull request
        │
        ▼
 ┌───────────────┐   ┌──────────────────┐   ┌──────────────────────────────────────┐
 │ unit          │──►│ integration      │──►│ image + E2E + UI                     │
 │ build + unit  │   │ in-process tests │   │ build image (Lab 6) → run container  │
 │ + coverage    │   │                  │   │ → E2E tests → UI tests → push image  │
 │ gate          │   │                  │   │   (push only on main and v* tags)    │
 └───────────────┘   └──────────────────┘   └──────────────────────────────────────┘
   seconds              seconds–1 min          the slowest, so it runs last
```

Principles behind it:

- **Fail fast:** the cheapest tests run first. If a unit test fails, the expensive jobs are never started (`needs:`).
- **Build once, test the artifact you ship:** the image that the E2E tests run against is **the same image** that is pushed. It is not rebuilt for publishing.
- **Gate publishing on tests:** the image is pushed only when everything is green, and never from a pull request.
- **Keep the evidence:** test results, coverage and logs are uploaded as artifacts, even (and especially) when the run fails.

Mapping to the tags and filters you already created:

| Job | C# command | Java command |
|-----|------------|--------------|
| unit | `dotnet test tests/LendIt.Tests --filter "Category=Unit"` (+ coverage gate) | `./mvnw verify -Dgroups=unit` (JaCoCo gate in `verify`) |
| integration | `dotnet test tests/LendIt.Tests --filter "Category=Integration"` | `./mvnw test -Dgroups=integration` |
| E2E | `dotnet test tests/LendIt.E2E.Tests` | `./mvnw test -Dgroups=e2e` |
| UI | `dotnet test tests/LendIt.UI.Tests` | `./mvnw test -Dgroups=ui` |

> **Note.** In Lab 6 you made a `test` stage inside the Dockerfile. That is another valid place to run *unit* tests (and it makes `docker build` fail when they fail). In this lab, tests run as separate pipeline steps, because the pipeline needs test **reports**, coverage and separate stages. Decide for your project which one you want (or both), and justify it.

### 5.2 The C# workflow

Create `.github/workflows/dotnet-ci.yml`:

```yaml
name: dotnet-ci

on:
  push:
    branches: [main]
    tags: ["v*"]
    paths:
      - "dotnet-app/**"
      - ".github/workflows/dotnet-ci.yml"
  pull_request:
    paths:
      - "dotnet-app/**"
      - ".github/workflows/dotnet-ci.yml"
  workflow_dispatch:

concurrency:                         # a new push to the same PR cancels the previous, now useless, run
  group: dotnet-ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

env:
  DOTNET_NOLOGO: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

defaults:
  run:
    working-directory: dotnet-app

jobs:
  # ---------------------------------------------------------------- 1) build + unit tests
  unit:
    name: build + unit tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: 10.0.x

      - uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('dotnet-app/**/*.csproj') }}
          restore-keys: nuget-${{ runner.os }}-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      # coverlet instruments the assembly while building, so this step must not use --no-build
      - name: Unit tests with a coverage gate (domain >= 80 %)
        run: >
          dotnet test tests/LendIt.Tests --configuration Release --no-restore
          --filter "Category=Unit"
          --logger "trx;LogFileName=unit.trx" --results-directory TestResults
          -p:CollectCoverage=true -p:CoverletOutputFormat=cobertura
          -p:CoverletOutput=../../TestResults/coverage/
          -p:Include="[LendIt.Api]LendIt.Api.Domain.*"
          -p:Threshold=80 -p:ThresholdType=line -p:ThresholdStat=total

      - name: Upload test results and coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dotnet-unit-results
          path: dotnet-app/TestResults

  # ---------------------------------------------------------------- 2) integration tests
  integration:
    name: integration tests
    needs: unit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: 10.0.x

      - uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('dotnet-app/**/*.csproj') }}
          restore-keys: nuget-${{ runner.os }}-

      - name: Integration tests
        run: >
          dotnet test tests/LendIt.Tests --configuration Release
          --filter "Category=Integration"
          --logger "trx;LogFileName=integration.trx" --results-directory TestResults

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dotnet-integration-results
          path: dotnet-app/TestResults

  # ---------------------------------------------------------------- 3) image + E2E + UI (+ publish)
  e2e:
    name: image + E2E + UI tests
    needs: integration
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                 # needed only to push the image to ghcr.io
    env:
      BASE_URL: http://localhost:8080
    steps:
      - uses: actions/checkout@v6

      - name: Compute the (lowercase) image name
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/lendit-dotnet" >> "$GITHUB_ENV"

      - uses: docker/setup-buildx-action@v3

      - name: Build the image (the one that is tested, and later published)
        uses: docker/build-push-action@v6
        with:
          context: ./dotnet-app
          file: ./dotnet-app/Dockerfile
          load: true                  # load the image into the runner's Docker, do not push yet
          tags: ${{ env.IMAGE }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Start the container and wait until it answers
        run: |
          docker run -d --name lendit -p 8080:8080 "$IMAGE:$GITHUB_SHA"
          curl --fail --silent --retry 30 --retry-delay 1 --retry-connrefused "$BASE_URL/items" > /dev/null

      - uses: actions/setup-dotnet@v5
        with:
          dotnet-version: 10.0.x

      - name: E2E tests
        run: >
          dotnet test tests/LendIt.E2E.Tests --configuration Release
          --logger "trx;LogFileName=e2e.trx" --results-directory TestResults

      - name: Install the Playwright browser
        run: |
          dotnet build tests/LendIt.UI.Tests --configuration Release
          pwsh tests/LendIt.UI.Tests/bin/Release/net10.0/playwright.ps1 install --with-deps chromium

      - name: UI tests
        run: >
          dotnet test tests/LendIt.UI.Tests --configuration Release --no-build
          --logger "trx;LogFileName=ui.trx" --results-directory TestResults

      - name: Show the container log on failure
        if: failure()
        run: docker logs lendit

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dotnet-e2e-ui-results
          path: dotnet-app/TestResults

      # ---- only reached when every test above passed; never on pull requests
      - name: Log in to ghcr.io
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push the tested image
        if: github.event_name != 'pull_request'
        run: |
          docker push "$IMAGE:$GITHUB_SHA"
          if [ "$GITHUB_REF_TYPE" = "tag" ]; then
            VERSION="${GITHUB_REF_NAME#v}"            # git tag v0.2.0 -> image tag 0.2.0
            docker tag "$IMAGE:$GITHUB_SHA" "$IMAGE:$VERSION"
            docker push "$IMAGE:$VERSION"
          elif [ "$GITHUB_REF_NAME" = "main" ]; then
            docker tag "$IMAGE:$GITHUB_SHA" "$IMAGE:main"
            docker push "$IMAGE:main"
          fi
```

### 5.3 The Java workflow

Create `.github/workflows/java-ci.yml`. The structure is the same: `on`, `concurrency`, `permissions`, `defaults.run.working-directory: java-app`. The jobs `unit` and `integration` look like this:

```yaml
jobs:
  unit:
    name: build + unit tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: 21
          cache: maven
          cache-dependency-path: java-app/pom.xml

      - name: Unit tests with a coverage gate (domain >= 80 %)
        run: ./mvnw -B -ntp verify -Dgroups=unit

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: java-unit-results
          path: |
            java-app/target/surefire-reports
            java-app/target/site/jacoco

  integration:
    name: integration tests
    needs: unit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: 21
          cache: maven
          cache-dependency-path: java-app/pom.xml

      - name: Integration tests
        run: ./mvnw -B -ntp test -Dgroups=integration

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: java-integration-results
          path: java-app/target/surefire-reports
```

The third job, `e2e` (image + E2E + UI), is the same as the C# one, with these changes:

| Part | C# workflow | Java workflow |
|------|-------------|---------------|
| image name | `.../lendit-dotnet` | `.../lendit-java` |
| `context` / `file` | `./dotnet-app` | `./java-app` |
| toolchain step | `actions/setup-dotnet@v5` | `actions/setup-java@v5` (Temurin 21, `cache: maven`) |
| E2E | `dotnet test tests/LendIt.E2E.Tests` | `./mvnw -B -ntp test -Dgroups=e2e` |
| browser install | `pwsh .../playwright.ps1 install --with-deps chromium` | `./mvnw -B -ntp exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="install --with-deps chromium" -Dexec.classpathScope=test` |
| UI | `dotnet test tests/LendIt.UI.Tests` | `./mvnw -B -ntp test -Dgroups=ui` |
| results to upload | `dotnet-app/TestResults` | `java-app/target/surefire-reports` |

Both workflows need `BASE_URL: http://localhost:8080` in the job's `env`. Write the complete Java workflow yourself, and push the **Java** and **C#** workflow in the same repository (each triggers only for its own folder, thanks to `paths`).

> If a step fails with `./mvnw: Permission denied`, run `git update-index --chmod=+x java-app/mvnw` and commit.

### 5.4 Understand every part

Answer in the report:

1. Why do jobs `integration` and `e2e` declare `needs:`? What is the effect on the total time when the unit tests fail?
2. Why is the image built with `load: true` and pushed in a *later* step, instead of using `push: true` in the build step?
3. Why is the push step guarded by `github.event_name != 'pull_request'`, and why does the `e2e` job still run the tests for pull requests?
4. What are the `concurrency` settings good for?
5. How do the git tag `v0.2.0` (Lab 4) and the image tag `0.2.0` relate to each other in this workflow?
6. Why are the test results uploaded with `if: always()`?
7. How does the `curl --retry` step prevent a flaky pipeline? What would happen without it?

### 5.5 Make the pipeline matter

A pipeline that does not block anything is only advice.

1. In the repository ruleset for `main` (**Settings → Rules**, as in Lab 6), add the jobs `build + unit tests`, `integration tests` and `image + E2E + UI tests` as **required status checks**. (They appear in the list after they have run at least once.)
2. Open a pull request that **breaks a unit test** (for example the bug from Task 1.5). Verify that the later jobs do not run, and that the **Merge** button is blocked. Screenshot it.
3. Open a pull request that breaks only an **E2E or UI test** (for example change the text `Lent out` in `index.html`). Which job turns red? Which jobs were green? Why did the earlier levels not detect this?
4. Fix both, merge, and verify that the push to `main` publishes `ghcr.io/<owner>/<repo>/lendit-<track>:main`. Then create the tag `v0.2.0` and verify that the image gets the tag `0.2.0`.
5. Download the artifacts of one run, and open the coverage report or a `.trx`/surefire report.
6. Add status badges for the two workflows to `README.md`:

```markdown
![dotnet-ci](https://github.com/<owner>/testing-demo/actions/workflows/dotnet-ci.yml/badge.svg)
![java-ci](https://github.com/<owner>/testing-demo/actions/workflows/java-ci.yml/badge.svg)
```

### 5.6 Optional extensions

- **Scheduled run:** add `schedule: - cron: "0 4 * * *"` to the triggers so that the whole suite also runs every night, which finds problems that are not caused by a code change (expired dependencies, changed base images).
- **Test report in the run summary:** publish the results to the job summary (`$GITHUB_STEP_SUMMARY`), or use a community action that renders TRX/JUnit XML.
- **Flaky tests:** run the UI suite 10 times in a loop (a matrix with `run: [1,2,3,...]`). If any run fails, find out why. A test that fails sometimes is worse than no test: people stop trusting the pipeline.
- **Browser matrix:** run the UI tests in Chromium, Firefox and WebKit.
- **Security scan of the image** (for example Trivy) as one more job before the push.

### Expected result of Task 5

- Two working workflows (C# and Java) with three jobs each.
- A blocked pull request (unit test failure) and a pull request with an E2E/UI failure, both with screenshots.
- A published image from `main` and a versioned image from a `v*` tag.
- Badges in `README.md`, and the answers to the questions.

---

## Task 6: Tests and the pipeline in your own project

Apply what you learned to the **project repository** from Lab 1.

1. **Test plan.** Create `docs/testing.md` with a table that maps each **user story** (from Lab 2 and your GitHub Project) to its tests:

   | User story | Unit | Integration | E2E | UI | Manual |
   |------------|------|-------------|-----|----|--------|
   | US-1.1 register-user | `PasswordHasherTests`, ... | `POST /register` returns 201 | register → login journey | registration form | usability check |

   For each test type that you *do not* use for a story, write down why (for example, "no UI yet").
2. **Write the tests.** At least: 10 unit tests (with test doubles where needed), 4 integration tests (use Testcontainers if you have a database), 1 E2E journey. If your project has a web UI, add 2 UI tests. If it is not a web application, replace the UI tests with tests of whatever is its outermost interface (a CLI, a mobile UI, a message consumer) and explain.
3. **Quality gate.** Choose a coverage threshold and add it to the pipeline. Justify the number.
4. **Pipeline.** Extend the build workflow from Lab 6 with the test jobs, following the pattern of Task 5: fast tests first, the image tested before it is published, results saved as artifacts.
5. **Rules.** Make the test jobs required status checks for `main`.
6. **Process.** Work as in Labs 3 to 5: issues in the GitHub Project for each test task (this is exactly the "testing task" under each user story from Lab 2), a branch per issue, pull requests with `Closes #<n>`, and a failed-then-fixed pipeline run that you describe in `docs/testing.md`.

### Expected result of Task 6

- `docs/testing.md` with the test plan, the pipeline description and the chosen thresholds.
- Tests and a green pipeline on `main` in your project repository, with the test jobs as required checks.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| C#: plain `dotnet test` fails in the E2E/UI projects | They need a running application. Run a specific project (`dotnet test tests/LendIt.Tests`) |
| C#: `WebApplicationFactory<Program>`: "`Program` is inaccessible" | Add `public partial class Program { }` at the end of `Program.cs` |
| C#: coverage threshold is ignored | With coverlet.msbuild, do not use `--no-build`; make sure the `-p:Threshold...` properties are on the `dotnet test` command |
| C#: `playwright.ps1` not found | Build the UI test project first. The script is in `bin/<Configuration>/net10.0/` |
| C#: browser not installed | `pwsh .../playwright.ps1 install chromium` (in CI with `--with-deps`) |
| C#: xUnit `IAsyncLifetime` does not compile | You have xUnit v3 (`ValueTask`); the lab uses v2 (`Task`) |
| Java: tests of other levels run, too | Pass `-Dgroups=unit` (or `integration`, `e2e`, `ui`). Without `-Dgroups` Maven runs every tag |
| Java: E2E/UI tests silently skipped | `BASE_URL` is not set (the `@EnabledIfEnvironmentVariable` check) |
| Java: `Unnecessary stubbings detected` | Mockito strict stubs: a `when(...)` was never used by the code. Remove the stub or fix the test |
| Java: `${local.server.port}` cannot be resolved | The test is not a `@SpringBootTest` with `RANDOM_PORT` |
| Java: JaCoCo `check` fails with *Rule violated* | The domain coverage is below 80%. Open `target/site/jacoco/index.html` and add tests |
| UI test fails with *strict mode violation* | The locator matches several elements. Use unique test data or a more specific locator |
| UI test passes locally, fails in CI | Different speed or data. Remove `sleep`s, use unique data, upload the container log and a screenshot |
| E2E fails in CI with *connection refused* | The container was not ready. Check the `curl --retry` step and `docker logs lendit` |
| `docker push` denied in CI | Missing `permissions: packages: write`, no login step, or the image name is not lowercase |
| Required check is "Expected: waiting for status" forever | The job name in the ruleset does not match the workflow job's `name`, or `paths:` filtered the workflow out for that PR |

---

## Deliverables

- Repository `testing-demo` (public) with the application and tests of your track(s), both workflows, and badges in `README.md`.
- A report `docs/testing-lab.md` in your **project** repository with: answers to all questions of Tasks 1 to 5, the coverage screenshot, the "bug on purpose" findings, screenshots of the blocked pull requests, and links to successful and failed pipeline runs and to the published images.
- `docs/testing.md`, the tests, the pipeline and the required checks in your project repository (Task 6).

Final checklist:

- [ ] Sample application runs; page and API verified by hand
- [ ] 15+ unit tests, domain coverage ≥ 80% with a gate, boundary bug found
- [ ] 8+ integration tests (in-process), time controlled in one of them
- [ ] E2E journeys against a container started from the Docker image
- [ ] 4+ automated UI tests with stable locators, API-prepared data and a page object
- [ ] Pipeline: unit → integration → image + E2E + UI → push, with `needs:` and artifacts
- [ ] Image published only from `main` or a `v*` tag, and only after all tests pass
- [ ] Test jobs are required checks; a failing PR is demonstrably blocked
- [ ] Your own project: test plan, tests, pipeline and quality gate

---

## Command cheat sheet

| Goal | C# | Java |
|------|----|------|
| Unit tests | `dotnet test tests/LendIt.Tests --filter "Category=Unit"` | `./mvnw test -Dgroups=unit` |
| Integration tests | `dotnet test tests/LendIt.Tests --filter "Category=Integration"` | `./mvnw test -Dgroups=integration` |
| E2E tests | `BASE_URL=... dotnet test tests/LendIt.E2E.Tests` | `BASE_URL=... ./mvnw test -Dgroups=e2e` |
| UI tests | `BASE_URL=... dotnet test tests/LendIt.UI.Tests` | `BASE_URL=... ./mvnw test -Dgroups=ui` |
| One test only | `dotnet test --filter "FullyQualifiedName~Lend_throws"` | `./mvnw test -Dtest=LoanServiceTest#lend_throwsWhenTheItemDoesNotExist` |
| Coverage | `-p:CollectCoverage=true -p:Threshold=80` | `./mvnw verify` (JaCoCo: `target/site/jacoco`) |
| Install Playwright browser | `pwsh .../playwright.ps1 install --with-deps chromium` | `./mvnw exec:java -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="install --with-deps chromium" -Dexec.classpathScope=test` |
| Start the app as a container | `docker run -d --name lendit -p 8080:8080 lendit:local` | same |
| Wait for the container | `curl --fail --retry 30 --retry-delay 1 --retry-connrefused http://localhost:8080/items` | same |
| Container log / clean up | `docker logs lendit` / `docker rm -f lendit` | same |

---

## Useful resources

- [Microsoft Learn – Unit testing in .NET](https://learn.microsoft.com/en-us/dotnet/core/testing/)
- [Microsoft Learn – Integration tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [xUnit.net documentation](https://xunit.net/)
- [NSubstitute](https://nsubstitute.github.io/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [AssertJ documentation](https://assertj.github.io/doc/)
- [Spring Boot – Testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Playwright for .NET](https://playwright.dev/dotnet/) and [Playwright for Java](https://playwright.dev/java/)
- [Testcontainers](https://testcontainers.com/)
- [Coverlet](https://github.com/coverlet-coverage/coverlet) and [JaCoCo](https://www.jacoco.org/jacoco/)
- [Martin Fowler – The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [GitHub Docs – Building and testing your code](https://docs.github.com/en/actions/use-cases-and-examples/building-and-testing)
