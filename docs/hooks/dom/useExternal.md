# useExternal

动态注入 JS 或 CSS 资源，useExternal 可以保证资源全局唯一。

## 实现思路

原理：通过 script 标签加载 JS 资源 / 创建 link 标签加载 CSS 资源，再通过创建标签返回的 Element 元素监听 load 和 error 事件 获取加载状态

1. 正则判断传入的路径 path 是 JS 还是 CSS
2. 加载 CSS/JS：创建 link/script 标签传入 path，支持传入 link/script 标签支持的属性，添加到 head/body 中，并返回 Element 元素与加载状态；这里需判断标签路径匹配是否存在，存在则返回上一次结果，以保证资源全局唯一
3. 利用创建标签返回的 Element 元素监听 load 和 error 事件，并在回调中改变加载状态

## 核心实现

主体实现结构：

```ts
export interface Options {
  type?: 'js' | 'css';
  js?: Partial<HTMLScriptElement>;
  css?: Partial<HTMLStyleElement>;
}

const useExternal = (path?: string, options?: Options) => {
  const [status, setStatus] = useState<Status>(path ? 'loading' : 'unset');

  const ref = useRef<Element>();

  useEffect(() => {
    if (!path) {
      setStatus('unset');
      return;
    }
    const pathname = path.replace(/[|#].*$/, '');
    if (
      options?.type === 'css' ||
      (!options?.type && /(^css!|\.css$)/.test(pathname))
    ) {
      const result = loadCss(path, options?.css);
    } else if (
      options?.type === 'js' ||
      (!options?.type && /(^js!|\.js$)/.test(pathname))
    ) {
      const result = loadScript(path, options?.js);
    } else {
    }

    if (!ref.current) {
      return;
    }

    const handler = (event: Event) => {};

    ref.current.addEventListener('load', handler);
    ref.current.addEventListener('error', handler);
    return () => {
      // 移除监听 & 清除操作
    };
  }, [path]);

  return status;
};
```

主函数中判断加载 CSS 还是 JS 资源：

```ts
const pathname = path.replace(/[|#].*$/, '');
if (
  options?.type === 'css' ||
  (!options?.type && /(^css!|\.css$)/.test(pathname))
) {
  const result = loadCss(path, options?.css); // 加载 css 资源并返回结果
  ref.current = result.ref; // 返回创建 link 标签返回的 Element 元素，用于后续绑定监听 load 和 error事件
  setStatus(result.status); // 设置加载状态
} else if (
  options?.type === 'js' ||
  (!options?.type && /(^js!|\.js$)/.test(pathname))
) {
  const result = loadScript(path, options?.js);
  ref.current = result.ref;
  setStatus(result.status);
} else {
  // do nothing
  console.error(
    "Cannot infer the type of external resource, and please provide a type ('js' | 'css'). " +
      'Refer to the https://ahooks.js.org/hooks/dom/use-external/#options',
  );
}
```

loadCss 方法：

> 往 HTML 标签上添加任意以 "data-" 为前缀来设置我们需要的自定义属性，可以进行一些数据的存放

```ts
const loadCss = (path: string, props = {}): loadResult => {
  const css = document.querySelector(`link[href="${path}"]`);
  // 不存在则创建
  if (!css) {
    const newCss = document.createElement('link');

    newCss.rel = 'stylesheet';
    newCss.href = path;
    // 设置 link 标签支持的属性
    Object.keys(props).forEach(key => {
      newCss[key] = props[key];
    });
    // IE9+
    const isLegacyIECss = 'hideFocus' in newCss;
    // use preload in IE Edge (to detect load errors)
    if (isLegacyIECss && newCss.relList) {
      newCss.rel = 'preload';
      newCss.as = 'style';
    }
    // 设置自定义属性[data-status]为loading状态
    newCss.setAttribute('data-status', 'loading');
    // 添加到 head 标签
    document.head.appendChild(newCss);

    // 标签路径匹配存在则直接返回现有结果，保证全局资源全局唯一
    return {
      ref: newCss,
      status: 'loading',
    };
  }
  // 如果标签存在则直接返回，并取 data-status 中的值
  return {
    ref: css,
    status: (css.getAttribute('data-status') as Status) || 'ready',
  };
};
```

loadScript 方法的实现也类似：

```ts
const loadScript = (path: string, props = {}): loadResult => {
  const script = document.querySelector(`script[src="${path}"]`);

  if (!script) {
    const newScript = document.createElement('script');
    newScript.src = path;
    // 设置 script 标签支持的属性
    Object.keys(props).forEach(key => {
      newScript[key] = props[key];
    });

    newScript.setAttribute('data-status', 'loading');
    // 添加到 body 标签
    document.body.appendChild(newScript);

    return {
      ref: newScript,
      status: 'loading',
    };
  }

  return {
    ref: script,
    status: (script.getAttribute('data-status') as Status) || 'ready',
  };
};
```

前面获取到 Element 元素后，监听 Element 的 load 和 error 事件，判断其加载状态并更新状态

```ts
const handler = (event: Event) => {
  const targetStatus = event.type === 'load' ? 'ready' : 'error';
  ref.current?.setAttribute('data-status', targetStatus);
  setStatus(targetStatus);
};

ref.current.addEventListener('load', handler);
ref.current.addEventListener('error', handler);
```
