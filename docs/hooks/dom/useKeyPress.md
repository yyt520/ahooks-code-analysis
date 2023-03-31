# useKeyPress

监听键盘按键，支持组合键，支持按键别名。

## KeyEvent 基础

**JS 的键盘事件**

- [keydown](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keydown_event)：触发于键盘按键按下的时候。
- [keyup](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keyup_event)：在按键被松开时触发。
- ~~（已过时）[keypress](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/keypress_event)：按下有值的键时触发，即按下 Ctrl、Alt、Shift、Meta 这样无值的键，这个事件不会触发。对于有值的键，按下时先触发 keydown 事件，再触发 keypress 事件~~

**关于 keyCode**
`（已过时）event.keyCode（返回按下键的数字代码）`，虽然目前大部分代码依然使用并保持兼容。但如果我们自己实现的话应该尽可能使用 `event.key（按下的键的实际值）`属性。具体可见[KeyboardEvent](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent)

**如何监听按键组合**

修饰键有四个

```ts
const modifierKey = {
  ctrl: (event: KeyboardEvent) => event.ctrlKey,
  shift: (event: KeyboardEvent) => event.shiftKey,
  alt: (event: KeyboardEvent) => event.altKey,
  meta: (event: KeyboardEvent) => {
    if (event.type === 'keyup') {
      // 这里使用数组判断是因为 meta 键分左边和右边的键(MetaLeft 91、MetaRight 93)
      return aliasKeyCodeMap['meta'].includes(event.keyCode);
    }
    return event.metaKey;
  },
};
```

- 当按下的组合键包含 `Ctrl` 键时，`event.ctrlKey` 属性为 true
- 当按下的组合键包含 `Shift` 键时，`event.shiftKey` 属性为 true
- 当按下的组合键包含 `Alt` 键时，`event.altKey` 属性为 true
- 当按下的组合键包含 `meta` 键时，`event.meta` 属性为 true（Mac 是 command 键，Windows 电脑是 win 键）

如按下 `Alt+K` 组合键，会触发两次 `keydown`事件，其中 `Alt` 键和 `K` 键打印的 `altKey` 都为 true，可以这么判断：

```js
if (event.altKey && keyCode === 75) {
  console.log('按下了 Alt + K 键');
}
```

## 在线测试

