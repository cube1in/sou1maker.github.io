# 含泛型反射

假设先在需要通过反射调用实现了`IMagrationTask`的类

```csharp
  public interface IMigrationTask
  {
      ValueTask Up<TDbContext>(TDbContext context) where TDbContext : DbContext;

      ValueTask Down<TDbContext>(TDbContext context) where TDbContext : DbContext;
  }
```

那么可以使用以下这种方法

```csharp
typeof(IMigrationTask).GetMethod("Up", BindingFlags.Static | BindingFlags.NonPublic)!
                      .MakeGenericMethod(dbContextType)
                      .Invoke(null, new object[] {dbContext});
```

或者使用以下取巧

```csharp
  public abstract class MigrationTask<TDbContext> : IDataMigrationTask where TDbContext : DbContext
  {
      protected abstract ValueTask Up(TDbContext context);

      protected abstract ValueTask Down(TDbContext context);

      #region [IDataMigrationTask]

      ValueTask IDataMigrationTask.Up(DbContext context)
      {
          return Up((TDbContext) context);
      }

      ValueTask IDataMigrationTask.Down(DbContext context)
      {
          return Down((TDbContext) context);
      }

      #endregion
  }

  internal interface IDataMigrationTask
  {
      public ValueTask Up(DbContext context);

      public ValueTask Down(DbContext context);
  }
```

在使用时

```csharp
  var dataMigration = (IDataMigrationTask) Activator.CreateInstance(dataMigrationTaskType)!;
  await dataMigration.Up(currentContext);
```

