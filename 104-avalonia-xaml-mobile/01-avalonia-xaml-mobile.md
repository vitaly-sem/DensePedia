# Avalonia & Cross-Platform XAML

---

## Сравнение XAML-фреймворков

| Framework | Platforms | Language | Open Source | Performance | UI Thread | Best for |
|---|---|---|---|---|---|---|
| **WPF** | Windows | C# | ✅ (.NET Core) | Средняя | 1 | Desktop Windows |
| **Avalonia UI** | Win, Mac, Linux, iOS, Android, Browser | C# | ✅ (MIT) | Высокая | 1 | Cross-platform desktop |
| **.NET MAUI** | Win, Mac, iOS, Android | C# | ✅ (MIT) | Средняя | 1 | Mobile + Desktop |
| **Uno Platform** | Win, Web(Wasm), iOS, Android | C# | ✅ (Apache 2.0) | Средняя | 1 | WinUI → Web/Mobile |

---

## Avalonia UI — подробно

**Факты:**
- **Кроссплатформенный WPF** (Windows, macOS, Linux, iOS, Android, Web via Wasm)
- Свой render engine (SkiaSharp) — не зависит от DirectX/Windows
- WPF-совместимый XAML + расширения
- MVVM, Binding, Styles, DataTemplates — как в WPF
- **Avalonia 11** — стабильный, production-ready

```xml
<!-- Avalonia XAML — почти как WPF -->
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Avalonia App" Width="800" Height="600">
  <Grid>
    <Grid.RowDefinitions>
      <RowDefinition Height="Auto" />
      <RowDefinition Height="*" />
    </Grid.RowDefinitions>
    
    <TextBox Text="{Binding SearchText}" Watermark="Search..." Margin="10" />
    
    <ListBox Grid.Row="1" ItemsSource="{Binding Users}" Margin="10">
      <ListBox.ItemTemplate>
        <DataTemplate>
          <Border Padding="8" Margin="4" BorderBrush="Gray" BorderThickness="1"
                  CornerRadius="4">
            <TextBlock Text="{Binding Name}" />
          </Border>
        </DataTemplate>
      </ListBox.ItemTemplate>
    </ListBox>
  </Grid>
</Window>
```

**Avalonia — отличия от WPF:**

| Аспект | WPF | Avalonia |
|---|---|---|
| Rendering | DirectX | SkiaSharp (GPU/CPU) |
| Платформы | Windows | Win/Mac/Linux/iOS/Android/Web |
| Styles | XAML styles only | CSS-like selectors |
| Transitions | Code | XAML (built-in animations) |
| DevTools | Snoop, WPF Inspector | Avalonia DevTools (built-in) |
| Community | Large | Growing (~40K GitHub stars) |

**Факт:** Avalonia поддерживает **CSS-подобные стили**:

```xml
<Style Selector="Button.primary /template/ ContentPresenter">
  <Setter Property="Background" Value="Blue" />
</Style>
```

---

## .NET MAUI

**Факты:**
- Преемник Xamarin.Forms (с 2022)
- **Один проект**, один XAML → все платформы
- Hot Reload, MVVM, DI built-in
- **MAUI 8** — стабильный (с .NET 8)

```xml
<!-- MAUI — единый UI для всех платформ -->
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="MyApp.MainPage">
  
  <ScrollView>
    <VerticalStackLayout Padding="30" Spacing="10">
      <Label Text="Welcome to .NET MAUI!" FontSize="32" />
      
      <Entry Placeholder="Username" Text="{Binding Username}" />
      <Entry Placeholder="Password" IsPassword="True" />
      
      <Button Text="Login" Command="{Binding LoginCommand}" 
              BackgroundColor="Blue" TextColor="White" />
      
      <CollectionView ItemsSource="{Binding Items}">
        <CollectionView.ItemTemplate>
          <DataTemplate>
            <Label Text="{Binding Name}" />
          </DataTemplate>
        </CollectionView.ItemTemplate>
      </CollectionView>
    </VerticalStackLayout>
  </ScrollView>
</ContentPage>
```

```csharp
// MAUI lifecycle
public partial class MainPage : ContentPage
{
    protected override void OnAppearing() { base.OnAppearing(); /* Load data */ }
    protected override void OnDisappearing() { /* Cleanup */ }
}

// Platform-specific code
#if ANDROID
    // Android-specific
#elif IOS
    // iOS-specific
#elif WINDOWS
    // Windows-specific
#endif
```

**Ограничения MAUI:**
- Performance уступает нативным (но лучше Xamarin.Forms)
- Баги на релизе (Microsoft активно фиксит)
- Плагины/компоненты всё ещё в разработке
- Не все WPF/Xamarin библиотеки портированы

---

## Uno Platform

**Факты:**
- Запускает **WinUI XAML** на Web (Wasm) + Mobile
- **Ближе всех к WinUI 3** — можно портировать UWP приложения
- Использует SkiaSharp для rendering

```xml
<!-- Uno — WinUI совместимый XAML -->
<Page x:Class="MyApp.MainPage">
  <Grid>
    <NavigationView>
      <NavigationView.MenuItems>
        <NavigationViewItem Content="Home" Icon="Home" />
        <NavigationViewItem Content="Settings" Icon="Setting" />
      </NavigationView.MenuItems>
      
      <ScrollViewer>
        <TextBlock Text="WinUI on Web!" Style="{StaticResource TitleTextBlockStyle}" />
      </ScrollViewer>
    </NavigationView>
  </Grid>
</Page>
```

---

## Выбор фреймворка

| Сценарий | Лучший выбор |
|---|---|
| Desktop (Win/Mac/Linux) только | **Avalonia** |
| Mobile + Desktop | **.NET MAUI** |
| UWP/WinUI → Web | **Uno Platform** |
| WPF → Cross-platform | **Avalonia** (прямой перенос) |
| SPA Web apps | Blazor / React / Vue |

**Факт:** Avalonia имеет **WPF compatibility mode** — можно переиспользовать XAML код с минимальными изменениями.

---

## Чек-лист

- [ ] Avalonia: SkiaSharp, CSS-like selectors, Win/Mac/Linux/iOS/Android/Web
- [ ] MAUI: один проект на все платформы, MVVM, DI
- [ ] Uno: WinUI XAML на Web + Mobile
- [ ] WPF → cross-platform: Avalonia (проще миграция)
- [ ] Mobile + Desktop: MAUI
- [ ] WebAssembly: Uno or Avalonia
