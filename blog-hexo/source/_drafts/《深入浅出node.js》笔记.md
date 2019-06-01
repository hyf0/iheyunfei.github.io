---
title: 《深入浅出node.js》笔记
tags:
- nodejs
- 编程
---

# 第二章 模块机制

- commonJS规范
    - 模块引用（怎么引用一个模块？）
    - 模块定义（如何定义一个模块？）
    - 模块标识（怎么找到一个模块？）

- Node
    - 引入模块的三个步骤:
        1. 路径分析
        2. 文件定位
        3. 编译执行
    - 分类：
        - 核心模块（内建）
        - 文件模块（自定义）
    - 特性：
        - 存在缓存机制，且核心模块的缓存优先于文件模块
        - node原生支持引入.json文件，加载后，node会调用JSON.parse方法，后把值赋给接收的变量
    - node模块标识符：
        - 核心模块名称：http、path...
        - 相对路径或绝对路径
        - 自定义非路径形式的模块模块名称
    - 模块加载顺序
        1. 核心模块缓存
        2. 核心模块
        3. 自定义模块缓存
        4. 自定义模块
    - 模块查找策略
        - 路径策略:
            - 沿路径递归查找每一层的node_modules文件夹，直到根目录
        - 后缀分析：
            - 当引入的模块没有指定后缀时，会按 .js, .json, .node以此尝试加载
            - 若尝试加载一个文件夹，一般情况下，node会尝试读package.json指定的入口文件，没有或非法的情况下，会尝试加载文件夹下的名index的文件，后缀策略同上
    - 关于exports和module.exports的问题
        - 书中仅仅指出如果直接对exports进行了赋值，并不会导出值内容。并且表示这是因为exports是引用类型，直接赋值，并不会改变域外的值。然而如果细细思考，就会发现这个逻辑上存在一些问题，如果你对modules.exports直接赋值，同样改变了其 **指向**，为什么这种情况，node可以正确识别导出的模块的内容呢？我相信作者是知道为什么的，只是表述上出现了一些偏差，以及少提了一些内容
        - 其根本原因在于，node是通过读取module对象上**exports属性**上的值，**而非变量exports的值**来加载模块，请看以下的示例代码就很容易理解了
```javascript
module.exports = {};
exports = module.exports;
(function (exports, require, module, __filename, __dirname) {
    // 你编写的模块文件
})
```

        - node通过执行这个函数，随后通过module.exports得到模块导出的内容，这时我们就可以清晰的看到，如果你在模块内部仅仅是改变exports对象的属性，那么没关系，因为exports此时和module.exports指向的是同一个对象，node通过module.exports拿到的是同一个对象。但是如果直接改变exports的值，那么此时函数内部exports指向的对象就和module.exports不一样了，此时node通过module.exports就拿不到exports上的内容。如果，直接对module.exports赋值一个新对象，则没有任何问题。因为无论如何改变module.exports的值，module这个对象始终在 **属性exports** 上存储着对新对象的引用，node依旧可以通过module.exports拿到导出内容。

        - 再次强调根本原因：node是通过读取module对象上**exports属性**上的值，**而非变量exports的值**来加载模块

        - 全局安装包，是个容易误解的名字，并非是可以从任何地方导入这个包，而是将包安装为全局可用的可执行的命令