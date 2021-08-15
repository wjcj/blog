React SSR 的诸多细节
====

本文通过实现一个 React 服务端渲染应用，探究服务端渲染应用在同构过程中需要解决的诸多问题，包括项目构建、路由同构、数据同构、接入 Redux、SEO 优化、404和重定向等问题。
[这里](https://github.com/wjcj/react-ssr)查看源码。

## 项目构建

项目主要结构：

```
├── dist
│   ├── ...
│   ├── client.js
│   ├── server.js
│   └── index.html
├── node_modules
├── src
│   ├── client
│   │    ├── componments
│   │    ├── pages
│   │    ├── store
│   │    ├── App.js             
│   │    └── index.js
│   └── server
│        └── index.js
├── package.json
├── .babelrc
├── index.html
├── webpack.base.js
├── webpack.client.js
└── webpack.server.js
```

### 客户端

下面是客户端示例代码：

``` jsx
// src/client/index.js
import React from 'react';
import ReactDom from 'react-dom';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

const Index = () => {
  return (
    <BrowserRouter>
      <App/>
    </BrowserRouter>
  );
}
ReactDom.hydrate(<Index/>, document.getElementById('root'));
```

需要注意的是使用 `ReactDom.hydrate` 替换 `ReactDom.render`。服务端渲染使用 [ReactDOMServer](https://zh-hans.reactjs.org/docs/react-dom-server.html) 中的 `renderToString` 方法在服务端将组件渲染成初始的 html 字符串后传输到客户端，从而达到加快页面加载速度、允许搜索引擎爬取你的页面以达到 SEO 优化的目的。`hydrate` 方法可以尽可能复用这些节点，并补充原有的事件绑定。

客户端 webpack 配置：
``` js
// webpack.client.js
module.exports = {
  entry: './src/client/index.js',
  output: {
    path: path.resolve(__dirname, './dist'),
    filename: 'client.js',
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        loader: require.resolve('babel-loader')
      },
      {
        test: /\.(css|scss|sass)$/,
        use: [
          'style-loader',
          'css-loader',
          'sass-loader',
        ]
      },
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      inject: true,
      template: path.join(__dirname, './index.html'),
      filename: 'index.html',
      minify:{
        removeComments: false,
      }
    }),
  ],
}
```

此配置除了用于 `webpack-dev-server` 开启本地服务，解析 jsx 代码。最主要的目的是打包生成客户端代码 `client.js` 在服务端 HTML 模板中使用。

### 服务端

服务端的主要代码是使用 `express` 创建一个服务器，通过 `renderToString` 方法将 `<App/>` 渲染成 HTML 字符串插入到 HTML 模板中，最终将完整的 html 文件发送到客户端。html 模板中引用了客户端脚本 `client.js`，保证 HTML 字符串渲染后浏览器可以接管页面。

``` js
// src/server/index.js
import express from 'express';
import { renderToString } from 'react-dom/server';
import React from 'react';
import App from '../client/App';

const app = express();
app.use(express.static('dist'));
app.use('/*', (req, res) => {
  const content = renderToString(<App/>);
  const html = renderHtml({ content });
  res.send(html);
});

app.listen(5000, () => console.log(`http://localhost:5000/`));

function renderHtml({ content, state, css, helmet, bundles }) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width,initial-scale=1">
        <link rel="icon" href="data:;base64,=">
      </head>
      <body>
        <div id="root">${content}</div>
        <script src="/client.js"></script>
      </body>
    </html>
  `;
};
```

上面代码由于存在 jsx 不能直接在 nodeJS 环境下运行，所以需要通过 webpack 进行打包。配置 `webpack.server.js` 文件：

``` js
// webpack.server.js
const path = require('path');
const webpack = require('webpack');
const nodeExternals = require('webpack-node-externals');

module.exports = {
  target: 'node',
  entry: './src/server/index.js',
  output: {
    path: path.resolve(__dirname, './dist'),
    filename: 'server.js',
  },
  externals: [nodeExternals()],
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        loader: require.resolve('babel-loader')
      },
      {
        test: /\.(css|scss|sass)$/,
        use: [
          'isomorphic-style-loader',
          'css-loader',
          'sass-loader',
        ]
      },
    ]
  },
};
```

由于 `src/server/index.js` 在 nodeJS 环境运行，所以需要设置 `target：node`。同时服务端环境已经安装了 node 核心模块和其他三方模块，所以不在需要把这些模块代码打包到最终代码中，使用 `webpack-node-externals` 可以排除这些模块。在服务端编译 CSS 文件需要用到 `isomorphic-style-loader`，后面会专门讲到。

打包后在生成的 `dist/server.js` 即可在 nodeJS 环境运行，使用 `nodemon` 可以监听 `dist` 目录文件变更，变更时会自动重新启动 node 程序（`node ./dist/server.js`）。

``` json
"scripts": {
  "dev:ssr:start": "nodemon --watch dist --exec node \"./dist/server.js\""
}
```

## 路由同构

客户端通过浏览器地址匹配页面组件，而服务端根据请求路径（`req.baseUrl`）匹配渲染的组件，这是两种完全不同的机制。但为了保证两端匹配的组件一致则需要使用同一套路由规则（即下面示例中的 `routes` ）。

``` js
// src/client/App.js
import { renderRoutes } from 'react-router-config';
import Home from './pages/home';
import About from './pages/about';

const Root = ({ route }) => (
  <Layout>
    <Header>
      <Link to="/home">Home</Link>
      <Link to="/about">About</Link>
    </Header>
    {renderRoutes(route.routes)}
  </Layout>
);

export const routes = [
  {
    component: Root,
    routes: [
      {
        path: "/",
        exact: true,
        component: Home,
      },
      {
        path: "/about",
        exact: true,
        component: About,
      },
    ]
  }
];

const App = () => <div>{renderRoutes(routes)}</div>;
export default App;
```

`react-router-config` 中的 `renderRoutes` 方法用于根据路由规则动态生成路由组件。还有一个 `matchRoutes` 方法可用于服务端根据请求路径匹配路由组件，在数据同构中至关重要。

客户端使用 `BrowserRouter` 组件根据 `window.location` 匹配路由，使用 history API (pushState, replaceState 和 popstate 事件) 实现页面组件切换。

``` js
import React from 'react';
import ReactDom from 'react-dom';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

const Index = () => {
  return (
    <BrowserRouter>
      <App/>
    </BrowserRouter>
  );
};

ReactDom.hydrate(<Index/>, document.getElementById('root'));
```

由于服务端无需记录当前路由状态，所以服务端使用无状态的 `StaticRouter` 路由组件，服务端需要知道当前的请求匹配哪个组件，但服务端无法获取 `window.location` 对象，所以需将请求路径作为 props 传入。同时还可以传入一个上下文对象 `context`，处理 404 和重定向会用到。 

``` js
import express from 'express';
import { renderToString } from 'react-dom/server';
import React from 'react';
import { StaticRouter } from 'react-router-dom';
import App from '../client/App';

const app = express();

app.use('/*', (req, res) => {
  const context = {};
  const jsx = (
    <StaticRouter location={req.baseUrl} context={context}>
      <App/>
    </StaticRouter>
  );
  const content = renderToString(jsx);
  const html = renderHtml({ content }});
  res.send(html);
});

function renderHtml({ content }) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width,initial-scale=1">
        <link rel="icon" href="data:;base64,=">
      </head>
      <body>
        <div id="root">${content}</div>
        <script src="/client.js"></script>
      </body>
    </html>
  `;
};
```

## 数据同构

数据同构的目的在于同一套代码实现数据请求与组件渲染。数据同构的核心是脱水和注水：

1. 服务端完成数据预加载，即在服务端请求接口获取当前请求路径的组件所需的全部数据。获取到数据后通常的做法是将数据 `JSON.stringify` 字符串化后，随干瘪的组件 HTML 字符串一同发到到客户端。这个过程叫脱水（dehydrate），数据即水。

2. 浏览器接管页面后，为减少请求或避免重新请求需要使用 1 步骤的脱水数据给应用程序补水。这个过程叫注水（hydrate）。

### 脱水

这里使用 Redux 完成脱水和注水过程。`createServerStore` 保证每个用户都是独立的 store：

``` js
// src/client/store/index.js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk'; // 实现异步
import reducers from './reducers';
export const createServerStore = () => createStore(reducers, applyMiddleware(thunk));
```

`loadBranchData` 函数根据当前请求地址获取匹配的所有路由组件，通过直接组件上注册的静态方法完成数据请求并更新到 store 中。最后将 store 中的 state 随 HTML 字符串发送至客户端。

``` js
// src/server/index.js
app.use('/*', async (req, res) => {
  const store = createServerStore();

  await loadBranchData(req.baseUrl, store);

  const jsx = (
    <Provider store={store}>
      <StaticRouter context={context} location={req.baseUrl} >
        <App/>
      </StaticRouter>
    </Provider>
  );
  
  const content = renderToString(jsx);
  const html = renderHtml({ content, state: store.getState() });
  res.send(html);
});

function renderHtml({ content, state }) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width,initial-scale=1">
        <link rel="icon" href="data:;base64,=">
      </head>
      <body>
        <div id="root">${content}</div>
        <script>
          window.reduxState = ${JSON.stringify(store.getState())}
        </script>
        <script src="/client.js"></script>
      </body>
    </html>
  `;
};
```

**`loadBranchData`**

数据预加载的核心在于通过 `matchRoutes` 获取匹配的路由组件。路由组件注册的页面组件定义了数据请求的 API（下面的 `loadData`），调用 API 完成数据预先加载。

``` js
// src/server/utils.js
import { matchRoutes } from 'react-router-config';

async function loadBranchData (pathname, store) {
  const branch = matchRoutes(routes, pathname);
  const promises = branch.map(({ route }) => {
    const loadData = route.component.loadData;
    return typeof loadData === 'function' ? loadData(store) : Promise.resolve(null);
  });
  const result = await Promise.all(promises);
  return result;
}
```

类组件和函数组件中定义 `loadData` 的方式：

``` jsx
class Home extends React.Component {
  // 预加载数据，服务端调用
  static async loadData(store) {
    return store.dispatch(getTodos())
  }
  // ...
}
```

函数组件中：

``` jsx
// src/client/pages/home.js
import React, { useEffect } from 'react';
import { connect } from 'react-redux';
import TodoList from '../componments/todoList';
import { getTodos } from '../store/actions';

const Home = ({ todos, getInitialProps, ...props }) => {
  // 避免重复请求，如果数据为空则表示当前组件未被预加载数据，从其他页面中跳转至此组件所在页面时，组件挂载时将在客户端请求数据。
  useEffect(() => {
    if (todos.length) return;
    getInitialProps();
  }, []);

  return (
    <div className="home">
      <h1>Home</h1>
      <TodoList data={todos} {...props} />
    </div>
  );
}

// 获取异步数据并更新数据至 store。
// getTodos 是一个异步 actionCreator
Home.loadData = async store => store.dispatch(getTodos());

const mapStateToProps = state => ({ todos: state.todos });
const mapDispatchToProps = dispatch => {
  return {
    getInitialProps: () => dispatch(getTodos()),
  }
};
export default connect(mapStateToProps, mapDispatchToProps)(Home);
```

### 注水

如果你完成了上面的流程，会发现服务端的发送到客户的 html 字符串是已经渲染了异步数据的组件。
但在浏览器实际看到的却是空数据的页面。这是由于客户端 store 与服务端 store 的数据状态不一致（水不一样）。浏览器接管页面后使用客户端 store 的数据对组件进行补水，但此时客户端 store 是默认的空数据，导致组件 props 不一致而引发重新渲染。

注水的目的就是保证浏览器接管页面后，组件不会重新渲染。需要做的就是使用服务端的脱水数据 初始化客户端 store.

``` js
// src/client/store/index.js
export const getClientStore = () => {
  const defaultState = JSON.parse(window.reduxState) || {};
  return createStore(reducer, defaultState, applyMiddleware(thunk))
}
```

``` jsx
// src/client/index.js
import { createClientStore } from './store';
const Index = () => {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <App/>
      </BrowserRouter>
    </Provider>
  );
};
ReactDom.hydrate(<Index/>, document.getElementById('root'));
```

完成 store 注水过程后，就是在组件中使用注水数据，实现组件的补水。上面实例中的 `Home` 组件使用 `connect` 获取客户端 store 中 `state.todos` 就是完成这个过程。

为了避免浏览器接管页面后组件发起重复请求，并且保证从其他路由跳转到当前组件时能发起请求，还需要做一些额外判断：

``` jsx
const Home = ({ todos, getInitialProps, ...props }) => {
  // 如果数据为空则表示当前组件未被预加载数据。
  // 从其他路由页面跳转至当前组件时，组件挂载时能请求数据。
  useEffect(() => {
    if (todos.length) return;
    getInitialProps();
  }, []);

  return (
    <div className="home">
      <h1>Home</h1>
      <TodoList data={todos} {...props} />
    </div>
  );
}
```

**数据安全**

使用 `JSON.stringify` 脱水容易造成 XSS 攻击，可以将数据保存在一个隐藏的 `textarea` 的 `value` 中。

``` js
<textarea id="textarea-redux-state" style="display: none">${JSON.stringify(state)}</textarea>
```

``` js
export const createClientStore = () => {
  let defaultState = {};
  const textarea = document.getElementById('textarea-redux-state');
  if (textarea && textarea.value) {
    defaultState = JSON.parse(textarea.value);
  }
  return createStore(reducers, defaultState, applyMiddleware(logger, thunk));
};
```

## SEO 优化

我们希望服务端渲染的 html 中的 head 标签信息（如 title、description）能根据请求地址动态生成。单页应用在切换路由时也能动态修改 head 标签信息。

我们往往使用 `react-helmet` 这个库来解决上述问题。

``` jsx
// src/client/pages/home.js
import Helmet from 'react-helmet';

const Home = props => {
  return (
    <div className="home">
      <Helmet>
        <title>Home</title>
        <meta name="description" content="react ssr demo: todolist" />
      </Helmet>
      {/* ... */}
    </div>
  );
}
```

上面步骤实现了客户端动态修改 head 标签信息。服务端想实现 head 标签信息随请求地址动态生成只需要

``` js
// src/server/index.js
import { Helmet } from 'react-helmet';

app.use('/*', async (req, res) => {
  const store = createServerStore();
  // ...
  const content = renderToString(jsx);
  const helmet = Helmet.renderStatic();
  const html = renderHtml({
    content,
    state: store.getState(),
    helmet,
  });
  res.send(html);
});

function renderHtml({ content, state, helmet }) {
  return `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width,initial-scale=1">
        <link rel="icon" href="data:;base64,=">
        ${helmet.title.toString()}${helmet.meta.toString()}
      </head>
      <body>
        <div id="root">${content}</div>
        <textarea id="textarea-ssr-data" style="display: none">${JSON.stringify(state)}</textarea>
        <script defer="defer" src="/client.js"></script>
      </body>
    </html>
  `;
};
```

## 处理 404

404 错误处理思路是通过 `StaticRouter` 告诉服务端是否匹配到路由，如果没有匹配到则修改 `context` 对象。

``` jsx
app.use('/*', async (req, res) => {
  // ...
  const context = {};
  const jsx = (
    <Provider store={store}>
      <StaticRouter context={context} location={req.baseUrl} >
        <App/>
      </StaticRouter>
    </Provider>
  );
  // ...
  if (context.notFound) {
    res.status(404);
  }
  res.send(html);
});
```

定义一个 `NotFound` 组件，服务端被匹配到后设置 `staticContext.NotFound = true`。
``` jsx
// 模拟 componentWillMount 周期函数
const useComponentWillMount = cb => {
  const willMount = useRef(true);
  if (willMount.current) {
    cb();
  };
  willMount.current = false;
};

// src/client/page/404.js
const NotFound = ({ staticContext }) => {
  useComponentWillMount(() => {
    staticContext && (staticContext.NotFound = true);
  });
  return (
    <div className="not-found">
      <Helmet>
        <title>404</title>
        <meta name="description" content="react ssr demo: 404" />
      </Helmet>
      <h1>404</h1>
    </div>
  );
};
```

将 `NotFound` 组件注册到路由表：

``` js
import NotFound from './pages/404';
export const routes = [
  {
    component: Root,
    routes: [
      {
        path: "/",
        exact: true,
        component: Home,
      },
      // ...
      {
        path: "*",
        component: NotFound,
      },
    ]
  }
];
```

## 重定向页面

重定向的思路与 400 的处理类似，区别在于无需我们手动修改 `context` 对象，当我们在某路由页面组件（比如下面的 `Protected` 组件）使用 `<Redirect/>` 进行重定向后，服务端如果匹配到此路由页面，`react-router-dom` 将在 `context` 里会添加 `action`、`location` 和 `url` 属性。通过 `context.action === 'REPLACE'` 即可在服务端定义重定向逻辑。

``` jsx
import React from 'react';
import { Redirect } from 'react-router-dom';
import { fakeAuth } from '../utils';

const Protected = ({ location, history }) => {
  const toPath = { pathname: '/login', state: { from: location }};
  const signout = () => {
    fakeAuth.signout(() => history.push(toPath));
  };
  return fakeAuth.isAuthenticated ?
    <div>已登录。<button onClick={signout}>Sign out</button></div>
    : <Redirect to={toPath} />;
};
export default Protected;
```

通过 `context.action === 'REPLACE'` 判断是否重定向：

``` js
const context = {};
const jsx = (
  <Provider store={store}>
    <StaticRouter context={context} location={req.baseUrl} >
      <App/>
    </StaticRouter>
  </Provider>
);
if (context.action === 'REPLACE') {
  res.redirect(301, context.url);
  return;
}
```

## 代码拆分（Code-Splitting）

代码拆分是基于路由拆分和组件拆分来完成，这里使用 `react-loadable` 实现组件异步加载。

首先确保服务端预加载所有组件。`Loadable.preloadAll()` 返回一个 promise，当所有可加载组件准备就绪时 promise 兑现。

``` js
import Loadable from 'react-loadable';
Loadable.preloadAll().then(() => {
  app.listen(5000, () => console.log(`http://localhost:5000/`));
});
```

这里介绍基于路由拆分需要懒加载的组件。通过 `Loadable` 声明需要懒加载的组件并注册到路由：

``` jsx
const Loading = () => <div>Loading...</div>;

