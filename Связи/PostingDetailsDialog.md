## 📌 **НАЗНАЧЕНИЕ PostingDetailsDialog**

**`PostingDetailsDialog`** — это модальное окно (WPF `Window`) для **просмотра детальной информации о бухгалтерской проводке**. Оно отображает все атрибуты проводки в read-only режиме: счета дебета/кредита, суммы, валюту, организацию, сотрудника и другие реквизиты.

---

## 🎯 **ЧТО ДЕЛАЕТ (ФУНКЦИОНАЛ)**

### 1. **Отображение полной информации о проводке**
Принимает `PostingViewModel` и отображает 18 полей в формате "метка → значение":

| № | Поле (Label) | Свойство PostingViewModel | Формат |
|---|---|---|---|
| 1 | **Документ** | `DocumentNumber` | Текст |
| 2 | **Дата** | `Date` | `dd.MM.yyyy HH:mm` |
| 3 | **Тип** | `DocumentType` | Текст |
| 4 | **Модуль** | `ModuleName` | Текст |
| 5 | **Дебет** | `DebitAccount` | Код счета |
| 6 | **Расшифровка дебета** | `DebitAccountName` | Наименование счета (с переносом строк) |
| 7 | **Кредит** | `CreditAccount` | Код счета |
| 8 | **Расшифровка кредита** | `CreditAccountName` | Наименование счета (с переносом строк) |
| 9 | **Корр. счет** | `CorrespondentAccount` | Корреспондентский счет |
| 10 | **Операция** | `Direction` | Направление операции |
| 11 | **Сумма (сом)** | `Amount` | `N2` (2 знака после запятой) |
| 12 | **Сумма (валюта)** | `AmountCurrency` | `N2` |
| 13 | **Валюта** | `Currency` | Код валюты |
| 14 | **Организация** | `Organization` | Наименование (с переносом строк) |
| 15 | **Сотрудник** | `Employee` | ФИО (с переносом строк) |
| 16 | **Создана** | `CreatedAt` | `dd.MM.yyyy HH:mm` (если есть) |
| 17 | **Статус** | `IsActive` | "Активна" / "Отключена" |
| 18 | **Примечание** | `Note` | Текст (с переносом строк) |

### 2. **Форматирование заголовка окна**
- `Title = $"Детали проводки №{posting.DocumentNumber}"`

### 3. **Закрытие**
- Кнопка "Закрыть" → `Close()`

---

## 🔗 **ЗАВИСИМОСТИ (DEPENDENCIES)**

### **Модели (Models)**

| Класс | Где используется | Для чего |
|---|---|---|
| `PostingViewModel` | Конструктор `PostingDetailsDialog(PostingViewModel posting)` | Модель представления проводки со всеми отображаемыми полями |

### **Где используется PostingDetailsDialog (потребители)**

| Модуль | Файл | Для чего |
|---|---|---|
| **PostingsView** | `Views/PostingsView.xaml.cs` | Просмотр деталей проводки из общего журнала проводок |
| **PostingsJournalView** | `Views/PostingsJournalView.xaml.cs` | Просмотр деталей проводки из журнала проводок |
| **CashOrderWorkView** | `Views/CashOrderWorkView.xaml.cs` | Просмотр проводки, связанной с кассовым ордером |

### **Свойства PostingViewModel, используемые в диалоге**

```
PostingViewModel
├── DocumentNumber    → DocumentNumberText
├── Date              → DateText (dd.MM.yyyy HH:mm)
├── DocumentType      → DocumentTypeText
├── ModuleName        → ModuleText
├── DebitAccount      → DebitText
├── DebitAccountName  → DebitNameText
├── CreditAccount     → CreditText
├── CreditAccountName → CreditNameText
├── CorrespondentAccount → CorrespondentText
├── Direction         → DirectionText
├── Amount            → AmountText (N2)
├── AmountCurrency    → AmountCurrencyText (N2)
├── Currency          → CurrencyText
├── Organization      → OrganizationText
├── Employee          → EmployeeText
├── CreatedAt         → CreatedAtText (dd.MM.yyyy HH:mm)
├── IsActive          → StatusText ("Активна"/"Отключена")
└── Note              → NoteText
```

---

## 🏗️ **АРХИТЕКТУРА ВЗАИМОДЕЙСТВИЯ**

```
PostingDetailsDialog
  │
  ├── Вход: PostingViewModel (модель проводки)
  │
  ├── XAML: 18 строк "Label + TextBlock" в Grid (2 колонки)
  │         ScrollViewer для поддержки прокрутки
  │
  ├── Code-behind: простое присвоение значений текстовым полям
  │
  └── Потребители:
        ├── PostingsView → общий журнал проводок
        ├── PostingsJournalView → журнал проводок
        └── CashOrderWorkView → кассовые ордера
```

---

## 💡 **КЛЮЧЕВЫЕ ОСОБЕННОСТИ**

1. **Read-only режим** — только просмотр, никаких изменений
2. **18 отображаемых полей** — полная информация о проводке
3. **Форматирование дат** — `dd.MM.yyyy HH:mm` (день.месяц.год часы:минуты)
4. **Форматирование сумм** — `N2` (с двумя знаками после запятой)
5. **Локализация статуса** — `IsActive` → "Активна" / "Отключена"
6. **Перенос строк** — для длинных текстов (наименования счетов, организации, сотрудники, примечания)
7. **ScrollViewer** — поддержка прокрутки при большом количестве данных
8. **Запрет изменения размера** — `ResizeMode="NoResize"`, фиксированный размер 620x640
9. **Используется в 3 модулях** — централизованный просмотр деталей проводки
10. **Простая архитектура** — без ViewModel, без привязок (Binding), прямое присвоение в code-behind