# useDocumentVisibility

监听页面是否可见。

## 使用场景

当页面在背景中或窗口最小化时禁止或开启某些活动，如离开页面停止播放音视频、暂停轮询接口请求

## 实现思路

1. 定义并暴露给外部`document.visibilityState`状态值，通过该字段判断页面是否可见
2. 监听 `visibilitychange` 事件（使用 document 注册），触发回调时更新状态值

## Document.visibilityState 与 visibilitychange 事件

**[Document.visibilityState](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/visibilityState)**（只读属性）

返回 document 的可见性，即当前可见元素的上下文环境。由此可以知道当前文档 (即为页面) 是在背后，或是不可见的隐藏的标签页，或者 (正在) 预渲染，共有三个可能的值。

- visible: 此时页面内容至少是部分可见。即此页面在前景标签页中，并且窗口没有最小化。
- hidden: 此时页面对用户不可见。即文档处于背景标签页或者窗口处于最小化状态，或者操作系统正处于 '锁屏状态' .
- prerender: 页面此时正在渲染中，因此是不可见的。文档只能从此状态开始，永远不能从其他值变为此状态。（prerender 状态只在支持"预渲染"的浏览器上才会出现）。

---

**[visibilitychange](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/visibilitychange_event)**

当其选项卡的内容变得可见或被隐藏时，会在文档上触发 visibilitychange (能见度更改) 事件。

> 警告： 出于兼容性原因，请确保使用 document.addEventListener 而不是 window.addEventListener 来注册回调。Safari <14.0 仅支持前者。

推荐阅读：[Page Visibility API 教程](http://www.ruanyifeng.com/blog/2018/10/page_visibility_api.html)

## 核心实现

```ts
type VisibilityState = 'hidden' | 'visible' | 'prerender' | undefined;

const getVisibility = () => {
  if (!isBrowser) {
    return 'visible';
  }
  // 返回document的可见性，即当前可见元素的上下文环境
  return document.visibilityState;
};

function useDocumentVisibility(): VisibilityState {
  const [documentVisibility, setDocumentVisibility] = useState(() =>
    getVisibility(),
  );

  // 监听事件
  useEventListener(
    'visibilitychange',
    () => {
      setDocumentVisibility(getVisibility());
    },
    {
      target: () => document,
    },
  );

  return documentVisibility;
}

export default useDocumentVisibility;
```
