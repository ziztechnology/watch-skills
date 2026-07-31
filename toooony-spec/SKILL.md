---
name: toooony-spec
description: Implementation and review standards for JavaScript and TypeScript, covering basic TypeScript project configuration, naming, basic syntax, type design, file and test directory organization, control flow, private fields, and TypeScript error annotations. Use when writing, completing, modifying, fixing, refactoring, or reviewing .js, .jsx, .mjs, .cjs, .ts, .tsx, .mts, or .cts files, TypeScript project dependencies, or tsconfig files. Also applies to Node.js scripts, frontend logic, utility functions, and configuration code.
---

# Basic TypeScript Project Configuration

- TypeScript projects must install `ts-essentials` and list it under `dependencies`.
- `tsconfig` must enable at least `strictNullChecks` by setting `compilerOptions.strictNullChecks` to `true`. More comprehensive strict-checking options such as `strict` may also be enabled.

# Naming

## Callback Parameters

Callback function parameters must use descriptive names. Prefer `item` when a parameter represents a generic element in a collection.

## Descriptive Names

Variables, constants, functions, methods, and similar identifiers should use long, descriptive names.

## Abbreviations

When creating or renaming identifiers, abbreviations within a name must use consistent capitalization. In camelCase, PascalCase, or MixedCaps names, use all lowercase for a lowercase abbreviation at the beginning of the name and all uppercase for abbreviations elsewhere. In snake_case, kebab-case, or all-lowercase names, abbreviations must follow that naming style and remain lowercase. Existing APIs, framework conventions, and external names must retain their official capitalization.

```text
apiClient
APIClient
parseURL
userID
user_id
```

# Basic Syntax and Code Style

## Strings

Strings must use single quotes.

## Enum Values

Enum values must use a constant object with `as const`, with the union type derived from that object.

```ts
export const FOO = {
  BAR: "bar",
} as const;

export type Foo = (typeof FOO)[keyof typeof FOO];
```

## Comments

- Documentation comments for functions, methods, and classes should use JSDoc. Other comments should use concise, single-line `//` text.
- Comments should focus on design rationale, constraints, and context.
- In comments, descriptions, and explanatory text, retain one space between Chinese text and English text or numbers.

# Type Design

## Type Declarations

Explicitly define types for local variables, intermediate results, and auxiliary structures only when doing so improves readability, reusability, or boundary safety. Types required by function signatures, public APIs, and existing project conventions must be explicitly declared.

## JSON Objects

Use `JsonObject` from `ts-essentials` for data that can be serialized as a JSON Object. Use `JsonObject` directly by default and retain its broad type. Do not narrow the type or define an interface that extends `JsonObject` merely because the current code knows the object's structure.

Define an interface that extends `JsonObject` only when business constraints require specific fields to be guaranteed at compile time and the precise type serves as a reusable contract or an explicit system boundary. Local variables, intermediate results, auxiliary structures, pass-through data, and dynamically structured data must use `JsonObject` directly.

Default form:

```ts
import type { JsonObject } from "ts-essentials";

const metadata: JsonObject = getMetadata();
```

Only when the exception above applies:

```ts
interface SearchState extends JsonObject {
  keyword: string;
  page: number;
  filters: JsonObject;
}
```

Every level of the object may contain only objects, arrays, strings, numbers, booleans, and `null`. Manage functions, `undefined`, `symbol`, `bigint`, `Date`, `Map`, `Set`, class instances, and other non-JSON data with separate types and storage locations appropriate to their semantics.

## JSON Arrays

Use `JsonArray` from `ts-essentials` for data that can be serialized as a JSON Array. Use `JsonArray` directly by default and retain its broad type. Do not narrow the array type merely because the current code knows the structure of some elements.

Define a precise array element type only when business constraints require the element structure to be guaranteed at compile time and the precise type serves as a reusable contract or an explicit system boundary. Local variables, intermediate results, auxiliary structures, pass-through data, and arrays with dynamic element structures must use `JsonArray` directly.

