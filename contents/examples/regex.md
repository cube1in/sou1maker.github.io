# Regex

<!-- tabs:start -->

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
