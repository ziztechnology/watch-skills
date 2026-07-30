---
name: toooony-spec
description: JavaScript 和 TypeScript 的实现与审查规范，覆盖 TypeScript 项目基础配置、命名、基础语法、类型设计、文件和测试目录组织、控制流、私有字段及 TypeScript 错误标注。编写、补全、修改、修复、重构或审查 .js、.jsx、.mjs、.cjs、.ts、.tsx、.mts、.cts 文件、TypeScript 项目依赖和 tsconfig 时使用，也适用于 Node.js 脚本、前端逻辑、工具函数和配置代码。
---

# TypeScript 项目基础配置

- TypeScript 项目必须安装 `ts-essentials`，并将其放在 `dependencies` 中。
- `tsconfig` 必须至少启用 `strictNullChecks`，即将 `compilerOptions.strictNullChecks` 设置为 `true`；可以同时启用 `strict` 等更完整的严格检查选项。

# 命名

## 回调参数

回调函数的参数必须使用描述性名称。参数表示集合中的通用元素时，优先命名为 `item`。

## 描述性命名

变量、常量、函数和方法等应当使用描述性强的长名称。

## 缩写

创建或重命名标识符时，名称中的缩写必须保持统一大小写。使用 camelCase、PascalCase 或 MixedCaps 时，名称开头的小写缩写使用全小写，其他位置的缩写使用全大写。使用 snake_case、kebab-case 或全小写命名时，缩写遵循该命名形式使用全小写。已有 API、框架约定和外部名称必须保持其正式大小写。

```text
apiClient
APIClient
parseURL
userID
user_id
```

# 基础语法与代码风格

## 字符串

字符串必须使用单引号。

## 枚举值

枚举值必须使用带有 `as const` 的常量对象，并从该对象派生联合类型。

```ts
export const FOO = {
  BAR: 'bar',
} as const;

export type Foo = (typeof FOO)[keyof typeof FOO];
```

## 注释

- 函数、方法和类的文档注释应当使用 JSDoc；其他注释应当使用简洁的单行 `//` 文本。
- 注释应当重点说明设计原因、约束条件和上下文。
- 注释、描述和说明性文字中的中文与英文、数字之间应当保留一个空格。

# 类型设计

## 类型声明

仅在能够提升代码可读性、复用性或边界安全时，为局部变量、中间结果和辅助结构显式定义类型。函数签名、公开 API 及项目已有约定要求的类型必须显式声明。

## JSON 对象

表示可以序列化为 JSON Object 的数据时，必须使用 `ts-essentials` 提供的 `JsonObject`。默认直接使用 `JsonObject` 并保持其宽泛类型；不得仅因当前代码已知对象结构而收窄类型或定义继承 `JsonObject` 的接口。

仅当业务约束要求在编译期保证特定字段，且该精确类型用于可复用契约或明确的系统边界时，才定义继承 `JsonObject` 的接口。局部变量、中间结果、辅助结构、透传数据和结构动态的数据必须直接使用 `JsonObject`。

默认写法：

```ts
import type { JsonObject } from 'ts-essentials';

const metadata: JsonObject = getMetadata();
```

仅在满足上述例外条件时：

```ts
interface SearchState extends JsonObject {
  keyword: string;
  page: number;
  filters: JsonObject;
}
```

该对象的所有层级只能包含对象、数组、字符串、数字、布尔值和 `null`。使用适合其语义的独立类型和存储位置管理函数、`undefined`、`symbol`、`bigint`、`Date`、`Map`、`Set`、类实例及其他非 JSON 数据。

## JSON 数组

表示可以序列化为 JSON Array 的数据时，必须使用 `ts-essentials` 提供的 `JsonArray`。默认直接使用 `JsonArray` 并保持其宽泛类型；不得仅因当前代码已知部分元素结构而收窄数组类型。

仅当业务约束要求在编译期保证元素结构，且该精确类型用于可复用契约或明确的系统边界时，才定义精确的数组元素类型。局部变量、中间结果、辅助结构、透传数据和元素结构动态的数组必须直接使用 `JsonArray`。

```ts
import type { JsonArray } from 'ts-essentials';

const records: JsonArray = getRecords();
```

## 通用数组

参数或数据可以同时接受可变数组和只读数组时，必须使用 `ts-essentials` 提供的 `AnyArray<Type>`，并保持数组可变性这一外层形态宽松。元素类型已知时必须明确传入；元素类型未知时使用 `unknown`。

表示可变或只读的宽松 JSON 对象数组时，使用 `AnyArray<JsonObject>`。

```ts
import type { AnyArray, JsonObject } from 'ts-essentials';

const processItems = (items: AnyArray<Item>) => {
  for (const item of items) {
    processItem(item);
  }
};

const records: AnyArray<JsonObject> = getRecords();
```

## 同步或异步返回值

函数、回调、钩子或扩展接口可以同时返回同步值和异步值时，必须使用 `ts-essentials` 提供的 `AsyncOrSync<Type>`，并保持返回形态宽松。返回值类型已知时必须明确传入。

表示同步或异步返回的宽松 JSON 对象时，使用 `AsyncOrSync<JsonObject>`。

```ts
import type { AsyncOrSync, JsonObject } from 'ts-essentials';

type Handler = () => AsyncOrSync<Result>;
type MetadataLoader = () => AsyncOrSync<JsonObject>;
```

# 文件组织

## Src Layout

项目必须使用 Src Layout，将源代码放在 `src/` 目录中。

## 辅助内容

实现文件应当主要放置实现逻辑。常量、模块级变量、配置对象、数据映射或类型定义较多时，应当按用途分别放到独立文件中。这些文件必须放在原文件的同级目录，与使用它们的代码放在一起。

## 测试文件

测试文件必须集中放置在当前模块目录下的 `__tests__/` 子目录中，与该模块的源文件分目录组织。

# 控制流

## 卫语句和提前返回

分支逻辑应当优先使用卫语句（Guard Clauses）或提前返回（Early Return），保持较低的嵌套深度。

## 立即执行逻辑

需要立即执行的独立逻辑必须提取为具有描述性名称的函数，再显式调用该函数取得结果。

```ts
const createConfig = () => {
  return { environment: 'production' };
};

const config = createConfig();
```

## 可迭代对象遍历

遍历可迭代对象时，优先使用 `for...of` 循环。

```ts
for (const item of items) {
  processItem(item);
}
```

# 封装与类型检查

## 类的私有成员

类的私有成员必须使用 ES2022 的 `#` 私有字段表示。

```ts
class Cache {
  #data = { foo: 'bar' };
}
```

## TypeScript 错误指令

代码中需要保留已知类型错误时，必须使用 `@ts-expect-error` 标注，并说明出现该错误的原因。

```ts
// @ts-expect-error -- 第三方类型声明缺少 runtimeOnly 属性
legacyClient.runtimeOnly();
```