const Home = Loadable({
  loader: () => import('./pages/home'),
  loading: Loading,
  // 导入模块的可选路径
  modules: ['./pages/home'],
  // 模块id数组
  webpack: () => [require.resolveWeak('./pages/home')], // require.resolveWeak：已同步方式获取模块ID，但不会把模块引入到 bundle 中（即弱依赖）
});

export const routes = [
  {
    component: Root,
    routes: [
      {
        path: "/",
        exact: true,
        component: Home,
      },
      //  ...
    ]
  }
];
```

`modules` 和 `webpack` 选项无需手动设置，可以通过 Babel 插件 `react-loadable/babel` 来完成。
``` json
{
  "plugins": [
    "react-loadable/babel"
  ]
}
```

`Loadable` 返回一个 [LoadableComponent](https://github.com/jamiebuilds/react-loadable#loadablecomponent) 组件，调用其静态方法 `preload()` 即可进行组件预加载。比如你可以在某事件的处理函数中预加载组件：

``` js
onMouseOver = () => {
  LoadableComponent.preload();
}
```

所以在服务端渲染时，需要对服务端匹配到的组件进行额外判断，如果是 `LoadableComponent` 组件则需要先预加载组件，预加载完成后才可以发起数据请求。

``` js
async function loadBranchData (pathname, store) {
  const branch = matchRoutes(routes, pathname);
  const promises = branch.map(({ route }) => {
    const component = route.component;
    // 判断是否是 LoadableComponent 组件
    if (component.preload) {
      // 组件预加载完成后请求数据
      return component.preload().then(res => {
        return res.default.loadData ? res.default.loadData(store) : null;
      })
    }
    return typeof component.loadData === 'function' ? component.loadData(store) : Promise.resolve(null);
  });
  const result = await Promise.all(promises);
  return result;
}
```

我们还需要搜集当前请求实际要预加载的模块，找到每个模块所在的 bundle 文件，最后将这些 bundle 通过 `<script>` 渲染到 HTML 中。

``` js
`${bundles.map(bundle => `<script src="/${bundle.file}"></script>`).join('\n')}`
```

使用 `Loadable.Capture` 收集正在预加载的模块：

``` jsx
const modules = [];

