# useClickAway

监听目标元素外的点击事件。

```ts
type Target = Element | (() => Element) | React.MutableRefObject<Element>;

/**
 * 监听目标元素外的点击事件。
 * @param onClickAway 触发函数
 * @param target DOM 节点或者 Ref，支持数组
 * @param eventName DOM 节点或者 Ref，支持数组，默认事件是 click
 */
useClickAway<T extends Event = Event>(
  onClickAway: (event: T) => void,
  target: Target | Target[],
  eventName?: string | string[]
);
```

### 使用场景

比如点击显示弹窗之后，此时点击弹窗之外的任意区域时（如弹窗的全局蒙层），该弹窗要自动隐藏。简而言之，属于"点击页面其他元素，XX 组件自动关闭"的功能。

### 实现思路

1. 在 document 上绑定全局事件。如默认支持点击事件，组件卸载的时候移除事件监听
2. 触发事件后，可通过**事件代理**获取到触发事件的对象的引用 e，如果该目标元素 e.target 不在外部传入的 target 元素(列表)中，则触发 onClickAway 函数

### 核心实现

假如只支持点击事件，只能传单个元素且只能是 Ref 类型，实现代码如下：

```js
export default function useClickAway<T extends HTMLElement>(
  onClickAway: (event: MouseEvent) => void,
  refObject: React.RefObject<T>,
) {
  useEffect(() => {
    const handleClick = (e: MouseEvent) => {
      if (
        !refObject.current ||
        refObject.current.contains(e.target as HTMLElement)
      ) {
        return
      }
      onClickAway(e)
    }

    document.addEventListener('click', handleClick)

    return () => {
      document.removeEventListener('click', handleClick)
    }
  }, [refObject, onClickAway])
}
```

ahooks 则继续拓展，思路如下：

1. 同时支持传入 DOM 节点、Ref：需要区分是 DOM 节点、函数、还是 Ref，获取的时候要兼顾所有情况
2. 可传入多个目标元素（支持数组）：通过循环绑定事件，用数组 some 方法判断任一元素包含则触发
3. 可指定监听事件（支持数组）：eventName 由外部传入，不传默认为 click 事件

来看看源码整体实现：

第 1、2 点的实现

```ts
// documentOrShadow 这部分忽略不深究，一般开发场景就是 document
const documentOrShadow = getDocumentOrShadow(target);

const eventNames = Array.isArray(eventName) ? eventName : [eventName];
// 循环绑定事件
eventNames.forEach(event => documentOrShadow.addEventListener(event, handler));

return () => {
  eventNames.forEach(event =>
    documentOrShadow.removeEventListener(event, handler),
  );
};
```

第 3 点 handler 函数的实现：

```ts
const handler = (event: any) => {
  const targets = Array.isArray(target) ? target : [target];
  if (
    // 判断点击的目标元素是否在外部传入的元素（列表）中，是则 return 不执行回调
    targets.some(item => {
      const targetElement = getTargetElement(item); // 这里处理了传入的target是函数、DOM节点、Ref 类型的情况
      return !targetElement || targetElement.contains(event.target);
    })
  ) {
    return;
  }
  // 触发事件
  onClickAwayRef.current(event);
};
```

1. 这里注意触发事件的代码是：`onClickAwayRef.current(event);`，实际是为了保证能拿到最新的函数，可以避免闭包问题

```js
const onClickAwayRef = useLatest(onClickAway);

// 等同于
const onClickAwayRef = useRef(onClickAway);
onClickAwayRef.current = onClickAway;
```

2. `getTargetElement` 方法获取目标元素实现如下：

```js
if (isFunction(target)) {
  targetElement = target();
} else if ('current' in target) {
  targetElement = target.current;
} else {
  targetElement = target;
}
```

### 注意 React17+版本的坑

Reactv17 前，React 将事件委托到 document 上，在 Reactv17 及以后，则委托到根节点，具体见该文：

- [ahooks 的 useClickAway 在 React 17 中不工作了！](https://juejin.cn/post/7110470986419404813)

解决方案是给 useClickAway 的事件类型设置为 mousedown 和 touchstart

### 其它写法实现参考

总体来说 ahooks 的实现功能更齐全考虑的场景更多，但业务开发如果是自己写 Hooks 实现的话，推荐下面的写法，足以覆盖日常开发场景：

- [react-use 的 useClickAway](https://github.com/streamich/react-use/blob/master/src/useClickAway.ts)
- [useHooks 的 useOnClickOutside](https://usehooks.com/useOnClickOutside/)
