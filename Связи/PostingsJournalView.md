# PostingsJournalView — детальное описание структуры

## Назначение

`PostingsJournalView` — это `UserControl` для просмотра и анализа бухгалтерских проводок. Предоставляет интерфейс для фильтрации, поиска и навигации по проводкам из двух источников: таблицы `doc_postings` и импортированных `DynamicDocument`.

## Архитектура UI

```
┌─────────────────────────────────────────────────────────────┐
│ Заголовок: "📋 Журнал проводок"                              │
│ Подзаголовок: "Просмотр и анализ бухгалтерских проводок"     │
├─────────────────────────────────────────────────────────────┤
│ Панель фильтров (Row 1)                                      │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Период: [__.__.____] — [__.__.____]                     │ │
│ │ Быстрый поиск: [______________]                         │ │
│ │ [🔍 Обновить] [📊 Обороты по счету] [📈 Сводные обороты]│ │
│ ├─────────────────────────────────────────────────────────┤ │
│ │ [Счет Дебет] [Счет Кредит] [Организация] [Сотрудник]    │ │
│ │ [№ документа] [✖ Сбросить]                              │ │
│ └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ DataGrid: PostingsGrid (Row 2)                               │
│ ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐  │
│ │Дата  │Докум.│Тип   │Модуль│Дебет │Кредит│Сумма │...  │  │
│ ├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤  │
│ │...   │...   │...   │...   │...   │...   │...   │...   │  │
│ └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘  │
├─────────────────────────────────────────────────────────────┤
│ StatusBar (Row 3)                                            │
│ Статус | Общая сумма: X сом | Подсказка                     │
└─────────────────────────────────────────────────────────────┘
```

## Компоненты

### 1. Заголовок (Row 0)
- `Border` с белым фоном
- `TextBlock` "📋 Журнал проводок" (FontSize 18, Bold)
- `TextBlock` "Просмотр и анализ бухгалтерских проводок" (FontSize 12, Gray)

### 2. Панель фильтров (Row 1)
Состоит из двух строк:

**Строка 0 — Основные фильтры:**
- **Период** (Grid.Column 0):
  - `DatePicker` `dpStartDate` — начало периода (по умолчанию: 1-е число текущего месяца)
  - `DatePicker` `dpEndDate` — конец периода (по умолчанию: сегодня)
- **Быстрый поиск** (Grid.Column 1):
  - `TextBox` `txtQuickSearch` — поиск по всем полям (счетам, номеру, модулю, примечанию, организации, сотруднику)
  - Событие: `TextChanged` → `ApplyFilters()`
- **Кнопки** (Grid.Column 3):
  - `🔍 Обновить` — `OnRefreshClick` → перезагрузка данных из `PostingService`
  - `📊 Обороты по счету` — `OnTurnoversClick` (TODO: не реализовано)
  - `📈 Сводные обороты` — `OnSummaryClick` (TODO: не реализовано)

**Строка 1 — Детальные фильтры:**
- `Border` с фоном `#F8F9FA`
- `WrapPanel` с текстовыми полями:
  - `txtDebetFilter` — фильтр по счету дебета
  - `txtCreditFilter` — фильтр по счету кредита
  - `txtOrganizationFilter` — фильтр по организации
  - `txtEmployeeFilter` — фильтр по сотруднику
  - `txtDocumentFilter` — фильтр по номеру документа
  - `✖ Сбросить` — `OnClearFiltersClick` → очистка всех фильтров

### 3. DataGrid: PostingsGrid (Row 2)
- **Источник данных**: `ObservableCollection<PostingViewModel> _filteredPostings`
- **Виртуализация**: `VirtualizingPanel.IsVirtualizing="True"`, `VirtualizationMode="Recycling"`
- **События**:
  - `SelectionChanged` → `PostingsGrid_SelectionChanged` (обновляет статус)
  - `MouseDoubleClick` → `PostingsGrid_MouseDoubleClick` (открывает детали)

**Колонки:**
| Колонка | Binding | Ширина | Стиль |
|---------|---------|--------|-------|
| Дата | `{Date, StringFormat=dd.MM.yyyy}` | 100 | — |
| Документ | `{DocumentNumber}` | 120 | — |
| Тип | `{DocumentType}` | 180 | — |
| Модуль | `{ModuleName}` | 140 | — |
| Дебет | `{DebitAccount}` | 100 | Синий `#2980B9`, Bold |
| Кредит | `{CreditAccount}` | 100 | Красный `#C0392B`, Bold |
| Сумма в сомах | `{Amount, StringFormat=N2}` | 120 | Выравнивание вправо |
| Сумма в валюте | `{AmountCurrency, StringFormat=N2}` | 120 | Выравнивание вправо |
| Валюта | `{Currency}` | 80 | — |
| Организация | `{Organization}` | 150 | — |
| Сотрудник | `{Employee}` | 120 | — |
| Примечание | `{Note}` | * (MinWidth 150) | — |

