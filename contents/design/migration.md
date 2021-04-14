# 迁移 Migration

### 🚧 施工中...

> 迁移(Migration) or 升级(Upgrade)，可以分为两部分：1. 结构迁移  2. 数据迁移

微软里的*Entity Framework Core*提供了默认的 [Migration](https://docs.microsoft.com/zh-cn/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli)

提供以下特征：

1. 基础库静态类，不适用依赖注入(提供可复用行)
2. 管理类`MigrationMonitor`中的`MigrationAsync`为`DbContext`的拓展方法(简易调用)
3. 适用*Microsoft.NET.Sdk*，而不是*Microsoft.NET.Sdk.Web*(基础库)
4. 全局反射获取继承了`MigrationTask`的类，从而调用类中的`Up`和`Down`方法

#### 基础库`Infrastructure.DatabaseMigration`

目录结构：

    |-- DatabaseMigration
        |-- MigrationHistory
            |-- DatabaseMigrationContext.cs
            |-- DatabaseMigrationFactory.cs
            |-- MigrationHistory.cs
        |-- MigrationMonitor.cs
        |-- MigrationTask.cs
        |-- DataMigrationAttribute.cs


<!-- tabs:start -->

### **MigrationMonitor**

> Migration 管理器

```csharp
  public static class MigrationMonitor
  {
      private static readonly Dictionary<string, HashSet<(string, Type)>> PendingMigration = new();

      private static readonly DatabaseMigrationDbContext DatabaseMigrationDbContext;

      static MigrationMonitor()
      {
          var databaseMigrationDbContextFactory = new DatabaseMigrationDbContextFactory();
          DatabaseMigrationDbContext = databaseMigrationDbContextFactory.CreateDbContext(Array.Empty<string>());
      }

      public static ValueTask MigrateAsync(this DbContext dbContext)
      {
          // TODO:
          throw new NotImplementedException();
      }

      private static async ValueTask InternalMigrateAsync(this DbContext dbContext, string? targetMigration = null,
          CancellationToken cancellationToken = default)
      {
          await dbContext.GetInfrastructure().GetService<IMigrator>()!.MigrateAsync(targetMigration,
              cancellationToken);
      }

      private static async ValueTask SetMigrationHistory(string migrationId)
      {
          var migrationHistory = new MigrationHistory
          {
              MigrationId = migrationId
          };

          await DatabaseMigrationDbContext.MigrationHistories.AddAsync(migrationHistory);
          await DatabaseMigrationDbContext.SaveChangesAsync();
      }
  }
```

### **MigrationTask**

> 编写数据迁移类时需要继承的基类，用于给`MigraionMonitor`通过反射调用`Up`方法和`Down`方法

```csharp
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
  [AttributeUsage(AttributeTargets.Class)]
  public class DataMigrationAttribute : Attribute
  {
      public string TargetMigration { get; }

      public string MigrationId { get; }

      /// <summary>
      /// 数据迁移
      /// </summary>
      /// <param name="targetMigration">所属结构迁移类的名称</param>
      /// <param name="migrationId">自身Id(唯一)</param>
      public DataMigrationAttribute(string targetMigration, string migrationId)
      {
          TargetMigration = targetMigration;
          MigrationId = migrationId;
      }
  }
```

### **DatabaseMigrationContext**

> 不用多说，*EF Core*内容

```csharp
  public class DatabaseMigrationDbContext : DbContext
  {
      public DatabaseMigrationDbContext(DbContextOptions<DatabaseMigrationDbContext> options)
          : base(options)
      {
      }

      protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
      {
          optionsBuilder.UseSqlite("Data Source = ./migration_history.db");
      }

      protected override void OnModelCreating(ModelBuilder modelBuilder)
      {
      }

      public DbSet<MigrationHistory> MigrationHistories { get; set; } = null!;
  }
```

### **DatabaseMigrationFactory**

> 这里不使用依赖注入的方式获取`DbContext`，所以通过这种方法去创建

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

### **MigrationHistory**

```csharp
  public class MigrationHistory
  {
      /// <summary>
      /// 迁移Id
      /// </summary>
      [Key] 
      public string MigrationId { get; set; } = null!;

      /// <summary>
      /// 版本
      /// TODO: // 需要转换
      /// </summary>
      public Version Version { get; set; } = new Version();

      /// <summary>
      /// 迁移描述
      /// </summary>
      public string? Description { get; set; }
  }
```

<!-- tabs:end -->
