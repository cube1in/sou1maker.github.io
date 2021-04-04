# 插件热加载

> **Microsoft.CodeAnalysis.CSharp** [传送门](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp?view=roslyn-dotnet)

> 利用`AssemblyLoadContext`和`Microsoft.CodeAnalysis.CSharp`库实现插件热加载


#### 目录结构

    |-- HotReload
        |-- IPlugin.cs
        |-- PluginController.cs
        |-- Program.cs
        |-- guest
            |-- MyPlugin.cs

<!-- tabs:start -->

### **IPlugin**

> `IPlugin`继承`IDisposable`，提供销毁`Dispose`方法

```csharp
  public interface IPlugin : IDisposable
  {
      string GetMessage();
  }
```

<!-- tabs:end -->

<!-- tabs:start -->

### **PluginController**

> `PluginController`继承`IPlugin`

```csharp
  public class PluginController : IPlugin
  {
      private readonly string _pluginName;
      private readonly string _pluginDirectory;
      private List<Assembly> _defaultAssemblies;
      private AssemblyLoadContext _context;
      private object _reloadLock;
      private volatile bool _changed;
      private volatile IPlugin _instance;
      private FileSystemWatcher _watcher;

      public PluginController(string pluginName, string pluginDirectory)
      {
          _defaultAssemblies = AssemblyLoadContext.Default.Assemblies
              .Where(u => !u.IsDynamic)
              .ToList();

          _pluginName = pluginName;
          _pluginDirectory = pluginDirectory;
          _reloadLock = new object();
          ListenFileChanges();
      }

      private void ListenFileChanges()
      {
          Action<string> onFileChanged = path =>
          {
              if (Path.GetExtension(path).ToLower() == ".cs")
              {
                  _changed = true;
              }
          };

          _watcher = new FileSystemWatcher();
          _watcher.Path = _pluginDirectory;
          _watcher.IncludeSubdirectories = true;
          _watcher.NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.FileName;

          _watcher.Changed += (sender, e) => onFileChanged(e.FullPath);
          _watcher.Created += (sender, e) => onFileChanged(e.FullPath);
          _watcher.Deleted += (sender, e) => onFileChanged(e.FullPath);
          _watcher.Renamed += (sender, e) =>
          {
              onFileChanged(e.FullPath);
              onFileChanged(e.OldFullPath);
          };
          _watcher.EnableRaisingEvents = true;
      }

      private void UnloadPlugin()
      {
          _instance?.Dispose();
          _instance = null;

          _context?.Unload();
          _context = null;
      }

      private Assembly CompilePlugin()
      {
          var binDirectory = Path.Combine(_pluginDirectory, "bin");
          var dllPath = Path.Combine(binDirectory, $"{_pluginDirectory}.dll");

          if (!Directory.Exists(binDirectory))
          {
              Directory.CreateDirectory(binDirectory);
          }

          if (File.Exists(dllPath))
          {
              File.Delete($"{dllPath}.old");
              File.Move(dllPath, $"{dllPath}.old");
          }

          var sourceFiles = Directory.EnumerateFiles(
              _pluginDirectory, "*.cs", SearchOption.AllDirectories);

          var compilationOptions = new CSharpCompilationOptions(
              OutputKind.DynamicallyLinkedLibrary,
              optimizationLevel: OptimizationLevel.Debug);

          var references = _defaultAssemblies
              .Select(u => u.Location)
              .Where(path => !string.IsNullOrWhiteSpace(path) && File.Exists(path))
              .Select(path => MetadataReference.CreateFromFile(path))
              .ToList();

          var syntaxTrees = sourceFiles
              .Select(p => CSharpSyntaxTree.ParseText(File.ReadAllText(p)))
              .ToList();

          var compilation = CSharpCompilation.Create(_pluginName)
              .WithOptions(compilationOptions)
              .AddReferences(references)
              .AddSyntaxTrees(syntaxTrees);

          var emitResult = compilation.Emit(dllPath);

          if (!emitResult.Success)
          {
              throw new InvalidOperationException(string.Join("\r\n",
                  emitResult.Diagnostics.Where(d => d.WarningLevel == 0)));
          }

          // return _context.LoadFromAssemblyPath(Path.GetFullPath(dllPath));
          using (var stream = File.OpenRead(dllPath))
          {
              var assembly = _context.LoadFromStream(stream);
              return assembly;
          }
      }

      private IPlugin GetInstance()
      {
          var instance = _instance;
          if (instance != null && !_changed)
          {
              return instance;
          }

          lock (_reloadLock)
          {
              instance = _instance;

              if (instance != null && !_changed)
              {
                  return instance;
              }

              UnloadPlugin();

              _context = new AssemblyLoadContext(
                  name: $"Plugin-{_pluginName}", isCollectible: true);

              var assembly = CompilePlugin();

              var pluginType = assembly.GetTypes()
                  .First(t => typeof(IPlugin).IsAssignableFrom(t));

              instance = (IPlugin) Activator.CreateInstance(pluginType);

              _instance = instance;
              _changed = false;
          }

          return instance;
      }

      public string GetMessage()
      {
          return GetInstance().GetMessage();
      }

      public void Dispose()
      {
          UnloadPlugin();

          _watcher?.Dispose();
          _watcher = null;
      }
  }
```

<!-- tabs:end -->

<!-- tabs:start -->

### **MyPlugin**

> `MyPlugin`插件

```csharp
  public class MyPlugin:IPlugin
  {
      public MyPlugin()
      {
          Console.WriteLine("MyPlugin loaded");
      }

      public string GetMessage()
      {
          return "Hello there!";
      }

      public void Dispose()
      {
          Console.WriteLine("MyPlugin unloaded");
      }

  }
```

<!-- tabs:end -->

> `Program`演示

```csharp
  class Program
  {
      static void Main(string[] args)
      {
          using (var controller = new PluginController("MyPlugin", "./guest"))
          {
              bool keepRunning = true;
              Console.CancelKeyPress += (sender, e) =>
              {
                  e.Cancel = true;
                  keepRunning = false;
              };

              while (keepRunning)
              {
                  try
                  {
                      Console.WriteLine(controller.GetMessage());
                  }
                  catch (Exception ex)
                  {
                      Console.WriteLine($"{ex.GetType()}: {ex.Message}");
                  }

                  Thread.Sleep(1000);
              }
          }
      }
  }
```

#### 💡 在生成时可设置`MyPlugin`连同guest文件夹一起Copy到输出目录，这样在读取时更方便

![image](https://user-images.githubusercontent.com/58240137/113433966-bb710300-9412-11eb-9a69-49caa41211b9.png)

![image](https://user-images.githubusercontent.com/58240137/113433654-3685e980-9412-11eb-9dc6-7674cb9e5de8.png)


