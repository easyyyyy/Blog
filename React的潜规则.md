# React的潜规则

## setState异步

- React在setState函数实现中，会根据变量`isBatchingUpdates`判断是直接更新还是放入队列。`isBatchingUpdates`默认为false，表示同步更新this.state
- 在React事件处理函数中或生命周期中使用setState。React会调用batchingUpdate方法，将`isBatchingUpdates`属性修改为true。使setState不会同步更新。
- 而原生的事件监听和setTimeout跳出react的体系，不会调用batchingUpdate，所以会直接更新

