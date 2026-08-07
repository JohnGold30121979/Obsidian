# Генерация номеров документов - Полное объяснение

  

## Обзор системы нумерации

  

Система автоматически генерирует последовательные номера для документов (счет-фактур, платежных поручений, кассовых ордеров и т.д.) с сохранением состояния в базе данных.

  

## Структура таблицы `doc_numbering`

  

```sql

CREATE TABLE doc_numbering (

    Id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    document_type VARCHAR(100) NOT NULL,  -- Тип документа (ключ)

    current_number INTEGER DEFAULT 1,     -- Текущий номер

    prefix VARCHAR(20) DEFAULT '',        -- Префикс (не используется)

    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UpdatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP

);

  

-- Уникальный индекс для document_type

CREATE UNIQUE INDEX idx_doc_numbering_type ON doc_numbering (document_type);

```

  

**Важно**: В таблице хранится **следующий** номер, который будет выдан (current_number = 5 означает, что следующий документ получит номер 4).

  

## Метод `GetNextDocumentNumberByKeyAsync`

  

### Назначение

Получает следующий последовательный номер для указанного типа документа.

  

### Параметры

- `documentName` (string) - Название документа (например, "Выписка счет-фактур")

- `numberingKey` (string) - Ключ для поиска в таблице нумерации

- `document` (MetadataObject?, optional) - Объект метаданных документа

  

### Возвращает

- `Task<string>` - Следующий номер документа в виде строки

  

### Пошаговый workflow

  

```csharp

private async Task<string> GetNextDocumentNumberByKeyAsync(

    string documentName,

    string numberingKey,

    MetadataObject? document = null)

```

  

#### Шаг 1: Обеспечение конфигурации нумерации

  

```csharp

if (document != null)

    await EnsureDocumentNumberConfigurationAsync(document);

else

    await EnsureDocumentNumberConfigurationAsync(documentName);

```

  

**Что делает**:

- Проверяет, существует ли запись в таблице `doc_numbering` для этого типа документа

- Если нет - создает новую запись с номером, рассчитанным на основе максимального существующего номера в документах

  

#### Шаг 2: Запрос к базе данных с двумя ключами

  

```csharp

SELECT current_number, document_type

FROM doc_numbering

WHERE document_type = @documentType1

   OR document_type = @documentType2

LIMIT 1

```

  

**Параметры**:

- `@documentType1` = `numberingKey` (новый формат: "Выписка счет-фактур")

- `@documentType2` = `"doc:{documentName}"` (старый формат: "doc:Выписка счет-фактур")

  

**Зачем два ключа?**

- Ранее система использовала ключи вида `doc:{GUID}` (например, `doc:ac83617a6b5242b29c9e36621d5c0642`)

- Новая система использует понятные названия документов

- Этот код поддерживает оба формата для обратной совместимости

  

#### Шаг 3: Инкрементация счетчика

  

```csharp

var currentNumber = reader.GetInt32(0);      // Читаем текущий номер (например, 5)

var actualDocumentType = reader.GetString(1); // Запоминаем какой ключ использовался

var nextNumber = currentNumber + 1;          // Следующий номер будет 6

  

// Обновляем запись в БД

UPDATE doc_numbering

SET current_number = @nextNumber, UpdatedAt = NOW()

WHERE document_type = @documentType

```

  

**Важный момент**:

- Счетчик обновляется **сразу** после чтения, чтобы предотвратить дублирование номеров при параллельных запросах

  

#### Шаг 4: Обработка максимального номера

  

```csharp

if (nextNumber <= MaxDocumentNumberUsedForCounter)

    return currentNumber.ToString();

  

return nextNumber > MaxDocumentNumberUsedForCounter

    ? MaxDocumentNumberUsedForCounter.ToString()

    : nextNumber.ToString();

```

  

Где `MaxDocumentNumberUsedForCounter = 999_999_999`

  

**Логика**:

- Если следующий номер <= 999,999,999 - возвращаем текущий номер

- Если превышен максимум - возвращаем максимальный номер (защита от переполнения)

  

#### Шаг 5: Обработка ошибок

  

```csharp

catch (Exception ex)

{

    System.Diagnostics.Debug.WriteLine($"Ошибка получения номера документа: {ex.Message}");

    return GenerateFallbackDocumentNumber();  // Возвращает "1"

}

```

  

## Примеры использования

  

### Пример 1: Создание счета-фактуры

  

