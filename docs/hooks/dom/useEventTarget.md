# useEventTarget

常见表单控件(通过 e.target.value 获取表单值) 的 onChange 跟 value 逻辑封装，支持自定义值转换和重置功能。

```ts
export interface Options<T, U> {
  initialValue?: T; // 初始值
  transformer?: (value: U) => T; // 自定义回调值的转化
}
```

## 使用场景

适用于较为简单的表单受控控件（如 input 输入框）管理

## 实现思路

1. 监听表单的 onChange 事件，拿到值后更新 value 值
2. 支持自定义回调值的转化，对外暴露 value 值、onChange 和 reset 方法

## 核心实现

这个实现比较简单，这里结尾代码有个`as const`，它表示强制 TypeScript 将变量或表达式的类型视为不可变的

具体可以看下这篇文章： [杀手级的 TypeScript 功能：const 断言](https://juejin.cn/post/6844903848939634696)

```ts
function useEventTarget<T, U = T>(options?: Options<T, U>) {
  const { initialValue, transformer } = options || {};
  const [value, setValue] = useState(initialValue);

  const transformerRef = useLatest(transformer);

  const reset = useCallback(() => setValue(initialValue), []);

  const onChange = useCallback((e: EventTarget<U>) => {
    const _value = e.target.value;
    if (isFunction(transformerRef.current)) {
      return setValue(transformerRef.current(_value));
    }
    // no transformer => U and T should be the same
    return setValue((_value as unknown) as T);
  }, []);

  return [
    value,
    {
      onChange,
      reset,
    },
  ] as const; // 将数组变为只读元组，可以确保其内容不会在其声明和函数调用之间发生变化
}
```
