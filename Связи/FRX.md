# FRX в FoxPro и конвертация в [[JSON]]
в BIS.ERP

## Что такое FRX

**FRX** (FoxPro Report eXecution) — это формат файлов макетов отчетов из Visual FoxPro. 
Это двоичный/DBF-подобный файл, который описывает визуальную структуру отчета:
- **Банды** (полосы): PageHeader, Header, Detail, Footer, PageFooter, Summary
- **Элементы**: текстовые метки, поля с выражениями, линии, рамки, изображения
- **Позиционирование**: координаты X, Y, ширина, высота
- **Шрифты**: имя, размер, bold/italic, выравнивание
- **Выражения**: формулы, привязка к полям данных

## Два файла: `.FRX` и `.FRT`

В FoxPro отчет всегда состоит из **пары файлов**:

### 1. `filename.FRX` — основной файл макета
- Содержит заголовок отчета, определения банд (полос)
- Описание элементов: позиция, размер, шрифт, тип
- Поля (`OBJTYPE = 8`) с ссылками на выражения
- Метаданные: размер страницы, ориентация

### 2. `filename.FRT` — memo-файл выражений
- Содержит текстовые memo-блоки с полными выражениями полей
- В DBF-терминах: поля типа `M` (Memo)
- Используется когда выражение слишком длинное для прямого хранения в FRX
- FRX хранит только числовой указатель (`DATAHANDLE`) на запись в FRT

**Почему два файла:** В DBF-формате поле memo не может храниться inline, поэтому выносится в отдельный FPT/FRT файл.

## Как это превращается в JSON в нашей системе

### Процесс импорта FRX

1. **Загрузка файла** (`ReportDesignerWindow.OnLoadFrxClick` или [[FrxParser.ParseFrxFile]])
   - Читается `.FRX` файл как массив байт
   - Если есть `.FRT` — тоже читается

2. **Парсинг структуры** (`FrxParser.ParseFrxFile`)
   - **Определение формата**: проверяется расширение или сигнатура
     - `.json` / `.fpx` → парсится как JSON
     - Иначе → парсится как DBF-подобный бинарный формат
   
3. **Чтение DBF-записей** (`ReadVisualFoxProRecords`)
   - Парсится заголовок DBF: количество записей, длина записи, кодировка
   - Читаются описания полей (дескрипторы)
   - Для каждой записи извлекаются значения полей по偏移
   - **Memo-поля (тип 'M')** читаются из FRT: определяется блок, тип блока (текст/изображение), длина
   - Поддерживаемые типы полей: N, F, L, D, T, I, B, M, C

4. **Построение структуры отчета** (`BuildVisualFoxProTemplate`)
   - **Запись с OBJTYPE=1** → параметры страницы (WIDTH, HEIGHT)
   - **Записи с OBJTYPE=9** → банды (полосы) с типом по OBJCODE:
     - 0=Title, 1=PageHeader, 2=ColumnHeader, 3=GroupHeader, 4=Detail, 5=GroupFooter, 6=ColumnFooter, 7=PageFooter, 8=Summary
   - **Элементы макета** (OBJTYPE 5,6,7,8,10,12,14,17):
     - OBJTYPE=5 → Text (текстовая метка, EXPR содержит текст)
     - OBJTYPE=6 → Line
     - OBJTYPE=7 → Box
     - OBJTYPE=8 → Expression (поле данных, NAME=имя поля, EXPR=выражение)
     - И др.
   - **Важно:** Если EXPR выглядит как число → это DATAHANDLE, и выражение ищется в другой записи с таким DATAHANDLE
   - **Переменные** (OBJTYPE=14, TYPE="V") → добавляются в `report.Expressions`

5. **Сериализация в JSON** (`JsonSerializer.Serialize`)
   - Внутренний формат `FrxReport` сериализуется в `FrxXml` (строку JSON)
   - Поддерживаются два варианта:
     - **PrintFormTemplate** — простой формат: `Bands` + `Elements`
     - **FrxFullData** — расширенный: `Template` + `Bands` + `Fields` в каждой банде
   
   Пример структуры JSON:
   ```json
   {
     "SourceFormat": "FoxProFRX",
     "OriginalFileName": "report.frx",
     "PageWidth": 2100,
     "PageHeight": 2970,
     "Bands": [
       {
         "Type": "PageHeader",
         "Top": 0,
         "Height": 120,
         "Order": 0
       }
     ],
     "Elements": [
       {
         "Type": "Text",
         "Text": "ПРИХОДНЫЙ КАССОВЫЙ ОРДЕР",
         "BandType": "PageHeader",
         "Left": 200,
         "Top": 20,
         "Width": 2500,
         "Height": 60,
         "FontName": "Arial",
         "FontSize": 18,
         "Bold": true,
         "Alignment": "Center"
       }
     ]
   }
   ```

6. **Сохранение в БД**: Этот JSON сохраняется в поле `Report.Template` как строка.

### Использование JSON

- **В дизайнере отчетов** (`ReportDesignerWindow`):
  - `PrintFormService.DeserializePrintTemplate(templateJson)` → `PrintFormTemplate`
  - Заполняет вкладку "FRX поля" соответствиями элементов полям данных
  - Позволяет редактировать нативную версию макета

- **Для экспорта PDF** (`PrintFormService.BuildTemplateLayoutPdf`):
  - Десериализуется `PrintFormTemplate`
  - На основе элементов строится PDF через QuestPDF

- **Извлечение полей** (`PrintFormService.ExtractReportFieldsFromTemplate`):
  - Анализирует JSON, находит все `{ИмяПоля}` в тексте элементов
  - Создает список `ReportField` для отображения в дизайнере

## Поддержка FRT memo

Метод `ReadMemo` в `FrxParser`:
- Читает блок по номеру `block * blockSize`
- Проверяет тип блока (0x00000001 = текст)
- Декодирует как текст в кодировке 1251 (русская Windows)

Через эту систему старые FoxPro-отчеты импортируются в современную JSON-модель, где их можно редактировать в `ReportDesignerWindow` и экспортировать в PDF.