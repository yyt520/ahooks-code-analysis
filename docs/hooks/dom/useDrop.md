# useDrop

处理元素拖拽的 Hook。

## 使用场景

- useDrop 可以单独使用来接收文件、文字和网址的拖拽。
- 向节点内触发粘贴动作也会被视为拖拽

## 涉及的拖拽 API

拖拽相关事件：

- [dragenter](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragenter_event)：事件在可拖动的元素或者被选择的文本进入一个有效的放置目标时触发。
- [dragleave](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragleave_event)：在拖动的元素或选中的文本离开一个有效的放置目标时被触发。
- [dragover](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/dragover_event)：在可拖动的元素或者被选择的文本被拖进一个有效的放置目标时（每几百毫秒）触发。
- [drop](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/drop_event)：当一个元素或是选中的文字被拖拽释放到一个有效的释放目标位置时，drop 事件被抛出。
- [paste](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/paste_event)：当用户在浏览器用户界面发起“粘贴”操作时，会触发 paste 事件。

## 实现思路

1. 监听以上 5 个事件
2. 另外在 drop 和 paste 事件中获取到 DataTransfer 数据，并根据数据类型进行特定的处理，将处理好的数据通过回调（onText/onFiles/onUri/onDom）给外部直接获取使用。

```ts
export interface Options {
  // 根据 drop 事件数据类型自定义回调函数
  onFiles?: (files: File[], event?: React.DragEvent) => void;
  onUri?: (url: string, event?: React.DragEvent) => void;
  onDom?: (content: any, event?: React.DragEvent) => void;
  onText?: (text: string, event?: React.ClipboardEvent) => void;
  // 原生事件
  onDragEnter?: (event?: React.DragEvent) => void;
  onDragOver?: (event?: React.DragEvent) => void;
  onDragLeave?: (event?: React.DragEvent) => void;
  onDrop?: (event?: React.DragEvent) => void;
  onPaste?: (event?: React.ClipboardEvent) => void;
}

const useDrop = (target: BasicTarget, options: Options = {}) => {};
```

## 核心实现

主函数实现比较简单，需要注意的时候在特定事件需要阻止默认事件`event.preventDefault();`和阻止事件冒泡`event.stopPropagation();`，让拖拽能正常的工作

```ts
const useDrop = (target: BasicTarget, options: Options = {}) => {
  const optionsRef = useLatest(options);

  // https://stackoverflow.com/a/26459269
  const dragEnterTarget = useRef<any>();

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(target);
      if (!targetElement?.addEventListener) {
        return;
      }

      // 处理 DataTransfer 不同数据类型数据
      const onData = (
        dataTransfer: DataTransfer,
        event: React.DragEvent | React.ClipboardEvent,
      ) => {};

      const onDragEnter = (event: React.DragEvent) => {
        event.preventDefault();
        event.stopPropagation();

        dragEnterTarget.current = event.target;
        optionsRef.current.onDragEnter?.(event);
      };

      const onDragOver = (event: React.DragEvent) => {
        event.preventDefault(); // 调用 event.preventDefault() 使得该元素能够接收 drop 事件
        optionsRef.current.onDragOver?.(event);
      };

      const onDragLeave = (event: React.DragEvent) => {
        if (event.target === dragEnterTarget.current) {
          optionsRef.current.onDragLeave?.(event);
        }
      };

      const onDrop = (event: React.DragEvent) => {
        event.preventDefault();
        onData(event.dataTransfer, event);
        optionsRef.current.onDrop?.(event);
      };

      const onPaste = (event: React.ClipboardEvent) => {
        onData(event.clipboardData, event);
        optionsRef.current.onPaste?.(event);
      };

      targetElement.addEventListener('dragenter', onDragEnter as any);
      targetElement.addEventListener('dragover', onDragOver as any);
      targetElement.addEventListener('dragleave', onDragLeave as any);
      targetElement.addEventListener('drop', onDrop as any);
      targetElement.addEventListener('paste', onPaste as any);

      return () => {
        targetElement.removeEventListener('dragenter', onDragEnter as any);
        targetElement.removeEventListener('dragover', onDragOver as any);
        targetElement.removeEventListener('dragleave', onDragLeave as any);
        targetElement.removeEventListener('drop', onDrop as any);
        targetElement.removeEventListener('paste', onPaste as any);
      };
    },
    [],
    target,
  );
};
```

在 drop 和 paste 事件中，获取到 DataTransfer 数据并传给 onData 方法，根据数据类型进行特定的处理

- [DataTransfer](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransfer)：DataTransfer 对象用于保存拖动并放下（drag and drop）过程中的数据。它可以保存一项或多项数据，这些数据项可以是一种或者多种数据类型。关于拖放的更多信息，请参见 [Drag and Drop](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_Drag_and_Drop_API)
- [DataTransfer.getData()](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransfer/getData)接受指定类型的拖放（以 DOMString 的形式）数据。如果拖放行为没有操作任何数据，会返回一个空字符串。数据类型有：text/plain，text/uri-list
- [DataTransferItem](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransferItem)：拖拽项。

```ts
const onData = (
  dataTransfer: DataTransfer,
  event: React.DragEvent | React.ClipboardEvent,
) => {
  const uri = dataTransfer.getData('text/uri-list'); // URL格式列表（链接）
  const dom = dataTransfer.getData('custom'); // 自定义数据，需要与 useDrag 搭配使用

  // 根据数据类型进行特定的处理
  // 拖拽/粘贴自定义 DOM 节点的回调
  if (dom && optionsRef.current.onDom) {
    let data = dom;
    try {
      data = JSON.parse(dom);
    } catch (e) {
      data = dom;
    }
    optionsRef.current.onDom(data, event as React.DragEvent);
    return;
  }

  // 拖拽/粘贴链接的回调
  if (uri && optionsRef.current.onUri) {
    optionsRef.current.onUri(uri, event as React.DragEvent);
    return;
  }

  // 拖拽/粘贴文件的回调
  // dataTransfer.files：拖动操作中的文件列表，操作中每个文件的一个列表项。如果拖动操作没有文件，此列表为空
  if (
    dataTransfer.files &&
    dataTransfer.files.length &&
    optionsRef.current.onFiles
  ) {
    optionsRef.current.onFiles(
      Array.from(dataTransfer.files),
      event as React.DragEvent,
    );
    return;
  }

  // 拖拽/粘贴文字的回调
  if (
    dataTransfer.items &&
    dataTransfer.items.length &&
    optionsRef.current.onText
  ) {
    // dataTransfer.items：拖动操作中 数据传输项的列表。该列表包含了操作中每一项目的对应项，如果操作没有项目，则列表为空
    // getAsString：使用拖拽项的字符串作为参数执行指定回调函数
    dataTransfer.items[0].getAsString(text => {
      optionsRef.current.onText!(text, event as React.ClipboardEvent);
    });
  }
};
```
