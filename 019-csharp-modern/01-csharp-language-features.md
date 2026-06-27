# Современные фишки C# (v8 → v13) — факты + примеры

---

## Эволюция C#: почему это важно

C# — один из самых быстро развивающихся мейнстрим-языков. С 2019 года (C# 8) команда .NET перешла на yearly release cycle, и каждый год приносит фичи, которые раньше были доступны только в F#, Rust, Kotlin или TypeScript. Знание современных конструкций — маркер Senior-разработчика.

---

## Records (C# 9, улучшены в C# 10)

**Факты:**
- `record` — reference type с **value equality** (сравнение по значениям полей, а не по ссылке)
- `record struct` (C# 10) — value type с value equality
- Автоматически генерирует: `Equals`, `GetHashCode`, `ToString`, `Deconstruct`, операторы `==`/`!=`
- `with` expression — non-destructive mutation (создание копии с изменением)

```csharp
// Positional record (C# 9) — компактный синтаксис
public record Person(string FirstName, string LastName, int Age);

var alice = new Person("Alice", "Smith", 30);
var olderAlice = alice with { Age = 31 };  // Копия, только Age изменён

// Value equality
var bob1 = new Person("Bob", "Jones", 25);
var bob2 = new Person("Bob", "Jones", 25);
Console.WriteLine(bob1 == bob2);  // True! (reference types, но value equality)

// Deconstruction
var (firstName, lastName, age) = alice;

// Наследование records
public record Employee(string FirstName, string LastName, int Age, string Department)
    : Person(FirstName, LastName, Age);

// record struct (C# 10) — value type, стековая аллокация
public readonly record struct Point3D(double X, double Y, double Z);
```

**Факт:** `record` компилируется в класс с переопределёнными методами равенства. `record struct` — в структуру. Выбирайте `record class` для DTO, моделей; `record struct` — для высокопроизводительных сценариев.

**Факт:** `with` expression для record class создаёт **shallow clone** через `protected` конструктор копирования. Reference-поля внутри record не клонируются глубоко.

---

## Pattern Matching (C# 7 → C# 11 — эволюция)

### Property Pattern (C# 8)
```csharp
if (order is { Status: OrderStatus.Pending, Total: > 1000 })
    ApplyDiscount(order);

// В switch expression
var discount = order switch
{
    { Status: OrderStatus.VIP, Total: > 10000 } => 0.20m,
    { Status: OrderStatus.VIP } => 0.10m,
    { Total: > 5000 } => 0.05m,
    _ => 0
};
```

### List Pattern (C# 11)
```csharp
int[] numbers = [1, 2, 3, 4];

// Проверка структуры массива/списка
var description = numbers switch
{
    [] => "Empty",
    [0] => "Starts with zero",
    [_, .., 4] => "Ends with 4",          // _ = любой один, .. = любые 0+
    [1, .. var middle, 4] => $"1...{string.Join(",", middle)}...4",
    [_, _, ..] => "At least 2 elements"
};

// List pattern в if
if (args is ["--verbose", var fileName])
    ProcessFile(fileName, verbose: true);
```

### Span/ReadOnlySpan pattern matching (C# 11)
```csharp
ReadOnlySpan<char> text = "Hello, World!";
if (text is "Hello, World!")
    Console.WriteLine("Match on Span<char>!");
```

### Extended Property Pattern (C# 10)
```csharp
// Было (C# 9):  { Address: { City: "London" } }
// Стало (C# 10):
if (person is { Address.City: "London", Address.Country: "UK" })
    Console.WriteLine("Londoner");
```

---

## Switch Expressions (C# 8, улучшены в C# 9)

```csharp
// Классический switch statement
string GetCategory(int points)
{
    switch (points)
    {
        case >= 90: return "A";
        case >= 80: return "B";
        case >= 70: return "C";
        default: return "F";
    }
}

// Switch expression (C# 8) — функциональный стиль
string GetCategory(int points) => points switch
{
    >= 90 => "A",
    >= 80 => "B",
    >= 70 => "C",
    _ => "F"
};

// Tuple pattern + switch expression
decimal GetDiscount(CustomerType type, int years) => (type, years) switch
{
    (CustomerType.VIP, >= 5) => 0.25m,
    (CustomerType.VIP, _) => 0.15m,
    (CustomerType.Regular, >= 3) => 0.10m,
    _ => 0
};

// Switch expression с guard clauses (C# 9)
string DescribeNumber(int n) => n switch
{
    < 0 and > -10 => "Small negative",
    < 0 => "Negative",
    0 => "Zero",
    > 0 and < 10 => "Small positive",
    > 0 => "Positive",
};
```

**Факт:** Switch expression **требует исчерпывающего перебора** (exhaustiveness) — компилятор предупредит, если не все случаи покрыты.

---

## Primary Constructors (C# 12)

```csharp
// C# 11: много boilerplate
public class UserService
{
    private readonly ILogger<UserService> _logger;
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;
    
    public UserService(ILogger<UserService> logger, IUserRepository repository, IEmailService emailService)
    {
        _logger = logger;
        _repository = repository;
        _emailService = emailService;
    }
}

// C# 12: Primary Constructor — минимум кода
public class UserService(
    ILogger<UserService> logger,
    IUserRepository repository,
    IEmailService emailService)
{
    public async Task<User> CreateAsync(string email)
    {
        logger.LogInformation("Creating user {Email}", email);
        var user = await repository.CreateAsync(email);
        await emailService.SendWelcomeAsync(email);
        return user;
    }
}

// Primary Constructors в структурах и record (C# 12 для class/struct)
public readonly struct Distance(double dx, double dy)
{
    public double Magnitude => Math.Sqrt(dx * dx + dy * dy);
}
```

**Факт:** Параметры primary constructor доступны **во всём теле класса** (поля, свойства, методы). Это не просто захват в конструкторе — компилятор захватывает параметры как поля, если они используются за пределами конструктора.

**Факт:** Для `record` первичный конструктор существовал с C# 9. C# 12 распространил это на все `class` и `struct`.

---

## Collection Expressions (C# 12)

```csharp
// C# 11: громоздко
int[] numbers = new[] { 1, 2, 3 };
List<string> names = new() { "Alice", "Bob" };
Span<int> span = stackalloc int[] { 1, 2, 3 };

// C# 12: унифицированный синтаксис — collection expressions
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob", "Charlie"];
Span<char> chars = ['H', 'e', 'l', 'l', 'o'];

// Spread operator (..) — разворачивание коллекций
int[] combined = [.. numbers, 6, 7, .. [8, 9, 10]];

// Поддерживают любой тип с collection initializer или builder
HashSet<int> set = [1, 2, 3, 2, 1];  // → { 1, 2, 3 }
ImmutableArray<string> immutable = ["a", "b", "c"];

// Параметры методов
void Process(string[] items) { }
Process(["one", "two", "three"]);

// Многомерные? Пока нет, но обсуждается для C# 13+
```

**Факт:** Collection expressions компилируются в оптимальный код: для `T[]` — `new T[]`, для `ReadOnlySpan<T>` — `RuntimeHelpers.CreateSpan`, для `List<T>` — через `CollectionsMarshal`. Никакого лишнего копирования.

---

## Span\<T\>, ReadOnlySpan\<T\>, ref struct (C# 7.2+)

```csharp
// Span — stack-only reference type (ref struct)
// Не может быть полем класса, не может быть в async, не может быть в IEnumerable<T>

// Без Span: аллокация подстроки
string text = "Hello, World!";
string substring = text.Substring(7, 5);  // Новая строка в куче!

// Со Span: ноль аллокаций
ReadOnlySpan<char> span = text.AsSpan();
ReadOnlySpan<char> world = span[7..12];  // View, не копия

// Span над массивом
int[] array = [1, 2, 3, 4, 5];
Span<int> slice = array.AsSpan(1, 3);  // { 2, 3, 4 } — view

// Span над stackalloc
Span<int> buffer = stackalloc int[64];  // Вообще без GC

// Span над unmanaged memory
var ptr = NativeMemory.Alloc(1024);
Span<byte> nativeSpan = new Span<byte>(ptr.ToPointer(), 1024);
```

**Факт:** `ref struct` имеет ограничения: не может быть полем класса, элементом массива, захвачен в лямбду или async метод, приведён к `object`. Это плата за zero-overhead абстракцию.

### SearchValues (C# 12, .NET 8)
```csharp
// Оптимизированный поиск любого из набора символов
// Использует SIMD для поиска (SSE2/AVX2)
SearchValues<char> vowels = SearchValues.Create("aeiouAEIOU");

ReadOnlySpan<char> text = "Hello, World!";
int index = text.IndexOfAny(vowels);  // 1 ('e') — SIMD-ускоренный поиск
```

---

## Nullable Reference Types (C# 8)

```csharp
// Включение: <Nullable>enable</Nullable> в .csproj или #nullable enable

// Non-nullable (по умолчанию при NRT enabled)
string name = "Alice";    // OK
string? maybeNull = null; // OK — явно nullable

// Компилятор предупреждает
void Greet(string name)
{
    Console.WriteLine(name.Length);  // OK
}

string? maybeNull = GetName();
Greet(maybeNull);  // ⚠️ CS8600: Converting null literal to non-nullable type

// Null-forgiving operator (!) — «я знаю, что здесь не null»
string definitely = maybeNull!;
Greet(definitely);

// Атрибуты для статического анализа
public static bool IsNullOrEmpty([NotNullWhen(false)] string? value)
{
    return value is null or "";
}

string? input = Console.ReadLine();
if (!string.IsNullOrEmpty(input))
    Console.WriteLine(input.Length);  // OK — компилятор знает, что input не null
```

**Факт:** NRT — это **статический анализ на этапе компиляции**. В рантайме никаких проверок не добавляется. `string` и `string?` — один и тот же тип в IL.

---

## Raw String Literals (C# 11)

```csharp
// Обычные строки: экранирование
var json = "{\n  \"name\": \"Alice\",\n  \"age\": 30\n}";

// Raw string literal — любое количество кавычек
var json = """
{
  "name": "Alice",
  "age": 30
}
""";

// JSON в коде — без экранирования
var query = """
SELECT u.Id, u.Name, u.Email
FROM Users u
WHERE u.IsActive = 1
  AND u.CreatedAt > @since
ORDER BY u.Name
""";

// Интерполяция в raw strings ($$ — кол-во $ определяет кол-во фигурных скобок)
var name = "Alice";
var greeting = $$"""
Hello, {{name}}!
Welcome to C# 11.
""";
```

**Факт:** Отступ в raw string literal определяется по **наименьшему отступу** закрывающих кавычек. Вся строка сдвигается соответственно.

---

## File-Scoped Types (C# 11)

```csharp
// C# 10: тип доступен во всей assembly
public class InternalHelper { }

// C# 11: тип виден только в этом файле
file class JsonSerializerHelper
{
    // Используется только в этом файле — не загрязняет namespace
}

file interface IInternalValidator
{
    bool Validate(string input);
}
```

**Факт:** `file` модификатор — более гранулярный контроль видимости, чем `internal`. Идеально для source generators и хелперов, которые должны быть скрыты от остального проекта.

---

## init, required (C# 9, C# 11)

```csharp
// init (C# 9) — свойство можно установить только при инициализации
public class Configuration
{
    public string ConnectionString { get; init; }  // Только в конструкторе или инициализаторе
    public int MaxRetries { get; init; } = 3;
}

var config = new Configuration
{
    ConnectionString = "Server=...",
    MaxRetries = 5
};
// config.ConnectionString = "...";  // ❌ CS8852: Init-only property

// required (C# 11) — свойство обязательно заполнить при создании
public class CreateUserRequest
{
    public required string Name { get; init; }     // Обязательно!
    public required string Email { get; init; }    // Обязательно!
    public int Age { get; init; }                   // Опционально
}

var request = new CreateUserRequest
{
    Name = "Alice",
    Email = "alice@example.com"
    // Age опционально
};
// var request = new CreateUserRequest();  // ❌ CS9035: Required member not set
```

**Факт:** `required` проверяется **компилятором**. Если вызывать конструктор через рефлексию — проверки не будет.

**Факт:** `SetsRequiredMembers` атрибут — если ваш конструктор устанавливает все required свойства, пометьте его этим атрибутом, чтобы компилятор не требовал инициализатора.

---

## Default Interface Methods (C# 8)

```csharp
// Интерфейсы с реализацией по умолчанию
public interface ILogger
{
    void Log(string message);  // Абстрактный (без реализации)
    
    // Методы с default implementation
    void LogError(string message) => Log($"[ERROR] {message}");
    void LogWarning(string message) => Log($"[WARN] {message}");
    
    // Статические методы в интерфейсе (C# 11)
    static abstract ILogger Create(string name);
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
    // LogError и LogWarning — автоматически из интерфейса
    
    public static ILogger Create(string name) => new ConsoleLogger();
}

// Virtual static methods (C# 11) — для generic math
public interface IAdditive<T> where T : IAdditive<T>
{
    static abstract T operator +(T left, T right);
    static abstract T Zero { get; }
}
```

**Факт:** Default Interface Methods были добавлены для совместимости с Java/Android API (Xamarin). Используйте осторожно — это не замена абстрактным классам.

**Факт:** Static abstract members в интерфейсах (C# 11) — основа для **generic math** в .NET 7+. `int`, `float`, `double` теперь реализуют `INumber<T>`.

---

## Generic Math (C# 11, .NET 7+)

```csharp
// До C# 11: невозможно написать generic-алгоритм для чисел
// После: static abstract members в интерфейсах

public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var value in values)
        sum += value;
    return sum;
}

// Работает с любым числовым типом
int intSum = Sum(new[] { 1, 2, 3, 4, 5 });        // 15
double dSum = Sum(new[] { 1.5, 2.5, 3.0 });        // 7.0
decimal mSum = Sum(new[] { 1.0m, 2.0m, 3.0m });    // 6.0m

// Generic Parse
public static T Parse<T>(string s) where T : IParsable<T> =>
    T.Parse(s, CultureInfo.InvariantCulture);

int value = Parse<int>("42");
double pi = Parse<double>("3.14159");
```

**Факт:** `INumber<T>` включает: `+`, `-`, `*`, `/`, `Zero`, `One`, `MinValue`, `MaxValue`, `Parse`, `TryParse` и ещё ~60 членов. Это позволяет писать математический код generic.

---

## Global Usings & Implicit Usings (C# 10)

```xml
<!-- .csproj -->
<PropertyGroup>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>

<!-- Или явно: GlobalUsings.cs -->
```

```csharp
// GlobalUsings.cs — один файл на проект
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;

// Или в .csproj
<ItemGroup>
  <Using Include="System" />
  <Using Include="Microsoft.Extensions.Logging" />
</ItemGroup>
```

**Факт:** `ImplicitUsings` автоматически подключает `System`, `System.Collections.Generic`, `System.Linq`, `System.Threading.Tasks` и другие в зависимости от типа проекта (ASP.NET Core добавляет `Microsoft.AspNetCore.Builder` и т.д.).

---

## Source Generators (C# 9, .NET 5+)

```csharp
// Source Generator — генерирует код на этапе компиляции
// Пример: генерация INotifyPropertyChanged (CommunityToolkit.Mvvm)

// Пишете:
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name;
}

// Генерируется на этапе компиляции:
// public partial class MainViewModel
// {
//     public string Name
//     {
//         get => _name;
//         set { if (SetProperty(ref _name, value)) OnPropertyChanged(...); }
//     }
// }
```

**Преимущества Source Generators:**
- **Нет рефлексии** — всё на этапе компиляции
- **AOT-совместимость** — работает в Native AOT
- **Производительность** — сгенерированный код как ручной

**Популярные Source Generators:**
| Generator | NuGet | Что генерирует |
|---|---|---|
| CommunityToolkit.Mvvm | `CommunityToolkit.Mvvm` | `INotifyPropertyChanged`, `RelayCommand` |
| Microsoft.Extensions.Logging | Встроен | `LoggerMessage` (high-performance logging) |
| System.Text.Json | Встроен | JSON serialization metadata (.NET 8+) |
| MinimalApi | Встроен | Route handler metadata |
| AutoMapper | `AutoMapper.Extensions.Microsoft.DependencyInjection` | Mapping profiles |

---

## Native AOT (C# 7+, .NET 7/8+ стабильно)

```xml
<!-- .csproj -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
# Публикация — один файл, без рантайма
dotnet publish -c Release -r linux-x64 --self-contained
ls -lh ./bin/Release/net8.0/linux-x64/publish/MyApp
# MyApp — 3-10 MB (зависит от размера приложения)
```

**Факты:**
- Компиляция в нативный код через **CoreRT**/NativeAOT
- **Быстрый старт** — нет JIT-компиляции при запуске
- **Меньше памяти** — нет JIT-кода в памяти
- **Ограничения:** нет рефлексии (только размеченная), нет dynamic, нет Assembly.Load

**Факт:** ASP.NET Core теперь поддерживает Native AOT (с .NET 8) для Minimal API. Холодный старт контейнера: ~50ms вместо ~500ms.

---

## Другие фичи (кратко)

### Target-typed new (C# 9)
```csharp
List<int> numbers = new();  // Тип выводится из левой части
Dictionary<string, User> users = new();  // Меньше повторений
```

### Target-typed conditional (C# 9)
```csharp
int? result = condition ? 1 : null;  // Раньше нужно было: (int?)1 : null
```

### Constant Interpolated Strings (C# 10)
```csharp
const string BaseUrl = "https://api.example.com";
const string Version = "v1";
const string FullUrl = $"{BaseUrl}/{Version}/users";  // const + интерполяция!
```

### Extended nameof scope (C# 11)
```csharp
[MyAttribute(nameof(parameter))]  // Раньше нельзя было
void Method(string parameter) { }
```

### Alias any type (C# 12)
```csharp
using Matrix = System.Collections.Generic.List<int[]>;
using Point = (int X, int Y);
using StringMap = System.Collections.Generic.Dictionary<string, string>;
```

### Default Lambda Parameters (C# 12)
```csharp
var greet = (string name = "World") => $"Hello, {name}!";
Console.WriteLine(greet());       // Hello, World!
Console.WriteLine(greet("C#"));   // Hello, C#!
```

### Inline Arrays (C# 12)
```csharp
[System.Runtime.CompilerServices.InlineArray(10)]
public struct FixedBuffer10<T> { private T _first; }

var buffer = new FixedBuffer10<int>();
buffer[0] = 42;  // На стеке, без аллокаций!
```

### ref readonly parameters (C# 12)
```csharp
void Process(ref readonly int value)  // Передача по ссылке, но только для чтения
{
    Console.WriteLine(value);
}
```

### params Span (C# 13)
```csharp
void Log(params ReadOnlySpan<string> messages)  // Без аллокации массива!
{
    foreach (var msg in messages)
        Console.WriteLine(msg);
}

Log("one", "two", "three");  // Span — без аллокаций!
```

### Lock object (C# 13 / .NET 9)
```csharp
Lock myLock = new();  // Специальный тип для lock
lock (myLock) { }     // через using (myLock.EnterScope())
```

---

## Хронология версий

| Версия | Год | Ключевые фичи |
|---|---|---|
| **C# 8** | 2019 | NRT, Async streams, Default interface methods, Switch expressions, Ranges/Indexes, Null-coalescing assignment |
| **C# 9** | 2020 | Records, Init-only properties, Target-typed new, Pattern matching improvements, Top-level statements, Source generators |
| **C# 10** | 2021 | Global usings, File-scoped namespaces, Record structs, Constant interpolated strings, Extended property patterns |
| **C# 11** | 2022 | Raw string literals, List patterns, Required members, File-scoped types, Generic math (static abstract interfaces), UTF-8 string literals |
| **C# 12** | 2023 | Primary constructors (class/struct), Collection expressions, Inline arrays, Default lambda parameters, Alias any type, ref readonly parameters |
| **C# 13** | 2024 | params Span, Lock object, ref struct improvements in generics, partial properties, escape character `\e` |

---

## Чек-лист Senior C# Developer

- [ ] Records: когда class vs struct, with-выражения, value equality
- [ ] Pattern Matching: property, tuple, list patterns; switch expressions
- [ ] Primary Constructors: DI-регистрация (C# 12)
- [ ] Collection Expressions: `[...]`, spread operator
- [ ] Span/Memory: zero-allocation string/array slicing
- [ ] NRT: nullable reference types, null-forgiving operator, `[NotNullWhen]`
- [ ] Raw String Literals: `"""..."""`, интерполяция
- [ ] Source Generators: CommunityToolkit.Mvvm, LoggerMessage
- [ ] Generic Math: `INumber<T>`, static abstract interfaces
- [ ] Native AOT: publish как один бинарник, ограничения
- [ ] init/required: неизменяемые DTO, обязательные поля
- [ ] Global/Implicit Usings: минимизация boilerplate
