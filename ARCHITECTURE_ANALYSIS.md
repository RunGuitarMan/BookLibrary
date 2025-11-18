# Анализ архитектуры проекта BookLibrary

## Оглавление
1. [Введение](#введение)
2. [Обзор технологического стека](#обзор-технологического-стека)
3. [Интересные архитектурные решения](#интересные-архитектурные-решения)
4. [Паттерны Domain-Driven Design](#паттерны-domain-driven-design)
5. [CQRS и медиатор](#cqrs-и-медиатор)
6. [Работа с базой данных](#работа-с-базой-данных)
7. [Обработка ошибок и доменные исключения](#обработка-ошибок-и-доменные-исключения)
8. [Observability и мониторинг](#observability-и-мониторинг)
9. [Тестирование архитектуры](#тестирование-архитектуры)
10. [Выводы](#выводы)

---

## Введение

BookLibrary — это образцовый проект библиотечной системы, демонстрирующий современные подходы к разработке enterprise-приложений на .NET 9. Проект реализует систему управления книгами и абонентами с возможностью выдачи и возврата книг.

**Ключевые характеристики проекта:**
- Clean Architecture с чёткой изоляцией слоёв
- Domain-Driven Design (DDD) с агрегатами, value objects и доменными событиями
- CQRS паттерн для разделения команд и запросов
- Vertical Slices Architecture для организации фичей
- Railway-Oriented Programming через Result<T>
- Outbox Pattern для надёжной публикации событий
- Автоматическое тестирование архитектурных ограничений

---

## Обзор технологического стека

### Платформа и фреймворки
- **.NET 9.0** (последняя LTS версия) с C# 13.0
- **ASP.NET Core** для веб-API
- **Entity Framework Core 9.0.4** с PostgreSQL
- **Npgsql 9.0.4** — провайдер данных PostgreSQL

### Ключевые библиотеки

#### Domain-Driven Design
- **FluentResults 3.16.0** — Result<T> типы для Railway-Oriented Programming
- **StronglyTypedId** — генерация типобезопасных идентификаторов на уровне компиляции
- **Sstv.DomainExceptions** — доменные исключения с кодами ошибок и метаданными

#### CQRS и медиация
- **Mediator 2.1.7** — source-generated медиатор для CQRS (без рефлексии в рантайме)

#### Паттерны надёжности
- **Sstv.Outbox.EntityFrameworkCore.Npgsql** — Transactional Outbox Pattern

#### Валидация
- **FluentValidation.AspNetCore 11.3.0** — декларативная валидация

#### Observability
- **Serilog.AspNetCore 9.0.0** — структурированное логирование
- **OpenTelemetry 1.12.0**:
  - Instrumentation для ASP.NET Core, EF Core, HTTP, Runtime
  - Prometheus exporter для метрик
  - OTLP exporter для трейсинга

#### Тестирование
- **NUnit 4.3.2**
- **ArchUnitNET 0.11.4** — автоматическая проверка архитектурных ограничений

---

## Интересные архитектурные решения

### 1. 🎯 Vertical Slices Architecture

**Описание**: Вместо традиционной горизонтальной организации (Controllers/, Services/, Repositories/), код организован по вертикальным срезам — каждая фича находится в отдельной папке со всеми необходимыми компонентами.

**Структура:**
```
Application/Features/
├── Books/
│   ├── AddNewBook/
│   │   ├── AddNewBookCommand.cs
│   │   ├── AddNewBookUseCase.cs
│   │   └── AddNewBookCommandValidator.cs
│   ├── BorrowBook/
│   ├── ReturnBook/
│   ├── GetBook/
│   └── ServiceCollectionExtensions.cs  # Регистрация всех use-case книг
├── Abonents/
│   ├── RegisterAbonent/
│   └── ServiceCollectionExtensions.cs
```

**Преимущества:**
- **Высокая связность**: Всё, что касается одной фичи, находится в одном месте
- **Низкая связанность**: Фичи изолированы друг от друга
- **Простота навигации**: Не нужно прыгать между слоями
- **Легко удалять**: Удаление фичи = удаление папки

**Код из проекта** (`Books/ServiceCollectionExtensions.cs`):
```csharp
public static IServiceCollection AddBooksFeatures(this IServiceCollection services)
{
    return services
        .AddScoped<AddNewBookUseCase>()
        .AddScoped<BorrowBookUseCase>()
        .AddScoped<ReturnBookUseCase>()
        .AddScoped<GetBookUseCase>()
        .AddScoped<GetPagedBooksUseCase>()
        .AddScoped<GetBorrowedBooksUseCase>();
}
```

---

### 2. 🧩 Доменные события с механизмом редукции (Domain Events Reducer)

**Описание**: Вместо публикации множества одинаковых событий, проект использует **DomainEventsReducer**, который агрегирует события перед отправкой в обработчики.

**Проблема**: При добавлении 100 экземпляров одной книги создаётся 100 событий `BookCreatedEvent`, что приводит к:
- 100 вызовов обработчиков
- Избыточной нагрузке на БД
- Дублированию логов

**Решение** (`DomainEventsReducer.cs`):
```csharp
public IEnumerable<IDomainEvent> Reduce(IReadOnlyCollection<IDomainEvent> domainEvents)
{
    var bookCreatedEvents = new List<BookCreatedEvent>();

    foreach (var domainEvent in domainEvents)
    {
        if (domainEvent is BookCreatedEvent bookCreatedEvent)
            bookCreatedEvents.Add(bookCreatedEvent);
        else
            yield return domainEvent;
    }

    // Группируем по ISBN + Дате публикации
    var groups = bookCreatedEvents.GroupBy(x =>
        new { x.Title, x.Isbn, x.PublicationDate });

    foreach (var group in groups)
    {
        // Одно событие с Count = количество книг
        yield return new BookCreatedEvent(
            group.Key.Title,
            group.Key.Isbn,
            group.Key.PublicationDate,
            group.Count()  // 100 вместо 100 событий!
        );
    }
}
```

**Результат**: 100 событий → 1 событие с Count=100

**Интеграция** (`DomainEventDispatcher.cs:79`):
```csharp
var reducedDomainEvents = _domainEventsReducer.Reduce(domainEvents);

foreach (var domainEvent in reducedDomainEvents)
{
    await _mediator.Publish(domainEvent, ct);
}
```

**Преимущества:**
- ⚡ Значительное снижение нагрузки
- 📊 Семантически точнее: "Создано 100 копий книги X", а не "100 отдельных книг"
- 🔍 Меньше записей в логах
- 💾 Меньше записей в outbox

---

### 3. 🔄 Outbox Pattern для статистики книг

**Описание**: Проект использует Transactional Outbox Pattern для обеспечения eventual consistency при обновлении статистики книг.

**Архитектура:**

```
Domain Event (BookCreated)
    ↓
BookStatChangeWatcher (слушатель)
    ↓
BookStatChange (outbox item) → БД в той же транзакции
    ↓
Background Worker (Sstv.Outbox)
    ↓
BookStatChangeApplier (batch handler)
    ↓
BookStat (read model) — обновление статистики
```

**Код обработчика** (`BookStatChangeWatcher.cs:179`):
```csharp
public ValueTask Handle(BookCreated notification, CancellationToken ct)
{
    // Добавляем запись в outbox в той же транзакции, что и создание книги
    _ctx.BookStatChanges.Add(new BookStatChange
    {
        Id = _uuidGenerator.GenerateNew(),
        Isbn = notification.Book.Isbn,
        PublicationDate = notification.Book.PublicationDate,
        AvailableCount = 1,    // +1 доступная книга
        BorrowedCount = 0,
    });

    return ValueTask.CompletedTask;
}
```

**Batch обработка** (`BookStatChangeApplier.cs:64`):
```csharp
public async Task<OutboxBatchResult> HandleAsync(
    IReadOnlyCollection<BookStatChange> items,
    OutboxOptions options,
    CancellationToken ct)
{
    // Группируем изменения по ISBN + Дата
    var books = items
        .GroupBy(x => (x.Isbn, x.PublicationDate))
        .ToDictionary(x => x.Key, v => new BookStatChange
        {
            Isbn = v.Key.Isbn,
            PublicationDate = v.Key.PublicationDate,
            AvailableCount = v.Sum(x => x.AvailableCount),  // Суммируем
            BorrowedCount = v.Sum(x => x.BorrowedCount),
        });

    // Обновляем существующие записи или создаём новые
    // ...

    return OutboxBatchResult.FullyProcessed;
}
```

**Гарантии:**
- ✅ **Атомарность**: Событие и изменение статистики в одной транзакции
- ✅ **At-least-once delivery**: Worker переотправит при сбое
- ✅ **Идемпотентность**: Батч-обработка с группировкой
- ✅ **Производительность**: Батчинг снижает количество запросов к БД

---

### 4. 🏛️ Convention-based маппинг Value Objects в Entity Framework

**Проблема**: При большом количестве value objects конфигурация EF Core становится многословной:
```csharp
builder.Property(x => x.BookId).HasConversion(new BookIdEfValueConverter());
builder.Property(x => x.Isbn).HasConversion(new IsbnEfValueConverter());
builder.Property(x => x.Title).HasConversion(new BookTitleEfValueConverter());
// ... десятки строк для каждой сущности
```

**Решение**: Автоматическое обнаружение конвертеров по соглашению об именовании.

**Реализация** (`ServiceCollectionExtensions.cs`):
```csharp
options.ReplaceService<IValueConverterSelector,
    ValueObjectsConverterSelectorByConvention>();
```

**Соглашение:**
- Value Object: `BookId`
- Конвертер: `BookId` + `EfValueConverter` = `BookIdEfValueConverter`

**Кастомный селектор** (`ValueObjectsConverterSelectorByConvention.cs`):
```csharp
public class ValueObjectsConverterSelectorByConvention : ValueConverterSelector
{
    public override IEnumerable<ValueConverterInfo> Select(Type modelClrType, ...)
    {
        // Ищем класс {TypeName}EfValueConverter в том же assembly
        var converterType = modelClrType.Assembly
            .GetType($"{modelClrType.FullName}EfValueConverter");

        if (converterType != null)
        {
            yield return new ValueConverterInfo(
                modelClrType,
                converterType,
                ...
            );
        }

        // Fallback на стандартные конвертеры
        foreach (var info in base.Select(modelClrType, ...))
            yield return info;
    }
}
```

**Преимущества:**
- 🎯 Нулевая конфигурация для value objects
- 📏 Масштабируемость: добавляем новый VO → создаём конвертер → готово
- 🔒 Типобезопасность на этапе компиляции
- 📖 Явное соглашение об именовании

---

### 5. 🛤️ Railway-Oriented Programming через Result&lt;T&gt;

**Описание**: Вместо исключений для бизнес-логики проект использует паттерн Result из **FluentResults**.

**Пример доменной логики** (`Book.cs:108`):
```csharp
public Result Borrow(Abonement abonement, DateTimeOffset borrowedAt, DateOnly? returnBefore)
{
    // Валидация периода
    if (DateOnly.FromDateTime(borrowedAt.Date) >= returnBeforeDate)
    {
        return ErrorCodes.InvalidBookBorrowingPeriod.ToDomainError();
    }

    // Проверка: книга уже выдана?
    if (BorrowInfo is not null)
    {
        return BorrowInfo.AbonentId != abonement.AbonentId
            ? ErrorCodes.BookAlreadyBorrowed.ToDomainError()
            : Result.Ok();  // Тот же абонент — уже выдано
    }

    // Проверка: у абонента уже 3 книги?
    if (abonement.BorrowedBooksCount >= 3)
    {
        return ErrorCodes.TooManyBooksBorrowedAlready.ToDomainError();
    }

    // Успешная выдача
    BorrowInfo = new BorrowInfo(abonement.AbonentId, borrowedAt, returnBeforeDate);
    AddDomainEvent(new BookBorrowedEvent(...));

    return Result.Ok();
}
```

**Обработка в use-case** (`BorrowBookUseCase.cs`):
```csharp
var result = book.Borrow(abonement, command.BorrowedAt, command.ReturnBefore);

if (result.IsFailed)
{
    return result;  // Возвращаем ошибку без исключения
}

await _ctx.SaveChangesAsync(ct);
return Result.Ok();
```

**Маппинг в контроллере** (`ApiController.cs`):
```csharp
public IActionResult Ok(ResultBase result)
{
    if (result is { IsFailed: true })
    {
        var error = result.Errors.First();

        if (error is DomainErrorResult domainError)
        {
            return new ErrorCodeProblemDetails(domainError).ToActionResult();
        }
    }

    return base.Ok();
}
```

**Преимущества:**
- 🚫 **Нет try-catch** для бизнес-логики
- 🎯 **Явные сигнатуры**: `Result Borrow(...)` — сразу видно, что может быть ошибка
- 🔀 **Композиция**: Можно комбинировать Result'ы
- 🐛 **Легче отлаживать**: Ошибки не прерывают стек вызовов

---

### 6. 🔐 Strongly Typed IDs с Source Generation

**Описание**: Использование **StronglyTypedId** для генерации типобезопасных идентификаторов на уровне компиляции.

**Определение** (`BookId.cs`):
```csharp
[StronglyTypedId(generateJsonConverter: false)]
public partial struct BookId;
```

**Генерируемый код** (упрощённо):
```csharp
public partial struct BookId : IEquatable<BookId>
{
    public Guid Value { get; }

    public BookId(Guid value) => Value = value;

    public static implicit operator Guid(BookId id) => id.Value;
    public static implicit operator BookId(Guid value) => new(value);

    public bool Equals(BookId other) => Value.Equals(other.Value);
    public override int GetHashCode() => Value.GetHashCode();
}
```

**Использование:**
```csharp
// Создание
var bookId = new BookId(Guid.NewGuid());

// Неявное преобразование
Guid guid = bookId;  // Работает
BookId id = guid;    // Тоже работает

// Типобезопасность
void BorrowBook(BookId bookId, AbonentId abonentId) { }

BorrowBook(abonentId, bookId);  // ❌ Ошибка компиляции!
```

**Преимущества:**
- 🛡️ **Типобезопасность**: Невозможно перепутать BookId и AbonentId
- ⚡ **Нулевой overhead**: struct без boxing
- 🏗️ **Source generation**: Нет рефлексии в runtime
- 📝 **Читаемость**: `BookId` vs `Guid` — намного понятнее

---

### 7. 🧪 Автоматическое тестирование архитектурных ограничений (ArchUnitNET)

**Описание**: Проект содержит **автотесты для архитектуры**, которые гарантируют соблюдение Clean Architecture.

**Тест зависимостей Domain слоя** (`DomainDepsTests.cs`):
```csharp
[Test]
public void Domain_Should_Not_DependOn_OuterLayers()
{
    Types().That().Are(ProjectAssemblies.DomainLayer)
        .Should()
        .OnlyDependOnTypesThat()
        .Are(
            Types().That()
                .Are(ProjectAssemblies.DomainLayer).Or()
                .ResideInNamespace("FluentResults").Or()
                .ResideInNamespace("System").Or()
                .ResideInNamespace("Seedwork").Or()
                .ResideInNamespace("Mediator")
        )
        .Check(ProjectAssemblies.Architecture);
}
```

**Тест изоляции Application слоя** (`ApplicationDepsTests.cs`):
```csharp
[Test]
public void Application_Should_Not_DependOn_OuterLayers()
{
    Types().That().Are(ProjectAssemblies.ApplicationLayer)
        .Should()
        .NotDependOnAnyTypesThat()
        .ResideInAssembly(
            ProjectAssemblies.InfrastructureAssembly,
            ProjectAssemblies.ApiAssembly
        )
        .Check(ProjectAssemblies.Architecture);
}

[Test]
public void Application_Should_Not_DependOn_Frameworks()
{
    Types().That().Are(ProjectAssemblies.ApplicationLayer)
        .Should()
        .NotDependOnAnyTypesThat()
        .Are(ProjectAssemblies.FrameworkDependencies)  // ASP.NET, EF Core
        .Check(ProjectAssemblies.Architecture);
}
```

**Что проверяется:**
- ✅ Domain не зависит от Infrastructure/Application/API
- ✅ Application не зависит от Infrastructure/API
- ✅ Application не использует фреймворки (ASP.NET, EF Core)
- ✅ Все entities наследуются от базового класса
- ✅ Все value objects наследуются от ValueObject
- ✅ Все доменные события реализуют IDomainEvent

**Преимущества:**
- 🔒 **Защита от деградации**: CI сломается при нарушении архитектуры
- 📚 **Живая документация**: Тесты описывают правила архитектуры
- 👥 **Onboarding**: Новые разработчики не могут случайно нарушить правила
- 🔧 **Рефакторинг**: Безопасно менять структуру

---

### 8. 📊 Query Splitting для избежания Cartesian Explosion

**Описание**: EF Core настроен на использование **split queries** вместо одного JOIN.

**Конфигурация** (`ServiceCollectionExtensions.cs`):
```csharp
options.UseNpgsql(npgsqlDataSource, builder =>
{
    builder.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
})
```

**Проблема с одним запросом:**
```sql
-- Без split query: Cartesian product
SELECT b.*, a1.*, a2.*, a3.*
FROM books b
LEFT JOIN LATERAL jsonb_to_recordset(b.authors) AS a1(...)
LEFT JOIN LATERAL jsonb_to_recordset(b.authors) AS a2(...)
-- Результат: N книг × M авторов = огромный датасет
```

**Решение со split queries:**
```sql
-- Запрос 1: Книги
SELECT * FROM books WHERE ...;

-- Запрос 2: Связанные данные
SELECT * FROM ... WHERE book_id IN (...);
```

**Trade-offs:**
- ✅ Избегаем Cartesian explosion при сложных связях
- ✅ Меньше потребление памяти
- ✅ Лучше работает с JSONB колонками (массив авторов)
- ⚠️ Больше round-trips к БД
- ⚠️ Возможна несогласованность при изменении данных между запросами

---

### 9. 🏗️ Отсутствие Repository Pattern

**Описание**: Проект **не использует репозитории**, а работает напрямую с `DbSet<T>` через абстракцию `IApplicationContext`.

**Интерфейс контекста** (`IApplicationContext.cs`):
```csharp
public interface IApplicationContext
{
    DbSet<Abonent> Abonents { get; }
    DbSet<Book> Books { get; }
    DbSet<BookStat> BookStats { get; }

    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

**Использование в use-case** (`GetBookUseCase.cs`):
```csharp
public class GetBookUseCase
{
    private readonly IApplicationContext _ctx;

    public async Task<Result<BookDto>> ExecuteAsync(GetBookQuery query, CancellationToken ct)
    {
        var book = await _ctx.Books
            .TagWithFileMember()
            .Where(x => x.Id == query.BookId)
            .FirstOrDefaultAsync(ct);

        return book is null
            ? ErrorCodes.BookNotFound.ToDomainError<BookDto>()
            : Result.Ok(book.ToDto());
    }
}
```

**Почему нет репозиториев?**

1. **DbSet<T> уже является Generic Repository**
   ```csharp
   // Нет смысла делать обёртку
   interface IBookRepository
   {
       Task<Book?> FindByIdAsync(BookId id);
   }

   // Когда можно напрямую
   await _ctx.Books.FirstOrDefaultAsync(x => x.Id == id);
   ```

2. **Тестируемость не страдает**
   - Используется абстракция `IApplicationContext`
   - В тестах можно мокировать или использовать InMemory БД

3. **Гибкость запросов**
   ```csharp
   // С репозиторием пришлось бы добавлять методы
   Task<List<Book>> FindAvailableByIsbnAsync(Isbn isbn);
   Task<List<Book>> FindBorrowedByAbonentAsync(AbonentId id);

   // Или делать Specification Pattern (ещё больше кода)

   // С DbSet просто пишем LINQ
   await _ctx.Books
       .Where(x => x.BorrowInfo == null && x.Isbn == isbn)
       .ToListAsync();
   ```

**Преимущества:**
- 📉 Меньше кода
- 🎯 Прямолинейность
- 🔧 Гибкость при написании запросов
- 🧪 Тестируемость через интерфейс контекста

---

### 10. 📝 Структурированное логирование с LoggingScope

**Описание**: Централизованные константы для полей структурированного логирования.

**Определение констант** (`LoggingScope.cs`):
```csharp
public static class LoggingScope
{
    public static class User
    {
        public const string ID = "UserId";
        public const string NAME = "UserName";
    }

    public static class Book
    {
        public const string ID = "BookId";
        public const string ISBN = "ISBN";
        public const string TITLE = "Title";
        public const string PUBLICATION_DATE = "PublicationDate";
    }

    public static class Abonent
    {
        public const string ID = "AbonentId";
        public const string EMAIL = "Email";
    }
}
```

**Использование** (`AddNewBookUseCase.cs`):
```csharp
using var _ = _logger.BeginScope(new Dictionary<string, object?>
{
    [LoggingScope.Book.ISBN] = command.Isbn,
    [LoggingScope.Book.TITLE] = command.Title,
    [LoggingScope.Book.PUBLICATION_DATE] = command.PublicationDate.ToString(),
});

_logger.LogInformation("Adding {Count} books", command.Count);
```

**Результат в Serilog (JSON):**
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "Information",
  "message": "Adding 10 books",
  "ISBN": "978-5-17-983250-4",
  "Title": "Clean Architecture",
  "PublicationDate": "2017-09-10",
  "Count": 10
}
```

**Преимущества:**
- 🔍 **Единые поля** во всех логах
- 📊 **Легко парсить** в Elastic/Loki/etc
- 🎯 **Автодополнение** в IDE
- 🐛 **Нет опечаток**: `LoggingScope.Book.ISBN` vs `"isbn"` vs `"ISBN"` vs `"Isbn"`

---

## Паттерны Domain-Driven Design

### Агрегаты

Проект содержит два агрегата:

#### 1. Book Aggregate (`Book.cs`)
**Корневая сущность**: `Book`

**Value Objects:**
- `BookId` — строго типизированный идентификатор
- `BookTitle` — название книги с валидацией
- `Isbn` — ISBN с валидацией формата (regex для ISBN-10/13)
- `BookPublicationDate` — дата публикации
- `Author` — сложный VO (имя, фамилия, отчество)
- `BorrowInfo` — информация о выдаче книги

**Бизнес-правила:**
- Книга должна иметь хотя бы одного автора
- Один экземпляр книги может быть выдан только одному абоненту
- Период выдачи по умолчанию — 30 дней
- Дата возврата должна быть позже даты выдачи

**Доменные события:**
- `BookCreatedEvent` — книга добавлена в библиотеку
- `BookBorrowedEvent` — книга выдана
- `BookReturnedEvent` — книга возвращена

**Инкапсуляция:**
```csharp
public sealed class Book : Entity
{
    // Все сеттеры private!
    public BookId Id { get; private set; }
    public BookTitle Title { get; private set; }
    public BorrowInfo? BorrowInfo { get; private set; }

    // Изменение состояния только через бизнес-методы
    public Result Borrow(Abonement abonement, ...)
    {
        // Валидация + изменение состояния + событие
        if (BorrowInfo is not null)
            return ErrorCodes.BookAlreadyBorrowed.ToDomainError();

        BorrowInfo = new BorrowInfo(...);
        AddDomainEvent(new BookBorrowedEvent(...));
        return Result.Ok();
    }
}
```

#### 2. Abonent Aggregate (`Abonent.cs`)
**Корневая сущность**: `Abonent`

**Value Objects:**
- `AbonentId` — идентификатор
- `AbonentName` — структурированное имя (имя, фамилия)
- `Email` — валидированный email

**Доменные события:**
- `AbonentRegisteredEvent` — абонент зарегистрирован

### Domain Services

**Abonement** (`BookPublicationDate.cs:48`) — доменный сервис, представляющий контекст выдачи книги абоненту.

```csharp
public sealed class Abonement : ValueObject
{
    public AbonentId AbonentId { get; }
    public int BorrowedBooksCount { get; }  // Сколько книг уже на руках

    public Abonement(AbonentId abonentId, int borrowedBooksCount)
    {
        if (abonentId == default)
            throw ErrorCodes.InvalidBorrowerAbonentId.ToException();

        if (borrowedBooksCount < 0)
            throw ErrorCodes.InvalidBorrowerBooksCount.ToException();

        AbonentId = abonentId;
        BorrowedBooksCount = borrowedBooksCount;
    }
}
```

**Использование:**
```csharp
// В use-case создаём Abonement
var borrowedCount = await _ctx.Books
    .Where(x => x.BorrowInfo != null && x.BorrowInfo.AbonentId == command.AbonentId)
    .CountAsync();

var abonement = new Abonement(command.AbonentId, borrowedCount);

// Передаём в агрегат — он проверит бизнес-правила
var result = book.Borrow(abonement, command.BorrowedAt, command.ReturnBefore);
```

### Value Objects

**Базовый класс** (`ValueObject.cs`):
```csharp
public abstract class ValueObject : IEquatable<ValueObject>
{
    // Сравнение на основе компонентов
    public bool Equals(ValueObject? other)
    {
        return other is not null &&
               GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode()
    {
        var hash = new HashCode();
        foreach (var component in GetEqualityComponents())
            hash.Add(component);
        return hash.ToHashCode();
    }

    // Каждый VO определяет свои компоненты
    protected abstract IEnumerable<object?> GetEqualityComponents();
}
```

**Пример: BorrowInfo** (`BorrowInfo.cs`):
```csharp
public sealed class BorrowInfo : ValueObject
{
    public AbonentId AbonentId { get; private set; }
    public DateTimeOffset BorrowedAt { get; private set; }
    public DateOnly ReturnBefore { get; private set; }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return AbonentId.ToString();
        yield return BorrowedAt.ToString();
        yield return ReturnBefore.ToString();
    }
}
```

**Immutability:**
- Все сеттеры `private set`
- Конструктор `internal` — создание только внутри агрегата
- Сравнение по значению, а не по ссылке

### Доменные события

**Базовый интерфейс** (`IDomainEvent.cs`):
```csharp
public interface IDomainEvent : INotification  // Mediator INotification
{
}
```

**Хранение в Entity** (`Entity.cs:33`):
```csharp
public abstract class Entity : IEntity
{
    private List<IDomainEvent>? _domainEvents;

    protected virtual void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents ??= [];
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents?.Clear();
    }
}
```

**Диспетчеризация** (`DomainEventDispatcher.cs:58`):
```csharp
public async Task DispatchDomainEventsAsync(CancellationToken ct)
{
    while (HasUnpublishedDomainEvents())
    {
        // Собираем события из всех сущностей в ChangeTracker
        var entities = _dbContext.ChangeTracker
            .Entries<IEntity>()
            .Where(e => e.Entity.DomainEvents.Count > 0)
            .Select(e => e.Entity);

        var domainEvents = new List<IDomainEvent>();
        foreach (var entity in entities)
        {
            domainEvents.AddRange(entity.DomainEvents);
            entity.ClearDomainEvents();
        }

        // Редукция событий
        var reducedEvents = _domainEventsReducer.Reduce(domainEvents);

        // Публикация через Mediator
        foreach (var domainEvent in reducedEvents)
        {
            await _mediator.Publish(domainEvent, ct);
        }
    }
}
```

**Вызов перед сохранением** (`ApplicationContext.cs`):
```csharp
public override async Task<int> SaveChangesAsync(CancellationToken ct)
{
    // 1. Диспетчеризация доменных событий
    _domainEventDispatcher.SetDbContext(this);
    await _domainEventDispatcher.DispatchDomainEventsAsync(ct);

    // 2. Уведомление Mediator об изменениях
    await _mediator.DispatchPostSaveNotificationsAsync(...);

    // 3. Сохранение в БД
    return await base.SaveChangesAsync(ct);
}
```

**Гарантии:**
- ✅ События публикуются **в той же транзакции**, что и изменения
- ✅ События всегда согласованы с состоянием БД
- ✅ Обработчики могут добавлять новые сущности/события
- ✅ Цикл диспетчеризации (`while`) обрабатывает каскадные события

---

## CQRS и медиатор

### Source-Generated Mediator

Проект использует библиотеку **Mediator** с source generation вместо рефлексии.

**Регистрация:**
```csharp
services.AddMediator(o => o.ServiceLifetime = ServiceLifetime.Scoped);
```

**Генерация кода:**
- При компиляции создаётся класс `Mediator`, который знает обо всех handlers
- Нет рефлексии в runtime → быстрее
- Поддержка Ahead-of-Time (AOT) компиляции

### Commands

**Пример команды** (`BorrowBookCommand.cs`):
```csharp
public sealed record BorrowBookCommand(
    BookId BookId,
    AbonentId AbonentId,
    DateTimeOffset BorrowedAt,
    DateOnly? ReturnBefore
) : IRequest<Result>;
```

**Handler/Use-case** (`BorrowBookUseCase.cs`):
```csharp
public sealed class BorrowBookUseCase : IRequestHandler<BorrowBookCommand, Result>
{
    private readonly IApplicationContext _ctx;

    public async ValueTask<Result> Handle(BorrowBookCommand command, CancellationToken ct)
    {
        // 1. Загрузка агрегата
        var book = await _ctx.Books
            .FirstOrDefaultAsync(x => x.Id == command.BookId, ct);

        if (book is null)
            return ErrorCodes.BookNotFound.ToDomainError();

        // 2. Проверка абонента
        var abonent = await _ctx.Abonents
            .FirstOrDefaultAsync(x => x.Id == command.AbonentId, ct);

        if (abonent is null)
            return ErrorCodes.AbonentNotFound.ToDomainError();

        // 3. Подсчёт выданных книг
        var borrowedCount = await _ctx.Books
            .CountAsync(x => x.BorrowInfo!.AbonentId == command.AbonentId, ct);

        // 4. Создание доменного сервиса
        var abonement = new Abonement(command.AbonentId, borrowedCount);

        // 5. Вызов бизнес-логики
        var result = book.Borrow(abonement, command.BorrowedAt, command.ReturnBefore);

        if (result.IsFailed)
            return result;

        // 6. Сохранение (триггерит доменные события)
        await _ctx.SaveChangesAsync(ct);

        return Result.Ok();
    }
}
```

### Queries

**Read Model** (`BookStat.cs`):
```csharp
public sealed class BookStat
{
    public string Isbn { get; set; }
    public DateOnly PublicationDate { get; set; }
    public string Title { get; set; }
    public string Authors { get; set; }
    public int AvailableCount { get; set; }  // Денормализация
    public int BorrowedCount { get; set; }   // Денормализация
}
```

**Query** (`GetPagedBooksQuery.cs`):
```csharp
public sealed record GetPagedBooksQuery(
    int Page,
    int PageSize,
    string? SearchTerm
) : IRequest<Result<PagedResult<BookStatDto>>>;
```

**Handler** (`GetPagedBooksUseCase.cs`):
```csharp
public async ValueTask<Result<PagedResult<BookStatDto>>> Handle(
    GetPagedBooksQuery query,
    CancellationToken ct)
{
    var queryable = _ctx.BookStats.AsQueryable();

    // Фильтрация
    if (!string.IsNullOrWhiteSpace(query.SearchTerm))
    {
        queryable = queryable.Where(x =>
            EF.Functions.ILike(x.Title, $"%{query.SearchTerm}%") ||
            EF.Functions.ILike(x.Authors, $"%{query.SearchTerm}%") ||
            EF.Functions.ILike(x.Isbn, $"%{query.SearchTerm}%")
        );
    }

    // Подсчёт
    var totalCount = await queryable.CountAsync(ct);

    // Пагинация
    var items = await queryable
        .OrderBy(x => x.Title)
        .Skip((query.Page - 1) * query.PageSize)
        .Take(query.PageSize)
        .Select(x => new BookStatDto(...))
        .ToListAsync(ct);

    return Result.Ok(new PagedResult<BookStatDto>(items, totalCount, query.Page));
}
```

**Разделение:**
- **Write Model** (Commands): `Book`, `Abonent` — нормализованные агрегаты
- **Read Model** (Queries): `BookStat` — денормализованная таблица для быстрых запросов
- **Eventual Consistency**: `BookStat` обновляется асинхронно через Outbox

---

## Работа с базой данных

### Конфигурация Entity Framework

**DbContext** (`ApplicationContext.cs`):
```csharp
public sealed class ApplicationContext : DbContext, IApplicationContext
{
    public DbSet<Abonent> Abonents { get; set; }
    public DbSet<Book> Books { get; set; }
    public DbSet<BookStat> BookStats { get; set; }
    public DbSet<BookStatChange> BookStatChanges { get; set; }
}
```

**Опции подключения:**
```csharp
options
    .UseNpgsql(npgsqlDataSource, builder =>
    {
        builder.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
    })
    .UseSnakeCaseNamingConvention()  // camelCase → snake_case
    .UseExceptionProcessor()  // Обработка PostgreSQL исключений
    .ReplaceService<IValueConverterSelector, ValueObjectsConverterSelectorByConvention>();
```

### Fluent Configuration

**Конфигурация Book** (`BookConfiguration.cs`):
```csharp
public class BookConfiguration : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> builder)
    {
        builder.HasKey(x => x.Id);

        // Индекс для уникальности издания
        builder.HasIndex(x => new { x.Isbn, x.PublicationDate });

        // JSON хранение массива авторов
        builder.Property(x => x.Authors)
            .HasJsonConversion();

        // Owned Entity для BorrowInfo
        builder.OwnsOne(x => x.BorrowInfo, y =>
        {
            y.Property(x => x.AbonentId)
                .HasColumnName("borrowed_by_abonent_id");
            y.Property(x => x.BorrowedAt)
                .HasColumnName("borrowed_at");
            y.Property(x => x.ReturnBefore)
                .HasColumnName("borrowed_return_before");
        });
    }
}
```

**Результат в БД:**
```sql
CREATE TABLE books (
    id UUID PRIMARY KEY,
    title TEXT NOT NULL,
    isbn TEXT NOT NULL,
    publication_date DATE NOT NULL,
    authors JSONB NOT NULL,  -- [{"name": "Robert", "surname": "Martin"}]
    borrowed_by_abonent_id UUID NULL,
    borrowed_at TIMESTAMPTZ NULL,
    borrowed_return_before DATE NULL,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX ix_books_isbn_publication_date
    ON books(isbn, publication_date);
```

### Value Converters

**Пример конвертера** (`IsbnEfValueConverter.cs`):
```csharp
public class IsbnEfValueConverter : ValueConverter<Isbn, string>
{
    public IsbnEfValueConverter()
        : base(
            v => v.Value,           // Isbn → string
            v => new Isbn(v)        // string → Isbn
        )
    {
    }
}
```

### PostgreSQL-специфичные фичи

**Full-text search:**
```csharp
// ILike для case-insensitive поиска
.Where(x => EF.Functions.ILike(x.Title, $"%{searchTerm}%"))
```

**JSONB операции:**
```csharp
// Хранение массива авторов
builder.Property(x => x.Authors).HasJsonConversion();

// В БД: {"name": "...", "surname": "..."}[]
```

**Расширения:**
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- Trigram для поиска
```

### Миграции

**Последняя миграция**: `20241205181929_UpgradeDotnet9`

**Применение:**
```bash
dotnet ef migrations add MigrationName --project BookLibrary.Infrastructure
dotnet ef database update --project BookLibrary.Api
```

---

## Обработка ошибок и доменные исключения

### Система кодов ошибок

**Enum с метаданными** (`ErrorCodes.cs`):
```csharp
[ExceptionConfig(ClassName = "BookLibraryException")]
[ErrorDescription(Prefix = "BL", Level = Level.Medium)]
public enum ErrorCodes
{
    [ErrorDescription(Description = "Book already borrowed", Level = Level.Low)]
    BookAlreadyBorrowed = 13,

    [ErrorDescription(Description = "Too many books borrowed already", Level = Level.NotError)]
    TooManyBooksBorrowedAlready = 27,

    [ErrorDescription(Description = "Abonent not found", Level = Level.Low)]
    AbonentNotFound = 2,

    // ... всего 34 кода
}
```

**Генерируемое исключение** (через Sstv.DomainExceptions):
```csharp
public class BookLibraryException : Exception
{
    public string ErrorCode { get; }  // "BL0013"
    public string ErrorId { get; }     // UUID v7
    public Level Level { get; }

    public BookLibraryException(ErrorCodes code, ...)
    {
        ErrorCode = $"BL{(int)code:D4}";
        ErrorId = UuidV7.Generate().ToString();
        // ...
    }
}
```

### Extension методы

**Преобразование в Result:**
```csharp
public static class ErrorCodesExtensions
{
    public static Result ToDomainError(this ErrorCodes errorCode)
    {
        return Result.Fail(new DomainErrorResult(errorCode));
    }

    public static Result<T> ToDomainError<T>(this ErrorCodes errorCode)
    {
        return Result.Fail<T>(new DomainErrorResult(errorCode));
    }

    public static BookLibraryException ToException(this ErrorCodes errorCode)
    {
        return new BookLibraryException(errorCode);
    }
}
```

**Использование:**
```csharp
// В бизнес-логике (Result)
if (book is null)
    return ErrorCodes.BookNotFound.ToDomainError();

// В валидации агрегата (Exception)
if (authors.Count == 0)
    throw ErrorCodes.BookMustHaveAnAuthors.ToException();
```

### Маппинг на HTTP статусы

**ErrorCodeProblemDetails** (RFC 7807):
```csharp
public class ErrorCodeProblemDetails : ProblemDetails
{
    public ErrorCodeProblemDetails(DomainErrorResult error)
    {
        Status = MapToHttpStatus(error.ErrorCode);
        Title = error.Description;
        Detail = error.ErrorCode;
        Extensions["errorId"] = error.ErrorId;
        Extensions["errorCode"] = error.ErrorCode;
        Extensions["level"] = error.Level.ToString();
    }

    private static int MapToHttpStatus(string errorCode)
    {
        return errorCode switch
        {
            "BL0002" => 404,  // AbonentNotFound
            "BL0001" => 404,  // BookNotFound
            "BL0013" => 400,  // BookAlreadyBorrowed
            "BL0027" => 400,  // TooManyBooksBorrowedAlready
            _ => 500
        };
    }
}
```

**Пример ответа:**
```json
{
  "status": 400,
  "title": "Book already borrowed",
  "detail": "BL0013",
  "errorId": "01936c8e-7e92-7b3a-8c1a-3f2e1d0c9b8a",
  "errorCode": "BL0013",
  "level": "Low"
}
```

---

## Observability и мониторинг

### OpenTelemetry

**Конфигурация** (`ServiceCollectionExtensions.cs`):
```csharp
services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddRuntimeInstrumentation()
        .AddHttpClientInstrumentation()
        .AddAspNetCoreInstrumentation()
        .AddPrometheusExporter()
    )
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddConsoleExporter()
    );
```

**Что инструментируется:**
- 🌐 **HTTP запросы**: latency, status codes, paths
- 💾 **EF Core queries**: SQL, duration, результаты
- 🔧 **.NET Runtime**: GC, ThreadPool, Exceptions
- 📊 **Кастомные метрики**: счётчики создания/выдачи книг

### Кастомные метрики

**Интерфейс** (`IMetricCollector.cs`):
```csharp
public interface IMetricCollector
{
    void BooksCreated(int count);
    void BookBorrowed();
    void BookReturned();
    void AbonentRegistered();
}
```

**Реализация через OpenTelemetry:**
```csharp
public class MetricCollector : IMetricCollector
{
    private readonly Counter<int> _booksCreatedCounter;
    private readonly Counter<int> _booksBorrowedCounter;

    public MetricCollector(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("BookLibrary");
        _booksCreatedCounter = meter.CreateCounter<int>("books.created");
        _booksBorrowedCounter = meter.CreateCounter<int>("books.borrowed");
    }

    public void BooksCreated(int count) => _booksCreatedCounter.Add(count);
    public void BookBorrowed() => _booksBorrowedCounter.Add(1);
}
```

**Prometheus endpoint:** `/metrics`

**Пример метрик:**
```
# TYPE books_created_total counter
books_created_total 1543

# TYPE books_borrowed_total counter
books_borrowed_total 342

# TYPE http_server_request_duration_seconds histogram
http_server_request_duration_seconds_bucket{le="0.005"} 120
```

### Serilog

**Конфигурация** (`Program.cs`):
```csharp
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console(new CompactJsonFormatter())
    .CreateLogger();
```

**Формат логов** (Compact JSON):
```json
{
  "@t": "2025-01-15T10:30:00.123Z",
  "@mt": "Book {Title} created with ISBN {ISBN}",
  "Title": "Clean Architecture",
  "ISBN": "978-0134494166",
  "BookId": "01936c8e-...",
  "SourceContext": "BookLibrary.Application.Features.Books.AddNewBook"
}
```

---

## Тестирование архитектуры

### ArchUnitNET тесты

**Структура:**
```
BookLibrary.ArchTests/
├── Domain/
│   ├── DomainDepsTests.cs        # Зависимости Domain слоя
│   ├── EntitiesTests.cs          # Правила для Entity
│   ├── ValueObjectTests.cs       # Правила для ValueObject
│   ├── ErrorCodeTests.cs         # Валидация ErrorCodes
│   └── DomainEventTests.cs       # Правила для событий
├── Application/
│   ├── ApplicationDepsTests.cs   # Зависимости Application
│   └── UseCaseRules.cs           # Правила для use-cases
└── ProjectAssemblies.cs          # Определения assembly
```

**Примеры правил:**

**Entities должны наследоваться от Entity:**
```csharp
[Test]
public void Entities_Should_Inherit_From_BaseEntity()
{
    Types().That()
        .AreAssignableTo(typeof(IEntity))
        .And().AreNot(typeof(Entity))
        .Should()
        .BeAssignableTo(typeof(Entity))
        .Check(ProjectAssemblies.Architecture);
}
```

**Value Objects должны быть immutable:**
```csharp
[Test]
public void ValueObjects_Should_Be_Immutable()
{
    Types().That()
        .AreAssignableTo(typeof(ValueObject))
        .Should()
        .HavePropertySettersWithPrivateAccess()
        .Check(ProjectAssemblies.Architecture);
}
```

**Use-cases должны заканчиваться на "UseCase":**
```csharp
[Test]
public void UseCases_Should_Have_UseCase_Suffix()
{
    Types().That()
        .ImplementInterface(typeof(IRequestHandler<,>))
        .Should()
        .HaveNameEndingWith("UseCase")
        .Check(ProjectAssemblies.Architecture);
}
```

---

## Выводы

### Сильные стороны архитектуры

1. **🏗️ Чистая архитектура**
   - Чёткое разделение слоёв с автоматическими тестами
   - Domain не зависит от инфраструктуры
   - Легко тестировать и менять реализации

2. **📦 Vertical Slices**
   - Высокая связность внутри фичи
   - Низкая связанность между фичами
   - Простота навигации и поддержки

3. **🎯 Типобезопасность**
   - Strongly Typed IDs исключают ошибки
   - Value Objects инкапсулируют валидацию
   - Railway-Oriented Programming вместо исключений

4. **⚡ Производительность**
   - Source-generated Mediator без рефлексии
   - Query Splitting избегает Cartesian explosion
   - Outbox Pattern с batch обработкой

5. **🔍 Observability**
   - OpenTelemetry для трейсинга и метрик
   - Структурированное логирование с Serilog
   - Централизованные константы для логов

6. **🧪 Тестируемость**
   - ArchUnitNET защищает от деградации архитектуры
   - Абстракции позволяют мокировать зависимости
   - Domain-логику можно тестировать без БД

### Области для улучшения

1. **📊 CQRS можно развить**
   - Добавить отдельную read-only БД (например, Elasticsearch)
   - Использовать projections для сложной аналитики

2. **🔐 Аутентификация/авторизация**
   - Сейчас используется mock
   - Можно добавить OAuth2/OIDC

3. **📝 История операций**
   - TODO в коде: история выдач книг (Book.cs:162)
   - Event Sourcing для полного аудита

4. **🚀 Кэширование**
   - Добавить Redis для кэша read-моделей
   - Invalidation через доменные события

### Рекомендации для изучения

Этот проект отлично подходит для изучения:
- ✅ Clean Architecture на практике
- ✅ Domain-Driven Design с реальными примерами
- ✅ CQRS и Event-Driven Architecture
- ✅ Railway-Oriented Programming
- ✅ Source Generators в .NET
- ✅ OpenTelemetry и observability
- ✅ ArchUnitNET для проверки архитектуры

---

**Дата анализа:** 2025-01-15
**Версия проекта:** .NET 9.0
**Последний коммит:** 84d581c - "chore: update deps"
