# FrxParser.ParseFrxFile — детальный разбор механики парсинга

## Точка входа

```csharp
public FrxReport ParseFrxFile(string frxPath)
```

**Шаг 1: Определение формата файла**
- Читает файл как массив байт
- Пробует прочитать как текст (UTF-8)
- Если расширение `.json`/`.fpx` или текст начинается с `{`/`[` → вызывается `ParseFoxProJsonFile` (см. ниже)
- Иначе → обрабатывается как legacy FoxPro FRX (DBF-подобный или XML)

---

## Ветка 1: `ParseFoxProJsonFile` (современный формат)

Если файл уже JSON, парсер пытается определить под-формат:

1. **Если есть `Bands` как массив** → это `FrxFullData` (расширенный формат с полными полями)
2. **Если есть `Elements`/`Bands`** → это готовый `PrintFormTemplate`
3. **Если есть `records`/`rows`/`data`/`items`** → это массив записей, который нужно преобразовать в `BuildVisualFoxProTemplate`

---

## Ветка 2: Legacy FoxPro FRX (DBF-подобный)

### Шаг 2.1: Чтение DBF-записей (`ReadVisualFoxProRecords`)

**Заголовок DBF:**
```
Байт 0: версия DBF
Байты 4-7: количество записей (int32 LE)
Байты 8-9: длина заголовка (uint16 LE)
Байты 10-11: длина записи (uint16 LE)
```

**Дескрипторы полей (начиная с байта 32):**
- Каждое поле — 32 байта
- Имя: ASCII строка до первого `\0` (макс 11 chars)
- Тип: `descriptor[11]` (char)
- Длина: `descriptor[16]` (byte)
- Количество десятичных знаков: `descriptor[17]` (byte)
- Конец дескрипторов: байт `0x0D`

**Чтение записей:**
- Каждая запись начинается с байта удаления (`*` = помечена на удаление)
- Смещение записи = `headerLength + recordIndex * recordLength`
- Пропускается байт удаления, затем читаются поля последовательно

**Типы полей FoxPro:**
- `'M'` → Memo → `ReadMemo` (из FRT)
- `'N'`/`'F'` → Number → `ReadNumber` (decimal через InvariantCulture)
- `'L'` → Logical → `T/t/Y/y` = true
- `'D'` → Date → `yyyyMMdd` → преобразуется в ISO
- `'T'` → DateTime → Julian Day + время
- `'I'` → Integer → int32 LE
- `'B'` → Double → double LE
- Иначе → строка (кодировка 1251, трим `\0` и пробелов)

### Шаг 2.2: Чтение Memo из FRT

**Структура memo-блока:**
```
Блок N: offset = blockNumber * blockSize
Байты 0-3: тип блока (0x00000001 = текст)
Байты 4-7: длина данных
Байты 8+: содержимое
```

Если тип = 1 → читается как текст в кодировке 1251

### Шаг 2.3: Построение шаблона (`BuildVisualFoxProTemplate`)

**Логика:**

1. **Найти запись отчета** (`OBJTYPE = 1`)
   - Ширина страницы = `WIDTH`
   - Высота страницы = `HEIGHT`

2. **Найти банды** (`OBJTYPE = 9`)
   - Сортируются по `VPOS` (вертикальная позиция)
   - `OBJCODE` преобразуется в тип банды:
     - 0 → Title, 1 → PageHeader, 2 → ColumnHeader, 3 → GroupHeader
     - 4 → Detail, 5 → GroupFooter, 6 → ColumnFooter, 7 → PageFooter, 8 → Summary
   - Параметры: `Top`, `Height`, `Width`, `Left`, `Order`

3. **Найти элементы макета**
   - Фильтр по `OBJTYPE`: 5, 6, 7, 8, 10, 12, 14, 17, 0
   - Сортировка по `VPOS`, затем `HPOS`

4. **Для каждого элемента определить банду** (`FindBandByPosition`)
   - Точное попадание: `top >= band.Top && top < band.Top + band.Height`
   - Если нет → ближайшая банда сверху (с допуском 50px) или снизу (с допуском 20px)
   - Fallback → Detail или первая банда

5. **Извлечь выражение (EXPR)**
   - Если EXPR == число > 0 → это DATAHANDLE на FRT
   - Ищется запись с `DATAHANDLE == это число` и `OBJTYPE == 8`
   - Из найденной записи берется EXPR как реальное выражение

6. **Определить имя поля**
   - OBJTYPE 5 (Label) → имя = EXPR (текст метки)
   - OBJTYPE 8 (Field) → имя = поле `NAME`, fallback = `FieldN`
   - Иначе → `{GetVfpElementType(objectType)}{fieldOrder}`

