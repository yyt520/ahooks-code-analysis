# useSize

监听 DOM 节点尺寸变化的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-size)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usesize-demo1/)

```ts
import React, { useRef } from 'react';
import { useSize } from 'ahooks';

export default () => {
  const ref = useRef(null);
  const size = useSize(ref);
  return (
    <div ref={ref}>
      <p>Try to resize the preview window </p>
      <p>
        width: {size?.width}px, height: {size?.height}px
      </p>
    </div>
  );
};
```

## 核心实现

这里涉及 [ResizeObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/ResizeObserver)

源码较容易理解，就不展开了

```ts
// 	目标 DOM 节点的尺寸
type Size = { width: number; height: number };

function useSize(target: BasicTarget): Size | undefined {
  const [state, setState] = useRafState<Size>();

  useIsomorphicLayoutEffectWithTarget(
    () => {
      const el = getTargetElement(target);

      if (!el) {
        return;
      }
      // Resize Observer API 提供了一种高性能的机制，通过该机制，代码可以监视元素的大小更改，并且每次大小更改时都会向观察者传递通知
      const resizeObserver = new ResizeObserver(entries => {
        entries.forEach(entry => {
          // 返回 DOM 节点的尺寸
          const { clientWidth, clientHeight } = entry.target;
          setState({
            width: clientWidth,
            height: clientHeight,
          });
        });
      });

      // 监听目标元素
      resizeObserver.observe(el);
      return () => {
        resizeObserver.disconnect();
      };
    },
    [],
    target,
  );

  return state;
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useSize/index.ts)
