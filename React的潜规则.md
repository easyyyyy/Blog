# React的潜规则

## setState异步

- React在setState函数实现中，会根据变量`isBatchingUpdates`判断是直接更新还是放入队列。`isBatchingUpdates`默认为false，表示同步更新this.state
- 在React事件处理函数中或生命周期中使用setState。React会调用batchingUpdate方法，将`isBatchingUpdates`属性修改为true。使setState不会同步更新。
- 而原生的事件监听和setTimeout跳出react的体系，不会调用batchingUpdate，所以会直接更新



## memo和useCallback()

```jsx
import React, { useState, memo, useCallback } from 'react';

function Expensive({ onClick, name }) {
  console.log('Expensive渲染');
  return <div onClick={onClick}>{name}</div>
}

const MemoExpensive = memo(Expensive);

function Cheap({ onClick, name }) {
  console.log('cheap渲染');
  return <div onClick={onClick}>{name}</div>
}

export default function Comp() {
    const [dataA, setDataA] = useState(0);
    const [dataB, setDataB] = useState(0);

    const onClickA = () => {
        setDataA(o => o + 1);
    };
    
    const onClickB = useCallback(() => {
        setDataB(o => o + 1);
    }, []);
    
    return <div>
        <Cheap onClick={onClickA} name={`组件Cheap：${dataA}`}/>
        <MemoExpensive onClick={onClickB} name={`组件Expensive：${dataB}`} />
    </div>
}
```

- 使用memo()包裹function component后，React会对props浅比较，props没变化就不会更新该子组件
- useCallback()可以使onClickB不会随着父组件更新而变化

