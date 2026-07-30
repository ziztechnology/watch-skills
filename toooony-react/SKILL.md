---
name: toooony-react
description: React 和 JSX/TSX 的框架专属实现与审查规范，覆盖函数式组件、页面导出、Props、JSON 可序列化的组件状态、状态初始化、Effect、交互副作用、条件渲染、并发渲染、代码分割、回调命名及 React 文件命名。编写、补全、修改、修复、重构、优化或审查 React 组件、页面、JSX/TSX 文件和自定义 Hook 时使用，也适用于基于 React 的前端框架项目。
---

# 组件定义与导出

## 函数式组件与箭头函数

必须使用函数式组件。函数式组件本身和组件内部的工具函数都必须使用箭头函数定义。

```tsx
import type { FC } from "react";

export const Foo: FC = () => {
  const handleClick = () => {
    console.log("test");
  };

  return <div onClick={handleClick}>bar</div>;
};
```

## 页面组件的默认导出

组件用作页面且框架允许默认导出时，使用下列风格：

```tsx
import type { FC } from "react";

const FooPage: FC = () => {
  return <div>Foo Page</div>;
};

export default FooPage;
```

## 组件 Props 类型

组件的 Props 类型必须单独定义，并作为 `FC` 的类型参数传入。

```tsx
import type { FC } from "react";

interface FooProps {
  title: string;
  visible: boolean;
}

export const Foo: FC<FooProps> = (props) => {
  const { title, visible = false } = props;

  if (!visible) {
    return null;
  }

  return <div>{title}</div>;
};
```

# 组件状态

## 统一的 componentData

组件自身的可变状态统一定义在一个 `componentData` 对象中，并通过同一个 `useState` 管理。`componentData` 必须是可以序列化为 JSON Object 的纯数据，并直接使用 `ts-essentials` 的 `JsonObject` 作为类型。不得为 `componentData` 额外定义或使用继承、扩展 `JsonObject` 的专用类型。

```tsx
import { useState } from "react";
import type { JsonObject } from "ts-essentials";

const [componentData, setComponentData] = useState<JsonObject>({
  formData: {
    keyword: "",
  },
  listData: [],
  loading: false,
});
```

`componentData` 中的列表和表单数据必须使用统一命名：

- 列表条目必须使用 `listData`，即 `componentData.listData`；无论列表是否分页。分页页码、总数等元数据可以放在 `componentData.pagination` 或项目已有的等价字段中。
- 表单模型使用 `formData`，即 `componentData.formData`，并将各表单字段集中在该对象内。

普通组件状态默认集中在 `componentData` 中。React API、第三方 API 或数据语义要求独立声明时，使用对应的状态来源。典型情况包括：

- Props 和 Context 保持为外部数据源，直接从对应 API 读取。
- DOM 引用、子组件引用、定时器标识和渲染无关数据使用 `useRef` 或对应 API 管理。
- 能够由 Props 或现有状态直接计算得到的派生数据，直接在渲染期间计算；仅在计算开销确实较大时使用 `useMemo`。
- 自定义 Hook、外部 Store、Router 及第三方 Hook 返回的状态或实例。
- 静态常量和函数保留为普通绑定。

按不可变数据处理 React 状态。`setComponentData` 会替换整个对象，因此更新部分字段时保留其他字段；新状态依赖旧状态时使用函数式更新：

```tsx
const handleSearch = (keyword: string) => {
  setComponentData((previousData) => ({
    ...previousData,
    formData: {
      ...previousData.formData,
      keyword,
    },
    loading: true,
  }));
};
```

更新 `componentData` 时，为发生变化的嵌套对象和数组创建新引用，并通过 setter 写回。多个字段属于同一次状态转换时，在一次 `setComponentData` 调用中完成更新。

## 昂贵初始状态的惰性初始化

创建初始状态需要解析较大的序列化数据或执行其他开销较大的同步计算时，必须向 `useState` 传入纯初始化函数。初始化函数不得产生副作用，并且相同输入必须得到相同结果。

```tsx
import { useState } from "react";
import type { JsonObject } from "ts-essentials";

const [componentData, setComponentData] = useState<JsonObject>(() => ({
  settings: JSON.parse(serializedSettings),
}));
```

# 生命周期与交互

## 组件初始化

组件挂载时执行数据请求等副作用操作，使用 `ahooks` 的 `useMount`：

```tsx
import { useMount } from "ahooks";

useMount(() => {
  void fetchUserProfile();
});
```

## 应用级初始化

浏览器端且必须在每次应用启动时执行一次的初始化逻辑，应当放在应用入口中执行。组件挂载时需要执行的局部逻辑继续使用 `useMount`。

```tsx
import { createRoot } from "react-dom/client";

const initializeApplication = () => {
  loadSettings();
  initializeMonitoring();
};

initializeApplication();

createRoot(document.getElementById("root")!).render(<App />);
```

## 交互副作用

由点击、提交、拖拽等明确用户交互触发的请求、通知和其他副作用，必须直接放在对应的事件处理函数中执行。

```tsx
const handleSubmit = async () => {
  await submitForm(componentData.formData);
  showToast("提交成功");
};
```

