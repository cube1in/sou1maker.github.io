# 桥接模式(BridgeMode)

提供`IJsonSerializerProvider`接口

```csharp
public interface IJsonSerializerProvider
{
    string Serialize(object value, object jsonSerializerOptions = default);

    T Deserialize<T>(string json, object jsonSerializerOptions = default);
}
```

默认实现`DefaultSerializerProvider`

```csharp
public class DefaultSerializerProvider : IJsonSerializerProvider
{
    public string Serialize(object value, object jsonSerializerOptions = default)
    {
        return JsonSerializer.Serialize(value, jsonSerializerOptions as JsonSerializerOptions);
    }

    public T Deserialize<T>(string json, object jsonSerializerOptions = default)
    {
        return JsonSerializer.Deserialize<T>(json, jsonSerializerOptions as JsonSerializerOptions);
    }
}
```

示例：

```csharp

// 主程序
class Program
{
    static void Main(string[] args)
    {
        DoSomething.Serializer = new NewtonSoftSerializerProvider();
        Console.WriteLine(DoSomething.Do());
    }

    // 继承于`IJsonSerializerProvider`的`Provider`
    class NewtonSoftSerializerProvider : IJsonSerializerProvider
    {
        public string Serialize(object value, object jsonSerializerOptions = default)
        {
            return JsonConvert.SerializeObject(value, jsonSerializerOptions as JsonSerializerSettings);
        }

        public T Deserialize<T>(string json, object jsonSerializerOptions = default)
        {
            return JsonConvert.DeserializeObject<T>(json, jsonSerializerOptions as JsonSerializerSettings);
        }
    }
}

// 用于处理的静态类, 提供默认实现, 但允许外部提供继承于`IJsonSerializerProvider`的`Provider`, 进行修改
public static class DoSomething
{
    public static IJsonSerializerProvider Serializer { get; set; } = new DefaultSerializerProvider();

    public static JsonObj Do()
    {
        var jsonStr = "{\"Text\":\"Hello World!\"}";
        return Serializer.Deserialize<JsonObj>(jsonStr);

        // do something
    }
}
```
