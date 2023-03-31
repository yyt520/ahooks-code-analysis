# useTitle

用于设置页面标题。

## 使用场景

当进入某页面需要改浏览器 Tab 中展示的标题时

## 核心实现

这个实现比较简单

```ts
const DEFAULT_OPTIONS: Options = {
  restoreOnUnmount: false, // 组件卸载时，是否恢复上一个页面标题
};

function useTitle(title: string, options: Options = DEFAULT_OPTIONS) {
  const titleRef = useRef(isBrowser ? document.title : '');
  useEffect(() => {
    document.title = title;
  }, [title]);

  useUnmount(() => {
    if (options.restoreOnUnmount) {
      // 组件卸载时，恢复上一个页面标题
      document.title = titleRef.current;
    }
  });
}
```

如果项目中我们自己实现的话，有个需要注意的地方，不要把`document.title = title;`写在外层，要写在 useEffect 里面，具体见该文：[检测意外的副作用](https://juejin.cn/post/6854573210663387149)
