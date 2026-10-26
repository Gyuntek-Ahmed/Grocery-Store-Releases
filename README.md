# GroceryStore

Desktop POS + складова система за хранителен магазин с един обект и два компютъра
(каса и склад), работещи в локална мрежа срещу общ SQL Server. Приложението работи
офлайн — интернет не е критичен.

## Възможности

- **Каса (POS):** сканиране на баркод, количка, плащане в брой или с карта, ресто,
  паркиране на продажба, сторно/връщане (цяло и частично), служебно изваждане на пари,
  кредити (продажба на изчакване), бързи бутони по категория.
- **Склад:** приемане на стока чрез USB баркод скенер, нов артикул при непознат баркод,
  следене на наличности, предупреждение и справка за ниски наличности, брак, поръчки.
- **Администрация (Manager):** CRUD на артикули, категории, доставчици, потребители;
  справки (продажби по ден/период, приходи по начин на плащане, печалба, наличности),
  експорт в Excel, преглед на одит лог.
- **Табло (Dashboard):** дневен/месечен оборот, печалба, графики.
- **Смени:** отваряне/затваряне на касова смяна с отчет и разлика.
- **Настройки:** език (на живо), връзка с БД, период за пазене на логове, backup.

## Технологичен стек

| Слой | Технологии |
|------|-----------|
| UI | WPF (`net10.0-windows`), CommunityToolkit.Mvvm, MaterialDesignThemes |
| Backend | .NET 10, C# |
| ORM | Entity Framework Core 10 (SQL Server, code-first) |
| DI / Hosting | Microsoft.Extensions.DependencyInjection + Hosting |
| Логове | Serilog (+ Sinks.File) |
| Сигурност | BCrypt.Net-Next (хеширане на пароли) |
| Експорт | ClosedXML (.xlsx) |
| Инсталатор | WiX Toolset → .msi |

## Архитектура

Clean / Onion архитектура. Посока на зависимостите: **Domain ← Application ← Infrastructure ← WPF**.

- **GroceryStore.Domain** — entities, enums, value objects, изключения, репозиторни
  интерфейси (`Interfaces/Repositories/`), `IUnitOfWork` и `IUnitOfWorkFactory`.
- **GroceryStore.Application** — use case-ове (`Features/<Област>/<Действие>UseCase.cs`),
  DTO-та, сервизни интерфейси и `Result` тип. Без MediatR — ViewModel-ите викат use case-овете директно.
- **GroceryStore.Infrastructure** — EF Core (`AppDbContext`, конфигурации, миграции),
  репозитории, `UnitOfWork`, INI настройки, услуги (auth, експорт, backup, update, scale).
- **GroceryStore.WPF** — Views/ViewModels (MVVM), локализация, конвертори, стартиране.

### Ключови конвенции

- **Бизнес грешките се връщат, не се хвърлят** — чрез `Result` / `Result<T>`
  (`Application/Common/Result.cs`). Викащият проверява `IsSuccess` / `Error`.
- **Unit of Work за всяка операция:** use case-овете получават свеж, кратко-живеещ
  `DbContext` чрез `IUnitOfWorkFactory.Create()` (с `using`). Така един context не се
  споделя между екрани, не изтича памет и няма едновременни операции върху него.
- **Optimistic concurrency:** `Product.RowVersion` пази наличността от едновременни
  промени (каса + склад). При конфликт операцията се проваля приятелски, а не разваля данните.
- **Регистрация на услуги (ръчно, без скан):** use case-ове → `ApplicationDependencyInjection`,
  инфраструктура → `Infrastructure/DependencyInjection`, ViewModels/Views → `WPF/Extensions/ServiceCollectionExtensions`.

## Изисквания

- .NET 10 SDK
- SQL Server (стои на касовия компютър; складовият се свързва по LAN)
- Windows (заради WPF)

## Билд и стартиране

```powershell
# Билд
dotnet build GroceryStore.slnx

# Стартиране на приложението
dotnet run --project GroceryStore.WPF
```

Решението ползва новия `.slnx` формат (нужен е скорошен .NET 10 SDK / VS 2026).

## База данни и миграции

DbContext-ът е в `GroceryStore.Infrastructure`; startup проектът е `GroceryStore.WPF`.
**Миграциите се прилагат автоматично при стартиране** (`App` → `MigrateAsync`) и базата
се seed-ва (вкл. създаване на началния Manager акаунт). Ръчно се налага само добавяне на миграция:

```powershell
dotnet ef migrations add <Име> --project GroceryStore.Infrastructure --startup-project GroceryStore.WPF
```

## Конфигурация

Настройките се четат от INI файл (не от `appsettings.json` — той е само за Serilog):

```
C:\ProgramData\GroceryStore\GroceryStore.ini   ← настройки (мени се и от UI)
C:\ProgramData\GroceryStore\logs\              ← Serilog логове (нов файл всеки ден)
```

Connection string-ът се сглобява от секцията `[Database]`. Двата компютъра имат еднакъв
INI файл, различен само по `Server=`. При първо стартиране или липса на връзка приложението
отваря екрана за настройки преди да продължи.

## Роли и достъп

| Роля | Достъп |
|------|--------|
| **Manager** | Всичко — модули, справки, администрация, настройки |
| **Cashier** | Само каса |
| **StockClerk** | Само склад |
| **CashierAndStock** | Каса и склад |

`StationRole` (`CashierStation` / `StockStation`) определя началния модул на станцията.

При първо стартиране се създава акаунт **`admin` / `admin123`** — сменете паролата веднага.

## Локализация

Български (`bg-BG`, по подразбиране) и турски (`tr-TR`). Смяната на език е на живо, без рестарт.
В XAML: `{loc:Localize Ключ}`; ресурси в `GroceryStore.WPF/Resources/Strings*.resx`.

## Документна номерация

| Тип | Формат |
|-----|--------|
| Продажба | `S-000001` |
| Приемане на стока | `R-000001` |
| Сторно | `RT-000001` |