7. **Создать FrxField + PrintFormElement**
   - Свойства: `Left`, `Top`, `Width`, `Height`, `FontName`, `FontSize`, `FontBold/Italic/Underline`, `Alignment`, `Format`, `BorderStyle`
   - FontStyle битовый флаг: `& 1` = bold, `& 2` = italic, `& 4` = underline
   - Alignment: 1=Right, 2=Center, иначе Left

8. **Переменные отчета** (`OBJTYPE = 14`, `TYPE = "V"`, `OBJTYPE = 10`)
   - Добавляются в `report.Expressions` как `FrxExpression`

9. **Автоматический расчет размера страницы**
   - `PageWidth = max(Element.Left + Element.Width) + 500`
   - `PageHeight = max(Element.Top + Element.Height) + 500`

10. **Fallback**: если банды не найдены → создается Detail банда Height=100

### Шаг 2.4: Fallback при ошибках

Если парсинг упал с исключением:
- Создается `FrxReport` с дефолтной структурой (6 банд: PageHeader, Header, Detail, Footer, PageFooter, Summary)
- В `FrxXml` записывается `PrintFormTemplate` с `SourceFormat = "FoxProFRX"`
- В `Description` добавляется "(fallback)"

---

## Ветка 3: XML FRX

Если файл — XML (содержит `<Bands>`, `<Expressions>`):
- Парсится через `XmlDocument`
- Банды: `SelectSingleNode("//Bands")` → дочерние узлы = банды
- Поля: `SelectSingleNode("Fields")` внутри каждой банды
- Атрибуты: Height, Top, Visible, Name, Type, Left, Width, Format, Alignment, FontName, FontSize, FontBold
- Выражения: `SelectSingleNode("//Expressions")` → `InnerText` = тело выражения

---

## Ветка 4: Бинарный FRX

Если файл — бинарный (заголовок 10 байт, затем записи):
- Читаются байты через `BinaryReader`
- Тип записи: `0x42` = Band, `0x46` = Field
- Длина записи: `int16` после типа
- Band: Top, Height, Left, Width, Order, Visible + пропуск 40 байт
- Field: Name (10b), Expression (254b), Left, Top, Width, Height, Format (50b), FontName (20b), FontSize, FontBold, Alignment, Order
- Неизвестные записи → пропуск

---

## Возвращаемое значение

Всегда возвращается `FrxReport` со структурой:
```csharp
public class FrxReport
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string OriginalFileName { get; set; }
    public byte[] FrxData { get; set; }      // сырые байты
    public string FrxXml { get; set; }       // JSON-сериализованная структура
    public string ReportType { get; set; }   // "FoxProLayout"
    public string Description { get; set; }
    public List<FrxBand> Bands { get; } = new();
    public List<FrxField> Fields { get; } = new();
    public List<FrxExpression> Expressions { get; } = new();
}
```

---

## Пример потока данных

1. Пользователь загружает `cash_receipt.frx` + `cash_receipt.frt`
2. `FrxParser.ParseFrxFile(path)`:
   - Детектирует: не JSON → DBF-формат
   - `ReadVisualFoxProRecords` → 150 записей
   - Находит 1 запись с `OBJTYPE=1` (WIDTH=2100, HEIGHT=2970)
   - Находит 3 банды (`OBJTYPE=9`): PageHeader, Detail, Summary
   - Находит 25 элементов (Labels, Fields, Lines, Boxes)
   - Для полей с числовым EXPR ищет выражения в FRT через `ReadMemo`
   - Строит `PrintFormTemplate` с 3 бандами и 25 элементами
   - Сериализует в JSON → `report.FrxXml`
3. `ReportDesignerWindow` получает `FrxReport`:
   - `LoadFrxElementMappings(templateJson)` → десериализует, строит `_frxElementMappings`
   - `ExtractReportFieldsFromTemplate(templateJson)` → извлекает поля `{Номер}`, `{Дата}`, `{Сумма}`
   - Загружает в `ReportFieldsGrid`
4. При сохранении JSON сохраняется в `Report.Template` (поле в БД)
5. При экспорте PDF: `PrintFormService.BuildTemplateLayoutPdf` десериализирует JSON и строит PDF через QuestPDF

---

## Ключевые особенности

- **Кодировка**: файлы читаются в CP1251 (русская локализация FoxPro)
- **Memo**: поддержка FRT как memo-хранилища для длинных выражений
- **DATAHANDLE**: опосредованные ссылки на выражения через числовые идентификаторы
- **Неважно, если что-то пропущено**: парсер пытается найти банду по позиции, создает дефолтные структуры
- **Двойное представление**: `FrxReport` (внутренний объект) → JSON строка (`FrxXml`) для хранения