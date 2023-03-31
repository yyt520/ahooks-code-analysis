# useFullscreen

管理 DOM 全屏的 Hook。

## 原生全屏 API

- [Element.requestFullscreen()](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/requestFullscreen)：用于发出异步请求使元素进入全屏模式
- [Document.exitFullscreen()](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/exitFullscreen)：用于让当前文档退出全屏模式。调用这个方法会让文档回退到上一个调用 Element.requestFullscreen()方法进入全屏模式之前的状态
- ~~[已过时不建议使用]：[Document.fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreen)：只读属性报告文档当前是否以全屏模式显示内容~~
- [Document.fullscreenElement](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenElement)：返回当前文档中正在以全屏模式显示的 Element 节点，如果没有使用全屏模式，则返回 null
- [Document.fullscreenEnabled](https://developer.mozilla.org/en-US/docs/Web/API/Document/fullscreenEnabled)：返回一个布尔值，表明浏览器是否支持全屏模式。全屏模式只在那些不包含窗口化的插件的页面中可用
- [fullscreenchange](https://developer.mozilla.org/en-US/docs/Web/API/Element/fullscreenchange_event)：元素过渡到或过渡到全屏模式时触发的全屏更改事件的事件
- [fullscreenerror](https://developer.mozilla.org/en-US/docs/Web/API/Element/fullscreenerror_event)：在 Element 过渡到或退出全屏模式发生错误后处理事件

## screenfull 库

useFullscreen 内部主要是依赖 [screenfull](https://github.com/sindresorhus/screenfull) 这个库进行实现的。

screenfull 对各种浏览器全屏的 API 进行封装，兼容性好。

下面是该库的 API：

- .request(element, options?)：使元素或者页面切换到全屏
- .exit()：退出全屏
- .toggle(element, options?)：在全屏和非全屏之间切换
- .on(event, function)：添加一个监听器，监听全屏切换或者错误事件。event 支持 `change` 或者 `error`
- .off(event, function)：移除之前注册的事件监听
- .isFullscreen：判断是否为全屏
- .isEnabled：判断当前环境是否支持全屏
- .element：返回该元素是否是全屏模式展示，否则返回 undefined

## 实现思路

看看 `useFullscreen` 的导出值：

```ts
return [
  state,
  {
    enterFullscreen: useMemoizedFn(enterFullscreen),
    exitFullscreen: useMemoizedFn(exitFullscreen),
    toggleFullscreen: useMemoizedFn(toggleFullscreen),
    isEnabled: screenfull.isEnabled,
  },
] as const;
```

那么实现的方向就比较简单了：

1. 内部封装并暴露 toggleFullscreen、enterFullscreen、exitFullscreen 方法，暴露内部是否全屏的状态，还有是否支持全屏的状态
2. 通过 screenfull 库监听`change`事件，在`change`事件里面改变全屏状态与处理执行回调

## 核心实现

三个方法的实现：

```ts
// 进入全屏方法
const enterFullscreen = () => {
  const el = getTargetElement(target);
  if (!el) {
    return;
  }

  if (screenfull.isEnabled) {
    try {
      screenfull.request(el);
      screenfull.on('change', onChange);
    } catch (error) {
      console.error(error);
    }
  }
};

// 退出全屏方法
const exitFullscreen = () => {
  const el = getTargetElement(target);
  if (screenfull.isEnabled && screenfull.element === el) {
    screenfull.exit();
  }
};

const toggleFullscreen = () => {
  if (state) {
    exitFullscreen();
  } else {
    enterFullscreen();
  }
};
```

onChange 方法

```ts
const onChange = () => {
  if (screenfull.isEnabled) {
    const el = getTargetElement(target);
    // screenfull.element：当前元素以全屏模式显示
    if (!screenfull.element) {
      // 退出全屏
      onExitRef.current?.();
      setState(false);
      screenfull.off('change', onChange); // 卸载 change 事件
    } else {
      // 全屏模式展示
      const isFullscreen = screenfull.element === el; // 判断当前全屏元素是否为目标元素
      if (isFullscreen) {
        onEnterRef.current?.();
      } else {
        onExitRef.current?.();
      }
      setState(isFullscreen);
    }
  }
};
```

上方`onChange`以及`exitFullscreen`执行退出全屏前有行需要判断的代码注意下，具体原因可以看下[修复 useFullScreen 当全屏后，子元素重复全屏和退出全屏操作后父元素也会退出全屏](https://github.com/alibaba/hooks/pull/1862)

```ts
// 判断当前全屏元素是否为目标元素，支持对多个元素同时全屏
const isFullscreen = screenfull.element === el;
```

screenfull.element 的实现：

```js
element: {
  enumerable: true,
  get: () => document[nativeAPI.fullscreenElement] ?? undefined,
},
```
