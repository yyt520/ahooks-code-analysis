# useScroll

监听元素的滚动位置。

[官方文档](https://ahooks.js.org/zh-CN/hooks/use-scroll)

## 基本用法

[官方在线 Demo，下方代码的执行结果](https://ahooks.js.org/~demos/usescroll-demo1/)

```ts
import React, { useRef } from 'react';
import { useScroll } from 'ahooks';

export default () => {
  const ref = useRef(null);
  const scroll = useScroll(ref);
  return (
    <>
      <p>{JSON.stringify(scroll)}</p>
      <div
        style={{
          height: '160px',
          width: '160px',
          border: 'solid 1px #000',
          overflow: 'scroll',
          whiteSpace: 'nowrap',
          fontSize: '32px',
        }}
        ref={ref}
      >
        <div>
          Lorem ipsum dolor sit amet, consectetur adipisicing elit. A aspernatur
          atque, debitis ex excepturi explicabo iste iure labore molestiae neque
          optio perspiciatis
        </div>
        <div>
          Aspernatur cupiditate, deleniti id incidunt mollitia omnis! A
          aspernatur assumenda consequuntur culpa cumque dignissimos enim eos,
          et fugit natus nemo nesciunt
        </div>
        <div>
          Alias aut deserunt expedita, inventore maiores minima officia porro
          rem. Accusamus ducimus magni modi mollitia nihil nisi provident
        </div>
        <div>
          Alias aut autem consequuntur doloremque esse facilis id molestiae
          neque officia placeat, quia quisquam repellendus reprehenderit.
        </div>
        <div>
          Adipisci blanditiis facere nam perspiciatis sit soluta ullam!
          Architecto aut blanditiis, consectetur corporis cum deserunt
          distinctio dolore eius est exercitationem
        </div>
        <div>
          Ab aliquid asperiores assumenda corporis cumque dolorum expedita
        </div>
        <div>
          Culpa cumque eveniet natus totam! Adipisci, animi at commodi delectus
          distinctio dolore earum, eum expedita facilis
        </div>
        <div>
          Quod sit, temporibus! Amet animi fugit officiis perspiciatis, quis
          unde. Cumque dignissimos distinctio, dolor eaque est fugit nisi non
          pariatur porro possimus, quas quasi
        </div>
      </div>
    </>
  );
};
```

## 核心实现

```ts
function useScroll(
  target?: Target, // DOM 节点或者 ref
  shouldUpdate: ScrollListenController = () => true, // 控制是否更新滚动信息
): Position | undefined {
  const [position, setPosition] = useRafState<Position>();

  const shouldUpdateRef = useLatest(shouldUpdate); // 控制是否更新滚动信息，默认值： () => true

  useEffectWithTarget(
    () => {
      const el = getTargetElement(target, document);
      if (!el) {
        return;
      }
      // 核心处理
      const updatePosition = () => {};

      updatePosition();

      // 监听 scroll 事件
      el.addEventListener('scroll', updatePosition);
      return () => {
        el.removeEventListener('scroll', updatePosition);
      };
    },
    [],
    target,
  );

  return position; // 滚动容器当前的滚动位置
}
```

接下来看看`updatePosition`方法的实现：

```ts
const updatePosition = () => {
  let newPosition: Position;
  // target属性传 document
  if (el === document) {
    // scrollingElement 返回滚动文档的 Element 对象的引用。
    // 在标准模式下，这是文档的根元素， document.documentElement。
    // 当在怪异模式下，scrollingElement 属性返回 HTML body 元素（若不存在返回 null）
    if (document.scrollingElement) {
      newPosition = {
        left: document.scrollingElement.scrollLeft,
        top: document.scrollingElement.scrollTop,
      };
    } else {
      // 怪异模式的处理：取 window.pageYOffset, document.documentElement.scrollTop, document.body.scrollTop 三者中最大值
      // https://developer.mozilla.org/zh-CN/docs/Web/API/Document/scrollingElement
      // https://stackoverflow.com/questions/28633221/document-body-scrolltop-firefox-returns-0-only-js
      newPosition = {
        left: Math.max(
          window.pageXOffset,
          document.documentElement.scrollLeft,
          document.body.scrollLeft,
        ),
        top: Math.max(
          window.pageYOffset,
          document.documentElement.scrollTop,
          document.body.scrollTop,
        ),
      };
    }
  } else {
    newPosition = {
      left: (el as Element).scrollLeft, // 获取滚动条到元素左边的距离（滚动条滚动了多少像素）
      top: (el as Element).scrollTop,
    };
  }
  // 	判断是否更新滚动信息
  if (shouldUpdateRef.current(newPosition)) {
    setPosition(newPosition);
  }
};
```

- [Element.scrollLeft](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollLeft) 获取滚动条到元素左边的距离
- [Element.scrollTop](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollTop) 获取滚动条到元素顶部的距离
