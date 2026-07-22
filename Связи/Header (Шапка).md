# Header (Шапка) MainWorkWindow — детальное описание

## Расположение и структура

Шапка находится в левой панели навигации (`AppSidebarBackgroundBrush`) наверху, внутри `Border` с фоном `AppBrandHeaderBackgroundBrush` и отступом 15px. Состоит из двух частей:

```
┌──────────────────────────────┐
│  ┌──┐                        │
│  │🖼│ НАЗВАНИЕ СИСТЕМЫ    🌙│
│  └──┘                        │
│  ┌──┐                        │
│  │🏢│ Имя инфобазы          │
│  └──┘                        │
└──────────────────────────────┘
```

## 1. Верхняя строка — бренд системы

### SystemLogoImage / SystemIconText

**Назначение:** Отображение логотипа системы (иконка/изображение).

**XAML структура:**
```xml
<Border Width="28" Height="28" CornerRadius="7"
        Background="{DynamicResource AppAccentBrush}"
        Margin="0,0,8,0">
    <Grid>
        <Image x:Name="SystemLogoImage" Stretch="UniformToFill" Visibility="Collapsed"/>
        <TextBlock x:Name="SystemIconText" FontSize="18"
                   HorizontalAlignment="Center" VerticalAlignment="Center"
                   Foreground="{DynamicResource AppBrandIconForegroundBrush}"/>
    </Grid>
</Border>
```

**Логика отображения (LogoDisplayHelper.Apply):**
```csharp
public static void Apply(Image image, TextBlock fallbackText, byte[]? imageBytes, string? fallbackIcon)
{
    var imageSource = LogoFileService.CreateImageSource(imageBytes);
    image.Source = imageSource;
    image.Visibility = imageSource == null ? Visibility.Collapsed : Visibility.Visible;
    fallbackText.Text = string.IsNullOrWhiteSpace(fallbackIcon) ? "🏢" : fallbackIcon.Trim();
    fallbackText.Visibility = imageSource == null ? Visibility.Visible : Visibility.Collapsed;
}
```

**Механизм:**
1. Если в `SystemConfiguration.LogoImage` (byte[] — изображение, загруженное пользователем) есть данные → `LogoFileService.CreateImageSource` создает `BitmapImage`, показывается `SystemLogoImage`
2. Если изображения нет → показывается `SystemIconText` с иконкой из `SystemConfiguration.Icon` (символ emoji, например "⚙", "🏢")
3. Если иконка не задана — fallback "🏢"

**Вызывается при загрузке окна (OnLoaded, строка 104):**
```csharp
var systemConfiguration = await new SystemConfigurationService().GetAsync();
SystemNameText.Text = systemConfiguration.SystemName;
LogoDisplayHelper.Apply(SystemLogoImage, SystemIconText, systemConfiguration.LogoImage, systemConfiguration.Icon);
```

### SystemNameText

**XAML:**
```xml
<TextBlock x:Name="SystemNameText" FontSize="22" FontWeight="Bold"
           Foreground="{DynamicResource AppBrandTitleForegroundBrush}"/>
```

**Назначение:** Отображение названия системы.

**Источник:** `SystemConfigurationService.GetAsync().SystemName`
- Устанавливается в настройках системы (через `SettingsWindow`)
- По умолчанию может быть пустым или заданным при первой настройке

---

### ThemeToggleButton 🌙

**XAML:**
```xml
<Button Grid.Column="1" x:Name="ThemeToggleButton" 
        Content="🌙" FontSize="20" 
        Background="Transparent" BorderThickness="0"
        Foreground="{DynamicResource AppBrandTitleForegroundBrush}" Cursor="Hand"
        Click="OnThemeToggleClick"
        ToolTip="Сменить тему"/>
```

**Назначение:** Переключение между светлой (Light) и темной (Dark) темой интерфейса.

**Логика инициализации (OnLoaded, строки 113-118):**
```csharp
var settings = AppSettings.Instance;
if (ThemeToggleButton != null)
{
    ThemeToggleButton.Content = ThemeService.GetThemeToggleIcon(settings.Theme);
    ThemeToggleButton.ToolTip = ThemeService.GetThemeToggleToolTip(settings.Theme);
}
```

**При нажатии (OnThemeToggleClick):**
- Переключает `AppSettings.Instance.Theme` между `"Light"` и `"Dark"`
- Вызывает `ThemeService.ApplyTheme(newTheme)`, который меняет `ResourceDictionary` приложения
- Обновляет иконку кнопки: 🌙 для светлой темы, ☀️ для темной
- Обновляет ToolTip: "Сменить тему" или "Сменить на светлую"

**Доступные темы:** Light, Dark (также есть Dracula в файлах тем, но в UI только 2 варианта)

---

## 2. Нижняя строка — информация об информационной базе

### InfoBaseLogoImage / InfoBaseIconText

**Назначение:** Отображение логотипа/иконки текущей информационной базы.

