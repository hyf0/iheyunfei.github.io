使用react编写了简书的主页、文章页，登录页，也算一个比较完整的react项目，收获不少。知道了一个项目正常开发流程，目录组织，一些常见的问题的应对方案，在此记录下。

# 搭建项目

- 使用create-react-app创建项目
- 使用yarn安装依赖

# 目录组织

- project
    - src
        - store（全局store）
            - index.js（store出口）
            - reducer.js（组合来自其他组件的reducer）
        - common（公共组件）
            - OneCommonComponent（一个公共组件的例子）
                - index.js（组件代码出口）
                - style.js（使用styled-components编写的css in js样式文件，使得组件的css私有化)
                - store 
                    - index.js（store出口文件）
                    - reducer.js（组件的reducer）
                    - actionTypes.js（action的type以常量的形式定义在这里）
                    - actionCreators.js（创建action的函数，以及异步的被包装成action的请求函数定义在这里）
                - components（组件的私有组件）
                    - OnePrivateCom1.js
                    - OnePriveteCom2.js
        - pages（非公共组件定义处）
            - home（例子：一个Home组件)
                - index.js（组件代码出口）
                - style.js（使用styled-components编写的css in js样式文件，使得组件的css私有化)
                - store 
                    - index.js（store出口文件）
                    - reducer.js（组件的reducer）
                    - actionTypes.js（action的type以常量的形式定义在这里）
                    - actionCreators.js（创建action的函数，以及异步的被包装成action的请求函数定义在这里）
                - components（组件的私有组件）
                    - OnePrivateCom1.js
                    - OnePriveteCom2.js


# 数据管理

- 使用redux
- 使用react-redux简化对redux的操作
- 使用redux中间件redux-thunk来支持异步action
- 使用immutable.js
    - 对immutable对象命名作出约定，如 $list.get()
- 分割store，通过combineReducers组装
    - 使用redux-immutable提供的combineReducers来使得合成reducer变成immutabe
- 使用redux-immutable在redux中使用immutable.js
- 使用redux-devtools调试工具

# css解决方案

- 使用styled-components来管理css
- vscode扩展vscode-styled-components提供语法高亮和自动补全

# 路由

- 使用react-router@4管理路由

# 如何实现动画效果

- 手动切换class
- 通过定义keyframes
- 使用css-transition-group

# 数据类型检查
- 使用React提供的propTypes检查数据类型

# 性能调优
- 重载shouldComponentUpdate函数，手动判断是否需要rerender
- 组件尽量继承pureComponent（配合immutable.js使用）
