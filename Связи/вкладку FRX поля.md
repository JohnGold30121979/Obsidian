# Вкладка «📊 Поля» в ReportDesignerWindow — детальное описание

## Назначение

Вкладка **Поля** предназначена для управления составом полей отчета: выбор полей из источника данных, настройка их отображения, агрегации, порядка и видимости.

## Структура UI

Вкладка разделена на две колонки:

```
┌──────────── 280px ────────────┬──────────── * ──────────────┐
│  📋 Доступные поля            │  📋 Поля отчета               │
│                               │                              │
│  ┌─────────────────────────┐  │  ┌────┬────┬────┬───┬───┐   │
│  │ 📄 Поле1 (String)       │  │  │Имя │Отоб│Агр.│Шир│Вид│   │
│  │ 📄 Поле2 (Decimal)      │  │  ├────┼────┼────┼───┼───┤   │
│  │ 📄 Поле3 (DateTime)     │  │  │    │    │    │   │   │   │
│  │ 📄 Поле4 (Reference)    │  │  │    │    │    │   │   │   │
│  │ ...                      │  │  └────┴────┴────┴───┴───┘   │
│  └─────────────────────────┘  │                              │
│                               │  [⬆ Вверх] [⬇ Вниз]        │
└───────────────────────────────┴──────────────────────────────┘
```

## Левая панель: «📋 Доступные поля»

### ListBox — `AvailableFields`
- **Источник данных**: `AvailableSourceFields` (ObservableCollection<FieldDef>)
- **Привязка**: `ItemsSource="{Binding AvailableSourceFields}"`
- **Режим**: Extended (множественный выбор)
- **Событие**: `MouseDoubleClick` → `OnFieldDoubleClick` (добавляет поле в отчет)

### Элемент отображения (ItemTemplate)
Каждый элемент содержит:
- Иконка: 📄
- Имя поля: `{Binding Name}` (SemiBold)
- Тип поля: `({Binding Type})` (серым цветом)

### Модель `FieldDef`
```csharp
public class FieldDef {
    public string Name { get; set; }        // Отображаемое имя
    public string DbColumnName { get; set; } // Имя колонки в БД
    public string Type { get; set; }         // Тип данных
}
```

### Загрузка полей
Поля загружаются при выборе источника данных (ComboBox на вкладке **Общие**):
1. Вызывается `LoadAvailableFields(MetadataObject catalog)`
2. Проходим по `catalog.Fields`, исключая поля уже добавленные в `_reportFields`
3. Заполняются: `AvailableSourceFields`, `AvailableDataFields`, `AvailableFilterFields`
4. Добавляются вычисляемые поля из `GetPrintFormComputedFields()`

## Правая панель: «📋 Поля отчета»

### DataGrid — `ReportFieldsGrid`
- **Источник данных**: `ObservableCollection<ReportField> _reportFields`
- **CanUserAddRows**: False
- **CanUserDeleteRows**: False
- **RowHeight**: 40
- **MaxHeight**: 450

### Колонки DataGrid

| Колонка | Привязка | Ширина | Редактирование | Описание |
|---------|----------|--------|----------------|----------|
| Имя поля | `{FieldName}` | 120 | Нет (IsReadOnly) | Техническое имя колонки БД |
| Отображаемое имя | `{DisplayName}` | 150 | Да | Человекочитаемое название колонки |
| Тип агрегации | `{AggregateType}` | 120 | Да (ComboBox) | `""`, `Sum`, `Average`, `Count`, `Min`, `Max` |
| Ширина | `{Width}` | 80 | Да | Ширина колонки в пикселях |
| Видимость | `{IsVisible}` | 80 | Да (CheckBox) | Показать/скрыть колонку |
| Выравнивание | `{Alignment}` | 100 | Да (ComboBox) | `Left`, `Center`, `Right` |
| Удалить | — | 50 | Кнопка ❌ | Удаляет поле из отчета |

### Кнопки управления

**⬆ Вверх** / **⬇ Вниз** — перемещение полей:
- `OnMoveUpClick`: сдвиг выбранного поля на одну позицию вверх
- `OnMoveDownClick`: сдвиг выбранного поля на одну позицию вниз
- После перемещения вызывается `ReorderFields()` — перенумерация `Order`

### Логика добавления поля

`AddSelectedFieldAsync()` (вызывается по двойному клику):
1. Получает выбранное поле из `AvailableFields`
2. Проверяет, нет ли уже поля с таким `DbColumnName` в `_reportFields`
3. Если нет — создает `ReportField`:
   - `Id` = новый Guid
   - `FieldName` = `DbColumnName`
   - `DisplayName` = `Name`
   - `Order` = количество полей + 1
   - `IsVisible` = true
   - `Width` = 120
   - `Alignment` = "Left"
4. Обновляет список доступных полей (убирает добавленное)

### Логика удаления поля

`OnRemoveFieldClick` (по кнопке ❌ в строке DataGrid):
1. Получает `ReportField` из `DataContext` кнопки
2. Удаляет из `_reportFields`
3. Обновляет список доступных полей (добавляет обратно доступное для выбора)

### Логика перемещения

Перемещение через кнопки ⬆/⬇:
```csharp
// Move Up: _reportFields.Move(index, index - 1)
// Move Down: _reportFields.Move(index, index + 1)
// ReorderFields(): перенумеровать Order от 1 до Count
```

### Commit изменений

`CommitDesignerGridEdits()` — вызывается перед сменой источника данных, чтобы зафиксировать текущие редактирования в DataGrid:
```csharp
TryCommitGrid(ReportFieldsGrid);  // Поля отчета
TryCommitGrid(ElementMappingGrid); // FRX маппинг
TryCommitGrid(NativeElementsGrid); // Нативный макет
```

## Поток данных при работе с полями

```
Выбор источника данных (вкладка Общие)
    │
    ▼
OnDataSourceChanged()
    │
    ▼
LoadAvailableFields(catalog)
    │
    ├── Получение полей из catalog.Fields
    ├── Исключение уже добавленных (_reportFields)
    ├── Заполнение AvailableSourceFields (левая панель)
    ├── Заполнение AvailableDataFields (для FRX маппинга)
    └── Заполнение AvailableFilterFields (для фильтров)
            │
            ▼
Двойной клик по полю → OnFieldDoubleClick → AddSelectedFieldAsync()
    │
    ▼
Добавление ReportField в _reportFields
    │
    ▼
Обновление AvailableSourceFields (убираем добавленные)
    │
    ▼
Сохранение → Report.Fields = _reportFields.ToList()
```

## Модель `ReportField`

```csharp
public class ReportField {
    public Guid Id { get; set; }
    public Guid ReportId { get; set; }          // FK → Report
    public string FieldName { get; set; }       // Техническое имя (DbColumnName)
    public string DisplayName { get; set; }     // Отображаемое имя
    public int Order { get; set; }              // Порядок
    public int Width { get; set; } = 120;       // Ширина колонки
    public bool IsVisible { get; set; } = true; // Видимость
    public string Alignment { get; set; } = "Left"; // Выравнивание
    public string AggregateType { get; set; }   // Тип агрегации
    public string Format { get; set; }          // Формат (не используется в UI)
}
```