---
title: redux源码学习以及实现（一）- 实现自己的redux和redux-thunk
date: 2019-05-25 12:02:28
tags:
- 编程
- 前端
- react
---

# 写在前面

一直听说redux的源码量小精巧，一直想手动实现的一个redux，并且学习下官方的实现，以求精进自己的编程水平。为了增加自己的学习效果，我把本次学习分成了两个步骤：

1. 在看源码之前，只根据以前用过的redux和redux-thuk的api，先实现一个redux和redux-thunk出来。只要能够正常运行，用于以前的项目即可。
2. 阅读redux的官方实现，对比参照自己的实现，查看思路上的异同，实现的细节，对一些问题的解决思路。最后，总结自己实现的不足，以及官方redux实现的优秀之处。

# 正文

## 实现

### 一个简单的todoList应用

- 手动实现的redux的完成标准就是至少支持App.jsx使用的所有功能。

```javascript
// App.jsx
import React from 'react';

import store from './store';

class App extends React.Component {

    constructor(props) {
        super(props);
        this.state = store.getState();
        store.subscrible(() => {
            // 订阅，当store产生更新时，拿到新的state
            this.setState(store.getState());
        });
    }

    handleInputOnChange(evt) {
        // 更新inputValue的值
        const inputValue = evt.target.value;
        store.dispatch({
            type: 'input',
            payload: inputValue,
        })
    }

    handleButtonOnClick() {
        // 添加内容到todoList
        store.dispatch({
            type: 'add',
        })
    }

    componentDidMount() {
        // 模仿挂载时请求远程数据，传入一个异步函数类型的action
        store.dispatch((dispatch) => {

            new Promise((resolve) => {
                setTimeout(() => {  // 模仿异步ajax请求
                    const todoData = ['1', '2, ', '3'];
                    resolve(todoData);
                }, 2000);
            })
                .then((todoList) => {
                    dispatch({
                        type: 'init',
                        payload: todoList,
                    })
                });
        });
    }

    render() {
        const { todoList, inputValue } = this.state;
        return (
            <div>
                <input type="text" 
                    value={inputValue} 
                    onChange={this.handleInputOnChange.bind(this)} 
                />
                <button 
                    onClick={this.handleButtonOnClick.bind(this)}
                >ADD</button>
                <ul>
                    {todoList.map((content, index) => (
                        <li key={index}>{content}</li>
                    ))}
                </ul>
            </div>
        );
    }
}

export default App;
```

### 创建store以及手动实现redux-thunk

```javascript
// store.js

const reduxThunk = (stroe) => {
    // redux中间件
    const currentAction = stroe.__action;
    // 拿到传入的action对象
    if (typeof currentAction  === 'function') {
        currentAction(store.dispatch);
        stroe.__action = {};
        // 设置action为空对象，防止后续中间件处理函数类型的action出错
    }
};

const defaultState = {
    todoList: ['defaultTodo1', 'defaultTodo2'],
    inputValue: 'defaultValue',
}

const reducer = (state = defaultState, action) => {
    switch (action.type) {
        case 'add': {
            return {
                inputValue: '',
                todoList: [state.inputValue, ...state.todoList],
            };
        }
        case 'input': {
            return {
                ...state,
                inputValue: action.payload,
            }
        }
        case 'init': {
            return {
                ...state,
                todoList: action.payload,
            }
        }
        default: return state;
    }
};

const store = createStore(reducer, reduxThunk);

export default store;
```

### 实现redux

```javascript
// redux/index.js

export const createStore = (reducer, ...middleWares) => {
    const store = {
        __state: undefined,
        __action: undefined,  
    };
    // 将action和state挂载到store上，这使得无论多少中间件访问和修改的都是同一个state，action对象
    const subscribedCallbacks = []; // 订阅的回调函数

    const getState = function() {
        return this.__state;
    }

    const dispatch = function(action) {
        this.__action = action; // 挂载
        middleWares.forEach(mid => mid(store)); // 按顺序触发中间件
        const newState = reducer(this.__state, this.__action); // 产生新的state
        this.__action = undefined; // 清空
        this.__state = newState; // 设置新的state
        subscribedCallbacks.forEach(cb => cb()); // 触发订阅的回调函数
    }

    const subscrible = function(cb) {
        subscribedCallbacks.push(cb);
    }

    store.getState = getState.bind(store);
    store.dispatch = dispatch.bind(store); 
    // 绑定this，因为dispatch被调用时不一定是通过.语法,即store.dispatch这样调用的
    store.subscrible = subscrible.bind(store);

    store.dispatch({});
    // 手动触发拿到reducer提供的默认state
    return store;
}
```

## 实现思路

### getState

没什么好说的，直接返回当前存储的state就好了。

### subscrible

这个也没什么好说的，把传入的回调函数，存储到一个数组里，更新state后，遍历调用即可。

### dispatch

手动实现的redux大部分逻辑都在这里，首先接收到action，把action挂载store对象上，然后开始触发中间件(把store传入中间件)，中间件至少可以访问传入的action然后对其做一些操作，否则中间件就没意义了。
随后把当前的state和处理后的action传入reducer计算出新的state，然后触发订阅的回调函数。

这里注意，平常在使用dispatch时，很少通过 **obj.dispatch调用** 且obj也不一定是store，所以要特别注意 **this** 的指向问题，因此使用 **bind** 手动绑定下this为store。

#### 问题：

- 如何修改action本身，而非在action对象上增添属性

##### 场景：

用过redux-thunk的人都知道，thunk给了redux处理函数类型action(即传入的action不是一个{}而是一个function)的能力，所以当你传入一个函数类型的action类型时，thunk调用了这个函数，把dispatch方法传给这个函数。此时就出现了问题，我不知道官方redux中间件是怎么设计的，比如提供了中间件可以中断中间件调用流程的方法。不管怎样，手动实现的redux会对一个action按顺序调用所有的中间件处理，而其他中间件期望得到的是一个object，而不是一个函数，所以中间件应该通过一种途径访问action，并且能够修改action，使得下一次再访问时访问的是新的action。

于是，我就把action对象挂载到store对象上，每一个中间件都通过store对象访问action（调用形式：someMiddleWare(store)），而不是直接传入action(调用形式：someMiddleWare(store, action))。

于此，手动实现的的redux-thunk在判断action是个函数后，就调用函数，然后把stroe上的action设置成一个空的object，防止后续中间件出错。

### redux-thunk的实现

原理很简单，判断action是不是function，是的话就dispatch方法传给这个function，然后把action设置成一个空的{}，不是的话就不做任何处理。
