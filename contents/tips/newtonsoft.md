# Newtonsoft

<!-- tabs:start -->

### **JsonPath Expression**

|  字符  |  描述  |
| -- | -- |
| $ | 根目录 |
| @ | 当前对象 |
| .. | 递归下沉 |
| [] | 下标操作符 |
| * | 通配符 |
| ?() | 过滤条件 |

```scharp
var jTokens = jo.SelectTokens("..$ref");
```

### **JsonSerializerSettings**

**Newtonsoft 序列化配置:**

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

<!-- tabs:end -->
