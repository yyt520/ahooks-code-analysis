# useCreation

useCreation 是 useMemo 或 useRef 的替代品。

因为 useMemo 不能保证被 memo 的值一定不会被重计算，而 useCreation 可以保证这一点。以下为 [React 官方文档](https://reactjs.org/docs/hooks-reference.html#usememo)中的介绍：

> You may rely on useMemo as a performance optimization, not as a semantic guarantee. In the future, React may choose to “forget” some previously memoized values and recalculate them on next render, e.g. to free memory for offscreen components. Write your code so that it still works without useMemo — and then add it to optimize performance.

而相比于 useRef，你可以使用 useCreation 创建一些常量，这些常量和 useRef 创建出来的 ref 有很多使用场景上的相似，但对于复杂常量的创建，useRef 却容易出现潜在的性能隐患。

```ts
const a = useRef(new Subject()); // 每次重渲染，都会执行实例化 Subject 的过程，即便这个实例立刻就被扔掉了
const b = useCreation(() => new Subject(), []); // 通过 factory 函数，可以避免性能隐患
```

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-creation)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usecreation-demo1/)

确保实例不会被重复创建。点击 "Rerender" 按钮，触发组件的更新，但 Foo 的实例会保持不变

```ts
import React, { useState } from 'react';
import { useCreation } from 'ahooks';

class Foo {
  constructor() {
    this.data = Math.random();
  }

  data: number;
}

export default function() {
  const foo = useCreation(() => new Foo(), []);
  const [, setFlag] = useState({});
  return (
    <>
      <p>{foo.data}</p>
      <button
        type="button"
        onClick={() => {
          setFlag({});
        }}
      >
        Rerender
      </button>
    </>
  );
}
```

我们发现看下来用法与 useMemo 完全一致，算是 useMemo 的再优化版本

## 实现思路

useCreation 的核心依赖是 useRef

1. 把相关值（依赖，具体值，是否初始化）保存在 useRef 中，将值进行缓存
2. 初始化或依赖变更（检测 useRef 的旧值依赖与当前依赖用 `Object.is()`比对）时，不一致则进行更新。

## 核心实现

```ts
// 通过 Object.is 比较依赖数组的值是否相等
function depsAreSame(oldDeps: DependencyList, deps: DependencyList): boolean {
  if (oldDeps === deps) return true;
  for (let i = 0; i < oldDeps.length; i++) {
    if (!Object.is(oldDeps[i], deps[i])) return false;
  }
  return true;
}

function useCreation<T>(factory: () => T, deps: DependencyList) {
  const { current } = useRef({
    deps,
    obj: undefined as undefined | T,
    initialized: false,
  });
  // 初始化或依赖变更时，重新初始化
  if (current.initialized === false || !depsAreSame(current.deps, deps)) {
    current.deps = deps; // 更新依赖
    current.obj = factory(); // 执行创建所需对象的函数
    current.initialized = true; // 初始化标识为 true
  }
  return current.obj as T;
}
```

这个 Hooks 属于是精益求精了，本来使用 `useMemo` 这个优化型的 Hook 就要考量场景，暂时还不知道哪些精细化场景用 `useCreation` 比较好
