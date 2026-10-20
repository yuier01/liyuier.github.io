---
title: 一种针对 OneBot 协议消息链特化的命令系统
titleTemplate: '%s - yuier'
date: 2026-10-20 18:38:17
updated: 2026-10-20 18:38:17
author: yuier
categories:
    - qq机器人
tags:
    - qq机器人
cover: 
hide: 
---

## 1. 概述

### 1.1 项目背景

[Yuni3](https://github.com/yuier01/Yuni3) 是一个基于 Spring Boot 的 OneBot 协议 QQ 机器人框架。命令系统是插件与用户交互的主要入口——用户在群聊或私聊中通过 `/` 开头的指令调用插件功能。

原有命令系统（`CommandModel` + `CommandMatcher`）存在以下局限：

- 不支持子命令嵌套
- 选项仅支持 0 或 1 个参数
- 不支持变长参数
- 无法约束选项的必填性
- 解析器缺乏回溯机制

新命令系统基于树形 `CommandNode` 结构，从根本上解决了上述问题。

### 1.2 设计目标

| 目标 | 说明 |
|---|---|
| **树形命令结构** | 支持任意深度的子命令嵌套（`/admin user ban <id>`） |
| **变长参数** | `variadic=true` 可消费多个 token，非贪婪停止于已知选项/子命令 |
| **多参数子命令** | 子命令可携带 0..N 个位置参数 |
| **选项独立建模** | 选项是 `Set<String>`，纯 token 无子结构，O(1) 匹配 |
| **回溯解析** | 子命令匹配失败时回溯，每个子节点从同一位置尝试 |
| **类型安全** | 匹配结果提供 `asText()` / `asImageList()` 等无需强制转型的方法 |

### 1.3 架构定位

```
                        ┌─────────────────────┐
                        │   CommandNode        │  模型层（yuni-event）
                        │   ArgDef             │  开发者构建命令树
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   CommandTokens      │  词法层（yuni-event）
                        │   tokenize + reply   │  文本拆分 + 回复抽离
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   CommandMatcher │  解析层（yuni-event）
                        │   CommandDetector│  递归下降 + 回溯
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   CommandResult      │  结果层（yuni-core）
                        │   ArgResult          │  类型化取值
                        └─────────────────────┘
```

---

## 2. 核心概念

### 2.1 概念定义

| 术语 | 定义 | 示例 |
|---|---|---|
| **命令（Command）** | 用户通过 `/` 前缀触发的完整指令 | `/git commit -m "fix"` |
| **命令树（Command Tree）** | 以 `CommandNode` 为节点的多叉树，描述命令的完整结构 | — |
| **节点（Node）** | 命令树中的一个元素。拥有名字、参数、选项和子命令 | `git`、`commit`、`-m` |
| **位置参数（Positional Arg）** | 按顺序消费的消息段。分必选和可选 | `add` 的 `files`、`push` 的 `remote` |
| **变长参数（Variadic Arg）** | 消费多个 token 的特殊位置参数，非贪婪 | `echo` 的 `message`、`upload` 的 `images` |
| **选项（Option）** | 预定义的裸标志。`Set<String>`，无参数无子结构 | `-f`、`-a`、`-O2` |
| **子命令（Child）** | 嵌套的 `CommandNode`，可携带自身的参数、选项和子命令 | `add`、`commit`、`-m` |
| **回溯（Backtracking）** | 匹配失败时恢复现场，尝试下一个候选的解析策略 | — |
| **Token** | 消息链经词法分析后产出的最小匹配单元 | `["git", "add", "file.txt"]` |

### 2.2 关键区分

**`-m` 是子命令，`-f` 是选项。** 区分标准：是否携带参数。

```
git commit -m "hello world"    ←  "-m" 有参数 → 是子命令
git push -f                    ←  "-f" 无参数 → 是选项
```

**选项与子命令在匹配方式上的不同**：

| | 选项 | 子命令 |
|---|---|---|
| 数据建模 | `Set<String>` | `CommandNode` |
| 参数 | 无 | 0..N 个 |
| 子节点 | 无 | 0..N 个 |
| 匹配方式 | O(1) Set 查找，恒成功 | 递归匹配，回溯 |
| 匹配优先级 | 高于子命令 | 低于选项 |

### 2.3 命令结构示意

```
CommandNode("gcc")
├── options:        {"-O2", "-Wall", "-g"}
├── children:
│   └── CommandNode("-o")
│       └── args:   [required("output")]
└── args:           [variadic("files")]
```

```
CommandNode("git")
├── requiresChild: true
└── children:
    ├── CommandNode("add")
    │   └── args:   [variadic("files")]
    ├── CommandNode("commit")
    │   ├── children:
    │   │   └── CommandNode("-m")── required ── args: [required("msg")]
    │   └── options: {"-a"}
    └── CommandNode("push")
        ├── options: {"-f"}
        └── args:   [required("remote"), optional("branch")]
```

---

## 3. 模型定义

### 3.1 CommandNode

```
包路径: com.yuier.yuni.event.detector.message.command.model
```

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `name` | `String` | — | 节点名。根命令、子命令均以此标识 |
| `description` | `String` | `""` | 帮助文本 |
| `args` | `List<ArgDef>` | `[]` | 位置参数列表 |
| `options` | `Set<String>` | `{}` | 选项（裸标志集合） |
| `children` | `List<CommandNode>` | `[]` | 子命令列表 |
| `permission` | `UserPermission` | `USER` | 所需权限。取值: MASTER(30), ADMIN(20), USER(10), BLOCKED(0) |
| `required` | `boolean` | `false` | 父节点是否要求本子命令必须匹配 |
| `requiresChild` | `boolean` | `false` | 是否要求至少一个选项或子命令被匹配 |

### 3.2 ArgDef

```
包路径: com.yuier.yuni.event.detector.message.command.model
```

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `name` | `String` | `""` | 参数名 |
| `description` | `String` | `""` | 参数描述 |
| `type` | `CommandArgRequireType` | `PLAIN` | 期望的消息段类型 |
| `required` | `boolean` | `true` | 是否必须 |
| `variadic` | `boolean` | `false` | 是否变长 |
| `defaultValue` | `String` | `null` | 默认值（仅可选参数有意义） |

**工厂方法**：

| 方法 | 返回 | 说明 |
|---|---|---|
| `required(name, desc)` | `ArgDef` | 必选 PLAIN 参数 |
| `required(name, desc, type)` | `ArgDef` | 必选，指定类型 |
| `optional(name, desc)` | `ArgDef` | 可选 PLAIN 参数 |
| `optional(name, desc, type)` | `ArgDef` | 可选，指定类型 |
| `optional(name, desc, defaultValue)` | `ArgDef` | 可选，含默认值 |
| `variadic(name, desc)` | `ArgDef` | 变长 PLAIN 参数（可选） |
| `variadic(name, desc, type)` | `ArgDef` | 变长参数（可选，指定类型） |
| `requiredVariadic(name, desc)` | `ArgDef` | 必选变长 PLAIN 参数（至少消费一个 token） |
| `requiredVariadic(name, desc, type)` | `ArgDef` | 必选变长参数（至少消费一个 token，指定类型） |

### 3.3 CommandTokens

```
包路径: com.yuier.yuni.event.detector.message.command
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `tokens` | `List<MessageSegment>` | 已 tokenize 的消息段列表（不含回复） |
| `reply` | `ReplySegment` | 已提取的回复消息段（可为 null） |

| 方法 | 返回 | 说明 |
|---|---|---|
| `hasReply()` | `boolean` | 是否存在回复消息段 |
| `get(index)` | `MessageSegment` | 取指定位置的 token |
| `size()` | `int` | token 数量 |
| `isEmpty()` | `boolean` | token 列表是否为空 |
| `subList(from, to)` | `List<MessageSegment>` | 子列表 |

**设计意图**：QQ 客户端强制回复消息段位于消息链之首且至多一个。开发者不可能以"回复在命令中可能出现的位置"来定义命令，因此 tokenize 阶段将回复单独抽离。解析器匹配 `REPLY` 类型参数时不从当前位置消耗 token，而是从 `CommandTokens.getReply()` 读取。

### 3.4 CommandArgRequireType

```
包路径: com.yuier.yuni.core.enums
```

| 枚举值 | 匹配的消息段类型 | 说明 |
|---|---|---|
| `PLAIN` | 文本（`TEXT`） | 任意文本 token |
| `NUMBER` | 文本（`TEXT`） | 必须匹配数字正则 `-?\d+(\.\d+)?` |
| `AT` | @ 消息段（`AT`） | QQ @ 某人 |
| `IMAGE` | 图片消息段（`IMAGE`） | 图片 |
| `URL` | 文本（`TEXT`） | 必须是合法 URL |
| `REPLY` | 回复消息段（`REPLY`） | 引用回复 |

### 3.5 参数排序规则

`CommandNode.Builder.build()` 在构建时自动对位置参数排序：

1. **必选参数在前**（按添加顺序）
2. **可选参数居中**（按添加顺序）
3. **变长参数在末**（最多一个）

开发者可以以任意顺序调用 `.arg()` 和 `.variadicArg()`，Builder 内部自动整理。唯一硬约束：最多定义一个变长参数，否则构建时抛 `IllegalArgumentException`。

**排序示例**：

```java
// 开发者乱序添加
CommandNode.builder("cmd")
    .arg(ArgDef.optional("c", "..."))
    .arg(ArgDef.required("a", "..."))
    .variadicArg("d", "...")
    .arg(ArgDef.optional("b", "..."))
    .build();

// build() 自动排序为: [required("a"), optional("c"), optional("b"), variadic("d")]
```

---

## 4. 匹配结果类型

### 4.1 CommandResult

```
包路径: com.yuier.yuni.core.event.matched
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `matchedKey` | `String` | 匹配到的节点名 |
| `matchSuccess` | `boolean` | 是否匹配成功 |
| `consumedCount` | `int` | 消耗的 token 数量 |
| `args` | `Map<String, ArgResult>` | 位置参数匹配结果（key = 参数名） |
| `options` | `Set<String>` | 匹配到的选项标志 |
| `children` | `Map<String, CommandResult>` | 子命令匹配结果（key = 子命令名） |

**查询方法**：

| 方法 | 返回 | 说明 |
|---|---|---|
| `hasArg(name)` | `boolean` | 是否有某位置参数 |
| `getArg(name)` | `ArgResult` | 获取位置参数结果 |
| `hasOption(name)` | `boolean` | 某选项是否被匹配 |
| `hasAnyOption()` | `boolean` | 是否有任何选项被匹配 |
| `hasChild(name)` | `boolean` | 某子命令是否被匹配 |
| `getChild(name)` | `CommandResult` | 获取子命令匹配结果 |
| `getChildNames()` | `Set<String>` | 所有匹配到的子命令名 |
| `hasAnyChild()` | `boolean` | 是否有任何子命令被匹配 |
| `hasAnyOptionOrChild()` | `boolean` | 是否有任何选项或子命令被匹配 |

### 4.2 ArgResult

```
包路径: com.yuier.yuni.core.event.matched
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | `String` | 参数名 |
| `type` | `CommandArgRequireType` | 参数类型 |
| `value` | `MessageSegment` | 单值参数的值（非变长时有效） |
| `values` | `List<MessageSegment>` | 变长参数的值列表（变长时有效） |
| `variadic` | `boolean` | 是否变长 |
| `isEmpty()` | `boolean` | 是否无值 |
| `size()` | `int` | 值的数量 |

**单值便捷方法**（变长参数调用时抛 `IllegalStateException`）：

| 方法 | 返回 | 说明 |
|---|---|---|
| `asText()` | `String` | 取 value 的文本内容 |
| `asImage()` | `ImageSegment` | 取 value 为 ImageSegment |
| `asAt()` | `AtSegment` | 取 value 为 AtSegment |
| `asNumber()` | `Long` | 取 value 解析为 Long |

**变长便捷方法**（非变长参数调用时抛 `IllegalStateException`）：

| 方法 | 返回 | 说明 |
|---|---|---|
| `asTextList()` | `List<String>` | 每个元素取文本 |
| `asImageList()` | `List<ImageSegment>` | 每个元素 cast 为 ImageSegment |
| `asAtList()` | `List<AtSegment>` | 每个元素 cast 为 AtSegment |
| `asNumberList()` | `List<Long>` | 每个元素解析为 Long |

**通用方法**（均可使用）：

| 方法 | 返回 | 说明 |
|---|---|---|
| `asList()` | `List<MessageSegment>` | 单值时返回单元素列表，变长时返回全部 |

**使用示例**：

```java
// 单值
String remote = r.getArg("remote").asText();
ImageSegment img = r.getArg("target").asImage();

// 变长
List<String> files = r.getArg("files").asTextList();
for (ImageSegment img : r.getArg("images").asImageList()) { ... }
```

---

## 5. Builder API

### 5.1 构造函数

```java
CommandNode.builder(String name) → Builder
```

`name` 为节点的唯一标识。以 `-` 开头是命名约定（如 `-m`、`-f`），不影响解析行为。

### 5.2 Builder 方法

| 方法 | 参数 | 说明 |
|---|---|---|
| `.description(String)` | 描述文本 | 设置帮助文本 |
| `.permission(UserPermission)` | 权限枚举 | 设置所需权限 |
| `.arg(ArgDef)` | 参数定义 | 添加位置参数（原始方式） |
| `.arg(String name, String desc)` | 参数名、描述 | 便捷：添加必选 PLAIN 参数 |
| `.variadicArg(String name, String desc)` | 参数名、描述 | 便捷：添加变长 PLAIN 参数 |
| `.variadicArg(String name, String desc, type)` | 参数名、描述、类型 | 便捷：添加指定类型的变长参数 |
| `.option(String flag)` | 选项标志 | 添加选项（裸标志） |
| `.child(CommandNode)` | 子命令节点 | 添加子命令 |
| `.requiresChild()` | — | 至少一个选项或子命令必须匹配 |
| `.required()` | — | 本节点自身必须被匹配（父节点用） |
| `.build()` | — | 构建。自动排序位置参数 |

### 5.3 约束条件与自动排序

`build()` 在构建时执行三类排序：

**位置参数排序**：必选在前 → 可选居中 → 变长在末。开发者可以任意顺序添加，Builder 内部整理。

**子命令排序**：`required=true` 的子命令排在非必选子命令前面，保证必选子命令优先获得匹配机会，避免被非必选兄弟节点的贪婪参数抢走 token。

| 约束 | 校验时机 | 违反时行为 |
|---|---|---|
| 至多一个变长参数 | `.variadicArg()` 调用时 | 抛 `IllegalArgumentException` |
| 必选在前，可选在后 | `.build()` 调用时 | 自动排序 |
| 变长在末尾 | `.build()` 调用时 | 自动排序 |
| 必选子命令优先 | `.build()` 调用时 | 自动排序 |

---

## 6. 匹配引擎

### 6.1 入口

```java
CommandMatcher matcher = new CommandMatcher();
CommandResult result = matcher.match(commandTokens, rootNode);
```

- `commandTokens`: `CommandTokens`，已 tokenize 的列表 + 已提取的回复消息段
- `rootNode`: `CommandNode`，命令树的根节点
- 返回: `CommandResult`，失败时 `matchSuccess = false`

**根层级校验**：`match()` 在递归匹配成功后额外检查——若 `consumedCount < tokens.size()`（存在未消费的 token），则返回失败。`/git commit -m "hello" extraGarbage` 不会静默通过。子层级不受此限制（未消费 token 是留给同级兄弟节点或父节点处理的）。

### 6.2 对 REPLY 类型参数的特殊处理

QQ 客户端的特性：回复消息段必定位于完整消息链的第一个消息段处，且至多一个。开发者定义命令时不可能以此为假设，因此 `CommandTokens` 将回复单独抽离。

解析器匹配 `REPLY` 类型参数时：

- **不从当前位置取 token**，而是检查 `tokens.hasReply()`
- 匹配成功后**不推进 `pos`**——回复本就不在 token 流中
- 必选参数无回复时直接失败；可选参数无回复时跳过
- `countRemainingRequired()` 排除 `REPLY` 类型（它不消耗 token，不影响后续参数）

### 6.3 匹配算法（伪代码）

```
matchInternal(tokens, start, node) → CommandResult:
    ┌─────────────────────────────────────────────────────────────┐
    │ Step 0: 匹配节点标识                                        │
    │   if tokens[start] 非文本 或 文本 ≠ node.name → return FAIL │
    │   pos = start + 1                                           │
    └─────────────────────────────────────────────────────────────┘
    ┌─────────────────────────────────────────────────────────────┐
    │ Step 1: 匹配位置参数（按 arg 列表顺序）                      │
    │   for each arg in node.args:                                │
    │     if arg.type == REPLY:                                   │
    │       if arg.required 且 !tokens.hasReply() → FAIL          │
    │       若 tokens.hasReply(): 从 tokens.getReply() 取值       │
    │       不推进 pos（回复不在 token 流中），continue            │
    │     if tokens 已耗尽:                                       │
    │       if arg.required → FAIL, else skip                     │
    │     if arg.variadic:                                        │
    │       非贪婪消费: 逐 token 校验类型, 遇到 node.options 或    │
    │       node.children[*].name 时停止                          │
    │       pos += consumed, 不 break（继续 step 2）              │
    │     if arg.required:                                        │
    │       token 类型匹配 → 消费; 否则 → FAIL                    │
    │     if arg.optional:                                        │
    │       仅当剩余 token > 后继必选参数数量时才尝试消费           │
    │       类型不匹配则跳过                                       │
    └─────────────────────────────────────────────────────────────┘
    ┌─────────────────────────────────────────────────────────────┐
    │ Step 2: 匹配选项 + 子命令                                    │
    │   while pos < tokens.length:                                │
    │     token = tokens[pos]                                     │
    │     if token.text ∈ node.options:                           │
    │       result.options.add(text); pos++; continue             │
    │     尝试子命令（回溯）:                                      │
    │       for child in node.children:                           │
    │         childResult = matchInternal(tokens, pos, child)     │
    │         if success: 记录; pos += consumed; break            │
    │         else: 回溯到同一位置，尝试下一个 child               │
    │       if no child matched → break                           │
    └─────────────────────────────────────────────────────────────┘
    ┌─────────────────────────────────────────────────────────────┐
    │ Step 3: 校验约束                                            │
    │   for child in node.children:                               │
    │     if child.required 且未命中 → FAIL                       │
    │   if node.requiresChild 且 options 和 children 均为空 → FAIL│
    └─────────────────────────────────────────────────────────────┘
    return SUCCESS(result)
```

### 6.4 回溯策略

子命令匹配采用**确定性回溯**：对当前位置，按 `children` 列表顺序逐一尝试。匹配成功则推进位置并重新扫描全部子节点；全部失败则停止。

```
输入: /gcc -o output main.c
       children: [-o]

pos=1: token="-o"
  try "-o": matchInternal([output, main.c], "-o") → success, consumed 2 tokens
  pos = 1 + 2 = 3
pos=3: tokens exhausted → done
result: child["-o"] matched, args: {output: "output"}

─────────────────────────────────────

输入: /backup xyz
       options: {"-full", "-incremental"}
       children: []

pos=1: token="xyz"
  token ∉ options → skip
  no children → break
result: options empty, children empty
requiresChild=true → FAIL
```

### 6.5 变长参数非贪婪策略

变长参数遇到以下两类 token 时停止消费：

1. **当前节点的选项**（`node.options`）
2. **当前节点的直接子命令名**（`node.children[*].name`）

兄弟节点的名称**不在停止条件内**。理由：兄弟节点名（如 `add`、`commit`）是自然语言常用词，若设为保留字将导致 `/git add file named commit` 中 `commit` 无法作为合法的文件名被消费。选项（`-f`、`-O2`）不同——以约定前缀区分，不会与参数值混淆。

```
输入: /gcc -O2 -Wall main.c util.c
       options: {"-O2", "-Wall"}
       args: [variadic("files", PLAIN)]

Step 1:
  variadic "files":
    pos=1: "-O2" ∈ options → STOP (consumed nothing)
pos=1

Step 2:
  "-O2" ∈ options → pos=2
  "-Wall" ∈ options → pos=3
pos=3: "main.c" ∉ options, no children → break
result: options={"-O2", "-Wall"}, consumedCount=3, remaining=["main.c","util.c"] → 根层级校验失败
```

子命令内部的变长参数只受该子命令自身的 options/children 约束，互不干扰。

> **关于同时执行多条子命令**：Yuni 定位为 QQ 聊天机器人命令系统，不鼓励"允许用户在一次输入中同时执行多条子命令"的行为（如 `/git add a.txt commit -m "x"`）。这与传统 CLI 有本质区别——CLI 面向脚本和自动化，而聊天机器人面向单次人机交互。若插件确需此能力，应定义显式的分隔符（如 `--`）或拆分为多次交互。

### 6.6 可选参数前瞻

可选参数在消费 token 前检查是否会导致后续必选参数无 token 可用。只有 `剩余 token 数 > 后续必选参数数` 时才尝试匹配。

```
定义: required("x"), optional("y"), required("z")
       （Builder 已自动排序为此顺序）

输入: /cmd a c
       tokens: [a, c]

Step 1:
  required "x": matches "a", pos=1
  optional "y": 后继有 1 个必选参数("z"), 剩余 token=1, 不满足 > 条件 → 跳过
  required "z": matches "c", pos=2
result: x="a", y=未匹配, z="c"
```

### 6.7 参数与子命令的顺序约定

**位置参数必须在子命令之前。** 解析器在单值参数处遇到子命令名时停止参数循环，不再回头。非变长参数和变长参数均不参与子命令匹配之后的 token 消费。

```
定义: optional("filter"), variadic("files")
       children: ["commit", "push"]

输入: /cmd commit file1 file2
  → filter 遇 "commit"（子命令名）→ break
  → merge → 匹配 child "commit"
  → file1, file2 残留 → 根层级校验失败

输入: /cmd file1 file2 commit
  → filter 匹配 "file1", variadic 消费 "file2" 遇 "commit" 停止
  → child "commit" 匹配
  → 全部 token 消费完毕 → 成功
```

**理由**：通用 CLI 工具（gcc、git）的惯例是参数在前、子命令/选项在后。Yuni3 对齐此惯例。若需参数与子命令混合（无论前后），应使用选项（`-x`）或显式分界符（`--`）。

### 7.1 发生时机

`CommandDetector.match(event)` 在收到消息时执行，位于 `PassivePluginMatcher.matchMessageEvent()` 中。

### 7.2 输出：CommandTokens

Tokenize 的输出是 `CommandTokens`，而非单纯的 `List<MessageSegment>`：

```
CommandTokens
├── tokens  : List<MessageSegment>   已 tokenize 的列表（不含回复）
└── reply   : ReplySegment           已抽离的回复消息段（可为 null）
```

### 7.3 Tokenize 规则

```
输入: MessageChain（消息链，可能包含文本、图片、@、回复等）

步骤:
  1. 文本消息段按空格拆分为多个 TextSegment
  2. 非文本消息段（图片、@ 等）原样保留
  3. 若 token 列表首部为 REPLY 类型:
     a. 将其移除并存入 CommandTokens.reply
     b. 查询回复目标的 QQ 号（调用 get_msg API）
     c. 从 token 列表中删除与该 QQ 号对应的 @ 消息段
     其他非 REPLY 的消息段不受影响

输出: CommandTokens
```

### 7.4 为什么回复要单独抽离

QQ 客户端的两个特性：

1. **回复消息段必定位于消息链之首**（无论用户编辑时在何处插入）
2. **一条消息链至多一个回复消息段**

这两条特性意味着开发者**不可能**以"回复出现在命令中的任意位置"为前提定义命令——用户输入 `/cmd arg1 回复 arg2` 时，回复实际会被 QQ 客户端移到最前面，变成 `回复 /cmd arg1 arg2`。

因此 tokenize 阶段将回复单独抽离，解析器匹配 `REPLY` 参数时不从 token 流的位置读取，而是从 `CommandTokens.getReply()` 读取。这使得开发者可以用直观的方式定义命令：

```java
CommandNode.builder("reply-demo")
    .arg(ArgDef.required("target", "回复的消息", CommandArgRequireType.REPLY))
    .arg(ArgDef.optional("extra", "额外文本", CommandArgRequireType.PLAIN))
    .build();
```

### 7.5 命令前缀处理

Tokenize 后，`CommandDetector` 检查首个 token 是否以 `commandFlag`（默认为 `/`）开头：

```
原始消息: 回复 /git add file.txt
tokenize  → tokens: ["/git", "add", "file.txt"], reply: ReplySegment(...)
去掉前缀  → tokens: ["git", "add", "file.txt"], reply: ReplySegment(...)
匹配     → rootNode "git" → child "add" → variadic "files" = ["file.txt"]
```

---

## 8. 插件集成指南

### 8.1 新插件开发

```java
public class MyPlugin implements CommandPlugin {

    // 1. 定义命令树（静态常量，避免重复构建）
    private static final CommandNode ROOT = CommandNode.builder("mycmd")
        .description("我的命令")
        .arg(ArgDef.required("target", "目标用户", CommandArgRequireType.AT))
        .child(CommandNode.builder("-v").build())
        .build();

    // 2. 返回探测器
    @Override
    public CommandDetector getDetector() {
        return new CommandDetector(ROOT);
    }

    // 3. 读取结果
    @Override
    public void execute(YuniMessageEvent event) {
        CommandResult r = event.getCommandResult();
        // ...
    }
}
```

### 8.2 结果读取模式

```java
CommandResult r = event.getCommandResult();

// ── 位置参数 ──
if (r.hasArg("remote")) {
    String remote = r.getArg("remote").asText();
}

// ── 选项 ──
boolean verbose = r.hasOption("-v");
boolean force   = r.hasOption("-f");

// ── 变长参数 ──
for (String file : r.getArg("files").asTextList()) {
    processFile(file);
}

// ── 子命令（递归） ──
if (r.hasChild("commit")) {
    CommandResult commit = r.getChild("commit");
    String msg = commit.getChild("-m").getArg("msg").asText();
    boolean amend = commit.hasOption("-a");
}

// ── 遍历所有匹配的子命令 ──
for (String name : r.getChildNames()) {
    CommandResult child = r.getChild(name);
    // 处理每个子命令
}
```

### 8.3 与旧系统的共存

`PassivePluginMatcher.matchMessageEvent()` 使用 `instanceof MessageDetector` 判断探测器类型，因此新旧探测器均可正常工作：

```java
// PassivePluginMatcher 中
var rawDetector = commandPluginInstance.getDetector();
if (rawDetector instanceof MessageDetector detector && detector.match(event)) {
    // 匹配成功，异步执行插件
}
```

新的 `CommandDetector` 将结果存入 `event.commandResult`；旧的 `CommandDetector` 将结果存入 `event.commandMatched`。两个字段独立，不冲突。

---

## 9. 完整示例

### 9.1 gcc 风格命令

```java
CommandNode gcc = CommandNode.builder("gcc")
    .description("编译 C 程序")
    .option("-O2")
    .option("-Wall")
    .option("-g")
    .child(CommandNode.builder("-o")
        .arg(ArgDef.required("output", "输出文件"))
        .build())
    .variadicArg("files", "源文件")
    .build();
```

| 输入 | 匹配结果 |
|---|---|
| `/gcc main.c` | `getArg("files").asTextList()` = `["main.c"]` |
| `/gcc main.c util.c -O2 -Wall` | `hasOption("-O2")` = true, `hasOption("-Wall")` = true, `files` = `["main.c", "util.c"]` |
| `/gcc main.c -o a.out` | `getChild("-o").getArg("output").asText()` = `"a.out"`, `files` = `["main.c"]` |
| `/gcc main.c -o a.out -O2` | 选项与子命令混合，均正确匹配 |

> **注意**：变长参数非贪婪地在遇到选项时停止。因此 `-O2`、`-o` 等必须放在变长参数值**之后**。选项在前会导致变长参数消费为空，根层级校验失败。这是设计取舍——选项位置灵活性与变长参数的确定性不可兼得。

### 9.2 git 风格命令

```java
CommandNode git = CommandNode.builder("git")
    .description("Git 版本控制")
    .requiresChild()
    .child(CommandNode.builder("add")
        .description("添加文件到暂存区")
        .variadicArg("files", "要添加的文件")
        .build())
    .child(CommandNode.builder("commit")
        .description("提交更改")
        .child(CommandNode.builder("-m")
            .description("提交信息")
            .required()
            .arg(ArgDef.required("msg", "提交信息"))
            .build())
        .option("-a")
        .build())
    .child(CommandNode.builder("push")
        .description("推送到远程")
        .option("-f")
        .arg(ArgDef.required("remote", "远程仓库名"))
        .arg(ArgDef.optional("branch", "分支名"))
        .build())
    .build();
```

```java
// 处理逻辑
public void execute(YuniMessageEvent event) {
    CommandResult r = event.getCommandResult();

    if (r.hasChild("add")) {
        List<String> files = r.getChild("add").getArg("files").asTextList();
        gitAdd(files);
    }
    if (r.hasChild("commit")) {
        CommandResult c = r.getChild("commit");
        String msg = c.getChild("-m").getArg("msg").asText();
        boolean all = c.hasOption("-a");
        gitCommit(msg, all);
    }
    if (r.hasChild("push")) {
        CommandResult p = r.getChild("push");
        boolean force = p.hasOption("-f");
        String remote = p.getArg("remote").asText();
        String branch = p.hasArg("branch") ? p.getArg("branch").asText() : "main";
        gitPush(remote, branch, force);
    }
}
```

### 9.3 互斥选项命令

```java
CommandNode backup = CommandNode.builder("backup")
    .description("备份操作")
    .requiresChild()
    .option("-full")
    .option("-incremental")
    .build();

// /backup            → 不匹配（requiresChild）
// /backup -full      → hasOption("-full") = true
// /backup -incremental → hasOption("-incremental") = true
// /backup -full -incremental → 两者都匹配（不互斥，插件自行判断合法性）
```

### 9.4 多参数子命令

```java
CommandNode search = CommandNode.builder("search")
    .description("搜索")
    .child(CommandNode.builder("-filter")
        .arg(ArgDef.required("keyword", "关键词"))
        .arg(ArgDef.required("type", "类型"))
        .arg(ArgDef.optional("date", "日期"))
        .build())
    .build();

// /search -filter game RPG 2024-01-01
// getChild("-filter").getArg("keyword").asText() = "game"
// getChild("-filter").getArg("type").asText()    = "RPG"
// getChild("-filter").getArg("date").asText()    = "2024-01-01"
```

### 9.5 变长图片上传

```java
CommandNode upload = CommandNode.builder("upload")
    .description("上传图片")
    .variadicArg("images", "要上传的图片", CommandArgRequireType.IMAGE)
    .build();

// /upload <img1> <img2> <img3>
// for (ImageSegment img : getArg("images").asImageList()) { upload(img); }

// /upload hello
// "hello" 不是 IMAGE → 整个命令匹配失败
```

---

## 10. 旧 API 对照参考

> **旧 API（`CommandModel`、`CommandBuilder`、`CommandPlugin` 等）已被移除。** 以下对照表仅作历史参考。

### 10.1 API 对照表

| 旧 API | 新 API |
|---|---|
| `CommandBuilder.create("head")` | `CommandNode.builder("head")` |
| `.addRequiredArg(name, desc)` | `.arg(ArgDef.required(name, desc))` |
| `.addRequiredArg(name, desc, type)` | `.arg(ArgDef.required(name, desc, type))` |
| `.addOptionalArg(name, desc)` | `.arg(ArgDef.optional(name, desc))` |
| `.addOptionalArg(name, desc, type)` | `.arg(ArgDef.optional(name, desc, type))` |
| `.addOption(flag)` | `.option(flag)` |
| `.addOptionWithRequiredArg(flag, name, desc)` | `.child(CommandNode.builder(flag).arg(ArgDef.required(name, desc)).build())` |
| `.addOptionWithOptionalArg(flag, name, desc)` | `.child(CommandNode.builder(flag).arg(ArgDef.optional(name, desc)).build())` |
| `new CommandDetector(model)` | `new CommandDetector(rootNode)` |
| `event.getCommandMatched()` | `event.getCommandResult()` |
| `CommandMatched.getArg(name).getValue()` 强转 | `CommandResult.getArg(name).asText()` 等 |
| `CommandMatched.hasOption(flag)` | `CommandResult.hasOption(flag)` |
| `CommandMatched.getOptionRequiredArg(flag)` | `CommandResult.getChild(flag).getArg(name)` |
| `CommandMatched.getOptionOptionalArg(flag)` | `CommandResult.getChild(flag).getArg(name)` |

### 10.2 曾经缺失的能力

| 能力 | 旧 API | 新 API |
|---|---|---|
| 子命令嵌套 | 不支持（需手动 switch） | `.child(CommandNode.builder("sub")...)` |
| 多参数子命令 | 不支持（仅 0 或 1 个） | 链式 `.arg()` 任意数量 |
| 变长参数 | 不支持 | `ArgDef.variadic(name, desc)` |
| 纯选项 | 与带参选项混用 | `options: Set<String>` 独立建模 |
| 选项必须约束 | 不支持 | `.child(...).required()` |
| 子节点必须约束 | 不支持 | `.requiresChild()` |
| 类型化结果 | 强制转型 | `asText()` / `asImageList()` |
| 参数自动排序 | 不支持 | `build()` 内部自动排序 |

---

## 11. 设计决策

| 决策 | 结论 | 理由 |
|---|---|---|
| 选项是否独立建模 | **是** — `Set<String>` | 选项是纯 token，无参数无子结构。`-m "msg"` 是子命令而非选项 |
| 选项与子命令匹配顺序 | **选项优先** | 选项 O(1) 恒成功，无回溯成本 |
| 变长参数消费策略 | **非贪婪**，仅停止于当前节点的选项和直接子命令名 | 兄弟节点名不应设为保留字（如 `commit` 可作为文件名）。选项有 `-` 前缀区分 |
| `REPLY` 类型参数如何处理 | **Tokenize 阶段抽离，不从 token 流位置读取** | QQ 客户端强制回复位于消息链之首，开发者不可能以位置模型定义回复参数 |
| 根层级未消费 token | **匹配失败** | 命令不应包含无法识别的 token |
| 是否引入新类型枚举 | **否** | 复用 `CommandArgRequireType`，`PLAIN` + `variadic` 覆盖所有场景 |
| 参数顺序 | **Builder 自动排序**（必选→可选→变长） | 开发者无需关注顺序 |
| 子命令顺序 | **Builder 自动排序**（必选优先） | 防止非必选兄弟节点的贪婪参数抢走必选子命令的 token |
| `required` 与 `variadic` 关系 | **正交**（4 种组合均有效） | `requiredVariadic()` 至少消费一个 token |
| 是否支持别名（`-x`/`--xxx`） | **否** | 中文娱乐环境为主，简单优先 |
| 是否支持 `=` 语法 | **否** | 不引入不必要的复杂机制 |
| 同时执行多条子命令 | **不鼓励** | 聊天机器人面向单次交互，非 CLI 脚本场景。如需此能力应定义显式分隔符 |
| 引号支持 | **延后** | 需单独设计；变长参数可解决大部分场景 |

---

## 12. 异常处理

### 12.1 构建期异常

| 场景 | 异常 | 消息 |
|---|---|---|
| 重复定义变长参数 | `IllegalArgumentException` | `命令 'X': 最多只能定义一个变长参数` |
| 传入 null tokens | 无异常 | 返回 `CommandResult.fail()` |
| 传入 null rootNode | 无异常 | 返回 `CommandResult.fail()` |
| 实体缺少 `@Id` 注解 | `IllegalArgumentException` | `类 X 缺少 @Id 主键注解` |

### 12.2 解析期行为

| 场景 | 行为 |
|---|---|
| 必选参数未匹配 | 返回 `matchSuccess = false` |
| `requiresChild` 未满足 | 返回 `matchSuccess = false` |
| `required` 子命令未匹配 | 返回 `matchSuccess = false` |
| 变长参数遇到类型不匹配的元素 | 返回 `matchSuccess = false` |
| 可选参数输入不足 | 跳过（不触发失败） |
| 未知 token（非选项非子命令） | 停止匹配，根层级校验失败 |

### 12.3 运行时异常

| 场景 | 异常 |
|---|---|
| 变长参数调用 `asText()` | `IllegalStateException`: "参数 'X' 是变长参数，请使用 asXxxList() 方法取值" |
| 非变长参数调用 `asTextList()` | `IllegalStateException`: "参数 'X' 是单值参数，请使用 asXxx() 方法取值" |
| `ArgResult.value` 为 null 调用 `asText()` | 返回 `null`（不抛异常） |

---

## 13. 未来规划

| 工作项 | 优先级 | 状态 |
|---|---|---|
| 引号支持（`"..."` 包裹的文本不拆分） | 中 | 计划中 |
| 基于 CommandNode 的自动帮助文本生成 | 中 | 计划中（以 `/命令头 帮助` 形式） |
| `YuniMessageEvent` 中 tokenize 结果缓存（避免各插件重复 tokenize） | 低 | 计划中 |
| 反引号/代码块支持（`\`...\`` 内文本不拆分） | 低 | 未计划 |
| 选项组（互斥选项集约束） | 低 | 未计划 |
