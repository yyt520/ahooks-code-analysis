# useIsomorphicLayoutEffect

在 SSR 模式下，使用 `useLayoutEffect` 时，会出现警告，为了避免该警告,可以使用 `useIsomorphicLayoutEffect` 代替 `useLayoutEffect`。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-isomorphic-layout-effect)

### 核心实现

```ts
const isBrowser = !!(
  typeof window !== 'undefined' &&
  window.document &&
  window.document.createElement
);

const useIsomorphicLayoutEffect = isBrowser ? useLayoutEffect : useEffect;
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useIsomorphicLayoutEffect/index.ts)