**XAML:**
```xml
<Border Width="22" Height="22" CornerRadius="6"
        Background="{DynamicResource AppAccentBrush}" Margin="0,0,7,0">
    <Grid>
        <Image x:Name="InfoBaseLogoImage" Stretch="UniformToFill" Visibility="Collapsed"/>
        <TextBlock x:Name="InfoBaseIconText" FontSize="13"
                   HorizontalAlignment="Center" VerticalAlignment="Center"
                   Foreground="White"/>
    </Grid>
</Border>
```

**Логика (OnLoaded, строка 109):**
```csharp
LogoDisplayHelper.Apply(InfoBaseLogoImage, InfoBaseIconText, _currentInfoBase.LogoImage, _currentInfoBase.DisplayIcon);
```

**Механизм:** Аналогично SystemLogoImage:
1. Если `InfoBase.LogoImage` (byte[]) есть → показывается изображение
2. Если нет → показывается `InfoBase.DisplayIcon` (строка emoji)
3. Если icon не задан → fallback "🏢"

### CurrentInfoBaseText

**XAML:**
```xml
<TextBlock x:Name="CurrentInfoBaseText" Text="Рабочий режим" 
           FontSize="11" Foreground="{DynamicResource AppBrandSubtitleForegroundBrush}" 
           VerticalAlignment="Center"/>
```

**Назначение:** Отображение названия текущей информационной базы.

**Логика (OnLoaded, строки 106-110):**
```csharp
_currentInfoBase = await _infoBaseManager.GetCurrentInfoBaseAsync();
if (_currentInfoBase != null)
{
    CurrentInfoBaseText.Text = _currentInfoBase.Name;
    // ...
    this.Title = $"{systemConfiguration.SystemName} - {_currentInfoBase.Name}";
}
```

**По умолчанию** (до загрузки): `"Рабочий режим"`
**После загрузки:** название информационной базы (например, `"Предприятие ОСОО Пример"`)
**Также обновляет Title окна:** `"BIS ERP - Название системы - Имя инфобазы"`

---

## Модели данных

### SystemConfiguration (откуда берутся SystemLogoImage/SystemIconText/SystemNameText)
```csharp
public class SystemConfiguration {
    public Guid Id { get; set; }
    public string SystemName { get; set; }      // → SystemNameText
    public byte[]? LogoImage { get; set; }      // → SystemLogoImage
    public string? Icon { get; set; }           // → SystemIconText (emoji)
    // ...
}
```

### InfoBase (откуда берутся InfoBaseLogoImage/InfoBaseIconText/CurrentInfoBaseText)
```csharp
public class InfoBase {
    public Guid Id { get; set; }
    public string Name { get; set; }            // → CurrentInfoBaseText + Title
    public byte[]? LogoImage { get; set; }      // → InfoBaseLogoImage
    public string? Icon { get; set; }           // → InfoBaseIconText (emoji)
    public string DisplayIcon => string.IsNullOrWhiteSpace(Icon) ? "🏢" : Icon;
    // ...
}
```

---

## Вспомогательные сервисы

### LogoDisplayHelper
```csharp
public static class LogoDisplayHelper
{
    public static void Apply(Image image, TextBlock fallbackText, 
        byte[]? imageBytes, string? fallbackIcon)
}
```
Универсальный helper для отображения логотипа: если есть изображение → показываем Image, иначе → TextBlock c иконкой-emoji.

### LogoFileService
```csharp
public static class LogoFileService
{
    public static ImageSource? CreateImageSource(byte[]? imageBytes)
}
```
Конвертирует массив байт в `BitmapImage` для WPF Image.Source. Возвращает null, если данные пустые.

### ThemeService
```csharp
public static class ThemeService
{
    public static string GetThemeToggleIcon(string theme)  // Light → 🌙, Dark → ☀️
    public static string GetThemeToggleToolTip(string theme)
    public static void ApplyTheme(string themeName)
}
```
Управление темами: переключение ResourceDictionary приложения, сохранение в AppSettings.

---

## Схема данных

```
SystemConfigurationService.GetAsync()
    │
    ├── SystemConfiguration.SystemName  → SystemNameText
    ├── SystemConfiguration.LogoImage   → LogoFileService → SystemLogoImage
    └── SystemConfiguration.Icon        → (если нет изображения) → SystemIconText
                                            Fallback: "🏢"
InfoBaseManager.GetCurrentInfoBaseAsync()
    │
    ├── InfoBase.Name                   → CurrentInfoBaseText
    ├── InfoBase.LogoImage              → LogoFileService → InfoBaseLogoImage
    └── InfoBase.Icon                   → (если нет изображения) → InfoBaseIconText
                                            Fallback: "🏢"
AppSettings.Instance.Theme
    │
    ├── "Light" → ThemeToggleButton.Content = 🌙, ToolTip = "Сменить тему"
    └── "Dark"  → ThemeToggleButton.Content = ☀️, ToolTip = "Сменить на светлую"
```