# Clone 克隆

继承`IClonable`, 并实现`Clone`方法
> PS: 此处我使用序列化和反序列化的方式实现了**深克隆**

```csharp
public class Table : ICloneable
{
    /// <summary>
    /// 浅克隆
    /// </summary>
    /// <returns></returns>
    public object Clone() => MemberwiseClone();

    /// <summary>
    /// 深克隆
    /// 通过序列化实现
    /// </summary>
    /// <returns></returns>
    public object DeepClone()
    {
        var jsonStr = JsonConvert.SerializeObject(this);
        return JsonConvert.SerializeObject(jsonStr);
    }


    public string Name = "Table";
}
```

为了更方便使用, 建立`CloneableExtensions`

```csharp
public static class CloneableExtensions
{
    public static T Clone<T>(this ICloneable self) => (T) self.Clone();
}
```

示例

```csharp
static class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello World!");

        var table = new Table();

        // this return Object, not good
        var table1 = table.Clone();

        // this return Table
        var table2 = table.Clone<Table>();
    }
}
```
