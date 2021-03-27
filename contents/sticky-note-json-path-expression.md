# JsonPath Expression

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