<Loadable.Capture report={moduleName => modules.push(moduleName)}>
  <StaticRouter context={context} location={req.baseUrl} >
    <App/>
  </StaticRouter>
</Loadable.Capture>
```

通过 `ReactLoadablePlugin` 插件获取模块到 bundles 的映射。
``` js
import { ReactLoadablePlugin } from 'react-loadable/webpack';

export default {
  plugins: [
    new ReactLoadablePlugin({
      filename: './dist/react-loadable.json',
    }),
  ],
};
```

使用 `react-loadable/webpack` 中的 `getBundles` 方法找到模块对应的所有 bundles。
如果根据你的 webpack 配置可能 `getBundles` 可能返回 js 以外的文件类型，比如 `.css` 或 `.map`，你还需要进行过滤：

``` js
const bundles = getBundles(stats, modules);

const styles = bundles.filter(bundle => bundle.file.endsWith('.css'));
const scripts = bundles.filter(bundle => bundle.file.endsWith('.js'));

res.send(`
  <!doctype html>
  <html lang="en">
    <head>
      ...
      ${styles.map(style => {
        return `<link href="/dist/${style.file}" rel="stylesheet"/>`
      }).join('\n')}
    </head>
    <body>
      <div id="app">${html}</div>
      <script src="/dist/main.js"></script>
      ${scripts.map(script => {
        return `<script src="/dist/${script.file}"></script>`
      }).join('\n')}
    </body>
  </html>
`);
```

最后还需要在客户端使用 `Loadable.preloadReady()` 预加载页面中包含的可加载组件。并确保在 HTML 最后执行 `window.main()`。

``` js
window.main = () => {
  Loadable.preloadReady().then(() => {
    ReactDOM.hydrate(<App/>, document.getElementById('root'));
  });
};
```

``` js
res.send(`
  <!doctype html>
  <html lang="en">
    <head>
      ...
      ${styles.map(style => {
        return `<link href="/dist/${style.file}" rel="stylesheet"/>`
      }).join('\n')}
    </head>
    <body>
      <div id="app">${html}</div>
      <script src="/dist/main.js"></script>
      ${scripts.map(script => {
        return `<script src="/dist/${script.file}"></script>`
      }).join('\n')}
      <script>window.main();</script>
    </body>
  </html>
`);
```

## 参考

- [不只是同构应用](https://zhuanlan.zhihu.com/p/79203739)
- [react-loadable](https://github.com/jamiebuilds/react-loadable)
