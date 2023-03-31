# useEventEmitter

在多个组件之间进行事件通知有时会让人非常头疼，借助 EventEmitter ，可以让这一过程变得更加
简单。

```ts
const event$ = useEventEmitter();
```

通过 props 或者 Context ，可以将 `event$` 共享给其他组件。然后在其他组件中，可以调用 EventEmitter 的方法

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-event-emitter)

## 基本用法

[官方在线 Demo](https://ahooks.js.org/~demos/useeventemitter-demo1/)

父组件向子组件共享事件

父组件创建了一个 `focus$` 事件，并且将它传递给了两个子组件。在 MessageBox 中调用 `focus$.emit` ，InputBox 组件就可以收到通知。

```ts
import React, { useRef, FC } from 'react';
import { useEventEmitter } from 'ahooks';
import { EventEmitter } from 'ahooks/lib/useEventEmitter';

const MessageBox: FC<{
  focus$: EventEmitter<void>;
}> = function(props) {
  return (
    <div style={{ paddingBottom: 24 }}>
      <p>You received a message</p>
      <button
        type="button"
        onClick={() => {
          props.focus$.emit();
        }}
      >
        Reply
      </button>
    </div>
  );
};

const InputBox: FC<{
  focus$: EventEmitter<void>;
}> = function(props) {
  const inputRef = useRef<any>();
  props.focus$.useSubscription(() => {
    inputRef.current.focus();
  });
  return (
    <input
      ref={inputRef}
      placeholder="Enter reply"
      style={{ width: '100%', padding: '4px' }}
    />
  );
};

export default function() {
  const focus$ = useEventEmitter();
  return (
    <>
      <MessageBox focus$={focus$} />
      <InputBox focus$={focus$} />
    </>
  );
}
```

## 使用场景

- 同级跨组件（距离较远的组件）通信

> 对于子组件通知父组件的情况，我们仍然推荐直接使用 props 传递一个 onEvent 函数。而对于父组件通知子组件的情况，可以使用 forwardRef 获取子组件的 ref ，再进行子组件的方法调用。 useEventEmitter 适合的是在距离较远的组件之间进行事件通知，或是在多个组件之间共享事件通知。

## 实现思路

通过发布订阅模式实现。主要实现两个方法`useSubscription`和`emit`

1. 首先要保证每次渲染调用 `useEventEmitter` 得到的返回值会保持不变： `useRef`来判断
2. `useSubscription` 会在组件创建时自动注册订阅，并在组件销毁时自动取消订阅：Set 数据结构记录订阅的事件列表，在`useEffect`里面实现监听和自动取消订阅操作
3. `emit` 方法，推送一个事件：循环 Set 事件列表取出时间执行

## 核心实现

主函数

```ts
function useEventEmitter<T = void>() {
  const ref = useRef<EventEmitter<T>>();
  if (!ref.current) {
    // 在组件多次渲染时，每次渲染调用 useEventEmitter 得到的返回值会保持不变，不会重复创建 EventEmitter 的实例。
    ref.current = new EventEmitter();
  }
  return ref.current;
}
```

核心其实就是实现 `EventEmitter` 类

```ts
class EventEmitter<T> {
  // Set 结构存放订阅的事件列表
  private subscriptions = new Set<Subscription<T>>();

  // 发送一个事件通知来触发事件
  emit = (val: T) => {
    // 触发订阅列表的所有事件
    for (const subscription of this.subscriptions) {
      subscription(val);
    }
  };

  // 订阅事件
  useSubscription = (callback: Subscription<T>) => {
    // eslint-disable-next-line react-hooks/rules-of-hooks
    const callbackRef = useRef<Subscription<T>>();
    callbackRef.current = callback; // 保证拿到最新引用
    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      function subscription(val: T) {
        if (callbackRef.current) {
          callbackRef.current(val);
        }
      }
      // 添加到订阅事件列表
      this.subscriptions.add(subscription);
      return () => {
        // 卸载的时候自动删除（取消订阅）
        this.subscriptions.delete(subscription);
      };
    }, []);
  };
}
```

个人建议如果是单独父子通信的话，没必要使用这个 Hook，直接传递参数就行了；而大量的事件管理建议使用全局状态库管理

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useEventEmitter/index.ts)
