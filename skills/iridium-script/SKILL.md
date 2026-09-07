---
name: iridium-script
description: 为 Iridium（Minecraft 1.20.1 Forge 客户端）编写 JavaScript 脚本，涵盖模块/HUD/命令注册、事件监听、Nashorn 风格 Java 互操作，以及 IridiumMCP 调试工具链。在以下任一情况使用：用户提到 Iridium 或客户端脚本；要求注册模块、HUD 或聊天命令；涉及 %APPDATA%\scripts 下的 .js 文件；当前工作目录位于 %APPDATA%\scripts；或用户只描述功能需求（如"帮我写个自动冲刺"）而未提及 Iridium，但上下文表明是该客户端。
---

## 简介

Iridium 是一个 Minecraft 1.20.1 客户端，基于Forge运行，带有一个基于Java Script的脚本执行器

脚本可以注册模块、设置项、HUD 元素和聊天命令，监听游戏事件，直接调用 Java API。

## 开始之前

确保以下材料已经获得：

- `types.d.ts`: 所有脚本API、Minecraft API大致定义，大部分为Minecraft API，你需要至少阅读Client API章节，Minecraft Types章节供速查
- `IridiumMCP`: 提供一些工具帮助你开发

如果用户未提供`types.d.ts`，通过`https://iridium.styles.wtf/assets/types.d.ts`下载，而不是要求用户提供。

所有的API定义都在`types.d.ts`务必看

文件位置：

| 用途 | 路径 |
|---|---|
| 脚本 | `%APPDATA%\scripts\*.js` |
| 本地库 | `%APPDATA%\scripts\.libs\*.js` |

## MCP

MCP 提供 Harness 和客户端交互的接口，对开发有很大帮助。

即使客户端没打开，MCP也可以连接因为MCP和客户端是分离的，你可以通过`ping`工具确保客户端在线。

如果你无法使用IridiumMCP，按照以下步骤安装或者排查

### 未安装MCP

下载`https://iridium.styles.wtf/assets/IridiumMCP.exe`到当前目录，不要放在临时目录不然下次可能找不到

根据任意方式配置，添加到你的harness mcp配置里，并且尽可能自主完成配置，如果有权限问题等实在无法自主完成的事情再找用户操作。

安装完毕之后用下方文案告知用户（根据用户的提示词语言翻译）：

```
┌─ IridiumMCP Server Setup Complete ────────────────────┐
│  ✓ Config  <path>                                    │
│  ✓ MCPs    <path>                                    │
│                                                      │
│  ⚡ Restart your agent to load the MCP servers       │
└──────────────────────────────────────────────────────┘
```

#### StreamHTTP

MCP会打开在`http://127.0.0.1:6767/mcp`

如果`6767`端口被占用应使用

`IridiumMCP.exe -http 127.0.0.1:6768`

#### stdio

`IridiumMCP.exe -stdio`

### 无法使用MCP

- 先判断是不是HTTP URL写错了，正确的URL应该包含`/mcp`，如果不包含`/mcp`就是错误的
- 重点检查Claude Code，Open Code等工具的settings代理配置，是否忘记配置NO_PROXY，放行127.0.0.1

## 库

先读取`reference/library.md`有一些预制的库目录，如果你认为用户的诉求可能需要这个库，使用`get_library`工具获得源码以便确认用法，然后用`client.import`来导入，示例：

```js
var clip = client.import("cubk/clipboard");
log("import ok, keys: " + Object.keys(clip).join(", "));
log("write: " + clip.write("MARKER-8964"));
```

## 开发流程

1. 理解用户诉求，创建，还是修改现有脚本
2. 尝试连接MCP，并使用`ping`工具确保客户端在线，如果客户端不在线告知用户需要启动客户端才能使用MCP，当然没有MCP也不是不能写，只是调试会很难
3. 阅读`types.d.ts`理解客户端API
4. 找到文件目录，寻找用户要修改的脚本或者创建脚本，可以使用MCP`list_scripts`工具快速查看
5. 编写测试版脚本，确保所有逻辑跑通，期间可以通过`eval`，`get_errors`，`grep_logs`，`get_logs`工具调试，使用`log()`接口输出的的日志将可以通过`grep_logs`和`get_logs`获得，优先使用`grep_logs`来节省上下文
6. 如果用户编写的是视觉功能，通过`screenshot`来获得游戏内截图确认效果
7. 修改脚本之后使用`reload_script`来重载，客户端**不会自动重载**
8. 使用`get_library`可以查看某个library源码
9. 确保所有功能跑通之后要求用户确认符合诉求
10. 用户如果提出了进一步修改意见，则修改
11. 如果用户确认符合诉求则问用户是否要清理调试代码等，清理掉调试代码等以便用户使用

