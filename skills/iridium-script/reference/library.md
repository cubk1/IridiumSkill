# 预制库目录

用 `client.import("<库名>")` 导入。写代码前先用 MCP `get_library` 工具拉源码确认实际
接口，本表只说明用途，不保证方法签名。

| 库名 | 用途 |
|---|---|
| `cubk/stopwatch` | 计时器，用于计算时间间隔 |
| `cubk/clipboard` | 系统剪贴板读写 |

## 用法

```js
var clip = client.import("cubk/clipboard");
log("import ok, keys: " + Object.keys(clip).join(", "));
```

`Object.keys()` 打印出来的就是这个库暴露的接口，配合 `get_library` 看源码即可确认参数。
