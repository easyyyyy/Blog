# CSS

## FLEX弹性布局

### 容器的属性

```css
.container {
    flex-direction: row | row-reverse | column | column-reverse; // 主轴方向
    flex-wrap: nowrap | wrap | wrap-reverse; // 是否换行
    flex-flow: <flex-direction> || <flex-wrap>; // 简写形式
    justify-content: flex-start | flex-end | center | space-between | space-around; // 主轴对齐方式
    align-items: flex-start(起点对齐) | flex-end(终点对齐) | center | baseline(第一行文字的基线对齐) | stretch<没有设置高度则占满容器高度(默认值)>; // 交叉轴的对齐方式
    align-content: flex-start | flex-end | center | space-between | space-around | stretch; // 定义多根轴线的对齐方式(flex-wrap: wrap时会出现多条轴线)
}
```

### 项目属性

```css
.item {
    order: <integer>; // 项目在容器中的排列顺序
	flex-basis: <length> | auto; // 在分配多余空间前，项目占据的主轴空间，默认值为auto，宽高取决于width或height
	flex-grow: <number>; // 项目放大比例
	flex-shrink: <number>; // 项目缩小比例
	flex: none | [ <'flex-grow'> <'flex-shrink'> || <'flex-basis'> ] // 简写
    // 默认值 0 1 auto
    // auto (1 1 auto)
    // none (0 0 auto)
    // 1 (1 1 0%)  非负数字，该数字是flex-grow的值
    // 0% (1 1 0%)  长度或百分比，是为flex-basis
    // 2 3 (2 3 0%) 两个非负，该数字是flex-grow和flex-shrink的值
    // 11 32px (11 1 32px) 非负和长度或百分比，该数字是flex-grow和flex-basis的值
	align-self: auto | flex-start | flex-end | center | baseline | stretch; // 默认值auto，继承父元素的align-items属性
}
```



## 动画

```css
.element {
    animation: fadein ease-in 2s;
    animation-fill-mode: forwards; //定义动画结束的样子
}

@keyframes fadein {
    from {
        opacity: 0;
    }
    to {
        opacity: 1;
    }
}
```

