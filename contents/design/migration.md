# 迁移 Migration

### 🚧 施工中...

> 迁移(Migration)：可以分为两部分：1. 结构迁移  2. 数据迁移

微软里的*Entity Framework Core*提供了默认的 [Migration](https://docs.microsoft.com/zh-cn/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli)

提供以下特征：

1. 基础库静态类，不适用依赖注入(提供可复用行)
2. 管理类`DbContextExtensions`中的`MigrationAsync`为`DbContext`的拓展方法(简易调用)
3. 使用*Microsoft.NET.Sdk*，而不是*Microsoft.NET.Sdk.Web*(基础库)
4. 全局反射获取继承了`MigrationTask`的类，从而调用类中的`Up`和`Down`方法

#### 基础库

> `Infrastructure.DatabaseMigration`

目录结构：

    |-- Infrastructure.DatabaseMigration
        |-- Extensions
            |-- DbContextCreateNewDbContextExtensions.cs
        |-- HistoryDbContext
            |-- MigrationDbContext.cs
            |-- MigrationHistory.cs
        |-- DbContextMigrateExtensions.cs
        |-- DataMigrationTask.cs
        |-- DataMigrationTaskAttribute.cs


<!-- tabs:start -->

### **DbContextMigrateExtensions**

> 作为`DbContext`的拓展，提供`MigrateAsync`方法

```csharp
  public static class DbContextMigrateExtensions
  {
      /// <summary>
      /// 执行迁移
      /// </summary>
      /// <param name="dbContext">需要执行迁移的DbContext</param>
      /// <returns></returns>
      public static async ValueTask MigrateAsync(this DbContext dbContext)
      {
          if (!MigrationEnabled()) return;

          // 创建 MigrationDbContext
          var migrationDbContext = dbContext.CreateNewDbContext<MigrationDbContext>();

          // 反射获取待办的MigrationTask
          var pendingMigrations = GetPendingMigrations();

          foreach (var (efMigrationIds, dataMigrations) in pendingMigrations)
          {
              // 结构迁移，14位时间戳数字排序(20210414100318)
              foreach (var efMigrationId in efMigrationIds.OrderBy(u => long.Parse(u.Substring(0, 14))))
              {
                  await dbContext.InternalMigrateAsync(efMigrationId);
              }

              // 数据迁移，14位时间戳数字排序(20210414100318)
              foreach (var (migrationId, dataMigrationType) in dataMigrations.OrderBy(u =>
                  long.Parse(u.Item1.Substring(0, 14))))
              {
                  if (await IsMigrationTaskRequired(migrationDbContext, migrationId))
                  {
                      var dataMigration = Activator.CreateInstance(dataMigrationType);
                      await (ValueTask) dataMigrationType.GetMethod("Up")!.Invoke(dataMigration,
                          new object[] {dbContext})!;

                      // 添加一条数据迁移记录
                      var migrationHistory = new MigrationHistory
                      {
                          MigrationId = migrationId,
                          ExecutionTime = DateTime.Now
                      };

                      await migrationDbContext.MigrationsHistories.AddAsync(migrationHistory);
                      await migrationDbContext.SaveChangesAsync();
                  }
              }
          }


          // 是否需要执行此数据迁移
          static async ValueTask<bool> IsMigrationTaskRequired(MigrationDbContext migrationDbContext,
              string migrationId)
          {
              await migrationDbContext.Database.EnsureCreatedAsync();
              var migrationsHistory =
                  migrationDbContext.MigrationsHistories.FirstOrDefault(u => u.MigrationId == migrationId);

              return migrationsHistory == null;
          }


          // 是否启用了环境变量
          static bool MigrationEnabled()
          {
#if DEBUG

              return true;

#else
                  return Environment.GetEnvironmentVariable("DB_MIGRATION_ENABLED")?.ToUpper() == "TRUE";

#endif
          }
      }


      /// <summary>
      /// 通过反射获取待办的MigrationTask
      /// </summary>
      /// <returns></returns>
      private static Dictionary<string[], HashSet<(string, Type)>> GetPendingMigrations()
      {
          // Dictionary<EfMigrationIds, HashSet<(MigrationId, DataMigrationTaskType)>
          var pendingMigrations = new Dictionary<string[], HashSet<(string, Type)>>();

          var migrationFileMatcher =
              new Microsoft.Extensions.FileSystemGlobbing.Matcher(StringComparison.OrdinalIgnoreCase);
          migrationFileMatcher.AddInclude("*Ef.Migrations.*.dll");

#if TEST
          migrationFileMatcher.AddInclude("Infrastructure.DatabaseMigration.Tests.dll");

#endif

          var migrationFiles = migrationFileMatcher
              .Execute(new DirectoryInfoWrapper(new DirectoryInfo(AppContext.BaseDirectory))).Files
              .Select(o => o.Path).ToList();

          foreach (var file in migrationFiles)
          {
              var assembly =
                  AssemblyLoadContext.Default.LoadFromAssemblyPath(Path.Combine(AppContext.BaseDirectory, file));

              var migrationTaskTypes =
                  assembly.ExportedTypes.Where(u =>
                      u.BaseType is {IsGenericType: true} &&
                      u.BaseType.GetGenericTypeDefinition() == typeof(DataMigrationTask<>));

              foreach (var migrationTaskType in migrationTaskTypes)
              {
                  var dataMigrationAttribute = migrationTaskType.GetCustomAttribute<DataMigrationTaskAttribute>(true);

                  if (dataMigrationAttribute != null)
                  {
#if DEBUG
                      // DEBUG模式下，检查是否存在该结构迁移
                      var structureMigrationAttributes = assembly.ExportedTypes
                          .Select(u => u.GetCustomAttribute<MigrationAttribute>());

                      if (dataMigrationAttribute.EfMigrationIds.Any(s =>
                          !structureMigrationAttributes.Any(u => u != null && u.Id == s)))
                      {
                          throw new ArgumentException("efMigrationId does not exist.");
                      }

#endif

                      if (!pendingMigrations.ContainsKey(dataMigrationAttribute.EfMigrationIds))
                      {
                          pendingMigrations.Add(dataMigrationAttribute.EfMigrationIds,
                              new HashSet<(string, Type)>());
                      }

                      pendingMigrations[dataMigrationAttribute.EfMigrationIds]
                          .Add((dataMigrationAttribute.MigrationId, migrationTaskType));
                  }
              }
          }

          return pendingMigrations;
      }


      /// <summary>
      /// 执行迁移(内部方法)
      /// </summary>
      /// <param name="dbContext"></param>
      /// <param name="targetMigration">目标Migration的文件名</param>
      /// <param name="cancellationToken"></param>
      /// <returns></returns>
      private static async ValueTask InternalMigrateAsync(this DbContext dbContext, string? targetMigration = null,
          CancellationToken cancellationToken = default)
      {
          await dbContext.GetInfrastructure().GetService<IMigrator>()!
              .MigrateAsync(targetMigration: targetMigration, cancellationToken: cancellationToken);
      }
  }
```

### **DataMigrationTask**

> 编写数据迁移类时需要继承的基类，用于给`DbContextMigrateExtensions`通过反射调用`Up`方法和`Down`方法

```csharp
  /// <summary>
  /// 数据迁移基类
  /// </summary>
  /// <typeparam name="TDbContext"></typeparam>
  public abstract class DataMigrationTask<TDbContext> where TDbContext : DbContext
  {
      /// <summary>
      /// 迁移
      /// </summary>
      /// <param name="dbContext"></param>
      /// <returns></returns>
      public abstract ValueTask Up(TDbContext dbContext);

      /// <summary>
      /// 回退
      /// </summary>
      /// <param name="dbContext"></param>
      /// <returns></returns>
      public abstract ValueTask Down(TDbContext dbContext);
  }
```

### **DataMigrationTaskAttribute**

> 编写数据迁移类时需要标记上的`Attribute`，用于给`DbContextMigrateExtensions`通过反射获取到数据迁移类所属的结构迁移和自身*MigrationId*

```csharp
  [AttributeUsage(AttributeTargets.Class)]
  public class DataMigrationTaskAttribute : Attribute
  {
      /// <summary>
      /// 数据迁移Id
      /// <example>
      /// 20210415022531_TestContext_UpdateNameMigrationTask_1
      /// </example>
      /// </summary>
      public string MigrationId { get; }

      /// <summary>
      /// 所属结构的文件名
      /// <example>
      /// 20210415063152_TestDbContext_init
      /// </example>
      /// </summary>
      public string[] EfMigrationIds { get; }

      /// <param name="migrationId">自身Id(唯一)</param>
      /// <param name="efMigrationIds">所属结构的文件名</param>
      public DataMigrationTaskAttribute(string migrationId, params string[] efMigrationIds)
      {
#if DEBUG
          var migrationIdTimeSpanStr = migrationId.Substring(0, 14);
          var efMigrationIdsTimeSpanStr = efMigrationIds.Select(u => u.Substring(0, 14)).ToList();
          // ReSharper disable once StringLiteralTypo
          var format = "yyyyMMddHHmmss";
          var cultureInfo = System.Globalization.CultureInfo.InvariantCulture;
          var dataTimeStyle = System.Globalization.DateTimeStyles.AdjustToUniversal;

          if (!long.TryParse(migrationIdTimeSpanStr, out _))
          {
              throw new ArgumentException($"migrationId: \"{migrationId}\" incorrect length.");
          }

          if (!DateTime.TryParseExact(migrationIdTimeSpanStr, format, cultureInfo, dataTimeStyle, out _))
          {
              throw new ArgumentException($"migrationId: \"{migrationId}\" timestamp format is incorrect.");
          }

          if (efMigrationIdsTimeSpanStr.Any(id => !long.TryParse(id, out _)))
          {
              throw new ArgumentException("efMigrationId incorrect length.");
          }

          if (efMigrationIdsTimeSpanStr.Any(id =>
              !DateTime.TryParseExact(id, format, cultureInfo, dataTimeStyle, out _)))
          {
              throw new ArgumentException("migrationId timestamp format is incorrect.");
          }

#endif

          MigrationId = migrationId;
          EfMigrationIds = efMigrationIds;
      }
  }
```

### **MigrationDbContext**

> 不用多说，*EF Core*内容，唯一需要注意的就是`Vsersion`与`String`直接的转换器

```csharp
  public class MigrationDbContext : DbContext
  {
      public MigrationDbContext(DbContextOptions<MigrationDbContext> options)
          : base(options)
      {
      }

      protected override void OnModelCreating(ModelBuilder modelBuilder)
      {
          modelBuilder.Entity<MigrationHistory>()
              .Property(q => q.ProductVersion)
              .HasConversion(new VersionToStringConverter());
      }

      public DbSet<MigrationHistory> MigrationsHistories { get; set; } = null!;

      private class VersionToStringConverter : ValueConverter<Version, string>
      {
          public VersionToStringConverter(ConverterMappingHints? mappingHints = null)
              : base(
                  m => m.ToString(),
                  s => Version.Parse(s)
                  , mappingHints)
          {
          }
      }
  }
```

### **MigrationHistory**

```csharp
  /// <summary>
  /// 数据迁移历史
  /// 需要和EfCore的__EFMigrationsHistory进行区分
  /// </summary>
  [Table("__MigrationsHistory")]
  public class MigrationHistory
  {
      /// <summary>
      /// 迁移Id
      /// </summary>
      [Key]
      public string MigrationId { get; set; } = null!;

      /// <summary>
      /// 产品版本
      /// </summary>
      public Version ProductVersion { get; set; } = new(4, 0, 0, 0);


      /// <summary>
      /// 执行时间
      /// </summary>
      public DateTime ExecutionTime { get; set; }

      /// <summary>
      /// 描述
      /// </summary>
      public string? Description { get; set; }
  }
```

### **DbContextCreateNewDbContextExtensions**

这个类时为了解决这两个问题而存在的：
> 1. 将`MigrationHistory`这个表建立到从`MigrateAsync(DbContext dbContext)`传入的`dbContext`中的数据库去(这里传入的数据库类型可能是多种多样的：`Postgrest`/`MySql`/`Oracle`等)。
> 2. 建立好`MigrationHistory`表之后，我们需要往表里添加数据。

方案：
> 1. 使用的是`MigrationDbContext`进行建表和添加数据操作。
> 2. 由于方案*1*，所以需要提取外部传入的`DbContext`中的`Options`。然后使用这个`Options`创建出`MigrationDbContext`。
> PS: 每个`DbContext`里的连接信息和使用数据库的信息都是由`Options`中的`Extensions`决定的，所以只要拿到了`Options`，就能创建出任何类型的数据库。

```csharp
  public static class DbContextCreateNewDbContextExtensions
  {
      private static readonly FieldInfo DbContextOptionsFieldInfo = typeof(DbContext).GetField("_options",
          BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.GetField)!;


      /// <summary>
      /// 从已有的 DbContext，创建一个新的类型的 DbContext
      /// </summary>
      /// <param name="context"></param>
      /// <typeparam name="TContext"></typeparam>
      /// <returns></returns>
      /// <exception cref="ArgumentNullException">给的 DbContext 没有 Options</exception>
      /// <exception cref="InvalidOperationException">没有合适的构造函数 DbContextOptions[TContext]</exception>
      public static TContext CreateNewDbContext<TContext>(this DbContext context) where TContext : DbContext
      {
          var options = DbContextOptionsFieldInfo.GetValue(context);
          if (!(options is DbContextOptions dbContextOptions))
          {
              throw new ArgumentNullException(nameof(context), "context.Options");
          }

          var builder = new DbContextOptionsBuilder<TContext>();

          var builderInfrastructure = (IDbContextOptionsBuilderInfrastructure) builder;
          foreach (var extension in dbContextOptions.Extensions)
          {
              builderInfrastructure.AddOrUpdateExtension(extension);
          }

          var constructor = typeof(TContext).GetConstructor(new[] {typeof(DbContextOptions<TContext>)});

          if (constructor == null)
          {
              throw new InvalidOperationException(
                  $"No suitable constructor found for DbContext type '${typeof(TContext).Name}'");
          }

          return (TContext) constructor.Invoke(new object?[] {builder.Options});
      }
  }
```

<!-- tabs:end -->
