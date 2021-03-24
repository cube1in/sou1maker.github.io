# [protobuf-net](https://github.com/protobuf-net/protobuf-net)

**前提：** 微软提供了在序列化过程中可回调的`Attribute`。 被这些`Attribute`标记的方法, 会在序列化/反序列化过程中被回调

[[OnSerializing]](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.serialization.onserializingattribute?view=net-5.0)

[[OnSerialized]](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.serialization.onserializedattribute?view=net-5.0)

[[OnDeserializing]](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.serialization.ondeserializingattribute?view=net-5.0)

[[OnDeserialized]](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.serialization.ondeserializedattribute?view=net-5.0)



**产生的问题：** protobuf-net 对继承于 `ICollection`/`IEnumerable`/`IDictionary` 等的类无法触发这些`Attribute`

示例代码：

> 自定义实现`IDictionary<string, string?>`类

```csharp
    [Serializable]
    [ProtoContract]
    public class ExampleDictionary : IDictionary<string, string?>
    {

        [OnDeserializing]
        internal void OnDeserializing(StreamingContext context)
        {
        }

        [OnDeserialized]
        internal void OnDeserialized(StreamingContext context)
        {
        }

        [OnSerializing]
        internal void OnSerializing(StreamingContext context)
        {
        }

        [OnSerialized]
        internal void OnSerialized(StreamingContext context)
        {
        }

        // 继承 IDictionary<string, string?> 时需要实现的方法省略，在文章最后给出。并不是重点
    }
```

> 调用类`Program`

```csharp
    static class Program
    {
        static void Main(string[] args)
        {
            var obj = new ExampleDictionary {{"hello", "world"}};
            
            // 关闭自动编译，增加可调试能力。正式环境中不需要写
            ProtoBuf.Meta.RuntimeTypeModel.Default.AutoCompile = false;

            var ms = new MemoryStream();

            ProtoBuf.Serializer.Serialize(ms, obj); // 序列化
            ms.Position = 0;
            var s1 = ProtoBuf.Serializer.Deserialize<ExampleDictionary>(ms); // 反序列化
            Console.WriteLine(s1);
        }
    }
```

##### 这种情况下, 被`Attribute`标记的方法并不会被回调.

> 原因是在序列化过程中, protobuf-net会生成两个`MetaType`, 一个是`ICollection<KeyValuePair<string,string>>`,
> 另一个是`ExampleDictionary`(也就是我们自定义类)

![image](https://user-images.githubusercontent.com/58240137/112269591-8696da80-8cb3-11eb-9024-12bd99087ff8.png)


> 查看`MetaType`类, 发现`BuildSerializer`方法(也就是说, 将每一个MetaType都Build一个Serializer对其进行处理)

![image](https://user-images.githubusercontent.com/58240137/112269862-e2f9fa00-8cb3-11eb-8351-55a49c976313.png)

![image](https://user-images.githubusercontent.com/58240137/112269928-ff963200-8cb3-11eb-8458-e4770ab0479e.png)

> 专门查看下对于`ExampleDictionary`这个`MetaType`所Build出来的Serializer, 发现是名为`TypeSerializer`

![image](https://user-images.githubusercontent.com/58240137/112271135-84ce1680-8cb5-11eb-9ab9-8c700a3f2bf6.png)

PS: 其实因为ExampleDictionary类没有给属性添加`[ProtoMember]`Attribute, 所以无法Read和Write

> 继续找下去, 发现`Callback`方法, 这里就会调用我们设置的回调`OnDeserializing`等Attribute

![image](https://user-images.githubusercontent.com/58240137/112271685-33725700-8cb6-11eb-8418-64bd02f030bf.png)

> 之所以没有回调, 是因为`this.callbacks`为空。而为空的原因正是在`TypeSerializer`的构造函数中传入的callbacks为空导致的。

![image](https://user-images.githubusercontent.com/58240137/112272051-a54aa080-8cb6-11eb-91ae-fe3443d00fa8.png)

##### 也就是说在`MetaType`执行`BuildSerializer`方法时, 没有传入`callbacks`

![image](https://user-images.githubusercontent.com/58240137/112272319-02deed00-8cb7-11eb-9fb1-049c69a779dc.png)

> 修改源码, 传入`callbacks`即可(后续下载源码修改/打包等操作不再说明)

![image](https://user-images.githubusercontent.com/58240137/112272428-24d86f80-8cb7-11eb-8d7f-128b730f4bcb.png)


<br/>

<br/>

<br/>

<br/>

##### 附文：完整`ExampleDictionary`

```csharp
    [Serializable]
    [ProtoContract]
    public class ExampleDictionary : IDictionary<string, string?>
    {
        
        [OnDeserializing]
        internal void OnDeserializing(StreamingContext context)
        {
        }

        [OnDeserialized]
        internal void OnDeserialized(StreamingContext context)
        {
        }

        [OnSerializing]
        internal void OnSerializing(StreamingContext context)
        {
        }

        [OnSerialized]
        internal void OnSerialized(StreamingContext context)
        {
        }

        private readonly Dictionary<string, string?> _store = new();

        #region [默认实现]

        public IEnumerator<KeyValuePair<string, string?>> GetEnumerator() => _store.GetEnumerator();

        IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

        public void Add(KeyValuePair<string, string?> item) =>
            this[item.Key] = item.Value;

        public void Clear() => _store.Clear();

        bool ICollection<KeyValuePair<string, string?>>.Contains(KeyValuePair<string, string?> item) =>
            ((ICollection<KeyValuePair<string, string?>>) _store).Contains(item);

        void ICollection<KeyValuePair<string, string?>>.CopyTo(KeyValuePair<string, string?>[] array, int arrayIndex) =>
            ((ICollection<KeyValuePair<string, string?>>) _store).CopyTo(array, arrayIndex);

        bool ICollection<KeyValuePair<string, string?>>.Remove(KeyValuePair<string, string?> item) =>
            ((ICollection<KeyValuePair<string, string?>>) _store).Remove(item);

        public int Count => _store.Count;
        public bool IsReadOnly => true;

        public void Add(string key, string? value) => _store.Add(key, value);

        public bool ContainsKey(string key) => _store.ContainsKey(key);

        public bool Remove(string key) => _store.Remove(key);

        public bool TryGetValue(string key, out string? value) => _store.TryGetValue(key, out value);

        public string? this[string key]
        {
            get => _store[key];
            set => _store[key] = value;
        }

        public ICollection<string> Keys => _store.Keys;
        public ICollection<string?> Values => _store.Values;

        #endregion
    }
```