```ts
import type { JsonArray } from "ts-essentials";

const records: JsonArray = getRecords();
```

## General Arrays

When a parameter or data value can accept both mutable and readonly arrays, use `AnyArray<Type>` from `ts-essentials` and keep the outer array mutability flexible. Pass the element type explicitly when it is known. Use `unknown` when the element type is unknown.

Use `AnyArray<JsonObject>` for a flexible array of JSON objects that may be mutable or readonly.

```ts
import type { AnyArray, JsonObject } from "ts-essentials";

const processItems = (items: AnyArray<Item>) => {
  for (const item of items) {
    processItem(item);
  }
};

const records: AnyArray<JsonObject> = getRecords();
```

## Synchronous or Asynchronous Return Values

When a function, callback, hook, or extension interface may return either a synchronous or asynchronous value, use `AsyncOrSync<Type>` from `ts-essentials` and keep the return form flexible. Pass the return value type explicitly when it is known.

Use `AsyncOrSync<JsonObject>` for a flexible JSON object returned synchronously or asynchronously.

```ts
import type { AsyncOrSync, JsonObject } from "ts-essentials";

type Handler = () => AsyncOrSync<Result>;
type MetadataLoader = () => AsyncOrSync<JsonObject>;
```

# File Organization

## Src Layout

Projects must use the Src Layout, with source code placed in the `src/` directory.

## Supporting Content

Implementation files should primarily contain implementation logic. When there are many constants, module-level variables, configuration objects, data mappings, or type definitions, place them in separate files according to purpose. These files must be placed in the same directory as the original file, alongside the code that uses them.

## Test Files

Test files must be grouped in a `__tests__/` subdirectory under the current module directory and organized separately from that module's source files.

# Control Flow

## Guard Clauses and Early Returns

Branching logic should prefer guard clauses or early returns to keep nesting depth low.

## Immediately Executed Logic

Independent logic that must execute immediately must be extracted into a descriptively named function, which is then called explicitly to obtain the result.

```ts
const createConfig = () => {
  return { environment: "production" };
};

const config = createConfig();
```

## Iterating Over Iterables

Prefer `for...of` loops when iterating over iterables.

```ts
for (const item of items) {
  processItem(item);
}
```

# Encapsulation and Type Checking

## Private Class Members

Private class members must use ES2022 `#` private fields.

```ts
class Cache {
  #data = { foo: "bar" };
}
```

## TypeScript Error Directives

When a known type error must remain in the code, annotate it with `@ts-expect-error` and explain why the error occurs.

```ts
// @ts-expect-error -- The third-party type declaration lacks the runtimeOnly property
legacyClient.runtimeOnly();
```

# UI Copy

## Spacing Between Chinese Text, English Text, and Numbers

In user-visible Chinese copy, retain one ASCII space between Chinese characters and English text or Arabic numerals.

Correct examples:

- `支持 iOS 18 及以上版本`
- `共找到 3 条记录`

Code, URLs, file paths, email addresses, official product names, and other indivisible identifiers must remain unchanged.

Do not add spaces between Chinese punctuation and adjacent English text or Arabic numerals, for example: `支持 iOS、Android 和 Windows`.

## Punctuation and Wording

- Chinese sentences should use full-width Chinese punctuation.
- Short text such as buttons, labels, and titles usually should not end with punctuation. Prompts, instructions, and error messages should use terminal punctuation when they are complete sentences.
- Button copy should use a clear action and express the operation's result, such as “保存修改”. For confirmation actions, use “取消” and the specific action name, such as “删除”; avoid using “确定”, “是”, or “否” alone.
- Error messages must explain what happened. When a recovery method can be provided, they should also explain the next action, such as “保存失败，请检查网络连接后重试。”
- The same concept must use a consistent name within the same interface. When a product glossary exists, use the names from that glossary.
