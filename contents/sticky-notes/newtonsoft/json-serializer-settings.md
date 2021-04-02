# JsonSerializerSettings

> Newtonsoft 序列化配置

```csharp
  public static readonly JsonSerializerSettings JsonSerializerSettings = new JsonSerializerSettings
  {
      //忽略循环引用
      ReferenceLoopHandling = ReferenceLoopHandling.Ignore,
      DefaultValueHandling = DefaultValueHandling.Ignore,
      TypeNameHandling = TypeNameHandling.Auto,
      SerializationBinder = new CustomSerializationBinder(typeof(Dashboard_V2).Assembly),
      //驼峰样式的key
      ContractResolver = new CamelCasePropertyNamesContractResolver(),
      Converters = new List<JsonConverter>
          {
              new DataSetConverter(),
              new DataTableConverter(),
              new ExpressionConverter(),
              new IntervalValueConverter(),
              new LookupConverter<string, string>(),
          },
  };
```
