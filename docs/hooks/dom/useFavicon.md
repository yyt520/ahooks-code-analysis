# useFavicon

设置页面的 favicon。

> favicon 指显示在浏览器收藏夹、地址栏和标签标题前面的个性化图标

## 使用场景

当需要改浏览器 Tab 中展示的图标 icon 时

## 核心实现

原理：通过 link 标签设置 favicon

更多 favicon 知识可见： [详细介绍 HTML favicon 尺寸 格式 制作等相关知识](https://www.zhangxinxu.com/wordpress/2019/06/html-favicon-size-ico-generator/)

源代码仅支持图标四种类型：

```ts
const ImgTypeMap = {
  SVG: 'image/svg+xml',
  ICO: 'image/x-icon',
  GIF: 'image/gif',
  PNG: 'image/png',
};

type ImgTypes = keyof typeof ImgTypeMap;
```

```ts
const useFavicon = (href: string) => {
  useEffect(() => {
    if (!href) return;

    const cutUrl = href.split('.');
    // 取出文件后缀
    const imgSuffix = cutUrl[cutUrl.length - 1].toLocaleUpperCase() as ImgTypes;

    const link: HTMLLinkElement =
      document.querySelector("link[rel*='icon']") ||
      document.createElement('link');

    link.type = ImgTypeMap[imgSuffix];
    // 指定被链接资源的地址
    link.href = href;
    // rel 属性用于指定当前文档与被链接文档的关系，直接使用 rel=icon 就可以，源码下方的 `shortcut icon` 是一种过时的用法
    link.rel = 'shortcut icon';

    document.getElementsByTagName('head')[0].appendChild(link);
  }, [href]);
};
```
