# 全局异常处理和本地化

提供以下特征：

> 1. 全局异常处理静态类`E`，允许调用此类的方法`E.OopsOh`抛出`FriendlyException`
> 2. 本地化提供依赖注入和全局静态类`L`，允许调用此类的方法`Lang.Text["name", args]`以提供本地化(读取`Json`文件)
> 3. 微软有默认的[`StringLocalizer<TResourceSource>`](https://docs.microsoft.com/zh-cn/dotnet/api/microsoft.extensions.localization.stringlocalizer-1?view=dotnet-plat-ext-5.0)，但读取的文件是`.resx`文件。

### 🚧 施工中...

## 一、本地化 Localization

> PS：这里实现的本地化，更多意义上只是读取文件内容，然后按`Dictionary<Key,Value>`键值对取值。是一种取(阉)巧(割)

目录结构：

    |-- Localization
        |-- Lang.cs
        |-- LocalizationSetting.cs
        |-- Resource.cs
        |-- UnifyLocalizer.cs
        |-- UnifyLocalizerFactory.cs

<!-- tabs:start -->

### **Lang**

> 全局静态类 提供`Lang.Text["name", args]`

```csharp
  public static class Lang
  {
      public static readonly IStringLocalizer @Text = new UnifyLocalizer(LocalizationSettings.Instance);
  }
```

### **LocalizationSetting**

>  相关的配置项

```csharp
  public class LocalizationSettings
  {
      /// <summary>
      /// 资源路径
      /// </summary>
      public string LangPath { get; set; } = "Lang";

      public static readonly LocalizationSettings Instance = new ();

      private LocalizationSettings()
      {
      }
  }
```

### **Resource**

>  静态获取`Json`。与异常处理共用

```csharp
  public static class Resources
  {
      private static readonly ILogger Logger = LoggerContext.For(typeof(Resources));

      private static readonly ConcurrentDictionary<string, Dictionary<string, string>> InternalResourcesCache = new();

      public static Dictionary<string, string> Get()
      {
          InternalResourcesCache.TryGetValue(CultureInfo.CurrentCulture.Name, out var dictionary);

          return dictionary ?? new Dictionary<string, string>();
      }

      public static void Search(string searchPath)
      {
          const string resourceFile = "*.json";

          var searchLocation = Path.Combine(AppContext.BaseDirectory, searchPath);

          if (!Directory.Exists(searchLocation))
          {
              Logger.Warn("No lang files found!");
              return;
          }

          var files = Directory.GetFiles(searchLocation, resourceFile, SearchOption.AllDirectories);

          if (files is {Length: 0})
          {
              Logger.Warn("No lang files found!");
              return;
          }

          foreach (var file in files)
          {
              var fileName = Path.GetFileNameWithoutExtension(file);
              var culture = Path.GetExtension(fileName).TrimStart('.');

              var content = File.ReadAllText(file, Encoding.UTF8);
              if (string.IsNullOrWhiteSpace(content))
              {
                  throw new FileLoadException($"{file} is null");
              }

              var dictionary = JsonSerializer.Deserialize<Dictionary<string, string>>(content);
              if (dictionary is not null)
              {
                  // 按culture分类
                  if (InternalResourcesCache.ContainsKey(culture))
                  {
                      foreach (var (key, value) in dictionary)
                      {
                          InternalResourcesCache[culture].TryAdd(key, value);
                      }
                  }
                  else
                  {
                      InternalResourcesCache.TryAdd(culture, dictionary);
                  }
              }
          }
      }
  }
```

### **UnifyLocalizer**

> 继承官方`IStringLocalizer` 为了支持依赖注入

```csharp
  public class UnifyLocalizer : IStringLocalizer
  {
      private readonly string _searchedLocation;

      public UnifyLocalizer(LocalizationSettings localizationSettings)
      {
          _searchedLocation = Path.Combine(AppContext.BaseDirectory, localizationSettings.LangPath);
          Resources.Search(localizationSettings.LangPath);
      }


      [Obsolete("This method is obsolete. Use `CurrentCulture` and `CurrentUICulture` instead.")]
      public IStringLocalizer WithCulture(CultureInfo culture) => this;

      public LocalizedString this[string name]
      {
          get
          {
              if (name is null)
              {
                  throw new ArgumentNullException(nameof(name));
              }

              var value = GetStringSafely(name);

              return new LocalizedString(name, value ?? name, value is null, _searchedLocation);
          }
      }

      public LocalizedString this[string name, params object[] arguments]
      {
          get
          {
              if (name is null)
              {
                  throw new ArgumentNullException(nameof(name));
              }

              var format = GetStringSafely(name);
              var value = string.Format(format ?? name, arguments);

              return new LocalizedString(name, value, format is null, _searchedLocation);
          }
      }

      public IEnumerable<LocalizedString> GetAllStrings(bool includeParentCultures)
      {
          var resourceNames = includeParentCultures
              ? GetAllStringsFromCultureHierarchy()
              : GetAllResourceStrings();

          foreach (var name in resourceNames)
          {
              var value = GetStringSafely(name);
              yield return new LocalizedString(name, value ?? name, value is null, _searchedLocation);
          }
      }

      private IEnumerable<string> GetAllStringsFromCultureHierarchy()
      {
          var currentCulture = CultureInfo.CurrentCulture;
          var resourceNames = new HashSet<string>();

          while (!currentCulture.Equals(currentCulture.Parent))
          {
              var cultureResourceNames = GetAllResourceStrings();
              foreach (var resourceName in cultureResourceNames)
              {
                  resourceNames.Add(resourceName);
              }

              currentCulture = currentCulture.Parent;
          }

          return resourceNames;
      }

      private static IEnumerable<string> GetAllResourceStrings()
      {
          var resources = Resources.Get();
          return resources.Select(r => r.Key);
      }

      private static string? GetStringSafely(string name)
      {
          if (name is null)
          {
              throw new ArgumentNullException(nameof(name));
          }

          var resources = Resources.Get();
          resources.TryGetValue(name, out var value);

          return value;
      }
  }
```

### **UnifyLocalizerFactory**

>  继承官方`IStringLocalizerFactory`。为了支持依赖注入

```csharp
  public class UnifyLocalizerFactory : IStringLocalizerFactory
  {
      public IStringLocalizer Create(Type resourceSource) => Lang.Text;

      public IStringLocalizer Create(string baseName, string location) => Lang.Text;
  }
```

<!-- tabs:end -->
