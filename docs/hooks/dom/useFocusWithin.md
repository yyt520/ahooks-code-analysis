# useFocusWithin

监听当前焦点是否在某个区域之内，同 css 属性: [focus-within](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-within)

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-focus-within)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/usefocuswithin-demo1/)

使用 ref 设置需要监听的区域。可以通过鼠标点击外部区域，或者使用键盘的 tab 等按键来切换焦点。

```ts
import React, { useRef } from 'react';
import { useFocusWithin } from 'ahooks';
import { message } from 'antd';

export default () => {
  const ref = useRef(null);
  const isFocusWithin = useFocusWithin(ref, {
    onFocus: () => {
      message.info('focus');
    },
    onBlur: () => {
      message.info('blur');
    },
  });
  return (
    <div>
      <div
        ref={ref}
        style={{
          padding: 16,
          backgroundColor: isFocusWithin ? 'red' : '',
          border: '1px solid gray',
        }}
      >
        <label style={{ display: 'block' }}>
          First Name: <input />
        </label>
        <label style={{ display: 'block', marginTop: 16 }}>
          Last Name: <input />
        </label>
      </div>
      <p>isFocusWithin: {JSON.stringify(isFocusWithin)}</p>
    </div>
  );
};
```

## 核心实现

主要还是监听了 [focusin](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusin_event) 和 [focusout](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusout_event) 事件

- focusin：当元素聚焦时会触发。和 focus 一样，只是 focusin 事件支持冒泡；
- focusout：当元素即将失去焦点时会被触发。和 blur 一样，只是 focusout 事件支持冒泡。

触发顺序：

在同时支持四种事件的浏览器中，当焦点在两个元素之间切换时，触发顺序如下（不同浏览器效果可能不同）：

- focusin 在第一个目标元素获得焦点前触发
- focus  在第一个目标元素获得焦点后触发
- focusout  第一个目标失去焦点时触发
- focusin  第二个元素获得焦点前触发
- blur  第一个元素失去焦点时触发
- focus  第二个元素获得焦点后触发

参考：[focus/blur VS focusin/focusout](https://juejin.cn/post/6888302279753990158#heading-7)

`MouseEvent.relatedTarget` 属性返回与触发鼠标事件的元素相关的元素：
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bab123a0cf794e4dbe3075d183ff7535~tplv-k3u1fbpfcp-watermark.image?)

```ts
export default function useFocusWithin(target: BasicTarget, options?: Options) {
  const [isFocusWithin, setIsFocusWithin] = useState(false);
  const { onFocus, onBlur, onChange } = options || {};
  // 监听 focusin 事件
  useEventListener(
    'focusin',
    (e: FocusEvent) => {
      if (!isFocusWithin) {
        onFocus?.(e);
        onChange?.(true);
        setIsFocusWithin(true);
      }
    },
    {
      target,
    },
  );

  // 监听 focusout 事件
  useEventListener(
    'focusout',
    (e: FocusEvent) => {
      // relatedTarget 属性返回与触发鼠标事件的元素相关的元素。
      // https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/relatedTarget
      if (
        isFocusWithin &&
        !(e.currentTarget as Element)?.contains?.(e.relatedTarget as Element)
      ) {
        onBlur?.(e);
        onChange?.(false);
        setIsFocusWithin(false);
      }
    },
    {
      target,
    },
  );

  return isFocusWithin; // 焦点是否在当前区域
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useFocusWithin/index.ts)
