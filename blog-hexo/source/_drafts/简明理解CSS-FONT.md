---
title: 简明理解CSS-FONT
tags:
- css
- 前端
---

# 字体族

# 字体fallback

- 依次向后寻找字体
    - 字体名用引号
    - 字体族不需要引号
- 书写顺序
    - 首先满足平台特殊，最后使用通用字体

# 网络字体、自定义字体


```css
// 定义
font-family {
    font-family: 'IF',   //字体命名
    src: url("url");     //远程字体需要对付允许跨域
}

// 使用

.custom-text {
    font-family: "IF";
}
```

# icon-font

- iconfont.cn

# 行高的构成
