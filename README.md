# Лабораторная работа №31-32: Entity Framework Core + SQLite

**Студент:** Лепилкин Максим Александрович
**Группа:** ИСП-232 
**Дата:** 04.05.26

## 1. Краткое описание работы

В ходе работы разработано веб-API на ASP.NET Core с использованием Entity Framework Core (Code First) и базы данных SQLite. Реализованы CRUD-операции над задачами, а также расширенные возможности: поиск с фильтрацией, статистика, пагинация, управление сроками выполнения (DueDate), массовое обновление и удаление. Миграции EF Core позволяют версионировать схему базы данных и синхронизировать её с моделями C#.

## 2. Полезные команды dotnet ef

| Команда | Описание |
|---------|----------|
| `dotnet ef migrations add <Name>` | Создать новую миграцию на основе изменений в моделях |
| `dotnet ef database update` | Применить все неприменённые миграции к БД |
| `dotnet ef database update <MigrationName>` | Откатить/применить до указанной миграции |
| `dotnet ef migrations list` | Показать список миграций и их статус |
| `dotnet ef migrations remove` | Удалить последнюю миграцию (если она не применена) |
| `dotnet ef migrations script` | Сгенерировать SQL-скрипт без применения к БД |

## 3. Структура проекта
```
Lab31-32_EFCore/
├── TaskDb/
│ ├── Controllers/
│ │ └── TasksController.cs
│ ├── Data/
│ │ └── AppDbContext.cs
│ ├── Models/
│ │ ├── TaskItem.cs
│ │ └── TaskDtos.cs
│ ├── Migrations/
│ │ ├── 202..._InitialCreate.cs
│ │ ├── 202..._AddDueDateToTask.cs
│ │ └── AppDbContextModelSnapshot.cs
│ ├── appsettings.json
│ ├── appsettings.Development.json
│ ├── Program.cs
│ └── taskdb.db
├── img/
│ └── (скриншоты выполнения)
├── .editorconfig
└── README.md
```

## 4. Список реализованных маршрутов API

| HTTP метод | Маршрут | Описание |
|------------|---------|-----------|
| GET | `/api/tasks` | Получить все задачи (с фильтрацией по `completed`, `priority`) |
| GET | `/api/tasks/{id}` | Получить задачу по ID |
| POST | `/api/tasks` | Создать новую задачу |
| PUT | `/api/tasks/{id}` | Полностью обновить задачу |
| PATCH | `/api/tasks/{id}/complete` | Переключить статус выполнения |
| DELETE | `/api/tasks/{id}` | Удалить задачу по ID |
| GET | `/api/tasks/search?query=&priority=&completed=` | Поиск по тексту, приоритету и статусу |
| GET | `/api/tasks/stats` | Получить статистику (всего, выполнено, % по приоритетам и др.) |
| GET | `/api/tasks/paged?page=1&pageSize=10` | Пагинация (по умолчанию pageSize=10, максимум 50) |
| GET | `/api/tasks/overdue` | Список просроченных задач (DueDate < сегодня и не выполнены) |
| PATCH | `/api/tasks/complete-all` | Отметить все невыполненные задачи как выполненные |
| DELETE | `/api/tasks/completed` | Удалить все выполненные задачи |

## 5. Таблица применённых миграций

| Название миграции | Добавленные изменения |
|-------------------|------------------------|
| `InitialCreate` | Создание таблицы `Tasks` с полями: Id, Title, Description, Priority, IsCompleted, CreatedAt + начальные данные (3 задачи) |
| `AddDueDateToTask` | Добавление поля `DueDate` (DateTime?, nullable) в таблицу `Tasks` |

Проверить список можно командой:
```bash
dotnet ef migrations list
```
## 6. Сравнительная таблица LINQ → SQL
|LINQ (method syntax)|	Генерируемый SQL (SQLite)|
|---|---|
|_db.Tasks.Where(t => t.IsCompleted == false)	|SELECT * FROM "Tasks" WHERE "IsCompleted" = 0|
|_db.Tasks.OrderBy(t => t.CreatedAt)	|ORDER BY "CreatedAt" ASC|
|_db.Tasks.OrderByDescending(t => t.CreatedAt)|	ORDER BY "CreatedAt" DESC|
|_db.Tasks.Skip(20).Take(10)	|LIMIT 10 OFFSET 20|
|_db.Tasks.CountAsync()	|SELECT COUNT(*) FROM "Tasks"|
|_db.Tasks.CountAsync(t => t.Priority == "High")	|SELECT COUNT(*) FROM "Tasks" WHERE "Priority" = 'High'|
|_db.Tasks.GroupBy(t => t.Priority).Select(g => new { g.Key, Count = g.Count() })	|SELECT "Priority" AS "Key", COUNT(*) AS "Count" FROM "Tasks" GROUP BY "Priority"|

## 7. Итоговая сравнительная таблица (память vs EF Core + SQLite)
| Концепция | Хранение в памяти (`static List<T>`) | EF Core + SQLite |
|-----------|--------------------------------------|------------------|
| **Хранение данных** | `static List<T>` в RAM | Файл `.db` на диске |
| **После перезапуска** | Данные пропадают | Данные сохраняются |
| **Поиск по условию** | LINQ to Objects | LINQ to Entities → SQL |
| **Создание структуры** | Не нужно | Миграции (`dotnet ef`) |
| **Начальные данные** | Хардкод в коде | `HasData()` в миграции |
| **Получение данных** | `list.FirstOrDefault(...)` | `await db.Table.FindAsync(id)` |
| **Добавление** | `list.Add(item)` | `db.Table.Add(item)` + `SaveChangesAsync()` |
| **Удаление** | `list.Remove(item)` | `db.Table.Remove(item)` + `SaveChangesAsync()` |
| **Масштабируемость** | Ограничена RAM | Гигабайты данных |
| **Транзакции** | Нет | Встроены в EF Core |

## 8. Главные выводы

- **EF Core автоматически переводит LINQ-запросы в SQL** – разработчик работает с объектами `C#`, а не с текстом `SQL`, что снижает вероятность ошибок и ускоряет написание кода.
- **Миграции – это «Git для схемы базы данных»** – изменение модели `C#` порождает миграцию, которую можно применить, откатить или расшарить через репозиторий.
- **Code First удобен при разработке с нуля** – сначала пишем классы, затем через миграции создаём БД, а не наоборот.
- **Асинхронные методы (`async`/`await`) обязательны** – они не блокируют поток сервера во время ожидания ответа от БД, что повышает пропускную способность приложения.
- **Пагинация (`Skip`/`Take`) и агрегации (`CountAsync`, `GroupBy`) выполняются на стороне БД** – это экономит память и ускоряет работу, так как в приложение загружаются только нужные данные.