## 脚本的结构

一个脚本就是一个 `.js` 文件，加载时**整体执行一次**：先注册模块和设置，
再用 `events.on(...)` 挂行为：

```js
client.registerModule("Auto Sprint", -1, "Movement", false);
client.registerSlider("Auto Sprint", "Delay", 5, 1, 20, 1);

events.on("tick", function () {
    if (!client.isEnabled("Auto Sprint")) return;
    // ...
});
```

脚本被自动预处理包在一个 IIFE 里执行（`(function(){ ...你的代码... })()`），所以顶层的 `var` 和
`function` 都是私有的，不同脚本之间不会互相覆盖。要跨脚本共享代码请用库

分类请用这六个之一：`Combat` `Movement` `Render` `World` `Player` `Inventory`。

一个脚本可以注册多个模块、命令、HUD。

## Java调用

引擎提供 Nashorn 风格的 Java 互操作。`mc` 以及从它拿到的一切（玩家、实体、物品、世界）
都是 Java 对象，直接调它们的方法就行：

```js
mc.player.getX();
mc.player.getMainHandItem().getCount();
mc.level.getBlockState(pos).getBlock();
```

### 取类

`Java.type()` 拿到一个类，可以用来 `new`、读静态字段、调静态方法：

```js
var BlockPos = Java.type("net.minecraft.core.BlockPos");
var Items = Java.type("net.minecraft.world.item.Items");
var Component = Java.type("net.minecraft.network.chat.Component");

var pos = new BlockPos(100, 64, 200);          // 构造
var stick = Items.STICK;                        // 静态字段
var text = Component.literal("hello");          // 静态方法
```

用点号写全限定名（`net.minecraft.core.BlockPos`），不是斜杠。
数组类型写 `"int[]"`、`"java.lang.String[]"`。

**内部类（嵌套类、枚举）用 `$` 分隔，不是点。** 引擎直接把名字交给
`Class.forName`，不会自动把点换成 `$`，写错会报 `Java class not found`：

```js
Java.type("com.mojang.blaze3d.vertex.VertexFormat$Mode");        // 对
Java.type("net.minecraft.world.entity.Entity$RemovalReason");     // 对
Java.type("com.mojang.blaze3d.vertex.VertexFormat.Mode");         // 错，找不到类
```

枚举常量当静态字段取：

```js
var Mode = Java.type("com.mojang.blaze3d.vertex.VertexFormat$Mode");
var quads = Mode.QUADS;
```

`Java.type` 的结果会缓存，重复调用不会有额外开销，可以放心写在事件回调里，
不过放到脚本顶层更清晰。

### 名字映射

引擎在查字段和方法时会自己做重映射，所以**使用 Mojang 名**（`getX`、`getMainHandItem`）

如果你确定你写了正确的Mojang名仍然有问题，可以先使用`eval`进行探测，因为映射器不是100%能映射所有字段和方法。

类名同理，写 `net.minecraft.core.BlockPos` 即可。

### 重载和类型转换

参数会按 Java 方法签名自动转换：JS 数字转 `int`/`float`/`double`，字符串转 `String`，
JS 函数可以传给函数式接口（单方法接口）。有重载时引擎按参数个数和类型挑一个匹配的。

拿不准某个方法存不存在、返回什么，用`eval`当场试：

```js
mc.player.getMainHandItem().getItem().toString()
```

### 数组

Java 数组比集合友好，有 `.length`，也能用下标：

```js
var arr = someJavaMethodReturningArray();
arr.length;
arr[0];
```

但**集合不是数组**，见下面的提示一节。

