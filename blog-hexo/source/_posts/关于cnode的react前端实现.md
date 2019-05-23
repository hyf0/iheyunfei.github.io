---
title: 关于cnode的react前端实现
date: 2019-05-23 17:40:47
tags:
- react
- 编程
- 前端
---

# 演示

![演示图片](https://github.com/iheyunfei/cnode-react/blob/master/public/show.gif?raw=true)

- [点此查看程序演示](http://iheyunfei.com/cnode-react/)
- [github地址](https://github.com/iheyunfei/cnode-react)

# 写在前面

本项目是学习react一个非常好的案例，不论是入门还是想进阶的人都能得到帮助。如果你觉得本项目确实给你带来了收获，记得分享给他人和 **star** 一下。

通过本项目，你可以学到：
- 一个合理的react项目应该是什么样子
- react/redux项目的目录结构组织方式
- 模块化开发以及管理数据的方式
- 如何抽象容器型和展示型组件
- 布局(Layout)组件的作用
- 使用react-router-dom的最佳实践
- 如何使用immutable.js，并和react，redux结合
- 通过约定命名减少错误
- 本项目的各个组件的开发、抽象思路(我写了大量的注释来阐释自己的想法，对某一功能的取舍，某一bug的思考)

# 启动程序

- 推荐使用yarn作为包管理器

**注意：本项目代码遵循Airbnb编码规范并开启了eslint的严格模式，如果eslint出现error，则会编译失败**

```
# install dependencies
yarn install or npm install

# serve with hot reload
yarn start or npm start

# build for production with minification
yarn build or npm run build
```

# Todo
- [√] 话题列表页
- [√] 话题列表侧边栏
- [√] 话题列表无限加载
- [√] 话题详情页
- [√] 话题详情侧边栏
- [√] 登录状态保存管理

因为cnode关于发帖的API已经被禁止，没有实现发帖功能

# 技术栈

## 数据管理

本项目采用 **redux** 并结合了 **immutabe.js** 作为管理数据的方式。
并做了如下限制：

- redux中除了基本值类型，任何对象都要经过immutable提供fromJS方法转换成immutable对象才能存入
- 在上一条的约束下，redux store中的变量使用正常的命名方式
- 组件中的任何变量的值如果是immutable对象，必须以$开头

## 样式模块化

- 全局样式文件在统一在入口文件 **index.js** 处引入
- 使用 **styled-components** 作为组件样式私有化方案

# 目录组织

```
cnode-react
        ├─config（打包配置文件
        ├─docs（build生成的文件
        │  └─static
        │      ├─css
        │      ├─js
        │      └─media
        ├─public
        ├─scripts
        └─src
            ├─assets（公共资源
            │  ├─font
            │  ├─images
            │  └─styles
            ├─components（展示型且组件
            │  ├─Header
            │  ├─Image
            │  ├─Loading
            │  ├─Panel
            │  └─UserInfoPanel
            ├─containers（容器型组件
            │  ├─Detail
            │  │  ├─components（私有展示型组件
            │  │  │  ├─AuthorInfoPanel
            │  │  │  ├─ReplyList
            │  │  │  └─Topic
            │  │  └─store（模块化store和常量定义处
            │  ├─Home
            │  │  ├─components
            │  │  │  ├─TopicList
            │  │  │  └─TopicListItem
            │  │  └─store
            │  └─Login
            │      ├─components
            │      └─store
            ├─Layout（布局组件
            │  └─CommonLayout
            ├─store（全局store处，用来导入各处私有store
            └─utils（公用方法，如请求用request对象
```

# 其他

## 使用eslint强制代码规范

- 代码遵循Airbnb编码规范

# License

[MIT](https://opensource.org/licenses/MIT)

# 遇到的问题以及反思

## eslint无法解析class静态属性，表现为在针对下列代码显示unexpected token "="

```javascript
class Foo extends Component {
  static propTypes = {
//                 ↑ unexpected topken "="
    foo: PropTypes.string,
  }
}

```
### 解决方法

这个是因为eslint默认的解析器并不支持es7的语法，安装 **eslint-babel**，并在.babelrc里把 **parser** 设置成babel-eslint即可

```javascript
// .babelrc
module.exports = {
  // ...
  "parser": "babel-eslint",
  // ...
};

```

## 路由跳转时高度不会重置的问题

### 解决方法
  - react-router-dom会对路由组件注入match属性
  - 在[componentDidUpdate](https://zh-hans.reactjs.org/docs/react-component.html#componentdidupdate)里拿到新旧的props
  - 通过拿到新旧不同的路径，进行比对，如果不同执行scrollTo(0, 0);

这显然不是好的解决方法，每次路由跳转都回到顶部和路由不会重置都是糟糕的用户体验，我想了想，是不是可以自己维持一个路由跳转信息栈，用于保存路由信息和页面高度，当跳转时弹出栈中的一条路由信息和当前的路由进行比对，如果相符，就通过window.scrollTo滑动的保存的页面高度，我觉得，这个方法的难点在于如果在合适的时机清空栈中的信息

## 快速多次切换路由，导致大量的网络请求，数据被重复渲染或覆盖

### 场景

论坛的帖子是通过sub(主题)分类的，页面上的每一个tab(标签)对应一个sub，每一次点击tab都会跳转到相应路由，组件通过路由拿到当前sub，然后请求相应的数据。然而，当用户快速在路由间切换时，如果没做限制，就会导致大量的异步网络请求，并且由于异步特性，可能导致后发的请求比前发的先到，然后数据被前发的覆盖掉。为了解决这个问题，我尝试了三种方法，每一种都比上一种要好一点。

### 解决方法

1. 是当每一次新的请求时，就隐藏掉页面上的tab组件，只有请求完毕后才显示，这个方法缺点很明显，会造成非常不好
 且可以明确感知的用户体验
2. 随后，我决定在reducer里做判断，在action中把请求数据请求的那个tab信息带上，如果和当前的tab不符合就丢掉数据，这个解决方法比上一个好，tab区域不会被隐藏，但还是有问题，即用户大量点击的情况下，依然会导致大量 的网络请求，并且得到数据后还是会触发action，虽然不合法的数据被丢掉了，但是大量的数据会导致网页内存激增，性能下降。
3. 最后，我想出了最佳的解决办法就是，在每一次点击tab导致的请求后，把以前的所有请求全部取消掉，因为请求是前一个tab的数据所以取消掉，不会造成任何影响(这个得益于axios提供了取消promise请求的功能)

## 通用组件防止内存泄漏问题

在我写react时，总会出现一个下面的错误提示
```
Can't perform a React state update on an unmounted component. This is a no-op......
```

### 原因

我封装了Image组件，接受一个src和alt参数，与img标签不一样的地方在于，Image组件会在图片未加载时显示一个loading效果，我通过state管理组件是否处于loading的状态，通过设置img元素的onload属性，在触发onload事件后，会触发一个回调函数，设置state的loadding状态为false。然而，Image作为一个通用组件，我没有考虑到Image组件可能会请求未完成的情况下，就被销毁，而此时事件还在，当img元素加载完毕后，就去触发回调，调用setState，而此时组件已经被销毁，就导致了内存泄漏的问题。

分析出原因后，解决这个问题就变得简单了，在unmount的时候，设置 **img.onload = null;** 就行了。

### 反思

似乎，react常见的内存泄漏原因就是忘记组件unmount时销毁事件，这牵扯到react事件的实现原理，react把所有事件挂载到window上进行代理，所以当元素被销毁时，浏览器无法销毁与其相关的事件，因为事件没有绑定在元素身上。

# 写在最后

建议编码的时候使用airbnb的编码规范，airbnb作为一个大型企业一定在js上吃了不少的亏，所以使用它们的编码规范，不仅使代码看着舒服还可以少踩很多的坑。