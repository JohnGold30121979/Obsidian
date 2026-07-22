[[GetNextDocumentNumberAsync]]  - Увеличение счетчика номера документа

# MetadataService — полный обзор

## Общая структура

`MetadataService` — это центральный сервис для управления метаданными системы. Реализован как `partial class`, разделенный на 7 файлов:

```
MetadataService.cs               (основной — инициализация, CRUD, SQL-генерация)
MetadataService.Catalogs.cs      (создание/обслуживание всех предустановленных справочников)
MetadataService.DataSeed.cs      (начальные данные для справочников)
MetadataService.Fields.cs        (генерация полей для всех справочников и документов)
MetadataService.Finance.cs       (финансовые документы: кассовые ордера, платежки, проводки)
MetadataService.FinanceDocuments.cs (инвойсы, счет-фактуры, ESF)
MetadataService.Reports.cs       (стандартные отчеты, печатные формы)
```

## 1. Основной файл — `MetadataService.cs`

### Конструктор
```csharp
public MetadataService(AppDbContext context)
```
Принимает контекст БД.

### Константы
- `CashOrderDocumentName` = "Расходный/Приходный КО"
- `CashOrderReceiptKind/PaymentKind` — виды кассовых ордеров
- `IndependentDocumentNumberingNames` — независимая нумерация для КО

### Основные методы

#### Инициализация
| Метод | Описание |
|-------|----------|
| `InitializeDefaultMetadataAsync(infoBaseId)` | Создает документы: Приход товаров, Расход товаров, Проводки, Кассовые ордера, Платежные поручения |
| `InitializePredefinedCatalogsAsync(infoBaseId)` | Создает все предустановленные справочники (см. ниже) |

#### Работа с данными
| Метод | Описание |
|-------|----------|
| `GetCatalogDataAsync(catalogId)` | Читает данные динамической таблицы, возвращает `List<Dictionary<string, object>>` |
| `AddCatalogItemAsync(catalogId, itemData)` | INSERT записи в динамическую таблицу |
| `SaveCatalogAsync(catalog)` | UPDATE метаданных справочника |
| `DeleteCatalogAsync(catalogId)` | DROP TABLE + удаление метаданных |
| `CreateCatalogAsync(name, desc, icon, fields)` | Создание нового справочника |

#### Работа с записями
| Метод | Описание |
|-------|----------|
| `CreateDynamicRecordAsync(metadataId, data)` | Создание записи (документ/справочник) |
| `UpdateDynamicRecordAsync(metadataId, recordId, data)` | UPDATE записи |
| `DeleteDynamicRecordAsync(metadataId, recordId)` | DELETE записи |

#### SQL-генерация
- `BuildCatalogDataSelectSqlAsync` — SELECT с ORDER BY по колонкам
- `GetSqlTypeForField` — маппинг типа поля на PostgreSQL (VARCHAR, INTEGER, DECIMAL, TIMESTAMP, BOOLEAN, TEXT)
- `FormatSqlValue` — преобразование .NET-типов в SQL-литералы
- `CreateTableForCatalogAsync` — CREATE TABLE
- `UpdateTableStructureAsync` — DROP + CREATE
- `AddColumnToTableAsync` — ALTER TABLE ADD COLUMN
- `Transliterate` — кириллица → латиница для имён колонок

#### Системные
- `EnsureGlobalDocumentNumberConfigurationAsync` — настройка сквозной нумерации
- `NormalizeDocumentTableNumbersAsync` — синхронизация номеров
- `NormalizeLegacyDocumentNumber` — удаление префиксов/суффиксов
- `RemoveDuplicateMetadataFieldsAsync` — чистка дубликатов полей
- `GetModulesAsync` — получение модулей разделов
- `GetAssignedModuleNameAsync` — определение модуля для объекта

---

## 2. `MetadataService.Catalogs.cs` — Справочники

Содержит методы создания и обеспечения структуры **20+ предустановленных справочников**:

