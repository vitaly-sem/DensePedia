# Модульное тестирование

---

## xUnit — факты

**Факты:**
- xUnit.net — современный фреймворк (создатели NUnit)
- `[Fact]` — тест без параметров
- `[Theory]` — параметризованный тест
- `IAsyncLifetime` — async setup/cleanup

```csharp
public class OrderServiceTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    
    public OrderServiceTests(DatabaseFixture fixture) => _fixture = fixture;
    
    [Fact]
    public async Task PlaceOrder_WithValidItems_CreatesOrder()
    {
        // Arrange
        var service = CreateService();
        var request = new CreateOrderRequest(UserId, Items);
        
        // Act
        var orderId = await service.PlaceOrderAsync(request);
        
        // Assert
        orderId.Should().NotBeEmpty();
    }
    
    [Theory]
    [InlineData(0, "Invalid quantity")]
    [InlineData(-1, "Negative quantity")]
    [InlineData(1001, "Exceeds max quantity")]
    public async Task PlaceOrder_WithInvalidQuantity_Throws(int quantity, string expectedMessage)
    {
        var service = CreateService();
        var request = new CreateOrderRequest(UserId, [new OrderItem(ProductId, quantity)]);
        
        var ex = await Assert.ThrowsAsync<DomainException>(() => service.PlaceOrderAsync(request));
        Assert.Contains(expectedMessage, ex.Message);
    }
}
```

### Shared Context — Fixtures

| Fixture | Lifetime | Когда |
|---|---|---|
| `IClassFixture<T>` | Один instance на класс тестов | Тяжёлые ресурсы (БД, HTTP) |
| `ICollectionFixture<T>` | Один instance на коллекцию классов | Ресурсы между классами |
| Constructor | Новый instance на каждый тест | Лёгкие setup |

---

## NSubstitute — mocking

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrder_SendsEvent()
    {
        var repo = Substitute.For<IOrderRepository>();
        var bus = Substitute.For<IEventBus>();
        
        var service = new OrderService(repo, bus);
        await service.PlaceOrderAsync(request);
        
        // Проверка вызова
        await bus.Received(1).PublishAsync(Arg.Any<OrderPlacedEvent>());
        
        // Проверка с конкретными параметрами
        await repo.Received(1).AddAsync(Arg.Is<Order>(o => o.UserId == expectedUserId));
        
        // Проверка возврата
        repo.GetByIdAsync(testId).Returns(Task.FromResult(expectedOrder));
        
        // Dynamic return
        repo.GetByIdAsync(Arg.Any<Guid>())
            .Returns(callInfo => CreateOrder(callInfo.Arg<Guid>()));
    }
}
```

---

## TDD (Test-Driven Development)

**Факт:** Red → Green → Refactor.

```
1. Red: Написать тест, который падает (нет реализации)
2. Green: Минимальная реализация для прохождения теста
3. Refactor: Улучшить код, тесты проходят
```

```csharp
// 1. RED — тест падает (класс FizzBuzz не существует)
[Fact]
public void FizzBuzz_WhenMultipleOf3_ReturnsFizz()
{
    var result = FizzBuzz.Get(3);
    Assert.Equal("Fizz", result);
}

// 2. GREEN — минимальная реализация
public static class FizzBuzz
{
    public static string Get(int number)
        => number % 3 == 0 ? "Fizz" : number.ToString();
}

// 3. REFACTOR — после всех тестов
public static string Get(int number)
    => number switch
    {
        _ when number % 15 == 0 => "FizzBuzz",
        _ when number % 3 == 0 => "Fizz",
        _ when number % 5 == 0 => "Buzz",
        _ => number.ToString()
    };
```

---

## Test Patterns

### AAA (Arrange-Act-Assert)

```csharp
[Fact]
public void CalculateTotal_WithDiscount_AppliesDiscount()
{
    // Arrange
    var calculator = new PriceCalculator();
    var items = new[] { new Item(100), new Item(200) };
    
    // Act
    var total = calculator.Calculate(items, discountCode: "SAVE10");
    
    // Assert
    Assert.Equal(270, total);  // (100 + 200) * 0.9
}
```

### Parameterized Tests

```csharp
[Theory]
[ClassData(typeof(PriceTestData))]
public void Calculate_WithVariousPrices_ReturnsCorrectTotal(decimal[] prices, decimal expected)
{
    var calculator = new PriceCalculator();
    var items = prices.Select(p => new Item(p)).ToArray();
    
    var total = calculator.Calculate(items);
    
    Assert.Equal(expected, total);
}

public class PriceTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return [new[] { 10m, 20m }, 30m];
        yield return [new[] { 100m }, 100m];
        yield return [new decimal[] { }, 0m];
    }
}
```

### Mock Behavior Verification

```csharp
[Fact]
public void Save_WhenValidationFails_DoesNotCallRepository()
{
    var repo = Substitute.For<IOrderRepository>();
    var validator = Substitute.For<IValidator<Order>>();
    validator.Validate(Arg.Any<Order>())
             .Returns(new ValidationResult(new[] { new ValidationFailure("Name", "Required") }));
    
    var service = new OrderService(repo, validator);
    
    Assert.Throws<ValidationException>(() => service.Save(new Order()));
    
    repo.DidNotReceive().Add(Arg.Any<Order>());  // Repository не вызывался
}
```

---

## Code Coverage

**Факты:**
- **Line coverage** — сколько строк кода выполнено тестами
- **Branch coverage** — сколько веток (if/switch) пройдено (важнее!)
- **80% line, 70% branch** — хорошая цель для бизнес-логики

```xml
<!-- .runsettings — настройка coverage -->
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <ExcludeByFile>**/Migrations/**</ExcludeByFile>
          <ExcludeByAttribute>Obsolete,GeneratedCode</ExcludeByAttribute>
          <ModulePaths>
            <Exclude>.*.Tests.dll</Exclude>
          </ModulePaths>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

```bash
# Запуск с coverage
dotnet test --collect:"XPlat Code Coverage"
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"CoverageReport" -reporttypes:Html
```

**Факт:** Не гонитесь за 100% coverage — это не гарантирует качества. Tests должны проверять **behavior**, не implementation.

---

## Чек-лист

- [ ] xUnit: Fact, Theory, IClassFixture, IAsyncLifetime
- [ ] NSubstitute: Received, Returns, Arg.Any/Is
- [ ] TDD: Red → Green → Refactor
- [ ] AAA: Arrange → Act → Assert
- [ ] Parameterized tests: InlineData, ClassData, MemberData
- [ ] Code coverage: line + branch, не гнаться за 100%
