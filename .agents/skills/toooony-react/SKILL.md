---
name: toooony-react
description: React and JSX/TSX-specific implementation and review standards covering functional components, page exports, props, JSON-serializable component state, state initialization, Effects, interaction side effects, conditional rendering, concurrent rendering, code splitting, callback naming, and React file naming. Use when writing, completing, modifying, fixing, refactoring, optimizing, or reviewing React components, pages, JSX/TSX files, and custom Hooks, including projects built with React-based frontend frameworks.
---

# Component Definitions and Exports

## Functional Components and Arrow Functions

You must use functional components. Both functional components and their internal utility functions must be defined with arrow functions.

```tsx
import type { FC } from "react";

export const Foo: FC = () => {
  const handleClick = () => {
    console.log("test");
  };

  return <div onClick={handleClick}>bar</div>;
};
```

## Default Exports for Page Components

When a component serves as a page and the framework permits default exports, use the following style:

```tsx
import type { FC } from "react";

const FooPage: FC = () => {
  return <div>Foo Page</div>;
};

export default FooPage;
```

## Component Props Types

A component's Props type must be defined separately and passed as the type argument to `FC`.

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

# Component State

## Unified `componentData`

Define all mutable state owned by the component in a single `componentData` object and manage it with one `useState`. `componentData` must contain only plain data that can be serialized as a JSON object, and it must use `JsonObject` from `ts-essentials` directly as its type. Do not define or use a dedicated type for `componentData` that inherits from or extends `JsonObject`.

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

List and form data inside `componentData` must use consistent names:

- List items must be named `listData` and accessed through `componentData.listData`, regardless of whether the list is paginated. Metadata such as the page number and total count may be stored in `componentData.pagination` or an equivalent field already used by the project.
- Name the form model `formData`, access it through `componentData.formData`, and group all form fields within that object.

Keep ordinary component state in `componentData` by default. Use the corresponding state source when React APIs, third-party APIs, or data semantics require separate declarations. Typical cases include:

- Keep Props and Context as external data sources and read them directly from their respective APIs.
- Manage DOM references, child component references, timer identifiers, and render-independent data with `useRef` or the corresponding API.
- Calculate derived data directly during rendering when it can be computed from Props or existing state. Use `useMemo` only when the computation is genuinely expensive.
- Keep state or instances returned by custom Hooks, external stores, routers, and third-party Hooks in their original sources.
- Keep static constants and functions as regular bindings.

Treat React state as immutable data. `setComponentData` replaces the entire object, so preserve unchanged fields when updating part of it. Use a functional update when the next state depends on the previous state:

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

When updating `componentData`, create new references for every nested object and array that changes, then write them back through the setter. When multiple fields belong to the same state transition, update them in a single `setComponentData` call.

## Lazy Initialization for Expensive Initial State

When creating the initial state requires parsing a large serialized payload or performing another expensive synchronous computation, you must pass a pure initializer function to `useState`. The initializer must not produce side effects, and identical inputs must produce identical results.

```tsx
import { useState } from "react";
import type { JsonObject } from "ts-essentials";

const [componentData, setComponentData] = useState<JsonObject>(() => ({
  settings: JSON.parse(serializedSettings),
}));
```

# Lifecycle and Interactions

## Component Initialization

Use `useMount` from `ahooks` for side effects such as data requests that run when the component mounts:

```tsx
import { useMount } from "ahooks";

useMount(() => {
  void fetchUserProfile();
});
```

## Application-Level Initialization

Browser-side initialization logic that must run once on every application startup should be placed in the application entry point. Continue to use `useMount` for local logic that must run when a component mounts.

```tsx
import { createRoot } from "react-dom/client";

const initializeApplication = () => {
  loadSettings();
  initializeMonitoring();
};

initializeApplication();

createRoot(document.getElementById("root")!).render(<App />);
```

## Interaction Side Effects

Requests, notifications, and other side effects triggered by explicit user interactions such as clicks, submissions, and drag operations must be executed directly in the corresponding event handler.

```tsx
const handleSubmit = async () => {
  await submitForm(componentData.formData);
  showToast("Submitted successfully");
};
```

Use Effects to synchronize the current render result with external systems such as network connections, browser APIs, and third-party components.

## Effect Responsibilities and Dependencies

Each Effect should manage exactly one independent synchronization process. Synchronization processes that depend on different reactive values must be split into separate Effects.

```tsx
import { useEffect } from "react";

useEffect(() => {
  analytics.trackPageView(pathname);
}, [pathname]);

useEffect(() => {
  document.title = `${pageTitle} | Example`;
}, [pageTitle]);
```

An Effect's dependency list must include every reactive value read by the Effect. When the Effect uses only one field of an object, depend on that field to avoid rerunning the Effect when unrelated fields change.

```tsx
useEffect(() => {
  synchronizeUser(user.id);
}, [user.id]);
```

## Effect Events

When the project uses a React version that supports `useEffectEvent`, event logic that must read the latest Props or state inside an Effect but should not trigger resynchronization may be extracted into an Effect Event. Call an Effect Event only from an Effect or another Effect Event in the same component, and do not add it to the Effect dependency array.

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

## Callback Props and Handler Naming

Name component callback Props with the `onXxx` pattern and the corresponding internal handlers with the `handleXxx` pattern.

```tsx
interface SearchBoxProps {
  onSearch: (keyword: string) => void;
}

export const SearchBox: FC<SearchBoxProps> = (props) => {
  const { onSearch } = props;

  const handleSearch = () => {
    onSearch("React");
  };

  return <button onClick={handleSearch}>Search</button>;
};
```

# Rendering and Performance

## Explicit Conditional Rendering

When a condition may be a number or another value that React can render, you must first derive an explicit Boolean condition, then use a ternary expression to render the target content or `null`.

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

## Component Code Splitting

Large components that are not needed for the initial render should be lazy-loaded with `lazy`, and `Suspense` should provide a stable fallback UI at a location that can represent an independent loading state. Lazy-loaded components must be defined at the module's top level.

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

## Non-Urgent Updates

When a state update triggers an expensive render that blocks urgent interactions such as input or clicks, you may use `useTransition` to mark the update as non-urgent. `isPending` indicates only whether the Transition is still in progress. Keep request cancellation, result ordering, and error handling in the request logic.

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

When a new value received by a component triggers an expensive derived render and the current component cannot control where that value is updated, you may use `useDeferredValue` to defer updating the corresponding subtree. Use `useDeferredValue` to adjust rendering priority, not to reduce or delay network requests.

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

## Preserving State in Hidden Interfaces

When the project uses a React version that supports `Activity` and an interface that switches visibility frequently must preserve its internal state and DOM, you may use `Activity`. Effects in child components are cleaned up while hidden, so every Effect must implement complete cleanup logic.

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

# Component Definition Location

Components must be defined at the module's top level and must not be defined inside another component function.

# React File Naming

- Use PascalCase for React component files.
- Name Hook files `useXxx.ts` or `useXxx.tsx`.
