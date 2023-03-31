# useResponsive

获取响应式信息。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-mouse)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useresponsive-demo1/)

```ts
import React from 'react';
import { configResponsive, useResponsive } from 'ahooks';

configResponsive({
  small: 0,
  middle: 800,
  large: 1200,
});

export default function() {
  const responsive = useResponsive();
  return (
    <>
      <p>Please change the width of the browser window to see the effect: </p>
      {Object.keys(responsive).map(key => (
        <p key={key}>
          {key} {responsive[key] ? '✔' : '✘'}
        </p>
      ))}
    </>
  );
}
```

## 实现思路

1. 监听 [resize](https://developer.mozilla.org/en-US/docs/Web/API/Window/resize_event) 事件，在 resize 事件处理函数中需要计算，且判断是否需要更新处理（性能优化）。
2. 计算：遍历对比 `window.innerWidth` 与配置项的每一种屏幕宽度，大于设置为 true，否则为 false

## 核心实现

```ts
type Subscriber = () => void;

const subscribers = new Set<Subscriber>();

type ResponsiveConfig = Record<string, number>;
type ResponsiveInfo = Record<string, boolean>;

let info: ResponsiveInfo;

// 默认的响应式配置和 bootstrap 是一致的
let responsiveConfig: ResponsiveConfig = {
  xs: 0,
  sm: 576,
  md: 768,
  lg: 992,
  xl: 1200,
};

function handleResize() {
  const oldInfo = info;
  calculate();
  if (oldInfo === info) return; // 没有更新，不处理
  for (const subscriber of subscribers) {
    subscriber();
  }
}

let listening = false; // 避免多次监听

// 计算当前的屏幕宽度与配置比较
function calculate() {
  const width = window.innerWidth; // 返回窗口的的宽度
  const newInfo = {} as ResponsiveInfo;
  let shouldUpdate = false; // 判断是否需要更新
  for (const key of Object.keys(responsiveConfig)) {
    newInfo[key] = width >= responsiveConfig[key];
    if (newInfo[key] !== info[key]) {
      shouldUpdate = true;
    }
  }
  if (shouldUpdate) {
    info = newInfo;
  }
}

// 自定义配置响应式断点（只需配置一次）
export function configResponsive(config: ResponsiveConfig) {
  responsiveConfig = config;
  if (info) calculate();
}

export function useResponsive() {
  if (isBrowser && !listening) {
    info = {};
    calculate();
    window.addEventListener('resize', handleResize);
    listening = true;
  }
  const [state, setState] = useState<ResponsiveInfo>(info);

  useEffect(() => {
    if (!isBrowser) return;

    // In React 18's StrictMode, useEffect perform twice, resize listener is remove, so handleResize is never perform.
    // https://github.com/alibaba/hooks/issues/1910
    if (!listening) {
      window.addEventListener('resize', handleResize);
    }

    const subscriber = () => {
      setState(info);
    };
    // 添加订阅
    subscribers.add(subscriber);
    return () => {
      // 组件卸载时取消订阅
      subscribers.delete(subscriber);
      // 当全局订阅器不再有订阅器，则移除 resize 监听事件
      if (subscribers.size === 0) {
        window.removeEventListener('resize', handleResize);
        listening = false;
      }
    };
  }, []);

  return state;
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useResponsive/index.ts)
