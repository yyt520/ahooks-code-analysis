# useDrag

处理元素拖拽的 Hook。

## 使用场景

useDrag 允许一个 DOM 节点被拖拽，需要配合 useDrop 使用。

## 涉及的拖拽事件

- [dragstart](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragstart_event): 在用户开始拖动元素或被选择的文本时调用。
- [dragend](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragend_event): 在拖放操作结束时触发（通过释放鼠标按钮或单击 escape 键）。

## 实现思路

1. 内部监听 dragstart 和 dragend 方法触发回调给外部使用
2. dragstart 事件触发时支持设置自定义数据到 dataTransfer 中

## 核心实现

```ts
export interface Options {
  // 在用户开始拖动元素或被选择的文本时调用
  onDragStart?: (event: React.DragEvent) => void;
  // 在拖放操作结束时触发（通过释放鼠标按钮或单击 escape 键）
  onDragEnd?: (event: React.DragEvent) => void;
}

const useDrag = <T>(data: T, target: BasicTarget, options: Options = {}) => {
  const optionsRef = useLatest(options);
  const dataRef = useLatest(data);
  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target);
      if (!targetElement?.addEventListener) {
        return;
      }

      const onDragStart = (event: React.DragEvent) => {
        optionsRef.current.onDragStart?.(event);
        // 设置自定义数据到 dataTransfer 中，搭配 useDrop 的 onDom 回调可获取当前设置的内容
        event.dataTransfer.setData('custom', JSON.stringify(dataRef.current));
      };

      const onDragEnd = (event: React.DragEvent) => {
        optionsRef.current.onDragEnd?.(event);
      };

      targetElement.setAttribute('draggable', 'true');

      targetElement.addEventListener('dragstart', onDragStart as any);
      targetElement.addEventListener('dragend', onDragEnd as any);

      return () => {
        targetElement.removeEventListener('dragstart', onDragStart as any);
        targetElement.removeEventListener('dragend', onDragEnd as any);
      };
    },
    [],
    target,
  );
};
```
