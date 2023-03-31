# useMemoizedFn

持久化 function 的 Hook，理论上，可以使用 useMemoizedFn 完全代替 useCallback。

在某些场景中，我们需要使用 useCallback 来记住一个函数，但是在第二个参数 deps 变化时，会重新生成函数，导致函数地址变化。

使用 useMemoizedFn，可以省略第二个参数 deps，同时保证函数地址永远不会变化。
它的功能和 useCallback 类似，不过使用更简单，不需要提供 dep 数组。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-memoized-fn)

## 基本用法

useMemoizedFn 与 useCallback 可以实现同样的效果。但 useMemoizedFn 函数地址不会变化，可以用于性能优化。

示例中 memoizedFn 是不会变化的，callbackFn 在 count 变化时变化。

[官方在线 Demo](https://ahooks.js.org/~demos/usememoizedfn-demo2/)

```ts
import { useMemoizedFn } from 'ahooks';
import { message } from 'antd';
import React, { useCallback, useRef, useState } from 'react';

export default () => {
  const [count, setCount] = useState(0);

  const callbackFn = useCallback(() => {
    message.info(`Current count is ${count}`);
  }, [count]);

  const memoizedFn = useMemoizedFn(() => {
    message.info(`Current count is ${count}`);
  });

  return (
    <>
      <p>count: {count}</p>
      <button
        type="button"
        onClick={() => {
          setCount(c => c + 1);
        }}
      >
        Add Count
      </button>

      <p>
        You can click the button to see the number of sub-component renderings
      </p>

      <div style={{ marginTop: 32 }}>
        <h3>Component with useCallback function:</h3>
        {/* use callback function, ExpensiveTree component will re-render on state change */}
        <ExpensiveTree showCount={callbackFn} />
      </div>

      <div style={{ marginTop: 32 }}>
        <h3>Component with useMemoizedFn function:</h3>
        {/* use memoized function, ExpensiveTree component will only render once */}
        <ExpensiveTree showCount={memoizedFn} />
      </div>
    </>
  );
};

// some expensive component with React.memo
const ExpensiveTree = React.memo<{ [key: string]: any }>(({ showCount }) => {
  const renderCountRef = useRef(0);
  renderCountRef.current += 1;

  return (
    <div>
      <p>Render Count: {renderCountRef.current}</p>
      <button type="button" onClick={showCount}>
        showParentCount
      </button>
    </div>
  );
});
```

## 使用场景

useMemoizedFn 能保证每次运行过程中保持最新的函数地址引用，适用于用于较为复杂的场景/组件，在属性传递过程减少不必要的 re-render

## 实现思路

针对上面 Demo 来说，如果我们要自己实现 `useMemoizedFn`，就是解决 useCallback 在 Demo 存在的缺陷。

1. callbackFn 的引用地址不能随 render 而改变，并在需要 count 值的时候能实时拿到（ref 保持引用地址不变）
2. 无需添加 dep 依赖（在 render 期间 ref 就要维持赋值最新的引用）

## 核心实现

1. 一个 `useRef` 保持外部传入的 fn 的引用 fnRef
2. 另一个 `useRef` 负责返回持久化函数 memoizedFn（内部获取并执行 fnRef），当实例后引用地址永远保持不变

源码不难，但确实是巧妙的实现。

```ts
/**
 * 持久化 function 的 Hook
 * @param fn 需要持久化的函数
 * @returns 引用地址永远不会变化的 fn
 */
function useMemoizedFn<T extends noop>(fn: T) {
  if (isDev) {
    if (!isFunction(fn)) {
      console.error(
        `useMemoizedFn expected parameter is a function, got ${typeof fn}`,
      );
    }
  }
  // 通过 useRef 保证持有最新的引用
  const fnRef = useRef<T>(fn);

  // why not write `fnRef.current = fn`?
  // https://github.com/alibaba/hooks/issues/728
  // useMemo 在这里并不是核心，只是避免在 devtool 模式下的异常行为；
  fnRef.current = useMemo(() => fn, [fn]); // 在 render 期间实时拿到最新的fn，直观看就是：fnRef.current = fn

  const memoizedFn = useRef<PickFunction<T>>();
  if (!memoizedFn.current) {
    // 内部定义方法，赋值给 memoizedFn.current，这样返回出去的方法实例化后引用地址永远保持不变
    memoizedFn.current = function(this, ...args) {
      return fnRef.current.apply(this, args); // 调用的时候再去取 fnRef（存有最新的 fn 引用）
    };
  }

  // 返回的持久化函数
  return memoizedFn.current as T;
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMemoizedFn/index.ts)