这里推荐个在线网站 [Keyboard Events Playground](https://keyevents.netlify.app/)测试键盘事件，只需要输入任意键即可查看有关它打印的信息，还可以通过复选框来过滤事件，辅助我们开发验证。

## 使用

在看源码之前，需要了解下该 Hook 支持的用法：

```ts
// 支持键盘事件中的 keyCode 和别名
useKeyPress('uparrow', () => {
  // TODO
});

// keyCode value for ArrowDown
useKeyPress(40, () => {
  // TODO
});

// 监听组合按键
useKeyPress('ctrl.alt.c', () => {
  // TODO
});

// 开启精确匹配。比如按 [shift + c] ，不会触发 [c]
useKeyPress(
  ['c'],
  () => {
    // TODO
  },
  {
    exactMatch: true,
  },
);

// 监听多个按键。如下 a s d f, Backspace, 8
useKeyPress([65, 83, 68, 70, 8, '8'], event => {
  setKey(event.key);
});

// 自定义监听方式。支持接收一个返回 boolean 的回调函数，自己处理逻辑。
const filterKey = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9'];
useKeyPress(
  event => !filterKey.includes(event.key),
  event => {
    // TODO
  },
  {
    events: ['keydown', 'keyup'],
  },
);

// 自定义 DOM。默认监听挂载在 window 上的事件，也可以传入 DOM 指定监听区域，如常见的监听输入框事件
useKeyPress(
  'enter',
  (event: any) => {
    // TODO
  },
  {
    target: inputRef,
  },
);
```

useKeyPress 的参数：

```ts
type keyType = number | string;
// 支持 keyCode、别名、组合键、数组，自定义函数
type KeyFilter = keyType | keyType[] | ((event: KeyboardEvent) => boolean);
// 回调函数
type EventHandler = (event: KeyboardEvent) => void;

type KeyEvent = 'keydown' | 'keyup';
type Options = {
  events?: KeyEvent[]; // 触发事件
  target?: Target; // DOM 节点或者 ref
  exactMatch?: boolean; // 精确匹配。如果开启，则只有在按键完全匹配的情况下触发事件。比如按键 [shif + c] 不会触发 [c]
  useCapture?: boolean; // 是否阻止事件冒泡
};

// useKeyPress 参数
useKeyPress(
  keyFilter: KeyFilter,
  eventHandler: EventHandler,
  options?: Options
);
```

## 实现思路

1. 监听 `keydown` 或 `keyup` 事件，处理事件回调函数。
2. 在事件回调函数中传入 keyFilter 配置进行判断，兼容自定义函数、keyCode、别名、组合键、数组，支持精确匹配
3. 如果满足该回调最终判断结果，则触发 eventHandler 回调

## 核心实现

> - genKeyFormatter：键盘输入预处理方法
> - genFilterKey：判断按键是否激活

沿着上述三点，我们来看这部分精简代码：

```ts
function useKeyPress(
  keyFilter: KeyFilter,
  eventHandler: EventHandler,
  option?: Options,
) {
  const {
    events = defaultEvents,
    target,
    exactMatch = false,
    useCapture = false,
  } = option || {};
  const eventHandlerRef = useLatest(eventHandler);
  const keyFilterRef = useLatest(keyFilter);

  // 监听元素（深比较）
  useDeepCompareEffectWithTarget(
    () => {
      const el = getTargetElement(target, window);
      if (!el) {
        return;
      }

      // 事件回调函数
      const callbackHandler = (event: KeyboardEvent) => {
        // 键盘输入预处理方法
        const genGuard: KeyPredicate = genKeyFormatter(
          keyFilterRef.current,
          exactMatch,
        );
        // 判断是否匹配 keyFilter 配置结果，返回 true 则触发传入的回调函数
        if (genGuard(event)) {
          return eventHandlerRef.current?.(event);
        }
      };

      // 监听事件（默认事件：keydown）
      for (const eventName of events) {
        el?.addEventListener?.(eventName, callbackHandler, useCapture);
      }
      return () => {
        // 取消监听
        for (const eventName of events) {
          el?.removeEventListener?.(eventName, callbackHandler, useCapture);
        }
      };
    },
    [events],
    target,
  );
}
```

上面的代码看起来比较好理解，需要推敲的就是 `genKeyFormatter` 函数。

```ts
/**
 * 键盘输入预处理方法
 * @param [keyFilter: any] 当前键
 * @returns () => Boolean
 */
function genKeyFormatter(
  keyFilter: KeyFilter,
  exactMatch: boolean,
): KeyPredicate {
  // 支持自定义函数
  if (isFunction(keyFilter)) {
    return keyFilter;
  }
  // 支持 keyCode、别名、组合键
  if (isString(keyFilter) || isNumber(keyFilter)) {
    return (event: KeyboardEvent) => genFilterKey(event, keyFilter, exactMatch);
  }
  // 支持数组
  if (Array.isArray(keyFilter)) {
    return (event: KeyboardEvent) =>
      keyFilter.some(item => genFilterKey(event, item, exactMatch));
  }
  // 等同 return keyFilter ? () => true : () => false;
  return () => Boolean(keyFilter);
}
```

看完发现上面的重点实现还是在 `genFilterKey` 函数：

- [aliasKeyCodeMap](https://github.com/alibaba/hooks/blob/master/packages/hooks/src/useKeyPress/index.ts#L24)

这段逻辑需要各位代入实际数值帮助理解，如输入组合键 `shift.c`

```ts
/**
 * 判断按键是否激活
 * @param [event: KeyboardEvent]键盘事件
 * @param [keyFilter: any] 当前键
 * @returns Boolean
 */
function genFilterKey(
  event: KeyboardEvent,
  keyFilter: keyType,
  exactMatch: boolean,
) {
  // 浏览器自动补全 input 的时候，会触发 keyDown、keyUp 事件，但此时 event.key 等为空
  if (!event.key) {
    return false;
  }

  // 数字类型直接匹配事件的 keyCode
  if (isNumber(keyFilter)) {
    return event.keyCode === keyFilter;
  }

  // 字符串依次判断是否有组合键
  const genArr = keyFilter.split('.'); // 如 keyFilter 可以传 ctrl.alt.c，['shift.c']
  let genLen = 0;

  for (const key of genArr) {
    // 组合键
    const genModifier = modifierKey[key]; // ctrl/shift/alt/meta
    // keyCode 别名
    const aliasKeyCode: number | number[] = aliasKeyCodeMap[key.toLowerCase()];

    if (
      (genModifier && genModifier(event)) ||
      (aliasKeyCode && aliasKeyCode === event.keyCode)
    ) {
      genLen++;
    }
  }

  /**
   * 需要判断触发的键位和监听的键位完全一致，判断方法就是触发的键位里有且等于监听的键位
   * genLen === genArr.length 能判断出来触发的键位里有监听的键位
   * countKeyByEvent(event) === genArr.length 判断出来触发的键位数量里有且等于监听的键位数量
   * 主要用来防止按组合键其子集也会触发的情况，例如监听 ctrl+a 会触发监听 ctrl 和 a 两个键的事件。
   */
  if (exactMatch) {
    return genLen === genArr.length && countKeyByEvent(event) === genArr.length;
  }
  return genLen === genArr.length;
}

// 根据 event 计算激活键数量
function countKeyByEvent(event: KeyboardEvent) {
  // 计算激活的修饰键数量
  const countOfModifier = Object.keys(modifierKey).reduce((total, key) => {
    // (event: KeyboardEvent) => Boolean
    if (modifierKey[key](event)) {
      return total + 1;
    }

    return total;
  }, 0);

  // 16 17 18 91 92 是修饰键的 keyCode，如果 keyCode 是修饰键，那么激活数量就是修饰键的数量，如果不是，那么就需要 +1
  return [16, 17, 18, 91, 92].includes(event.keyCode)
    ? countOfModifier
    : countOfModifier + 1;
}
```
