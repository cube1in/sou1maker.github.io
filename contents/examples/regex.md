# Regex

:unicorn: [`在线测试`](https://regex101.com/)

<!-- tabs:start -->

### **常用正则**

**1. 匹配中国大陆手机号码的正则表达式**

[`更多...`](https://github.com/VincentSit/ChinaMobilePhoneNumberRegex/blob/master/README-CN.md)

```text
^(?:\+?86)?1(?:3\d{3}|5[^4\D]\d{2}|8\d{3}|7(?:[235-8]\d{2}|4(?:0\d|1[0-2]|9\d))|9[0-35-9]\d{2}|66\d{2})\d{6}$
```

**2. 无限嵌套括号的正则表达式**

周围组合一些递归逻辑来实现这一点

此正则表达式将匹配嵌套三层深度的开括号和近似括号，如{a{b{c}}}{{{d}e}f}

```text
\{((?:\{(?:\{.*?\}|.)*?\}|.)*?)\}
```

![nestgegex](https://user-images.githubusercontent.com/58240137/117565394-89bf1c00-b0e3-11eb-9ef1-f72ef749ab20.png)

虚线区域是基本搜索，其中搜索嵌套在自身内部，以满足您的需要。

在下面的示例中，我只是针对大多数示例运行正则表达式。

将此正则表达式与`foreach`循环组合，该循环将获取每个Group 1并从当前字符串`^[^{]*`的开头捕获所有非开放括号，然后通过上面的正则表达式递归地返回其余字符串以捕获内部值下一组括号，然后从字符串`[^}]*$`的末尾捕获所有非近括号

示例文本：

```text
{a}
{a:b}
{a:{b}}
{a:{b:c}}
{a}{b}
{a}{b}{c}
{a{b{c}}}{{{d}e}f}
```
**声明**

此表达式仅适用于第三级递归。外部文本需要单独处理。 .net正则表达式引擎确实提供递归计数，并且可以支持N层深度。如此处所写，此表达式可能无法按照`g`中的预期处理捕获`{a:{b}g{h}i}`。


### **Extensions**

**1. ReplaceAsync**

```csharp
  public static async Task<string> ReplaceAsync(this Regex regex, string input, Func<Match, Task<string>> replacementFn)
  {
    var sb = new StringBuilder();
    var lastIndex = 0;

    foreach (Match match in regex.Matches(input))
    {
      sb.Append(input, lastIndex, match.Index - lastIndex)
      .Append(await replacementFn(match).ConfigureAwait(false));

      lastIndex = match.Index + match.Length;
    }

    sb.Append(input, lastIndex, input.Length - lastIndex);
    return sb.ToString();
  }
```

<!-- tabs:end -->