| Справочник | Таблица | Метод создания |
|------------|---------|----------------|
| Организации | `catalog_organizations` | `CreateOrganizationsCatalog` |
| Сотрудники (Списочный состав) | `catalog_employees` | `CreateEmployeesCatalog` |
| Основные средства | `catalog_assets` | `CreateAssetsCatalog` |
| Государства | `catalog_countries` | `CreateCountriesCatalog` |
| Участки | `catalog_sites` | `CreateSitesCatalog` |
| МОЛ | `catalog_responsible_persons` | `CreateResponsiblePersonsCatalog` |
| Наименования категорий | `catalog_material_categories` | `CreateMaterialCategoriesCatalog` |
| Виды материалов | `catalog_material_types` | `CreateMaterialTypesCatalog` |
| Справочник материалов | `catalog_materials` | `CreateMaterialCatalog` |
| План счетов | `catalog_plan_schetov_...` | `CreateChartOfAccountsCatalog` |
| Банки | `catalog_banks` | `CreateBanksCatalog` |
| Справочник валют | `catalog_currencies` | `CreateCurrencyCatalog` |
| Курсы валют | `catalog_currency_rates` | `CreateCurrencyRatesCatalog` |
| Расчетные счета организаций | `catalog_bank_accounts` | `CreateBankAccountsCatalog` |
| Кассы | `catalog_cash_desks` | `CreateCashDesksCatalog` |
| Налоги | `catalog_taxes` | `CreateTaxCatalog` |
| Подразделения | `catalog_divisions` | `CreateDivisionCatalog` |
| Должности | `catalog_positions` | `CreatePositionsCatalog` |
| Виды поставки | `catalog_supply_kinds` | `CreateSupplyKindCatalog` |
| Виды оплаты | `catalog_payment_kinds` | `CreatePaymentKindCatalog` |
| Классификация платежей | `catalog_payment_classification` | `CreatePaymentClassificationCatalog` |
| Типы поставки | `catalog_delivery_types` | `CreateDeliveryTypeCatalog` |
| Авансовые платежи | `catalog_advance_payments` | `CreateAdvancePaymentsCatalog` |
| Настройка доступа к счетам | `catalog_account_access` | `CreateAccountAccessCatalog` |
| Расчет курсовой разницы | `catalog_exchange_rate_diff` | `CreateExchangeRateDiffCatalog` |
| Настройка доступа к счетам (аналитика) | `catalog_account_analytics_links` | `CreateAccountAnalyticsLinksCatalog` |

**Шаблон создания:**
1. Создать `MetadataObject` с Id, Name, TableName, ObjectType, Fields
2. `_context.MetadataObjects.AddAsync`
3. `CreateTableForCatalogAsync` (DDL)
4. Заполнить начальными данными (`AddXxxDataToTable`)

**Методы `EnsureXxxStructureAsync`** — проверяют существование/обновляют структуру при каждом запуске.

**Дополнительно:**
- `GetCurrencyRateForDateAsync` — получение курса валюты на дату
- `EnsureManagedDocumentAnalyticFieldsAsync` — аналитические поля для кассовых ордеров и платежек

---

## 3. `MetadataService.DataSeed.cs` — Начальные данные

Наполнение справочников данными:

| Метод | Что записывает |
|-------|----------------|
| `AddCurrencyDataToTable` | KGS, USD, EUR, KZT, RUB |
| `AddMaterialCategoriesDataToTable` | 6 категорий материалов |
| `AddChartOfAccountsDataToTable` | План счетов КР (из `InitialDataProvider.GetChartOfAccounts()`) |
| `EnsureCashOrderPostingAccountsAsync` | Счета 4010, 6010, 6810, 6850 |
| `AddBanksDataToTable` | Банки КР (из `InitialDataProvider.GetBanks()`) |
| `AddMaterialTypesDataToTable` | 14 видов материалов |
| `AddCountriesDataToTable` | 195 государств |
| `AddTaxDataToTable` | 8 видов налогов с кодами ЭСФ |
| `AddSupplyKindDataToTable` | Поставка товаров, работ/услуг, прочая |
| `AddPaymentKindDataToTable` | Наличные, карта, безналичный, чек |
| `AddDeliveryTypeDataToTable` | Облагаемая, необлагаемая, импорт, экспорт |
| `AddSiteDataToTable` | 5 участков |
| `AddPrimaryOrganizationDataToTable` | "Основное предприятие" |
| `AddPositionDataToTable` | Директор, бухгалтер, экономист |
| `UpsertCatalogSeedRowAsync` | Универсальный UPSERT по коду |

