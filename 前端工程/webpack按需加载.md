# webpack按需加载

## 为什么要按需加载
对应用分包，提高首屏的加载速度
## 如何实现按需加载
在项目中，实现按需加载的方式有

- react-lodable

```javascript
import React, { useState } from "react";
import ReactDom from "react-dom";
import Loadable from 'react-loadable';

const LoadableComponent = (loader) => Loadable({
  loader,
  loading: ({error}) => <>{error ? "Error":"Loading..."}</>
})

const AnotherCom = LoadablePage(() => import('./AnotherCom'));

const MainPage = () => {
  const [show, setShow] = useState(false);

  const loadComponent = () => {
    setShow(true)
  };

  return (
    <>
      <button onClick={loadComponent}>动态加载</button>

      {show && <AnotherCom />}

    </>
  );
}

ReactDom.render(<MainPage />, document.getElementById("root"));
```

- Suspense组件 + React.lazy

```javascript
import React, { useState, Suspense } from "react";
import ReactDom from "react-dom";

const MainPage = () => {
  const [Show, setShow] = useState(undefined);

  const loadComponent = () => {
    const AnotherCom = React.lazy(() => import('./AnotherCom'));
    setShow(AnotherCom)
  };

  return (
    <>
      <button onClick={loadComponent}>动态加载</button>

      <Suspense fallback={<div>Loading...</div>}>
        {Show && <Show />}
      </Suspense>

    </>
  );
}


ReactDom.render(<MainPage />, document.getElementById("root"));
```

## 利用import实现按需加载
react-lodable和React.lazy的实现，都是基于ECMA Script 标准中的一个提案，语法是import(ModuleName)，该提案为我们提供了动态加载模块的能力。
目前原生并不支持动态import，我们需要借助webpack等打包工具(webpack内部实现了该语法)，或者babel转换，插件是@babel/plugin-syntax-dynamic-import
```javascript
// .babelrc
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-react"
  ],
  "plugins": [
    "@babel/plugin-syntax-dynamic-import"
  ]
}
```
### 实现方式
```javascript
import React, { useState } from "react";
import ReactDom from "react-dom";

const MainPage = () => {
  const [LoadableCom, setLoadableCom] = useState();

  const loadComponent = () => {
    import('./AnotherCom').then((result) => {
      setLoadableCom(result.default)
    })
  };
  return (
    <>
      <button onClick={loadComponent}>动态加载</button>
      {
        LoadableCom
      }
    </>
  );
}


ReactDom.render(<MainPage />, document.getElementById("root"));
```