Effect 用于根据当前渲染结果与网络连接、浏览器 API、第三方组件等外部系统保持同步。

## Effect 职责与依赖

每个 Effect 应当只负责一个独立的同步过程。多个同步过程依赖不同的响应式值时，必须拆分为多个 Effect。

```tsx
import { useEffect } from "react";

useEffect(() => {
  analytics.trackPageView(pathname);
}, [pathname]);

useEffect(() => {
  document.title = `${pageTitle} | Example`;
}, [pageTitle]);
```

Effect 的依赖必须包含 Effect 中读取的响应式值。仅使用对象的某个字段时，依赖该字段，避免对象其他字段变化时重复执行 Effect。

```tsx
useEffect(() => {
  synchronizeUser(user.id);
}, [user.id]);
```

## Effect Event

项目使用支持 `useEffectEvent` 的 React 版本时，可以将 Effect 内需要读取最新 Props 或状态、但不应触发重新同步的事件逻辑提取为 Effect Event。Effect Event 仅在当前组件的 Effect 或其他 Effect Event 中调用，并且不加入 Effect 依赖数组。

```tsx
import { useEffect, useEffectEvent } from "react";

const handleConnected = useEffectEvent(() => {
  showNotification(theme);
});

useEffect(() => {
  const connection = createConnection(roomID);
  connection.on("connected", handleConnected);
  connection.connect();

  return () => {
    connection.disconnect();
  };
}, [roomID]);
```

## 回调 Props 与处理函数命名

组件回调 Props 使用 `onXxx` 命名，组件内部对应的处理函数使用 `handleXxx` 命名。

```tsx
interface SearchBoxProps {
  onSearch: (keyword: string) => void;
}

export const SearchBox: FC<SearchBoxProps> = (props) => {
  const { onSearch } = props;

  const handleSearch = () => {
    onSearch("React");
  };

  return <button onClick={handleSearch}>搜索</button>;
};
```

# 渲染与性能

## 明确的条件渲染

条件值可能是数字或其他会被 React 渲染的值时，必须先得到明确的布尔条件，再使用三元表达式渲染目标内容或 `null`。

```tsx
import type { FC } from "react";

interface BadgeProps {
  count: number;
}

export const Badge: FC<BadgeProps> = (props) => {
  const { count } = props;
  const hasCount = count > 0;

  return hasCount ? <span className="badge">{count}</span> : null;
};
```

## 组件代码分割

首屏不需要且体积较大的组件应当使用 `lazy` 延迟加载，并在能够表达独立加载状态的位置使用 `Suspense` 提供稳定的回退界面。懒加载组件必须在模块顶层定义。

```tsx
import { lazy, Suspense } from "react";
import type { FC } from "react";

const DataChart = lazy(() => import("./DataChart"));

export const Dashboard: FC = () => {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <DataChart />
    </Suspense>
  );
};
```

## 非紧急更新

状态更新会触发开销较大的渲染并阻塞输入、点击等紧急交互时，可以使用 `useTransition` 将该更新标记为非紧急更新。`isPending` 仅表示 Transition 是否仍在进行；异步请求的取消、结果顺序和错误处理继续由请求逻辑负责。

```tsx
import { useTransition } from "react";

const [isPending, startTransition] = useTransition();

const handleSelectTab = (selectedTab: string) => {
  startTransition(() => {
    setComponentData((previousData) => ({
      ...previousData,
      selectedTab,
    }));
  });
};

return <TabContent aria-busy={isPending} />;
```

组件接收的新值会触发开销较大的派生渲染，但当前组件无法控制该值的更新位置时，可以使用 `useDeferredValue` 延后对应子树的更新。`useDeferredValue` 用于调整渲染优先级，不用于减少或延迟网络请求。

```tsx
import { memo, useDeferredValue } from "react";
import type { FC } from "react";

interface SearchResultsProps {
  keyword: string;
}

interface ResultListProps {
  keyword: string;
}

const ResultList: FC<ResultListProps> = memo((props) => {
  const { keyword } = props;

  return <ExpensiveResultList keyword={keyword} />;
});

export const SearchResults: FC<SearchResultsProps> = (props) => {
  const { keyword } = props;
  const deferredKeyword = useDeferredValue(keyword);

  return <ResultList keyword={deferredKeyword} />;
};
```

## 保留隐藏界面的状态

项目使用支持 `Activity` 的 React 版本，且频繁切换显示状态的界面需要保留内部状态和 DOM 时，可以使用 `Activity`。隐藏期间子组件的 Effect 会被清理，因此 Effect 必须完整实现清理逻辑。

```tsx
import { Activity } from "react";
import type { FC } from "react";

interface SidebarProps {
  visible: boolean;
}

export const Sidebar: FC<SidebarProps> = (props) => {
  const { visible } = props;

  return (
    <Activity mode={visible ? "visible" : "hidden"}>
      <SidebarContent />
    </Activity>
  );
};
```

# 组件定义位置

组件必须在模块顶层定义，不得在另一个组件函数内部定义。

# React 文件命名

- React 组件文件使用 PascalCase。
- Hook 文件使用 `useXxx.ts` 或 `useXxx.tsx`。
