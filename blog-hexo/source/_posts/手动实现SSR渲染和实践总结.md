---
title: 手动实现SSR渲染和实践总结
date: 2019-05-15 20:46:45
tags:
- 前端
- 编程
---
# 思路

react-dom/server 提供了 renderToString 函数，可以从虚拟 DOM 结构生成 html 字符串，后端通过拼接字符串生成完整的 html 代码，然后返回给前端。前端通过 react-dom 提供的 hydrate 方法，在前端进行同构，实现事件方法的绑定等功能。

# 流程

## 搭建环境

### 配置 webpack

- 后端和前端的代码编译需要不同的 webpack 配置
  - 后端的 webpack 配置文件，注意把 target 设置成 node
- 后端编译注意不要把 node 的代码重复打包
  - 使用 webpackn-node-externals 来避免重复打包
- 使用 babel-loader 支持 jsx 和最新的语法
  - 安装@babel/core 使用最新的 babel
  - 安装@babel/preset-env 来支持最新的语法
  - 安装@babel/preset-react 编译 jsx
  - 注意不要重复编译 node_modules 里的文件

### 调试环境

- 安装使用 npm-run-all 来使得一键自动编译前端和后端文件，并且开启一个调试服务器

### 引入路由

- 使用 react-router-dom
- 服务器端需要使用 StaticRouter 而非 BrowserRouter
- 后端的 StaticRouter 需要获取请求路径和一个 context 参数
- 使用react-router-config的renderRoutes方法，代替手动来生成路由组件

### 引入 Redux

- 通过包裹 createStore 函数，使得前后端的可以创建不同的 store，以支持其特异的行为
- 后端针对每一个 request 都要创建一个新的 store
- 注意，服务器不会执行组件的 componentDidMount 方法

### 异步数据渲染

- 在组件上实现静态方法 loadData，当后端收到请求时，通过路径匹配，得到匹配路由下的组件，通过执行对应将要显示组件的 loadData 函数，将得到数据后传入 store，随后再执行 renderToString 方法，从使得异步数据直接在服务器端渲染
- 后端执行异步请求时，可以使用 Promise.all 来等待所有请求完成后再渲染
```javascript
matchedRoutes = routes.filter(
  route => route.loadData && isMatch(route.path, request.path)
// 拥有加载数据的请求且与当前请求路径相匹配
);
requests = Promise.all(matchedRoutes.map(route => route.loadData()));
// loadData返回一个Promise,这里有个小坑,暂且不表
```

- 使用 react-router-config 来帮助进行有关路由的操作
- 可以使用 react-router-config 的 matchRoutes 来匹配多级路由
- 前后端重复渲染的问题
    - 问题：后端的store数据没有传递给前端的store
        - 解决：通过脱水和注水，把后端的store数据传递给前端
    - 前端会再起Ajax请求获取数据
        - 解决：在请求代码前，增加判断操作，只当不存在数据时才请求数据
    - 注意，不能单纯的删除前端的请求代码
        - 因为：SSR渲染只在首次请求执行，后续的操作会由前端的React接管，这样的话只有首屏的页面有数据，而页面跳转时不会执行SSR渲染，前端又没有请求数据的函数，所以会出现无法显示数据的情况
        - 或者说：脱水的行为只发生在前端的第一次请求

#### 异步数据请求的问题

小坑：请针对以下代码考虑，A请求失败，BCD还在请求的情况。

```javascript
const coms = [A, B, C, D];
//假设一个页面拥有A, B, B, D四个组件且都需要异步请求数据来渲染,都拥有loadData方法来请求数据，且其返回一个Promise
Promise.all(coms.map(com => com.loadData))
    .then(() => {
        // 此时store数据已经被设置好
        response.send(render()) //执行渲染函数
    })
    catch(() => {
        response.send(render())
    });
//直接的写法
```

这种情况下，由于A失败，会直接执行catch方法，而BCD的请求会直接断掉，渲染的页面ABCD的数据都没有。这就导致了一个问题，ABCD的数据并不是互相依赖的，**A数据请求失败，不应该影响到BCD数据显示**，所以应该使用下面一种写法

```javascript
const coms = [A, B, C, D];
//假设一个页面拥有A, B, B, D四个组件且都需要异步请求数据来渲染,都拥有loadData方法来请求数据，且其返回一个Promise
Promise.all(coms.map((com) => (new Promise(resolve, reject) {
    com.loadData()
        .then(resolve)
        .catch(resolve);
}));
//更好的写法
```

通过在请求数据的Promise外面再包裹一层Promise,无论请求是否成功，都会使得外层的Promise变成resolved状态，不至于影响终止Promise.all的操作。
此时，尽管A请求数据失败，但是BCD的数据可以正常显示。

### 代理服务器

- 使用express-http-server使得express可以作为一个代理服务器代理前端请求
- 注意前端的后端的ajax的请求地址不同
    - 通过创建不同的axios实例来配置url
    - 通过redux-thunk提供的withExtraArgument方法，在store中标识当前环境是后端还是前端，并且传递相关的axios实例
- 代理前端请求时注意转发其Cookie

### 404页面的实现
- 通过在同级配置一个不带有path属性的路由来实现404页面
- 后端通过新建一个context空对象，传递给StaticRouter，StaticRouter会将context对象重命名为staticContext传递给路由组件，404路由组件在其内部对context对象就行修改来标识前端访问的页面不存在，后端根据context对象的标识，修改状态码为404

#### 301重定向的实现
- 前端通过Redirect组件实现
- 后端实现思路同上，react-router对于返回Redirect组件的组件，会对其props注入一些参数，标识此页面被重定向了

### SSR中的css

- 使用isomorphic-css-loader提供的_getCss方法在后端拼接上css样式
- 同样通过context来传递不同组件的style样式
- 注意，context只会传递给路由组件

# 其他

## 预渲染
- 对于首屏时间没有太多要求的话，使用预渲染代替SSR是更好的选择。
- 预渲染解决了搜索引擎SEO的问题
- preRender框架提供了预渲染的功能
- 原理：通过模拟一个浏览器，拿到网页源码，然后返回
- 需要判断请求来自用户还是搜索引擎

## 注意

- 当实现一个组件的静态方法的时候，要注意其可能被高阶组件包裹后，无法正常的访问其静态方法
- SSR渲染只在首次请求执行，后续的操作会由前端的React接管

## 了解

- 在 SSR 项目中，通常会启一个 Node 服务器作为中间层与服务器进行通信
- 高阶组件抽象，就是对一个组件中的代码公共部分进行抽离，通过参数来传递不同的部分

# 实践

我尝试跟随上述步骤将曾经的一个项目改写成SSR渲染，过程中我遇到了很多问题，有些简单，有些困难。
- 比如服务器渲染总是不成功，原因只是我忘了配置多级路由的二级路由。
- 我使用的styled-components框架，在提供SSR渲染方法上非常的简单，几乎只需不到5行的代码，比起单独导入css文件，再通过staticContext传递要方便很多
- react-router@4的相对路由用着很反直觉，我还是倾心于中心式的路由管理
- 由于项目使用immutable.js，数据的脱水注水快把我搞疯掉了
- 在最终改造完成后，我不得不同意那个观点 **除非万不得已，预渲染是个非常好的选择**