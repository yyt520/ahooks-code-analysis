# useMouse

监听鼠标位置。

### 基本用法

API：

```ts
const state: {
  screenX: number, // 距离显示器左侧的距离
  screenY: number, // 距离显示器顶部的距离
  clientX: number, // 距离当前视窗左侧的距离
  clientY: number, // 距离当前视窗顶部的距离
  pageX: number, // 距离完整页面左侧的距离
  pageY: number, // 距离完整页面顶部的距离
  elementX: number, // 距离指定元素左侧的距离
  elementY: number, // 距离指定元素顶部的距离
  elementH: number, // 指定元素的高
  elementW: number, // 指定元素的宽
  elementPosX: number, // 指定元素距离完整页面左侧的距离
  elementPosY: number, // 指定元素距离完整页面顶部的距离
} = useMouse(target?: Target);
```

[官方在线 Demo](https://ahooks.js.org/zh-CN/hooks/use-mouse)

```ts
import React, { useRef } from 'react';
import { useMouse } from 'ahooks';

export default () => {
  const ref = useRef(null);
  const mouse = useMouse(ref.current);

  return (
    <>
      <div
        ref={ref}
        style={{
          width: '200px',
          height: '200px',
          backgroundColor: 'gray',
          color: 'white',
          lineHeight: '200px',
          textAlign: 'center',
        }}
      >
        element
      </div>
      <div>
        <p>
          Mouse In Element - x: {mouse.elementX}, y: {mouse.elementY}
        </p>
        <p>
          Element Position - x: {mouse.elementPosX}, y: {mouse.elementPosY}
        </p>
        <p>
          Element Dimensions - width: {mouse.elementW}, height: {mouse.elementH}
        </p>
      </div>
    </>
  );
};
```

### 核心实现

实现原理：通过监听 [mousemove](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mousemove_event) 方法，获取鼠标的位置。通过 [getBoundingClientRect](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)（提供了元素的大小及其相对于视口的位置） 获取到 target 元素的位置大小，计算出鼠标相对于元素的位置。

```ts
export default (target?: BasicTarget) => {
  const [state, setState] = useRafState(initState);

  useEventListener(
    'mousemove',
    (event: MouseEvent) => {
      const { screenX, screenY, clientX, clientY, pageX, pageY } = event;
      const newState = {
        screenX,
        screenY,
        clientX,
        clientY,
        pageX,
        pageY,
        elementX: NaN,
        elementY: NaN,
        elementH: NaN,
        elementW: NaN,
        elementPosX: NaN,
        elementPosY: NaN,
      };
      const targetElement = getTargetElement(target);
      if (targetElement) {
        const {
          left,
          top,
          width,
          height,
        } = targetElement.getBoundingClientRect();

        // 计算鼠标相对于元素的位置
        newState.elementPosX = left + window.pageXOffset; // window.pageXOffset：window.scrollX 的别名
        newState.elementPosY = top + window.pageYOffset; // scrollY 的别名
        newState.elementX = pageX - newState.elementPosX;
        newState.elementY = pageY - newState.elementPosY;
        newState.elementW = width;
        newState.elementH = height;
      }
      setState(newState);
    },
    {
      target: () => document,
    },
  );

  return state;
};
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useMouse/index.ts)
