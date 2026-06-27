# Качество кода и code review

---

## Roslyn Analyzers

**Факты:**
- Анализ кода на этапе компиляции
- Встроенные: CA-правила (Code Analysis), IDE-правила
- NuGet: `Microsoft.CodeAnalysis.NetAnalyzers`, `SonarAnalyzer.CSharp`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>  <!-- Ошибки при нарушении -->
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="SonarAnalyzer.CSharp" Version="9.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

**Факт:** `TreatWarningsAsErrors` — заставляет фиксить все предупреждения перед сборкой.

**Факт:** Правила можно отключить для конкретных мест (`pragma warning disable CA1234`), но не глобально.

---

## .editorconfig — единый стиль

```ini
# .editorconfig — в корне решения

root = true

# Default
[*]
indent_style = space
indent_size = 4
charset = utf-8
end_of_line = lf
trim_trailing_whitespace = true
insert_final_newline = true

# C# specific
[*.cs]
csharp_indent_case_contents = true
csharp_new_line_before_open_brace = all
csharp_preferred_modifier_order = public, private, protected, internal, static, readonly, async
csharp_style_var_for_built_in_types = true:warning
csharp_style_var_when_type_is_apparent = true:suggestion

# Naming rules
csharp_naming_rules:
  - identifier = interface -> prefix: I
  - identifier = private_field -> prefix: _, underscore_required: true

# Tests
[*Test*.cs]
indent_size = 2
```

---

## SonarQube / SonarCloud

**Факты:**
- Статический анализ: bugs, vulnerabilities, code smells, duplications
- **Quality Gate** — порог прохождения (не более 3% duplications, 0 bugs, 0 vulnerabilities)
- Интеграция с CI/CD (GitHub Actions, Azure DevOps)

```yaml
# GitHub Actions — SonarCloud
- name: Analyze with SonarCloud
  uses: SonarSource/sonarcloud-github-action@v2
  with:
    args: >
      -Dsonar.organization=myorg
      -Dsonar.projectKey=myproject
      -Dsonar.cs.opencover.reportsPaths=**/coverage.opencover.xml
```

**Факт:** SonarQube rules часто пересекаются с Roslyn analyzers. Лучше отключить дублирующиеся в одном из инструментов.

---

## Code Review Checklist

### Архитектура
- [ ] Принципы SOLID соблюдены?
- [ ] Нет циклических зависимостей между слоями
- [ ] DI используется, Service Locator отсутствует
- [ ] Абстракции адекватны (не over-engineering)

### Безопасность
- [ ] Input validation на сервере (FluentValidation, Data Annotations)
- [ ] Нет SQL Injection (параметризованные запросы)
- [ ] Нет XSS (HTML-encoding, CSP)
- [ ] Authentication/Authorization проверена
- [ ] Secrets не в коде

### Производительность
- [ ] Нет множественных итераций IEnumerable
- [ ] Async/await не блокируется (.Result, .Wait())
- [ ] ConfigureAwait(false) в библиотеках
- [ ] Нет аллокаций в hot path
- [ ] LINQ не материализуется без необходимости

### Тесты
- [ ] Unit-тесты покрывают бизнес-логику
- [ ] Интеграционные тесты для critical paths
- [ ] Нет mock'а всего подряд (только внешние зависимости)
- [ ] Тесты детерминированы (не flaky)

### Код
- [ ] Магические числа вынесены в константы
- [ ] Имена методов/классов отражают назначение
- [ ] Методы не > 30 строк (SRP)
- [ ] Исключения не используются для flow control
- [ ] Null-проверки и Guard clauses

---

## Guard Clauses

```csharp
// Валидация входных параметров в начале метода
public async Task<Order> GetOrderAsync(Guid orderId, string userId)
{
    ArgumentNullException.ThrowIfNull(orderId);
    ArgumentException.ThrowIfNullOrWhiteSpace(userId);
    
    var order = await _repo.GetByIdAsync(orderId);
    
    if (order == null)
        throw new NotFoundException($"Order {orderId} not found");
    
    if (order.UserId != userId)
        throw new ForbiddenAccessException();
    
    return order;
}
```

---

## Exception Handling Patterns

```csharp
// ❌ Exception for flow control
try
{
    var user = _repo.Find(id);  // Returns null if not found
}
catch (NotFoundException) { ... }  // Exception как условие

// ✅ Guard clause
var user = _repo.Find(id);
if (user == null) return NotFound();

// ❌ Голый throw ex (теряет stack trace)
catch (Exception ex) { throw ex; }

// ✅ Правильно
catch (Exception ex) { throw; }  // Сохраняет stack trace

// ✅ Global exception handler
app.UseExceptionHandler(app =>
{
    app.Run(async context =>
    {
        var feature = context.Features.Get<IExceptionHandlerFeature>();
        var error = feature?.Error;
        
        context.Response.StatusCode = error switch
        {
            NotFoundException => 404,
            ValidationException => 400,
            UnauthorizedAccessException => 403,
            _ => 500
        };
        
        await context.Response.WriteAsJsonAsync(new
        {
            error = error?.Message ?? "An error occurred",
            type = error?.GetType().Name
        });
    });
});
```

---

## Nullability

```csharp
#nullable enable

public class UserService
{
    public async Task<User?> FindAsync(Guid id)  // Может вернуть null
        => await _db.Users.FirstOrDefaultAsync(u => u.Id == id);
    
    public async Task<User> GetAsync(Guid id)  // Никогда null
        => await _db.Users.FindAsync(id) 
           ?? throw new NotFoundException($"User {id} not found");
}

// Null-forgiving operator — только если уверены
var user = await FindAsync(id);
var name = user!.Name;  // ! — я знаю, что не null
```

---

## Чек-лист

- [ ] Roslyn Analyzers: CA, IDE, SonarAnalyzer
- [ ] .editorconfig: единый стиль, naming rules
- [ ] SonarQube: Quality Gate, duplications
- [ ] Code Review: архитектура, безопасность, производительность, тесты
- [ ] Guard clauses: ArgumentNullException.ThrowIfNull
- [ ] Exception handling: не для flow control, throw (не throw ex)
- [ ] Nullability: nullable enable, ? vs null-forgiving !
