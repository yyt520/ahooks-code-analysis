# useEventListener

优雅的使用 addEventListener。

## 使用场景

通用事件监听 Hook，简化写法（无需在 useEffect 卸载函数中手动移除监听函数，由内部去移除）

## 实现思路

1. 判断是否支持 addEventListener
2. 在单独只有 useEffect 实现事件监听移除的基础上，将相关参数都由外部传入，并添加到依赖项
3. 处理事件参数的 TS 类型，addEventListener 的第三个参数也需要由外部传入

## 核心实现

- [EventTarget.addEventListener()](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener)：将指定的监听器注册到 EventTarget 上，当该对象触发指定的事件时，指定的回调函数就会被执行

EventTarget 指任何其他支持事件的对象/元素 `HTMLElement | Element | Document | Window`

符合 EventTarget 接口的都具有下列三个方法

```js
EventTarget.addEventListener();
EventTarget.removeEventListener();
EventTarget.dispatchEvent();
```

- TS 函数重载

> 函数重载指使用相同名称和不同参数数量或类型创建多个方法，让我们定义以多种方式调用的函数。在 TS 中为同一个函数提供多个函数类型定义来进行函数重载

```ts
function useEventListener<K extends keyof HTMLElementEventMap>(
  eventName: K,
  handler: (ev: HTMLElementEventMap[K]) => void,
  options?: Options<HTMLElement>,
): void;
function useEventListener<K extends keyof ElementEventMap>(
  eventName: K,
  handler: (ev: ElementEventMap[K]) => void,
  options?: Options<Element>,
): void;
function useEventListener<K extends keyof DocumentEventMap>(
  eventName: K,
  handler: (ev: DocumentEventMap[K]) => void,
  options?: Options<Document>,
): void;
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (ev: WindowEventMap[K]) => void,
  options?: Options<Window>,
): void;
```

实现：

```ts
function useEventListener(
  eventName: string,
  handler: noop,
  options: Options = {},
) {
  const handlerRef = useLatest(handler);

  useEffectWithTarget(
    () => {
      const targetElement = getTargetElement(options.target, window);
      if (!targetElement?.addEventListener) {
        return;
      }

      const eventListener = (event: Event) => {
        return handlerRef.current(event);
      };

      // 添加监听事件
      targetElement.addEventListener(eventName, eventListener, {
        // true 表示事件在捕获阶段执行，false（默认） 表示事件在冒泡阶段执行
        capture: options.capture,
        // true 表示事件在触发一次后移除，默认 false
        once: options.once,
        // true 表示 listener 永远不会调用 preventDefault()。如果 listener 仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告
        passive: options.passive,
      });

      // 移除监听事件
      return () => {
        targetElement.removeEventListener(eventName, eventListener, {
          capture: options.capture,
        });
      };
    },
    [eventName, options.capture, options.once, options.passive],
    options.target,
  );
}
```
