# Templating в WPF — DataTemplate vs ControlTemplate

---

## DataTemplate vs ControlTemplate — ключевые различия

| | DataTemplate | ControlTemplate |
|---|---|---|
| **Что определяет** | Внешний вид **данных** | Внешний вид **элемента управления** |
| **TargetType** | Тип данных (модель) | Тип контрола (Button, ListBox) |
| **Binding** | `{Binding}` к свойствам данных | `{TemplateBinding}` к свойствам контрола |
| **Где используется** | `ItemTemplate`, `ContentTemplate` | `Template` (через Style или напрямую) |
| **Наследование** | Через `DataType` | Через `TargetType` |

---

## DataTemplate — отображение данных

```xml
<DataTemplate DataType="{x:Type models:Employee}">
  <Border BorderBrush="LightGray" BorderThickness="1" Padding="8" Margin="4">
    <Grid>
      <Grid.ColumnDefinitions>
        <ColumnDefinition Width="Auto" />
        <ColumnDefinition Width="*" />
      </Grid.ColumnDefinitions>
      
      <Ellipse Width="40" Height="40" Fill="{Binding AvatarColor}" />
      
      <StackPanel Grid.Column="1" Margin="8,0,0,0">
        <TextBlock Text="{Binding FullName}" FontWeight="Bold" />
        <TextBlock Text="{Binding Department}" FontStyle="Italic" Foreground="Gray" />
        <TextBlock Text="{Binding Email}" />
      </StackPanel>
    </Grid>
  </Border>
</DataTemplate>

<!-- Использование: автоматический выбор по типу -->
<ListBox ItemsSource="{Binding Employees}" />
<!-- WPF сам найдёт DataTemplate с DataType=Employee -->
```

**Факт:** Если DataTemplate не указан явно, WPF вызывает `ToString()` на объекте.

### DataTemplateSelector

```csharp
public class EmployeeTemplateSelector : DataTemplateSelector
{
    public DataTemplate? ManagerTemplate { get; set; }
    public DataTemplate? DeveloperTemplate { get; set; }
    public DataTemplate? DefaultTemplate { get; set; }
    
    public override DataTemplate SelectTemplate(object item, DependencyObject container)
    {
        return item switch
        {
            Manager _ => ManagerTemplate,
            Developer _ => DeveloperTemplate,
            _ => DefaultTemplate
        };
    }
}
```

```xml
<local:EmployeeTemplateSelector x:Key="TemplateSelector"
  ManagerTemplate="{StaticResource ManagerTemplate}"
  DeveloperTemplate="{StaticResource DeveloperTemplate}"
  DefaultTemplate="{StaticResource DefaultTemplate}" />

<ListBox ItemsSource="{Binding Employees}"
         ItemTemplateSelector="{StaticResource TemplateSelector}" />
```

---

## ControlTemplate — кастомизация контрола

```xml
<ControlTemplate x:Key="RoundButtonTemplate" TargetType="Button">
  <Border x:Name="border" 
          CornerRadius="8" 
          Background="{TemplateBinding Background}"
          BorderBrush="{TemplateBinding BorderBrush}" 
          BorderThickness="1"
          SnapsToDevicePixels="True">
    
    <ContentPresenter HorizontalAlignment="Center" 
                      VerticalAlignment="Center"
                      RecognizesAccessKey="True"
                      SnapsToDevicePixels="True" />
  </Border>
  
  <ControlTemplate.Triggers>
    <Trigger Property="IsMouseOver" Value="True">
      <Setter TargetName="border" Property="Background" Value="{StaticResource HoverBrush}" />
      <Setter TargetName="border" Property="BorderBrush" Value="DodgerBlue" />
    </Trigger>
    <Trigger Property="IsPressed" Value="True">
      <Setter TargetName="border" Property="Background" Value="{StaticResource PressedBrush}" />
      <Setter TargetName="border" Property="RenderTransform">
        <Setter.Value>
          <ScaleTransform ScaleX="0.97" ScaleY="0.97" />
        </Setter.Value>
      </Setter>
    </Trigger>
    <Trigger Property="IsEnabled" Value="False">
      <Setter TargetName="border" Property="Opacity" Value="0.5" />
    </Trigger>
  </ControlTemplate.Triggers>
</ControlTemplate>

<!-- Применение -->
<Button Template="{StaticResource RoundButtonTemplate}" Content="Click Me" />
```

