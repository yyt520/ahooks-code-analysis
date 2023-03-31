# useInViewport

观察元素是否在可见区域，以及元素可见比例。

## 使用场景

- 图片懒加载：当图片滚动到可见位置的时候才加载
- 无限滚动加载：滑动到底部时开始加载新的内容
- 检测广告的曝光率：广告是否被用户看到
- 用户看到某个区域时执行任务或播放动画

## IntersectionObserver API

- [Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API)
- 可参考学习：[IntersectionObserver API 使用教程](https://www.ruanyifeng.com/blog/2016/11/intersectionobserver_api.html)

## 实现思路

1. 监听目标元素，支持传入原生 `IntersectionObserver` API 选项
2. 对 `IntersectionObserver` 构造函数的回调函数设置可见状态与可见比例值
3. 借助 [intersection-observer](https://www.npmjs.com/package/intersection-observer) 库实现 polyfill

## 核心实现

```ts
export interface Options {
  // 根(root)元素的外边距
  rootMargin?: string;
  // 可以控制在可见区域达到该比例时触发 ratio 更新。默认值是 0 (意味着只要有一个 target 像素出现在 root 元素中，回调函数将会被执行)。该值为 1.0 含义是当 target 完全出现在 root 元素中时候 回调才会被执行。
  threshold?: number | number[];
  // 指定根(root)元素，用于检查目标的可见性
  root?: BasicTarget<Element>;
}
```

```ts
function useInViewport(target: BasicTarget, options?: Options) {
  const [state, setState] = useState<boolean>(); // 是否可见
  const [ratio, setRatio] = useState<number>(); // 当前可见比例

  useEffectWithTarget(
    () => {
      const el = getTargetElement(target);
      if (!el) {
        return;
      }
      // 可以自动观察元素是否可见，返回一个观察器实例
      const observer = new IntersectionObserver(
        entries => {
          // callback函数的参数（entries）是一个数组，每个成员都是一个IntersectionObserverEntry对象。如果同时有两个被观察的对象的可见性发生变化，entries数组就会有两个成员。
          for (const entry of entries) {
            setRatio(entry.intersectionRatio); // 设置当前目标元素的可见比例
            setState(entry.isIntersecting); // isIntersecting：如果目标元素与交集观察者的根相交，则该值为true
          }
        },
        {
          ...options,
          root: getTargetElement(options?.root),
        },
      );

      observer.observe(el); // 开始监听一个目标元素

      return () => {
        observer.disconnect(); // 停止监听目标
      };
    },
    [options?.rootMargin, options?.threshold],
    target,
  );

  return [state, ratio] as const;
}
```
