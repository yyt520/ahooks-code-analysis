# useHover

监听 DOM 元素是否有鼠标悬停。

## 鼠标监听事件

- [mouseenter](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mouseenter_event)：第一次移动到触发事件元素中的激活区域时触发
- [mouseleave](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mouseleave_event)：在定点设备（通常是鼠标）的指针移出某个元素时被触发

扩展下几个鼠标事件的区别：

- mouseenter：当鼠标移入某元素时触发。
- mouseleave：当鼠标移出某元素时触发。
- mouseover：当鼠标移入某元素时触发，移入和移出其子元素时也会触发。
- mouseout：当鼠标移出某元素时触发，移入和移出其子元素时也会触发。
- mousemove：鼠标在某元素上移动时触发，即使在其子元素上也会触发。

## 核心实现

原理是监听 `mouseenter` 触发 `onEnter` 回调，切换状态为 true；监听 `mouseleave` 触发 `onLeave`回调，切换状态为 false。

完整实现：

```ts
export interface Options {
  onEnter?: () => void;
  onLeave?: () => void;
  onChange?: (isHovering: boolean) => void;
}

export default (target: BasicTarget, options?: Options): boolean => {
  const { onEnter, onLeave, onChange } = options || {};

  // useBoolean：优雅的管理 boolean 状态的 Hook
  const [state, { setTrue, setFalse }] = useBoolean(false);

  // 监听 mouseenter 判断有鼠标进入目标元素
  useEventListener(
    'mouseenter',
    () => {
      onEnter?.();
      setTrue();
      onChange?.(true);
    },
    {
      target,
    },
  );

  // 监听 mouseleave 判断有鼠标是否移出目标元素
  useEventListener(
    'mouseleave',
    () => {
      onLeave?.();
      setFalse();
      onChange?.(false);
    },
    {
      target,
    },
  );

  return state;
};
```
