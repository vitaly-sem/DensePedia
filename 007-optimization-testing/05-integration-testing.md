# Интеграционное тестирование

---

## WebApplicationFactory — интеграционные тесты ASP.NET Core

**Факты:**
- `WebApplicationFactory<T>` — запускает приложение в памяти (TestServer)
- Реальный HTTP pipeline (middleware, auth, routing)
- Можно подменить зависимости через `ConfigureTestServices`

```csharp
public class ApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public ApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                // Подмена реального DbContext на in-memory
                services.RemoveAll<AppDbContext>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
                
                // Подмена внешних сервисов
                services.RemoveAll<IPaymentGateway>();
                services.AddScoped<IPaymentGateway, MockPaymentGateway>();
            });
        });
    }
    
    [Fact]
    public async Task PostOrder_WithValidData_Returns201()
    {
        var client = _factory.CreateClient();
        
        var response = await client.PostAsJsonAsync("/api/orders", new
        {
            UserId = Guid.NewGuid(),
            Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 1 } }
        });
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var location = response.Headers.Location;
        Assert.NotNull(location);
        
        // Проверка, что созданный ресурс доступен
        var getResponse = await client.GetAsync(location);
        Assert.Equal(HttpStatusCode.OK, getResponse.StatusCode);
    }
}
```

### Custom WebApplicationFactory

```csharp
public class CustomApiFactory : WebApplicationFactory<Program>
{
    private readonly string _dbName = $"TestDb_{Guid.NewGuid()}";
    
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        
        builder.ConfigureServices(services =>
        {
            // Remove real DB
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);
            
            // Add InMemory DB
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase(_dbName));
        });
    }
    
    // Helper methods
    public async Task<HttpClient> CreateAuthenticatedClientAsync()
    {
        var client = CreateClient();
        var token = await GetTestTokenAsync();
        client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        return client;
    }
}
```

---

## TestContainers — реальные БД в тестах

**Факты:**
- Запускает Docker-контейнеры для тестов
- Поддерживает: SQL Server, PostgreSQL, Redis, Kafka, RabbitMQ
- Контейнер живёт, пока жив тест (автоудаление)

```csharp
public class SqlServerContainerFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .WithPassword("Test123!Pass")
        .Build();
    
    public string ConnectionString => _container.GetConnectionString();
    
    public async Task InitializeAsync() => await _container.StartAsync();
    
    public async Task DisposeAsync() => await _container.DisposeAsync();
}

public class OrderRepositoryTests : IClassFixture<SqlServerContainerFixture>
{
    private readonly SqlServerContainerFixture _fixture;
    private readonly AppDbContext _db;
    
    public OrderRepositoryTests(SqlServerContainerFixture fixture)
    {
        _fixture = fixture;
        
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(fixture.ConnectionString)
            .Options;
        
        _db = new AppDbContext(options);
        _db.Database.EnsureCreated();
    }
    
    [Fact]
    public async Task GetByIdAsync_WithExistingOrder_ReturnsOrder()
    {
        // Arrange — используем реальную БД
        var order = new Order(UserId);
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        
        var repo = new OrderRepository(_db);
        
        // Act
        var result = await repo.GetByIdAsync(order.Id);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(order.Id, result.Id);
    }
}
```

**Факт:** TestContainers в .NET — NuGet `Testcontainers` (ранее `DotNet.Testcontainers`).

**Факт:** Для CI/CD нужно настроить Docker-in-Docker или использовать service containers (GitHub Actions).

---

## EF Core — тестирование репозиториев

```csharp
public class EfRepositoryTests
{
    [Fact]
    public async Task AddAsync_SavesToDatabase()
    {
        // Arrange
        await using var context = CreateContext();
        var repo = new OrderRepository(context);
        
        // Act
        await repo.AddAsync(new Order(UserId));
        await context.SaveChangesAsync();
        
        // Assert
        var saved = await context.Orders.FirstOrDefaultAsync();
        Assert.NotNull(saved);
    }
    
    private AppDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase($"TestDb_{Guid.NewGuid()}")  // Уникальное имя!
            .Options;
        return new AppDbContext(options);
    }
}
```

**Факт:** `UseInMemoryDatabase` — **не реляционная** БД. Не тестирует SQL, constraints, транзакции. Для этого — TestContainers.

---

## Health Checks — мониторинг

```csharp
// Тестирование health checks
[Fact]
public async Task HealthCheck_ReturnsHealthy()
{
    var client = _factory.CreateClient();
    
    var response = await client.GetAsync("/health");
    
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    var body = await response.Content.ReadFromJsonAsync<HealthReport>();
    Assert.Equal(HealthStatus.Healthy, body.Status);
}
```

---

## Чек-лист

- [ ] WebApplicationFactory: TestServer, ConfigureTestServices
- [ ] Custom factory: смена окружения, подмена DB
- [ ] TestContainers: SQL Server, Redis, Kafka
- [ ] EF Core: InMemory для быстрых тестов, TestContainers для реальных
- [ ] Health checks: /health endpoint
- [ ] CI/CD: Docker-in-Docker для TestContainers