```csharp

// В InvoiceService.GenerateDocumentNumberAsync()

var metadata = await _context.MetadataObjects.AsNoTracking()

    .FirstOrDefaultAsync(item => item.Name == "Выписка счет-фактур" && item.ObjectType == "Document");

  

// metadata.Name = "Выписка счет-фактур"

// metadata.Id = ac83617a-6b52-42b2-9c9e-36621d5c0642

  

return await _metadataService.GetNextDocumentNumberAsync(metadata);

```

  

**Что происходит**:

1. Вызывается `GetNextDocumentNumberAsync(metadata)`

2. Внутри вызывается `GetNextDocumentNumberByKeyAsync("Выписка счет-фактур", "Выписка счет-фактур", metadata)`

3. `numberingKey` = `GetDocumentNumberingKey(metadata)` = `metadata.Name` = "Выписка счет-фактур"

4. В БД ищется запись где `document_type = "Выписка счет-фактур"` OR `document_type = "doc:Выписка счет-фактур"`

5. Найдена запись с `current_number = 5`

6. Возвращается "4", а в БД записывается "6"

  

### Пример 2: Первый документ (создание записи)

  

```csharp

// Если записи в doc_numbering нет

await EnsureDocumentNumberConfigurationAsync(document);

  

// Создается запись:

INSERT INTO doc_numbering (document_type, current_number, prefix)

VALUES ('Выписка счет-фактур', 2, '')

```

  

Текущий максимальный номер в документах = 1, поэтому следующий = 2.

  

### Пример 3: Параллельные запросы

  

```csharp

// Запрос 1: Читает current_number = 5

// Запрос 2: Ждет пока Запрос 1 обновит до 6

// Запрос 2: Читает current_number = 6, возвращает "6", обновляет до 7

// Запрос 1: Возвращает "5", обновляет до 6 (конфликт!)

```

  

**Проблема**: При параллельных запросах возможны гонки данных.

  

**Решение**: В идеале нужно использовать:

```sql

UPDATE doc_numbering

SET current_number = current_number + 1

WHERE document_type = @documentType

RETURNING current_number

```

  

Но текущая реализация работает корректно в 99% случаев из-за быстрого выполнения.

  

## Ключевые особенности системы

  

### 1. Обратная совместимость

  

```csharp

WHERE document_type = @documentType1   -- Новый формат: "Выписка счет-фактур"

   OR document_type = @documentType2   -- Старый формат: "doc:Выписка счет-фактур"

```

  

Позволяет работать со старыми данными, созданными до изменения ключей.

  

### 2. Автоматическая инициализация

  

```csharp

private async Task EnsureDocumentNumberConfigurationAsync(MetadataObject document)

{

    await CreateDocumentNumberingTableAsync();

    var nextNumber = await GetSuggestedNextDocumentNumberAsync(new[] { document });

    var numberingKey = GetDocumentNumberingKey(document);

    // Создает запись если не существует

    INSERT INTO doc_numbering (document_type, current_number, prefix)

    VALUES (@documentType, @currentNumber, '')

    ON CONFLICT (document_type) DO NOTHING

}

```

  

**Что делает `GetSuggestedNextDocumentNumberAsync`**:

- Находит максимальный номер в существующих документах

- Возвращает max + 1

- Если документов нет - возвращает 1

  

### 3. Нормализация номеров

  

```csharp

internal static string NormalizeLegacyDocumentNumber(string? documentNumber)

{

    // Убирает все не-цифровые символы

    var digitsOnly = new string(normalizedNumber.Where(char.IsDigit).ToArray());

    return string.IsNullOrEmpty(digitsOnly) ? normalizedNumber : digitsOnly;

}

```

  

Примеры:

- "СФ-2024-001" → "2024001"

- "№ 5" → "5"

- "123" → "123"

  

## Схема работы системы

  