### 类型判断

```js
var Player = Java.type("net.minecraft.world.entity.player.Player");
if (Java.isType(entity, Player)) { ... }
```

`instanceof` 不按 Java 类型工作（它走 JS 原型链），判断 Java 类型一律用 `Java.isType`。
`Java.isJavaObject(x)` 判断一个值是不是 Java 对象。

## 提示

一些脚本系统的规范等

### Java 集合不是 JS 数组

`mc.level.players()` 返回的是 `java.util.List`，**没有 `.length`**。写
`for (var i = 0; i < list.length; i++)` 时条件是 `0 < undefined`，循环一次都不执行，
而且**不报错**——这是最难发现的一类 bug。

```js
var players = mc.level.players();
players.length                   // undefined
players.size(); players.get(0)   // 可以
Java.from(players).length        // 可以，转成真正的 JS 数组
```

`Java.from()` 接受 Java 数组和任何 `Iterable`（List/Set/Collection）。要遍历 `Map` 得先
`.entrySet()` 或 `.values()`。

### 设置项 id 永远是 `"模块名:设置名"`

注册时是两个参数，读取时是一个拼接的字符串：

```js
client.registerSlider("My Module", "Speed", 1, 0, 5, 0.1);
client.getNumber("My Module:Speed");
```

HUD 的设置项不属于模块本身，必须用 `client.hudProperty(hud, label)` 生成 id，不要手拼。

### HUD 必须回写 `ctx.width` / `ctx.height`

不写的话元素没有碰撞箱，用户在 HUD 编辑模式里**拖不动也点不着**，但画面看起来是正常的。

```js
client.registerHud("Coords", { anchor: "Left Bottom", x: 4, y: 4 }, function (graphics, ctx) {
    var text = "X " + Math.round(mc.player.getX());
    render.drawString(graphics, text, ctx.x, ctx.y, 0xFFFFFFFF);
    ctx.width = render.getStringWidth(text);   // 必须
    ctx.height = render.fontHeight();          // 必须
});
```

### 3D 标签要传实体，不要传坐标

传实体时引擎会用 `partialTicks` 做插值，实体移动时标签跟得住；自己传 `getX()` 拿到的是
tick 末的位置，标签会抖动、和实体渲染位置对不上。

```js
render.drawTag("§e" + p.getName().getString(), p, 0.4, { autoScale: true });   // 推荐
render.drawTag("§aSpawn", 100.5, 65, 200.5, {});                               // 固定位置才这么写
```

第三个参数是在实体身高之上的额外偏移，碰撞箱高度引擎已经算进去了。

### 颜色是 ARGB，alpha 在最前

`0xFF55FF55` 是不透明的绿色。漏掉 alpha 写成 `0x55FF55`，alpha 就是 0，画出来完全透明——
看起来像"没画出来"。

文本颜色代码用 `§`（U+00A7），例如 `"§c红色"`、`"§l粗体"`。


### 模块开关要自己判断

注册模块**不会**让事件只在开启时触发。每个回调都要自己检查：

```js
events.on("tick", function () {
    if (!client.isEnabled("My Module")) return;
    // ...
});
```

也可以监听 `enable` / `disable` 事件做初始化和清理。

当然，你可以写其他模块的附属功能，开启某个其他已有模块才执行的逻辑。

## 事件

三类，写法不同：

**通知型**——只是告诉你发生了什么：

```js
events.on("tick", function () { ... });
events.on("render3d", function (partialTicks) { ... });
```

**可取消型**——`return true` 阻止默认行为：

```js
events.on("chat", function (text) {
    return text === "secret";     // true = 不发送
});
```

**可修改型**——收到一个对象，改它的字段：

```js
events.on("movementInput", function (e) { e.jump = true; });
events.on("fov", function (e) { e.factor = 1.5; });
```

可修改型事件里 `return` 是没用的，必须写字段。哪个事件属于哪类、字段叫什么，
`types.d.ts` 里每条 `on(...)` 重载都有注明。

渲染分两个事件：`render2d` 画屏幕 UI（给你 `graphics`），`render3d` 画世界内容
（`render.drawTag` / `render.drawBox` 只能在这里调）。