---

## 4. `MetadataService.Fields.cs` — Определения полей

Содержит методы `GetXxxFields()` для генерации `List<MetadataField>` для каждого справочника/документа. Основные:

| Метод | Для кого |
|-------|----------|
| `GetStandardDocumentFields` | Документы (Приход/Расход товаров) |
| `GetPostingFields` | Журнал проводок |
| `GetCashOrderFields` | Кассовые ордера |
| `GetPaymentOrderFields` | Платежные поручения |
| `GetEmployeeCatalogFields` | Сотрудники |
| `GetOrganizationFields` | Организации |
| `GetFixedAssetFields` | Основные средства |
| `GetChartOfAccountsFields` | План счетов |
| `GetChartOfAccountsAnalyticFields` | Аналитические флаги Плана счетов |
| `GetCurrencyFields` | Валюты |
| `GetCurrencyRateFields` | Курсы валют |
| `GetBankFields` | Банки |
| `GetBankAccountFields` | Расчетные счета |
| `GetCashDeskFields` | Кассы |
| `GetMaterialFields` | Материалы |
| `GetCountryFields` | Государства |
| `GetTaxFields` | Налоги |
| `GetAccountAnalyticsLinkFields` | Связи аналитики |
| `GetDocumentAnalyticReferenceFields` | Ссылочные поля документов |
| `GetInvoiceFieldDefinitions` | Поля счет-фактур |

---

## 5. `MetadataService.Finance.cs` — Финансовые документы

- Создание и обеспечение структуры кассовых ордеров (`EnsureUnifiedCashOrderDocumentAsync`, `EnsureCashOrderDocumentStructureAsync`)
- Методы для инвойсов и счет-фактур
- Работа с нумерацией и проведением документов

---

## 6. `MetadataService.FinanceDocuments.cs` — Счет-фактуры

- `EnsureInvoiceDocumentStructureAsync` — структура полей счет-фактур
- `EnsureEsfExchangeDocumentStructureAsync` — структура для ЭСФ обмена
- Методы для работы с инвойсами (SalesIssue, PurchaseRegistration)

---

## 7. `MetadataService.Reports.cs` — Отчеты

- `EnsureStandardReportsAsync` — обеспечение стандартных отчетов
- `EnsureStandardReportTemplatesAsync` — шаблоны отчетов
- Стандартные FRX-шаблоны для отчетов

---

## Схема взаимодействия

```
ConfiguratorWindow / другие UI
    │
    ▼
MetadataService (partial class)
    │
    ├── main: CRUD, SQL, нумерация, инициализация
    ├── Catalogs: создание 26 справочников
    ├── Fields: 20+ генераторов полей
    ├── DataSeed: наполнение данными
    ├── Finance: КО, платежки, проводки
    ├── FinanceDocuments: счета-фактуры, ЭСФ
    └── Reports: стандартные отчеты
    │
    ▼
AppDbContext (Entity Framework)
    │
    ▼
PostgreSQL (через Npgsql)
```

## Ключевые особенности

- **Динамические таблицы**: каждый справочник/документ имеет свою таблицу, создаваемую через DDL
- **Транслитерация**: имена колонок генерируются из русских названий полей
- **UPSERT логика**: `UpsertCatalogSeedRowAsync` — UPDATE или INSERT с проверкой по `code`
- **Цепочка инициализации**: `GetCatalogsAsync()` → 12 Ensure* методов, `GetDocumentsAsync()` → 6 Ensure* методов
- **Проводки**: поддержка правил проводок в документах через `MetadataPostingRule`
- **Аналитика**: связь флагов плана счетов с полями документов через `AccountAnalyticsLinks`
- **Нумерация**: глобальная сквозная нумерация документов, независимая для кассовых ордеров