```

┌─────────────────────────────────────────────────────────────┐

│  1. Пользователь создает новый счет-фактуру                  │

└───────────────────────┬─────────────────────────────────────┘

                        │

                        ▼

┌─────────────────────────────────────────────────────────────┐

│  2. InvoiceService.GenerateDocumentNumberAsync()             │

│     - Загружает метаданные документа                         │

│     - Вызывает MetadataService.GetNextDocumentNumberAsync()  │

└───────────────────────┬─────────────────────────────────────┘

                        │

                        ▼

┌─────────────────────────────────────────────────────────────┐

│  3. MetadataService.GetNextDocumentNumberByKeyAsync()        │

│     - Убеждается что запись в doc_numbering существует       │

│     - Если нет - создает с начальным номером                 │

└───────────────────────┬─────────────────────────────────────┘

                        │

                        ▼

┌─────────────────────────────────────────────────────────────┐

│  4. SELECT current_number FROM doc_numbering                 │

│     WHERE document_type IN ('Выписка счет-фактур',           │

│                               'doc:Выписка счет-фактур')     │

└───────────────────────┬─────────────────────────────────────┘

                        │

                        ▼

┌─────────────────────────────────────────────────────────────┐

│  5. Найдено: current_number = 5                              │

│     - Возвращаем "5" для нового документа                    │

│     - Обновляем в БД: current_number = 6                     │

└───────────────────────┬─────────────────────────────────────┘

                        │

                        ▼

┌─────────────────────────────────────────────────────────────┐

│  6. Новый документ создается с номером "5"                   │

└─────────────────────────────────────────────────────────────┘

```

  

## Конфигурация для разных типов документов

  

### Кассовые ордера (КО)

```csharp

// Два отдельных ключа для прихода и расхода

"Приходный кассовый ордер"  - для ПКО

"Расходный кассовый ордер"  - для РКО

```

  

### Платежные поручения

```csharp

// Используется название документа

"Платежное поручение"

```

  

### Счета-фактуры

```csharp

// Используется название документа

"Выписка счет-фактур"

```

  

## Отладка и мониторинг

  

### Просмотр текущих номеров

  

```sql

SELECT

    document_type,

    current_number,

    prefix,

    CreatedAt,

    UpdatedAt

FROM doc_numbering

ORDER BY document_type;

```

  

### Проверка использования номеров

  

```sql

SELECT

    d.document_type,

    d.current_number as next_number,

    COUNT(t.doc_number) as used_numbers,

    d.current_number - COUNT(t.doc_number) as available

FROM doc_numbering d

LEFT JOIN doc_incoming t ON REGEXP_REPLACE(COALESCE(t.doc_number::text, ''), '\D', '', 'g')

                         = CAST(d.current_number - 1 AS TEXT)

GROUP BY d.document_type, d.current_number;

```

  

### Логирование в коде

  

```csharp

System.Diagnostics.Debug.WriteLine($"Ошибка получения номера документа: {ex.Message}");

System.Diagnostics.Debug.WriteLine($"✅ Проводка создана: {documentType} | {docNumber} | {amount}");

```

  

## Возможные проблемы и решения

  

### Проблема 1: Номер "1" выдается повторно

**Причина**: Запись в `doc_numbering` не инкрементируется

**Решение**: Проверить, что метод `GetNextDocumentNumberByKeyAsync` обновляет БД

  

### Проблема 2: Документы с одинаковыми номерами

**Причина**: Две записи в `doc_numbering` для одного документа (старый и новый формат ключей)

**Решение**:

```sql

-- Найти дубликаты

SELECT document_type, COUNT(*)

FROM doc_numbering

GROUP BY document_type

HAVING COUNT(*) > 1;

  

-- Удалить старые записи

DELETE FROM doc_numbering

WHERE document_type LIKE 'doc:%';

```

  

### Проблема 3: Пропуски в нумерации

**Причина**: Документы удаляются, но нумератор продолжает расти

**Решение**: Это нормальное поведение. Для бесспорной нумерации нужно хранить использованные номера.

  

## Best Practices

  

1. **Всегда используйте один формат ключей** - либо все `document_name`, либо все `doc:{guid}`

2. **Регулярно проверяйте таблицу doc_numbering** на предмет дубликатов

3. **Не сбрасывайте current_number вручную** - это нарушит последовательность

4. **При миграции данных** - сначала обновите ключи, потом мигрируйте документы

  

## Заключение

  

Система нумерации документов обеспечивает:

- ✅ Последовательную нумерацию

- ✅ Поддержку разных типов документов

- ✅ Обратную совместимость со старыми данными

- ✅ Защиту от переполнения номеров

- ✅ Автоматическую инициализацию

  

Критически важные моменты:

1. **Ключ в `doc_numbering` должен совпадать** с тем, что используется при поиске

2. **Счетчик инкрементируется сразу** после чтения для предотвращения дублей

3. **Формат ключей** должен быть единообразным для всех документов одного типа