### 4. StatusBar (Row 3)
- `StatusText` — текущий статус ("Готов", "Загрузка...", "Загружено N проводок")
- `TotalInfo` — общая сумма отфильтрованных проводок
- Подсказка: "двойной щелчок открывает счет-фактуру или подробности проводки"

## Логика работы (code-behind)

### Инициализация
```csharp
public PostingsJournalView(PostingService postingService)
```
- Принимает `PostingService` через конструктор
- Устанавливает период: с 1-го числа текущего месяца по сегодня
- При загрузке (`Loaded`) вызывает `LoadPostingsAsync()`

### Загрузка данных
```csharp
private async Task LoadPostingsAsync()
```
1. Получает `startDate` / `endDate` из DatePicker'ов
2. Вызывает `_postingService.GetAllPostingsAsync(startDate, endDate)`
3. Заполняет `_allPostings` (полный список)
4. Вызывает `ApplyFilters()` для применения фильтров
5. Обновляет статус и общую сумму

### Фильтрация
```csharp
private void ApplyFilters()
```
- Последовательно применяет фильтры к `_allPostings`:
  - `txtDebetFilter` — `DebitAccount.Contains()`
  - `txtCreditFilter` — `CreditAccount.Contains()`
  - `txtOrganizationFilter` — `Organization.Contains()`
  - `txtEmployeeFilter` — `Employee.Contains()`
  - `txtDocumentFilter` — `DocumentNumber.Contains()`
  - `txtQuickSearch` — поиск по всем полям (ToLower)
- Результат помещается в `_filteredPostings`
- Обновляется `PostingsGrid.ItemsSource` и `TotalInfo`

### Обработка двойного клика
```csharp
private async void PostingsGrid_MouseDoubleClick(object sender, MouseButtonEventArgs e)
```
1. Пытается открыть исходный документ через `PostingSourceDocumentOpener.TryOpenAsync`
2. Если не удалось — открывает `PostingDetailsDialog` с деталями проводки

### Открытие счет-фактуры
```csharp
private async Task<bool> TryOpenInvoiceFromPostingAsync(PostingViewModel posting)
```
- Проверяет, является ли тип документа счет-фактурой
- Ищет метаданные документа
- Через `InvoiceService.FindInvoiceIdByPostingNumberAsync` находит ID
- Открывает `InvoiceEditDialog` в режиме чтения

## Взаимодействие с другими модулями

```
PostingsJournalView
    │
    ├── PostingService
    │       ├── GetAllPostingsAsync(startDate, endDate)
    │       │       ├── doc_postings (SQL)
    │       │       └── DynamicDocument (JSONB)
    │       ├── GetPostingsByDocumentAsync()
    │       └── GetTurnoversByAccountAsync()
    │
    ├── PostingSourceDocumentOpener.TryOpenAsync()
    │       └── Открытие исходного документа (кассовый ордер, платежка и т.д.)
    │
    ├── PostingDetailsDialog
    │       └── Детальный просмотр проводки
    │
    └── InvoiceEditDialog (через TryOpenInvoiceFromPostingAsync)
            └── Просмотр счет-фактуры в режиме read-only
```

## Модель данных

`PostingViewModel` содержит:
- `Id`, `Date`, `DocumentNumber`, `DocumentType`
- `ModuleCode`, `ModuleName`
- `DebitAccount`, `DebitAccountName`
- `CreditAccount`, `CreditAccountName`
- `CorrespondentAccount`, `Direction`
- `Amount`, `AmountCurrency`, `Currency`
- `Note`, `Organization`, `Employee`
- `Site`, `ResponsiblePerson`
- `DocumentId`, `CreatedAt`, `IsActive`
- `DetailHint` — строка для статус-бара

## Особенности

- **Два источника данных**: проводки из `doc_postings` (SQL) и из `DynamicDocument` (JSONB)
- **Виртуализация**: DataGrid использует `VirtualizingPanel.Recycling` для производительности
- **Фильтрация на стороне клиента**: все данные загружаются, фильтры применяются в памяти
- **TODO**: кнопки "Обороты по счету" и "Сводные обороты" пока не реализованы
- **Навигация**: двойной клик пытается открыть исходный документ, если не удается — показывает детали проводки