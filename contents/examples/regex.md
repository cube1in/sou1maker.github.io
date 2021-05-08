# Regex

:unicorn: [`在线测试`](https://regex101.com/)

<!-- tabs:start -->

### **常用**

#### 1. 匹配中国大陆手机号码的正则表达式.

[`更多...`](https://github.com/VincentSit/ChinaMobilePhoneNumberRegex/blob/master/README-CN.md)

```text
^(?:\+?86)?1(?:3\d{3}|5[^4\D]\d{2}|8\d{3}|7(?:[235-8]\d{2}|4(?:0\d|1[0-2]|9\d))|9[0-35-9]\d{2}|66\d{2})\d{6}$
```

### **Extensions**

> ReplaceAsync

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
