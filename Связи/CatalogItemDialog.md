## 📌 **НАЗНАЧЕНИЕ CatalogItemDialog**

**`CatalogItemDialog`** — универсальное модальное окно (WPF `Window`) для **добавления и редактирования записей справочников** в системе BIS.ERP. Оно динамически строит форму ввода на основе метаданных справочника (`MetadataObject`), автоматически определяя типы полей и создавая соответствующие элементы управления.

---

## 🎯 **ЧТО ДЕЛАЕТ (ФУНКЦИОНАЛ)**

### 1. **Динамическое построение формы** (`BuildFieldsAsync`)
- Получает список полей справочника из `MetadataObject.Fields`
- Для каждого поля создает Label + InputControl, тип которого зависит от `FieldType` и атрибутов поля

### 2. **Типы создаваемых контролов**

| Тип поля (`FieldType`) | Создаваемый контрол |
|---|---|
| `Int` | `TextBox` (с парсингом int) |
| `Decimal` | `TextBox` (с парсингом decimal) |
| `DateTime` | `DatePicker` |
| `Bool` | `CheckBox` |
| `String` (default) | `TextBox` |

### 3. **Специализированные контролы**

- **Reference-поля** (`field.ReferenceCatalog != null`) → `ReferencePickerControl` (ComboBox + кнопки "?" выбрать, "+" добавить, "Изм." редактировать)
- **Поля плана счетов** (для справочника "План счетов") → `ComboBox` с ChoiceItem (Тип счета, Закрывает модуль, Признак печати, Сохранять остатки)
- **Аналитические счета** (`AccountAnalyticsRules.IsAccountSelectorField`) → `AccountPickerControlFactory` (TextBox + кнопка "?" выбора счета)

### 4. **Автоматическая генерация кода** (`ApplyGeneratedCodeAsync`)
- Если поле называется "Код"/"code" — автоматически генерирует следующий код на основе существующих записей
- Поле становится ReadOnly с подсказкой "Код формируется автоматически"

### 5. **Автозаполнение из сотрудника** (`AutoFillFromEmployee`)
- При выборе сотрудника в поле "Табельный номер" автоматически заполняет: ФИО, Должность, Телефон, Участок

### 6. **Синхронизация наименований** (`AttachOrganizationNameSync`)
- Для справочника "Организации": изменение "Наименование" → автокопирование в "Полное наименование"

### 7. **Сбор данных при сохранении** (`OnSaveClick`)
- Проходит по всем полям, собирает значения из контролов
- Для ReferencePickerControl → `SelectedReferenceItem.Id`
- Для ComboBox → определяет тип (ReferenceItem, ChoiceItem)
- Для AccountPicker → `AccountPickerControlFactory.GetSelectedAccountValue`
- Устанавливает `DialogResult = true` и закрывает окно

### 8. **Фильтрация полей** (`GetEditableFields`)
- Для "План счетов" — строго определенный набор колонок (code, name, account_type, closing_module_code и др.)
- Для "Авансовые платежи" — свой набор колонок
- Удаляет дубликаты по `DbColumnName` и `Name`

---

## 🔗 **ЗАВИСИМОСТИ (DEPENDENCIES)**

### **Модели (Models)**

| Класс | Где используется | Для чего |
|---|---|---|
| `MetadataObject` | Поле `_catalog` | Метаданные справочника: имя, поля, тип объекта |
| `MetadataField` | `GetEditableFields()`, `CreateRegularControl()` и др. | Описание поля: имя, тип, ReferenceCatalog, DisplayPattern |
| `ReferenceItem` | `ReferencePickerControl`, `OnSaveClick` | Элемент справочника для ComboBox (Id + DisplayName + LookupKeys) |
| `ChoiceItem` | Внутренний record в `CatalogItemDialog` | Элемент выбора для ComboBox (Code, Display, LookupKeys) |
| `AccountReferenceItem` | Через `AccountPickerControlFactory` | Элемент счета из плана счетов |

### **Сервисы (Services)**

| Сервис | Где используется | Для чего |
|---|---|---|
| `MetadataService` | Основной сервис (поле `_metadataService`) | Получение справочников, данных, модулей, создание/обновление записей |
| `LocalizationService` | `NormalizeChartOfAccountsChoice` | Локализация значений (Active → Активный) |
| `AccountAnalyticsRegistry` | Поле `_accountAnalytics` | Загрузка и поиск счетов для аналитики |
| `AccountAnalyticsRules` | `ShouldUseAccountPicker` | Определение, является ли поле полем выбора счета |

### **Фабрики/Контролы (Views)**

| Компонент | Где используется | Для чего |
|---|---|---|
| `ReferencePickerControlFactory` | `CreateReferenceControl()` | Создание универсального контрола выбора из справочника |
| `ReferencePickerControl` | `OnSaveClick`, `AutoFillFromEmployee` | UserControl-обертка над ComboBox для выбора справочных значений |
| `AccountPickerControlFactory` | `ShouldUseAccountPicker` | Создание контрола выбора счета (TextBox + кнопка выбора) |
| `ReferenceComboBoxSearchHelper` | Через `ReferencePickerControlFactory` | Быстрый поиск/фильтрация в ComboBox |
| `ReferenceDisplayHelper` | Через `ReferencePickerControlFactory` | Построение отображаемого имени для ReferenceItem |
| `ReferenceSelectionDialog` | Через `ReferencePickerControlFactory` | Диалог выбора записи из справочника |

### **Другие диалоги (взаимные вызовы)**

| Диалог | Где используется | Для чего |
|---|---|---|
| `AccountSelectionDialog` | В `AccountPickerControlFactory` | Диалог выбора счета из плана счетов |
| `CatalogItemDialog` (сам себя) | В `ReferencePickerControlFactory.AddReferenceAsync` | Рекурсивный вызов для добавления записи в связанный справочник |

---

## 🏗️ **АРХИТЕКТУРА ВЗАИМОДЕЙСТВИЯ**

```
CatalogItemDialog
  │
  ├── MetadataService ──── MetadataObject (справочник)
  │                           └── MetadataField[] (поля)
  │
  ├── ReferencePickerControlFactory ──── ReferencePickerControl
  │                                         └── ComboBox + кнопки (?, +, Изм.)
  │
  ├── AccountPickerControlFactory ──── UserControl
  │                                      └── TextBox + кнопка (?)
  │
  ├── AccountAnalyticsRegistry ──── AccountReferenceItem[]
  │
  └── Внутренние: ChoiceItem, GetEditableFields, AutoFillFromEmployee
```

---

## 💡 **КЛЮЧЕВЫЕ ОСОБЕННОСТИ**

1. **Универсальность** — один диалог для всех справочников системы
2. **Динамическое построение** — форма строится на лету по метаданным
3. **Поддержка ссылочных полей** — автоматическое создание ReferencePicker для связанных справочников
4. **Специализированная логика** — для "План счетов" и "Организации" своя логика отображения/валидации
5. **Автогенерация кода** — для полей "Код" автоматически вычисляется следующий номер
6. **Автозаполнение** — выбор сотрудника автоматически заполняет связанные поля (ФИО, должность, телефон)
7. **Рекурсивное создание** — при необходимости можно добавить запись в связанный справочник прямо из диалога
8. **Фильтрация по справочнику** — для разных справочников показывается разный набор полей