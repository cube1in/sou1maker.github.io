# JsonFormatter

**简述：** 在项目过程中遇到`Json`字符串，可能会出现格式很杂乱/没有缩进的情况

> 当然，你也可以使用*Newtonsoft*中的`JObject`对`json`文件进行处理（这样会多一个超大的依赖）

<!-- tabs:start -->

### **JsonFormatter**

> `JsonFormatter`

```csharp
public static class JsonFormatter
{
    private const string Indent = "  ";

    public static string PrettyPrint(string input)
    {
        var output = new StringBuilder(input.Length * 2);
        char? quote = null;
        int depth = 0;

        for (int i = 0; i < input.Length; ++i)
        {
            char ch = input[i];

            switch (ch)
            {
                case '{':
                case '[':
                    output.Append(ch);
                    if (!quote.HasValue)
                    {
                        output.AppendLine();
                        output.Append(Indent.Repeat(++depth));
                    }

                    break;
                case '}':
                case ']':
                    if (quote.HasValue)
                        output.Append(ch);
                    else
                    {
                        output.AppendLine();
                        output.Append(Indent.Repeat(--depth));
                        output.Append(ch);
                    }

                    break;
                case '"':
                case '\'':
                    output.Append(ch);
                    if (quote.HasValue)
                    {
                        if (!output.IsEscaped(i))
                            quote = null;
                    }
                    else quote = ch;

                    break;
                case ',':
                    output.Append(ch);
                    if (!quote.HasValue)
                    {
                        output.AppendLine();
                        output.Append(Indent.Repeat(depth));
                    }

                    break;
                case ':':
                    if (quote.HasValue) output.Append(ch);
                    else output.Append(" : ");
                    break;
                default:
                    if (quote.HasValue || !char.IsWhiteSpace(ch))
                        output.Append(ch);
                    break;
            }
        }

        return output.ToString();
    }
}
```

### **StringExtension**

> `StringExtension`

```csharp
public static class StringExtensions
{
    public static string Repeat(this string str, int count)
    {
        return new StringBuilder().Insert(0, str, count).ToString();
    }

    public static bool IsEscaped(this string str, int index)
    {
        bool escaped = false;
        while (index > 0 && str[--index] == '\\') escaped = !escaped;
        return escaped;
    }

    public static bool IsEscaped(this StringBuilder str, int index)
    {
        return str.ToString().IsEscaped(index);
    }
}
```

<!-- tabs:end -->