**Факт:** `{TemplateBinding}` — это оптимизированный `{Binding RelativeSource={RelativeSource TemplatedParent}}`. Он легче, но не поддерживает `Converter`, `StringFormat`, `FallbackValue`.

### ControlTemplate через Style (рекомендуется)

```xml
<Style TargetType="Button">
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="Button">
        <!-- ... -->
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

### Theme-стили (Generic.xaml)

```csharp
// Для кастомного контрола — тема в Themes/Generic.xaml
[ThemeInfo(ResourceDictionaryLocation.None, ResourceDictionaryLocation.SourceAssembly)]
public class CustomControl : Control
{
    static CustomControl()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(CustomControl),
            new FrameworkPropertyMetadata(typeof(CustomControl)));
    }
}
```

**Факт:** WPF ищет `DefaultStyleKey` в `Themes/Generic.xaml`. Если стиль не найден — контрол не имеет визуального представления.

---

## ItemTemplate vs DisplayMemberPath

```xml
<!-- DisplayMemberPath — простое свойство (быстрее, но ограничено) -->
<ListBox ItemsSource="{Binding Employees}" DisplayMemberPath="FullName" />

<!-- ItemTemplate — полный контроль (гибче, но тяжелее) -->
<ListBox ItemsSource="{Binding Employees}">
  <ListBox.ItemTemplate>
    <DataTemplate>
      <StackPanel>
        <TextBlock Text="{Binding FullName}" />
        <TextBlock Text="{Binding Department}" />
      </StackPanel>
    </DataTemplate>
  </ListBox.ItemTemplate>
</ListBox>
```

**Факт:** `DisplayMemberPath` создаёт лёгкую обёртку. `ItemTemplate` — полноценный DataTemplate с VisualTree. Для > 1000 элементов — `DisplayMemberPath` или виртуализация.

---

## HierarchicalDataTemplate (TreeView)

```xml
<TreeView ItemsSource="{Binding Departments}">
  <TreeView.ItemTemplate>
    <HierarchicalDataTemplate ItemsSource="{Binding Employees}">
      <TextBlock Text="{Binding Name}" FontWeight="Bold" />
      <HierarchicalDataTemplate.ItemTemplate>
        <DataTemplate>
          <TextBlock Text="{Binding FullName}" />
        </DataTemplate>
      </HierarchicalDataTemplate.ItemTemplate>
    </HierarchicalDataTemplate>
  </TreeView.ItemTemplate>
</TreeView>
```

---

## Resources — область видимости

```xml
<!-- Application.Resources — видно всему приложению -->
<Application.Resources>
  <SolidColorBrush x:Key="AppBrush" Color="SteelBlue" />
</Application.Resources>

<!-- Window.Resources — видно только этому окну -->
<Window.Resources>
  <Style TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
    <Setter Property="Background" Value="{StaticResource AppBrush}" />
  </Style>
</Window.Resources>

<!-- Local (inline) — только этому элементу -->
<Grid.Resources>
  <local:MyConverter x:Key="MyConverter" />
</Grid.Resources>
```

**Факт:** Resource lookup: Element → Application → Theme → System. Если ресурс найден на уровне Window, он не будет искаться выше.

**Факт:** `StaticResource` — один раз при загрузке. `DynamicResource` — при каждом изменении (с подпиской на изменения). DynamicResource медленнее, но позволяет менять тему на лету.

---

## Чек-лист

- [ ] DataTemplate vs ControlTemplate — различие, TargetType
- [ ] DataTemplateSelector — множественные шаблоны для данных
- [ ] TemplateBinding vs Binding — ограничения
- [ ] DefaultStyleKey + Themes/Generic.xaml
- [ ] DisplayMemberPath vs ItemTemplate — производительность
- [ ] HierarchicalDataTemplate для TreeView
- [ ] Resource lookup: StaticResource vs DynamicResource
- [ ] StaticResource — быстрее, DynamicResource — гибче (темы)
