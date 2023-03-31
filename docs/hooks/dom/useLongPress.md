# useLongPress

监听目标元素的长按事件。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-long-press)

## 基本用法

支持参数：

```ts
export interface Options {
  delay?: number;
  moveThreshold?: { x?: number; y?: number };
  onClick?: (event: EventType) => void;
  onLongPressEnd?: (event: EventType) => void;
}
```

[官方在线 Demo](https://ahooks.js.org/~demos/uselongpress-demo1/)

```ts
import React, { useState, useRef } from 'react';
import { useLongPress } from 'ahooks';

export default () => {
  const [counter, setCounter] = useState(0);
  const ref = useRef<HTMLButtonElement>(null);

  useLongPress(() => setCounter(s => s + 1), ref);

  return (
    <div>
      <button ref={ref} type="button">
        Press me
      </button>
      <p>counter: {counter}</p>
    </div>
  );
};
```

## touch 事件

- [touchstart](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/touchstart_event)：在一个或多个触点与触控设备表面接触时被触发
- [touchmove](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/touchmove_event)：在触点于触控平面上移动时触发
- [touchend](https://developer.mozilla.org/en-US/docs/Web/API/Element/touchend_event)：当触点离开触控平面时触发 touchend 事件

## 实现思路

1. 判断当前环境是否支持 touch 事件：支持则监听 `touchstart`、`touchend` 事件，不支持则监听 `mousedown`、`mouseup`、`mouseleave` 事件
2. 根据触发监听事件和定时器共同来判断是否达到长按事件，达到则触发外部回调
3. 如果外部有传 `moveThreshold（按下后移动阈值）`参数 ，则需要监听 `mousemove` 或`touchmove` 事件进行处理

## 核心实现

根据[实现思路]第一条，很容易看懂实现大致框架代码：

```ts
type EventType = MouseEvent | TouchEvent;

// 是否支持 touch 事件
const touchSupported =
  isBrowser &&
  // @ts-ignore
  ('ontouchstart' in window ||
    (window.DocumentTouch && document instanceof DocumentTouch));

function useLongPress(
  onLongPress: (event: EventType) => void,
  target: BasicTarget,
  { delay = 300, moveThreshold, onClick, onLongPressEnd }: Options = {},
) {
  const onLongPressRef = useLatest(onLongPress);
  const onClickRef = useLatest(onClick);
  const onLongPressEndRef = useLatest(onLongPressEnd);

  const timerRef = useRef<ReturnType<typeof setTimeout>>();
  const isTriggeredRef = useRef(false);
  // 是否有设置移动阈值
  const hasMoveThreshold = !!(
    (moveThreshold?.x && moveThreshold.x > 0) ||
    (moveThreshold?.y && moveThreshold.y > 0)
  );

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target);
      if (!targetElement?.addEventListener) {
        return;
      }

      const overThreshold = (event: EventType) => {};

      function getClientPosition(event: EventType) {}

      const onStart = (event: EventType) => {};

      const onMove = (event: TouchEvent) => {};

      const onEnd = (
        event: EventType,
        shouldTriggerClick: boolean = false,
      ) => {};

      const onEndWithClick = (event: EventType) => onEnd(event, true);

      if (!touchSupported) {
        // 不支持 touch 事件
        targetElement.addEventListener('mousedown', onStart);
        targetElement.addEventListener('mouseup', onEndWithClick);
        targetElement.addEventListener('mouseleave', onEnd);
        if (hasMoveThreshold)
          targetElement.addEventListener('mousemove', onMove);
      } else {
        // 支持 touch 事件
        targetElement.addEventListener('touchstart', onStart);
        targetElement.addEventListener('touchend', onEndWithClick);
        if (hasMoveThreshold)
          targetElement.addEventListener('touchmove', onMove);
      }

      // 卸载函数解绑监听事件
      return () => {
        // 清除定时器，重置状态
        if (timerRef.current) {
          clearTimeout(timerRef.current);
          isTriggeredRef.current = false;
        }
        if (!touchSupported) {
          targetElement.removeEventListener('mousedown', onStart);
          targetElement.removeEventListener('mouseup', onEndWithClick);
          targetElement.removeEventListener('mouseleave', onEnd);
          if (hasMoveThreshold)
            targetElement.removeEventListener('mousemove', onMove);
        } else {
          targetElement.removeEventListener('touchstart', onStart);
          targetElement.removeEventListener('touchend', onEndWithClick);
          if (hasMoveThreshold)
            targetElement.removeEventListener('touchmove', onMove);
        }
      };
    },
    [],
    target,
  );
}
```

对于是否支持 touch 事件的判断代码，需要了解一种场景，在搜的时候发现一篇文章可以看下：[touchstart 与 click 不得不说的故事](https://juejin.cn/post/6844903588683071495)

**如何判断长按事件**：

1. 在 onStart 设置一个定时器 setTimeout 用来判断长按时间，在定时器回调将 isTriggeredRef.current 设置为 true，表示触发了长按事件；
2. 在 onEnd 清除定时器并判断 isTriggeredRef.current 的值，true 代表触发了长按事件；false 代表没触发 setTimeout 里面的回调，则不触发长按事件。

```ts
const onStart = (event: EventType) => {
  timerRef.current = setTimeout(() => {
    // 达到设置的长按时间
    onLongPressRef.current(event);
    isTriggeredRef.current = true;
  }, delay);
};

const onEnd = (event: EventType, shouldTriggerClick: boolean = false) => {
  // 清除 onStart 设置的定时器
  if (timerRef.current) {
    clearTimeout(timerRef.current);
  }
  // 判断是否达到长按时间
  if (isTriggeredRef.current) {
    onLongPressEndRef.current?.(event);
  }
  // 是否触发点击事件
  if (shouldTriggerClick && !isTriggeredRef.current && onClickRef.current) {
    onClickRef.current(event);
  }
  // 重置
  isTriggeredRef.current = false;
};
```

实现了[实现思路]的前两点，接下来需要实现第三点，传 `moveThreshold` 的情况

```ts
const hasMoveThreshold = !!(
  (moveThreshold?.x && moveThreshold.x > 0) ||
  (moveThreshold?.y && moveThreshold.y > 0)
);
```

> clientX、clientY：点击位置距离当前 body 可视区域的 x，y 坐标

```ts
const onStart = (event: EventType) => {
  if (hasMoveThreshold) {
    const { clientX, clientY } = getClientPosition(event);
    // 记录首次点击/触屏时的位置
    pervPositionRef.current.x = clientX;
    pervPositionRef.current.y = clientY;
  }
  // ...
};

// 传 moveThreshold 需绑定 onMove 事件
const onMove = (event: TouchEvent) => {
  if (timerRef.current && overThreshold(event)) {
    // 超过移动阈值不触发长按事件，并清除定时器
    clearInterval(timerRef.current);
    timerRef.current = undefined;
  }
};

// 判断是否超过移动阈值
const overThreshold = (event: EventType) => {
  const { clientX, clientY } = getClientPosition(event);
  const offsetX = Math.abs(clientX - pervPositionRef.current.x);
  const offsetY = Math.abs(clientY - pervPositionRef.current.y);

  return !!(
    (moveThreshold?.x && offsetX > moveThreshold.x) ||
    (moveThreshold?.y && offsetY > moveThreshold.y)
  );
};

function getClientPosition(event: EventType) {
  if (event instanceof TouchEvent) {
    return {
      clientX: event.touches[0].clientX,
      clientY: event.touches[0].clientY,
    };
  }

  if (event instanceof MouseEvent) {
    return {
      clientX: event.clientX,
      clientY: event.clientY,
    };
  }

  console.warn('Unsupported event type');

  return { clientX: 0, clientY: 0 };
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useLongPress/index.ts)
