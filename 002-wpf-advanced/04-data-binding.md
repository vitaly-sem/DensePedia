# Data Binding — продвинутые сценарии

---

## Binding Modes — когда что использовать

| Mode | Описание | Когда |
|---|---|---|
| `OneWay` | Источник → Цель | Данные только для чтения (TextBlock) |
| `TwoWay` | Источник ↔ Цель | Редактируемые поля (TextBox, CheckBox) |
| `OneTime` | Источник → Цель (один раз) | Статические данные, конфигурация |
| `OneWayToSource` | Цель → Источник | Специфические случаи (Sliders как input только в VM) |
| `Default` | Зависит от DP (TextBox = TwoWay, TextBlock = OneWay) | По умолчанию |

**Факт:** `UpdateSourceTrigger` определяет когда TwoWay binding пишет в источник:

| Trigger | Когда | Пример |
|---|---|---|
| `PropertyChanged` | При каждом изменении | Поиск, фильтрация |
| `LostFocus` | При потере фокуса | Формы (экономия запросов) |
| `Explicit` | Только при `BindingExpression.UpdateSource()` | Массовое сохранение |
| `Default` | Зависит от DP (TextBox = LostFocus) | По умолчанию |

---

## FallbackValue и TargetNullValue

```xml
<!-- FallbackValue — когда binding не может разрешиться (ошибка пути и т.д.) -->
<TextBlock Text="{Binding Total, FallbackValue='N/A', TargetNullValue='-'}" />

<!-- TargetNullValue — когда значение null -->
<TextBlock Text="{Binding Name, TargetNullValue='(no name)'}" />
```

**Факт:** `FallbackValue` показывается при ошибке binding (BindingExpression имеет HasError=true). `TargetNullValue` — когда источник вернул null.

---

## PriorityBinding

```xml
<TextBlock>
  <TextBlock.Text>
    <PriorityBinding>
      <!-- Пробует по порядку: первое успешное значение -->
      <Binding Path="FullName" />
      <Binding Path="UserName" FallbackValue="(no username)" />
      <Binding Path="Email" FallbackValue="(no email)" />
      <Binding Source="Unknown User" />
    </PriorityBinding>
  </TextBlock.Text>
</TextBlock>
```

**Факт:** PriorityBinding проверяет каждый Binding по очереди. Если первый не может разрешиться (FallbackValue), пробует следующий.

---

## MultiBinding

```csharp
public class FullNameConverter : IMultiValueConverter
{
    // values — массив значений от каждого Binding
    // parameter — ConverterParameter из XAML
    public object Convert(object[] values, Type targetType, object parameter, CultureInfo culture)
    {
        var firstName = values[0] as string ?? "";
        var lastName = values[1] as string ?? "";
        return $"{firstName} {lastName}".Trim();
    }
    
    public object[] ConvertBack(object value, Type[] targetTypes, object parameter, CultureInfo culture)
    {
        // Для TwoWay — разделить обратно
        var parts = (value as string)?.Split(' ') ?? new[] { "", "" };
        return new object[] { parts[0], parts.Length > 1 ? parts[1] : "" };
    }
}
```

```xml
<TextBlock>
  <TextBlock.Text>
    <MultiBinding Converter="{StaticResource FullNameConverter}">
      <Binding Path="FirstName" />
      <Binding Path="LastName" />
    </MultiBinding>
  </TextBlock.Text>
</TextBlock>
```

### MultiBinding + StringFormat

```xml
<!-- StringFormat для MultiBinding с использованием индексного синтаксиса -->
<TextBlock>
  <TextBlock.Text>
    <MultiBinding StringFormat="{}{0} — {1:C}">
      <Binding Path="ProductName" />
      <Binding Path="Price" />
    </MultiBinding>
  </TextBlock.Text>
</TextBlock>
```

**Факт:** StringFormat работает **только** если target type — `string`. Для TextBlock.Text — подходит, для IsEnabled — нет (используйте Converter).

---

## Binding Validation

### ValidationRules

```csharp
public class AgeValidationRule : ValidationRule
{
    public int Min { get; set; } = 0;
    public int Max { get; set; } = 150;
    
    public override ValidationResult Validate(object value, CultureInfo cultureInfo)
    {
        if (value is string str && int.TryParse(str, out int age))
        {
            if (age < Min || age > Max)
                return new ValidationResult(false, $"Age must be {Min}-{Max}");
            return ValidationResult.ValidResult;
        }
        return new ValidationResult(false, "Not a valid integer");
    }
}
```

