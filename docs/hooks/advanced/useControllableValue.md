# useControllableValue

在某些组件开发时，我们需要组件的状态既可以自己管理，也可以被外部控制，useControllableValue 就是帮你管理这种状态的 Hook。

[官方文档](https://ahooks.js.org/zh-CN/hooks/https://ahooks.js.org/zh-CN/hooks/use-controllable-value)

## 基本用法

- 非受控组件：如果 props 中没有 value，则组件内部自己管理 state
- 受控组件：如果 props 有 value 字段，则由父级接管控制 state
- 无 value，有 onChange 的组件：只要 props 中有 onChange 字段，则在 state 变化时，就会触发 onChange 函数

[官方在线 Demo](https://ahooks.js.org/~demos/usecontrollablevalue-demo2/)

如果 props 有 value 字段，则由父级接管控制 state：

```ts
import React, { useState } from 'react';
import { useControllableValue } from 'ahooks';

const ControllableComponent = (props: any) => {
  const [state, setState] = useControllableValue<string>(props);

  return (
    <input
      value={state}
      onChange={e => setState(e.target.value)}
      style={{ width: 300 }}
    />
  );
};

const Parent = () => {
  const [state, setState] = useState<string>('');
  const clear = () => {
    setState('');
  };

  return (
    <>
      <ControllableComponent value={state} onChange={setState} />
      <button type="button" onClick={clear} style={{ marginLeft: 8 }}>
        Clear
      </button>
    </>
  );
};
```

## 受控组件与非受控组件

React 官方对[受控组件](https://zh-hans.reactjs.org/docs/forms.html#controlled-components)和[非受控组件](https://zh-hans.reactjs.org/docs/uncontrolled-components.html)的解释：

> - 受控组件：在 HTML 中，表单元素（如`<input>`、 `<textarea>` 和 `<select>`）通常自己维护 state，并根据用户输入进行更新。而在 React 中，可变状态（mutable state）通常保存在组件的 state 属性中，并且只能通过使用 setState()来更新。
> - 非受控组件：表单数据将交由 DOM 节点来处理。

但是受控组件/非受控组件又是一个相对的概念，子组件相对父组件来说才有受控/非受控的说法。当组件中有数据受父级组件的控制（比如数据的来源和修改的方式由父级组件的 props 提供），就是受控组件；而当组件的数据完全由组件自身维护，这样的组件是非受控组件。从这个角度看，antd 的 Input 组件既可以是受控也可以非受控，这取决于我们如何使用。

## 使用场景

- 表单组件既支持受控又要支持非受控的场景，目前很多 UI 库目前都基本支持这两种场景

## 实现思路

根据 props 是否有`[valuePropName]`属性来判断是否受控。如受控：则值由父级接管；否则组件内部状态维护；初始值的设置也遵循该逻辑。

## 核心实现

```ts
function useControllableValue<T = any>(
  props: StandardProps<T>,
): [T, (v: SetStateAction<T>) => void];
function useControllableValue<T = any>(
  props?: Props,
  options?: Options<T>,
): [T, (v: SetStateAction<T>, ...args: any[]) => void];
function useControllableValue<T = any>(
  props: Props = {},
  options: Options<T> = {},
) {
  const {
    defaultValue, // 默认值，会被 props.defaultValue 和 props.value 覆盖
    defaultValuePropName = 'defaultValue', // 默认值的属性名
    valuePropName = 'value', // 值的属性名
    trigger = 'onChange', // 修改值时，触发的函数
  } = options;
  // 外部（父级）传递进来的 props 值
  const value = props[valuePropName] as T;
  // 是否受控：判断 valuePropName（默认即表示value属性），有该属性代表受控
  const isControlled = props.hasOwnProperty(valuePropName);

  // 首次默认值
  const initialValue = useMemo(() => {
    // 受控：则由外部的props接管控制 state
    if (isControlled) {
      return value;
    }
    // 外部有传递 defaultValue，则优先取外部的默认值
    if (props.hasOwnProperty(defaultValuePropName)) {
      return props[defaultValuePropName];
    }
    // 优先级最低，组件内部的默认值
    return defaultValue;
  }, []);

  const stateRef = useRef(initialValue);
  // 受控组件：如果 props 有 value 字段，则由父级接管控制 state
  if (isControlled) {
    stateRef.current = value;
  }

  // update：调用该函数会强制组件重新渲染
  const update = useUpdate();

  function setState(v: SetStateAction<T>, ...args: any[]) {
    const r = isFunction(v) ? v(stateRef.current) : v;

    // 非受控
    if (!isControlled) {
      stateRef.current = r;
      update(); // 更新状态s
    }
    // 只要 props 中有 onChange（trigger 默认值未 onChange）字段，则在 state 变化时，就会触发 onChange 函数
    if (props[trigger]) {
      props[trigger](r, ...args);
    }
  }

  // 返回 [状态值, 修改 state 的函数]
  return [stateRef.current, useMemoizedFn(setState)] as const;
}
```

[完整源码](https://github.com/alibaba/hooks/blob/v3.7.4/packages/hooks/src/useControllableValue/index.ts)
