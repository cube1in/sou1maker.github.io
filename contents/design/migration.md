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
        |-- MigrationHistory
            |-- DatabaseMigrationDbContext.cs
            |-- DatabaseMigrationDbContextFactory.cs
            |-- DatabaseMigrationHistory.cs
        |-- DbContextExtensions.cs
        |-- MigrationTask.cs
        |-- DataMigrationAttribute.cs


<!-- tabs:start -->

### **DbContextExtensions**

> 作为`DbContext`的拓展，提供`MigrateAsync`方法

```csharp
  public static class DbContextExtensions
  {
      /// <summary>
      /// 执行迁移
      /// </summary>
      /// <param name="dbContext">需要执行迁移的DbContext</param>
      /// <returns></returns>
      public static async ValueTask MigrateAsync(this DbContext dbContext)
      {
          if (!MigrationEnabled()) return;

          // 创建DatabaseMigrationDbContext(使用using执行完后释放)
          var databaseMigrationDbContextFactory = new DatabaseMigrationDbContextFactory();
          await using var migrationDbContext =
              databaseMigrationDbContextFactory.CreateDbContext(new string[] { });

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
                      var migrationHistory = new DatabaseMigrationHistory
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
          static async ValueTask<bool> IsMigrationTaskRequired(DatabaseMigrationDbContext migrationDbContext,
              string migrationId)
          {
              await migrationDbContext.InternalMigrateAsync();

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
                  return Environment.GetEnvironmentVariable("DATABASE_AUTO_UPGRADE")?.ToUpper() == "TRUE" &&
                         Environment.GetEnvironmentVariable("DATA_AUTO_UPGRADE")?.ToUpper() == "TRUE";

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
                      u.BaseType != null && u.BaseType.IsGenericType &&
                      u.BaseType.GetGenericTypeDefinition() == typeof(MigrationTask<>));

              foreach (var migrationTaskType in migrationTaskTypes)
              {
                  var dataMigrationAttribute = migrationTaskType.GetCustomAttribute<DataMigrationAttribute>(true);

                  if (dataMigrationAttribute != null && !dataMigrationAttribute.Obsolete)
                  {
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
          await dbContext.GetInfrastructure().GetService<IMigrator>()
              .MigrateAsync(targetMigration: targetMigration, cancellationToken: cancellationToken);
      }
  }
```

### **MigrationTask**

> 编写数据迁移类时需要继承的基类，用于给`MigraionMonitor`通过反射调用`Up`方法和`Down`方法

```csharp
  /// <summary>
  /// 数据迁移基类
  /// </summary>
  /// <typeparam name="TDbContext"></typeparam>
  public abstract class MigrationTask<TDbContext> where TDbContext : DbContext
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

### **DataMigrationAttribute**

> 编写数据迁移类时需要标记上的`Attribute`，用于给`MigraionMonitor`通过反射获取到数据迁移类所属的结构迁移和自身*MigrationId*

```csharp
  public class DataMigrationAttribute : Attribute
  {
      /// <summary>
      /// 数据迁移Id
      /// <example>
      /// 20210415022531_TestContext_UpdateNameMigrationTask_1
      /// </example>
      /// </summary>
      public string MigrationId { get; }

      /// <summary>
      /// 是否过时
      /// 如果为true则不会执行此数据升级所属的结构升级，也不会执行此数据升级
      /// </summary>
      public bool Obsolete { get; }

      /// <summary>
      /// 所属结构的文件名
      /// <example>
      /// 20210415063152_TestDbContext_init
      /// </example>
      /// </summary>
      public string[] EfMigrationIds { get; }

      /// <param name="migrationId">自身Id(唯一)</param>
      /// <param name="obsolete">是否过时</param>
      /// <param name="efMigrationIds">所属结构的文件名</param>
      public DataMigrationAttribute(string migrationId, bool obsolete, params string[] efMigrationIds)
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
          Obsolete = obsolete;
          EfMigrationIds = efMigrationIds;
      }
  }
```

### **DatabaseMigrationDbContext**

> 不用多说，*EF Core*内容

```csharp
  public class DatabaseMigrationDbContext : DbContext
  {
      public DatabaseMigrationDbContext(DbContextOptions<DatabaseMigrationDbContext> options)
          : base(options)
      {
      }

      protected override void OnModelCreating(ModelBuilder modelBuilder)
      {
          modelBuilder.Entity<DatabaseMigrationHistory>()
              .Property(q => q.ProductVersion)
              .HasConversion(new VersionToStringConverter());
      }

      public DbSet<DatabaseMigrationHistory> MigrationsHistories { get; set; } = null!;
      
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

### **DatabaseMigrationDbContextFactory**

> 这里不使用依赖注入的方式获取`DbContext`，所以通过这种方法去创建

特别提醒：继承`IDesignTimeDbContextFactory`是在使用命令`dotnet ef migrations add xxx`增加迁移时，*ef core*将会读取目录是否有这个类，若无，则无法增加迁移

```csharp
  public class DatabaseMigrationDbContextFactory : IDesignTimeDbContextFactory<DatabaseMigrationDbContext>
  {
      public DatabaseMigrationDbContext CreateDbContext(string[] args)
      {
          var builder = new DbContextOptionsBuilder<DatabaseMigrationDbContext>();

          builder.UseSqlite("Data Source = ./migration_history.db");

          return new DatabaseMigrationDbContext(builder.Options);
      }
  }
```

### **DatabaseMigrationHistory**

```csharp
  /// <summary>
  /// 迁移历史
  /// </summary>
  [Table("__MigrationsHistory")]
  public class DatabaseMigrationHistory
  {
      /// <summary>
      /// 迁移Id
      /// </summary>
      [Key]
      public string MigrationId { get; set; } = null!;

      /// <summary>
      /// 产品版本
      /// </summary>
      public Version ProductVersion { get; set; } = new (4, 0, 0, 0);


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

<!-- tabs:end -->