```xml
<TextBox x:Name="ageBox">
  <TextBox.Text>
    <Binding Path="Age" UpdateSourceTrigger="PropertyChanged">
      <Binding.ValidationRules>
        <local:AgeValidationRule Min="0" Max="150" />
      </Binding.ValidationRules>
    </Binding>
  </TextBox.Text>
</TextBox>
<TextBlock Text="{Binding ElementName=ageBox, Path=(Validation.Errors)[0].ErrorContent}" 
           Foreground="Red" />
```

### INotifyDataErrorInfo (современная валидация)

```csharp
public class PersonViewModel : INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = new();
    
    public bool HasErrors => _errors.Count > 0;
    public event EventHandler<DataErrorsChangedEventArgs>? ErrorsChanged;
    
    public IEnumerable GetErrors(string? propertyName)
    {
        return propertyName != null && _errors.TryGetValue(propertyName, out var errors)
            ? errors
            : Enumerable.Empty<string>();
    }
    
    private void ValidateProperty(string propertyName)
    {
        _errors.Remove(propertyName);
        
        switch (propertyName)
        {
            case nameof(Age) when Age < 0 || Age > 150:
                AddError(propertyName, "Age must be 0-150");
                break;
            case nameof(Email) when !Email.Contains('@'):
                AddError(propertyName, "Invalid email");
                break;
        }
    }
    
    private void AddError(string propertyName, string error)
    {
        if (!_errors.ContainsKey(propertyName))
            _errors[propertyName] = new List<string>();
        _errors[propertyName].Add(error);
        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(propertyName));
    }
}
```

---

## Value Converters

### IValueConverter — лучшие практики

```csharp
// Аттрибут ValueConversion — для tooling поддержки
[ValueConversion(typeof(bool), typeof(Visibility))]
public class BoolToVisibilityConverter : IValueConverter
{
    // Параметр: invert=true инвертирует
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        var boolValue = (bool)value;
        
        if (parameter is string param && param.Equals("invert", StringComparison.OrdinalIgnoreCase))
            boolValue = !boolValue;
        
        if (targetType == typeof(Visibility))
            return boolValue ? Visibility.Visible : Visibility.Collapsed;
        
        if (targetType == typeof(Brush))
            return boolValue ? Brushes.Green : Brushes.Red;
        
        return boolValue;
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
        => throw new NotSupportedException();
}
```

### Converter-параметры и ресурсы

```xml
<!-- Параметр через ConverterParameter (статический) -->
<TextBlock Foreground="{Binding IsActive, Converter={StaticResource BoolToBrushConverter}, ConverterParameter=true}" />

<!-- Параметр через resource (для dynamic) -->
<TextBlock.Foreground>
  <Binding Path="Status">
    <Binding.Converter>
      <local:StatusToColorConverter>
        <local:StatusToColorConverter.ColorMappings>
          <local:ColorMapping Status="Active" Color="Green" />
          <local:ColorMapping Status="Inactive" Color="Gray" />
        </local:StatusToColorConverter.ColorMappings>
      </local:StatusToColorConverter>
    </Binding.Converter>
  </Binding>
</TextBlock.Foreground>
```

---

## Binding Expression — программный доступ

```csharp
// Получение BindingExpression
var expression = textBox.GetBindingExpression(TextBox.TextProperty);

// Обновление вручную (при UpdateSourceTrigger=Explicit)
expression?.UpdateSource();

// Проверка ошибок
if (expression?.HasError == true)
    var error = expression.ValidationError;

// Принудительное обновление binding цели (от VM к View)
expression?.UpdateTarget();

// Очистка binding
BindingOperations.ClearBinding(textBox, TextBox.TextProperty);
```

**Факт:** `UpdateTarget()` — форсирует перечитывание значения из источника. Полезно при `OneTime` binding после изменения источника.

**Факт:** `BindingOperations.GetBindingExpression` возвращает null, если binding не установлен.

---

## Binding Groups (GroupBox / Collection)

```xml
<!-- GroupBox с DataContext для вложенных элементов -->
<GroupBox Header="Address" DataContext="{Binding Address}">
  <StackPanel>
    <TextBox Text="{Binding Street}" />
    <TextBox Text="{Binding City}" />
  </StackPanel>
</GroupBox>
```

**Факт:** `{Binding}` без Path = DataContext целиком. Используется для вложенных объектов.

---

## Чек-лист

- [ ] OneWay vs TwoWay vs OneTime — когда что
- [ ] UpdateSourceTrigger: PropertyChanged, LostFocus, Explicit
- [ ] FallbackValue vs TargetNullValue
- [ ] PriorityBinding — fallback цепочка
- [ ] MultiBinding + IMultiValueConverter
- [ ] ValidationRules + INotifyDataErrorInfo
- [ ] IValueConverter с параметрами
- [ ] BindingExpression: UpdateSource, UpdateTarget, HasError
