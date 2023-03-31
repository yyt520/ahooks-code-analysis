# useLatest

返回当前最新值的 Hook，可以避免闭包问题。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-latest)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/uselatest-demo1/)

```ts
import React, { useState, useEffect } from 'react';
import { useLatest } from 'ahooks';

export default () => {
  const [count, setCount] = useState(0);

  const latestCountRef = useLatest(count);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(latestCountRef.current + 1);
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  return (
    <>
      <p>count: {count}</p>
    </>
  );
};
```

## 核心实现

通过 `useRef`，保持最新的引用值

```ts
function useLatest<T>(value: T) {
  const ref = useRef(value);
  // useRef 保存能保证每次获取到的都是最新的值
  ref.current = value;

  return ref;
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLatest/index.ts)

这个 Hook 还算比较实用点
