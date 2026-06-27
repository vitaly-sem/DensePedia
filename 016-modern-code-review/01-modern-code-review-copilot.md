# Modern Code Review & AI Coding

---

## Code Review Best Practices

**Факты:**
- Code Review — не поиск ошибок, а **knowledge sharing** и **design review**
- **Small PRs** (< 400 строк) — в 2x быстрее ревьюятся и реже содержат баги
- **DORA metrics**: время от commit до deploy (elite: < 1 день)

```markdown
# PR Checklist для ревьюера

## Architecture
- [ ] Соответствует SOLID и паттернам проекта
- [ ] Нет layer violations (UI → DB напрямую)
- [ ] DI правильно настроен (lifetimes)

## Security
- [ ] SQL Injection: нет конкатенации строк
- [ ] XSS: HTML-encoding, CSP headers
- [ ] Authentication/Authorization проверены
- [ ] Input validation на сервере
- [ ] Secrets не в коде

## Performance
- [ ] No N+1 queries (EF Core Include/ThenInclude)
- [ ] Async/await не блокируется (.Result, .Wait)
- [ ] Нет аллокаций в hot path
- [ ] LINQ не материализуется без необходимости

## Tests
- [ ] Unit tests для бизнес-логики
- [ ] Integration tests для critical paths
- [ ] Тесты детерминированы

## Code Style
- [ ] SonarQube quality gate passed
- [ ] .editorconfig соблюдён
- [ ] Магические числа → константы
- [ ] Имена классов/методов отражают назначение
```

---

## AI Pair Programming — Copilot, Cursor

**Факты:**
- **GitHub Copilot** — GPT-4 based автодополнение кода
- **Copilot Chat** — AI-ассистент для объяснения, рефакторинга
- **Cursor** — AI-first IDE (встроенная интеграция AI)

### Copilot — эффективное использование

```csharp
// 1. Хорошие промпты — опишите что нужно в комментарии
// Create a method that validates email format using regex
// and returns true/false with an error message
public bool TryValidateEmail(string email, out string error)
{
    // Copilot сгенерирует реализацию
}

// 2. Copilot как тест-писатель
// "Write unit tests for OrderService.PlaceOrderAsync"
// Copilot сгенерирует тест-методы

// 3. Copilot для boilerplate
// "Create constructor injection for IUserRepository, IEmailService, ILogger"
// Copilot: private readonly IUserRepository _userRepo; ...
```

### Copilot — чего НЕЛЬЗЯ делать

```csharp
// ❌ Не доверять коду без проверки (Copilot может сгенерировать уязвимый код)
// ❌ Не вставлять sensitive данные в промпты (API keys, passwords)
// ❌ Не игнорировать лицензии (Copilot может сгенерировать GPL код)

// ✅ Проверять сгенерированный код на:
// - Security (SQL injection, XSS, hardcoded secrets)
// - Performance (N+1, blocking calls)
// - Correctness (edge cases, null handling)
```

**Факт:** Copilot работает как **next-word prediction** для кода. Он не думает об архитектуре — только о локальном контексте.

---

## Agentic Coding — AGI в разработке

**Факт:** Современные AI-агенты (Devin, Cline, AutoGPT) умеют:

| Capability | Example | Status |
|---|---|---|
| **Plan** | Прочитать репозиторий, понять архитектуру | ✅ |
| **Code** | Сгенерировать код по спецификации | ✅ |
| **Test** | Написать и запустить тесты | ✅ |
| **Debug** | Найти и исправить баг | ✅ |
| **Deploy** | CI/CD, инфраструктура (Pulumi, Terraform) | ⚠️ Partial |
| **Architecture** | Многосервисная архитектура | ❌ (human needed) |

### Agent Workflow

```
User: "Add user authentication to the API"

Agent:
1. Read project structure → понимает что ASP.NET Core
2. Plan:
   - Add Identity packages
   - Create User entity
   - Configure JWT auth
   - Add login/register endpoints
3. Execute:
   - dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
   - Generate migration
   - Create AuthController
   - Write tests
4. Review:
   - Run tests (✅ pass)
   - Check for vulnerabilities
   - PR created
```

---

## Practical AI Coding Tips

```markdown
### Prompt Engineering для кода

✅ Хорошие промпты:
- "Create a rate-limited HTTP client with Polly retry policy"
- "Write a SQL query for top 10 customers by revenue this month"
- "Refactor this method to use strategy pattern: [code]"

❌ Плохие промпты:
- "Fix this code" (нет контекста)
- "Make it faster" (нет метрик)
- "Write the whole app" (слишком broad)

### Tools Comparison

| Tool | Type | Best for |
|---|---|---|
| GitHub Copilot | Autocomplete | Daily coding |
| Cursor | AI IDE | Full project context |
| Claude (Anthropic) | Chat + Code | Complex algorithms |
| GPT-4 (ChatGPT) | Chat | Architecture discussions |
| Gemini (Google) | Chat + Code | Long context (>1M tokens) |
| Devin | AI Agent | Autonomous development |
```

---

## Чек-лист

- [ ] Code Review: small PRs (< 400 lines), checklist
- [ ] Copilot: хорошие промпты, проверка сгенерированного кода
- [ ] AI Agents: Devin, plan→code→test→review workflow
- [ ] Prompt Engineering: конкретные задачи, контекст
- [ ] Security: не доверять AI коду без проверки
- [ ] DORA metrics: deploy frequency, lead time
