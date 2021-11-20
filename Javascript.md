# JavaScript 总结

## 数据类型

### number

- JavaScript 将数字存储为 64 位浮点数，但所有按位运算都以 32 位二进制数执行

- 位运算 负数是以补码(负数为原码取反+1)的方式存储 `-2^31 < a < 2^31 -1`

  ```
  1111 1111 1111 1111 1111 1111 1111 1111 	-1
  
  0000 0000 0000 0000 0000 0000 0001 0010     18
  1111 1111 1111 1111 1111 1111 1110 1110     -18
  
  0111 1111 1111 1111 1111 1111 1111 1111     2^31
  1000 0000 0000 0000 0000 0000 0000 0000		-2^31  1<<31    -1<<31
  
  ```

  



## 异步

### Promise

- 手动实现一个Promise.all

  ```javascript
  Promise.myall = function (args) {
      let len = args.length, count = 0
      let result = []
      return new Promise((resolve, reject) => {
          args.forEach((item, index) => {
              item.then((res) => {
                 result[index] = res
                  count++
                  if (count === len) {
                      resolve(result)
                  }
              })
          })
      })
  }
  
  // test
  let p1 = new Promise((resolve, reject) => {
      setTimeout(() => {
          resolve(1)
      }, 1000)
  })
  let p2 = new Promise((resolve, reject) => {
      setTimeout(() => {
          resolve(2)
      }, 500)
  })
  
  Promise.myall([p1, p2]).then((res)=> {
      console.log(res)
  })
  ```



- Promise.all 并发限制

  ```javascript
  let urls = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17]
  
  function asyncLimit (arr, limit) {
      let lim = Math.min(arr.length, limit)
      let exe = []
      
      for (let i = 0; i< lim; i++) {
          let p = enqueue()
          exe.push(p)
      }
      
      Promise.all(exe).then(()=>{
          console.log('全部发送完成')
      })
      
      function enqueue () {
          return new Promise((resolve, reject) => {
              let url = urls.pop()
              console.log(url)
              setTimeout((res)=>{
                  resolve()
              }, parseInt(Math.random()*5000))
          }).then(()=>{
              if (urls.length !== 0) {
                  return enqueue()
              }
          })
      } 
  }
  
  asyncLimit(urls, 3)
  ```

  

## 垃圾回收算法

无法判定一些内存是不是“不在需要”

- 引用
  - 一个对象有访问另一个对象的权限，叫一个对象引用另一个对象
  - 对象包括
    - Javascript对象
    - 函数作用域（全局词法作用域）
- 引用计数垃圾收集
  - “对象是否不再需要” -> "对象有没有其他对象引用到"
  - 循环引用问题
  - IE6，7使用引用技术对DOM对象垃圾回收
- 标记-清除算法
  - 设定一个根对象(Javascript是全局对象)
  - 垃圾回收器会从根节点找所有可以引用的对象



## 代理与反射

### 代理

#### 创建代理

```javascript
const target = {
    id: 'target'
}

const handler = {}
const proxy = new Proxy(target, handler)
```

#### 捕获器

- 在处理程序对象中定义的“基本操作拦截器”

- 在代理对象上调用这些操作时，代理可以在这些操作传播到目标对象之前先调用捕获器函数，从而拦截修改相应行为

  ```javascript
  const handler = {
      get () {
          return 'handle'
      }
  }
  ```

- 反射api，**全局Reflect对象**的同名方法可以重建原始行为

  ```javascript
  const handler = {
      get () {
          return Reflect.get(...arguments)
      }
  }
  
  //简洁
  const handler = {
      get: Reflect.get
  }
  
  //否则
  const handler = {
      get (target, property, receiver) {
          return target[property]
      }
  }
  ```

- 捕获器不变式

  - 如果目标对象有一个不可配置且不可写的数据属性，捕获器返回与该属性不同的值时会报错

- 可撤销代理

  - Proxy构造函数有`revocable()`方法，返回撤销函数`revoke()`

    ```javascript
    const {proxy, revoke} = Proxy.revocable(target, handler)
    
    // ...
    
    revoke()
    
    // 在使用代理会抛出TypeError
    ```

- 反射api

  - 反射api不限于捕获处理程序

  - 大多数反射api在Object类型上有对应的方法

  - 状态标记

    - 很多反射方法会返回“状态标记的布尔值”

      ```javascript
      const o = {}
      if(Reflect.defineProperty(o, 'foo', {value: 'bar'})) {
          console.log('success')
      } else {
          console.log('failure')
      }
      ```

    - 下面方法会返回**状态标记**

      - Reflect.defineProperty()
      - Reflect.preventExtensions()
      - Reflect.setPrototypeOf()
      - Reflect.set()
      - Reflect.deleteProperty()

    - 代替操作符

      - Reflect.get(); 代替对象访问操作符
      - Reflect.set(); 代替=赋值
      - Reflect.has(); 代替in或with()
      - Reflect.deleteProperty(); 代替delete操作符
      - Reflect.construct(); 代替new操作符

  - 代理另一个代理

  - 代理的问题

    - 代理的this

      ```javascript
      const target = {
          thisVal () {
              return this === proxy
          }
      }
      
      const proxy = new Proxy(target, {})
      
      target.thisVal() // false
      proxy.thisVal()  // true
      ```

    - 内部槽位

      - 有些内置类型会依赖代理无法控制的机制
      - Date类型方法依赖内部槽位[[NumberDate]]



#### 代理捕获器操作（13种）

- get()
- set()
- has()
- defineProperty()
- getOwnPropertyDescriptor()
- deleteProperty()
- ownKeys()
- getPrototypeOf()
- setPrototypeOf()
- isExtensible()
- preventExtensibles()
- apply()
- construct()



#### 代理模式（使用代理有用的编程模式）

- 跟踪属性访问
- 隐藏属性
- 属性验证
- 函数与构造函数属性验证
- 数据绑定与可观察对象



### vue3.0通过Proxy劫持对象



## 对象Object

### 创建对象

#### 工厂模式

- 一句话概括：在函数内定义一个对象并返回

```javascript
function create () {
    let o = {}
    o.name = '123'
    o.age = 22
    o.sayName = function () {
        console.log(this.name)
    }
    return o
}
```

#### 构造函数

- 一句话：使用`this`指定属性，`new`一个对象
- new做了什么：
  - 定义一个空对象`o`
  - `o.__proto__ = Person.prototype`
  - `Person.call(o)`
  - 返回新对象

都是实例属性

```javascript
function Person (name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = function () {
        console.log(this.name)
    }
}
```

#### 原型模式

会共享属性

```javascript
function Person () {}

Person.prototype.name = '123'
Person.prototype.age = 123
Person.prototype.sayName = function () {
    console.log(this.name)
}
```



### 继承

#### 原型链继承

- 一句话：是**子类**的`prototype`等于父类的实例

```javascript
function SuperType () {
    this.property = true;
}

SuperType.prototype.getSuperValue = function () {
    return this.property
}

function SubType () {
    this.subproperty = false;
}

SubType.prototype = new SuperType();

SubType.prototype.getSubValue = function () {
    return this.subproperty
}
```

- 问题
  - 父类的实例属性会变为子类的原型属性
  - 子类实例化不能给父类传参

#### 盗用构造函数

- 一句话在子类的构造函数执行父类构造函数

```javascript
function SuperType () {
    this.colors = ["red", "blue"]
}

function SubType () {
    SuperType.call(this)
}
```

- 问题：
  - 无法继承父类的原型属性



#### 组合继承

```javascript
function SuperType(name) {
    this.name = name
    this.colors = ['red', 'blue']
}

SuperType.prototype.sayName = function () {
    console.log(this.name)
}

function SubType (name, age) {
    SuperType.call(this, name)
    this.age = age
}

SubType.prototype = new SuperType()

SubType.prototype.sayAge = function () {
    console.log(this.age)
}
```



#### 原型式继承

- 继承的是一个对象，不是构造函数

```javascript
function object(o) {
    function F() {}
    F.prototype = o
    return new F()
}
```

- 有一个对象，想在它的基础上创建新对象
- `Object.create()`将原型式继承规范化



#### 寄生式继承

- 比原型式继承多的一步是，增加一个自身的属性

```javascript
function createAnother (original) {
    let clone = object(original)
    clone.sayHi = function () {
        console.log("hi")
    }
    return clone
}
```



#### 寄生式组合继承

```javascript
function inheritPrototype(subType, superType) {
    let prototype = object(superType.prototype)
    prototype.constructor = subType
    subType.prototype = prototype
}

function SuperType(name) {
    this.name = name
    this.colors = ['red', 'blue']
}

SuperType.prototype.sayName = function () {
    console.log(this.name)
}

function SubType (name, age) {
    SuperType.call(this, name)
    this.age = age
}

inheritPrototype(SubType, SuperType)

SubType.prototype.sayAge = function () {
    console.log(this.age)
}
```



### js获取对象

- Object.keys()
  - 获取对象实例自身可枚举属性键
- Object.getOwnPropertyNames()
  - 获取对象实例自身全部属性键
- for...in
  - 输出对象实例自身以及原型链上可枚举属性键



## 判断数据类型 

- **typeof**

| 输入值    | 输出值 (String) |
| :-------- | :-------------- |
| undefined | undefined       |
| Function  | Function        |
| **null**  | **Object**      |
| Boolean   | Boolean         |
| String    | String          |
| Number    | Number          |
| Symbol    | Symbol          |
| Object    | Object          |
| Array     | Object          |

- 



## 数据类型

### Set

- 成员不能重复
- 只要键值，没有键名
- 可以遍历，`add`,`delete`,`has`

### weakSet

- 成员都是对象
- 成员都是弱引用，不会阻止垃圾回收
- 不可迭代，`add`,`delete`,`has`

### Map

- 键值对的集合
- 可以迭代

### weakMap

- 只接受对象作为键名(null除外)，不接受其他类型值作为键名
- 键名所指向的对象，不计入垃圾回收机制
- 不可迭代
- 方法：`get`,`set`,`has`,`delete`



## 高级方法

### 柯里化

- 柯里化是一种将使用多个参数的一个函数转化为一系列使用一个参数的函数的技术

  ```javascript
  function add(a, b) {
      return a + b;
  }
  
  add(1, 2)
  
  var addCurry = curry(add)
  addCurry(1)(2)
  ```

- 用途
  - 参数复用，降低通用性，提高适用性

      ```javascript
      function ajax(type, url, data) {
          var xhr = new XMLHttpRequest()
          xhr.open(type, url, true)
          xhr.send()
      }

      ajax('POST', 'www.test.com', "name=kevin")

      var ajaxCurry = curry(ajax);

      // 以 POST 类型请求数据
      var post = ajaxCurry('POST');
      post('www.test.com', "name=kevin");

      // 以 POST 类型请求来自于 www.test.com 的数据
      var postFromTest = post('www.test.com');
      postFromTest("name=kevin");
      ```
      
  - 传给回调函数，如map
  
      ```javascript
      var person = [{name: 'kevin'}, {name: 'daisy'}]
      
      // 传统
      var name = person.map(function (item) {
          return item.name;
      })
      
      //curry
      var prop = curry(function (key, obj) {
          return obj[key]
      });
      
      var name = person.map(prop('name'))
      ```
  
- 实现

  - 第一版

    ```javascript
    var curry = function (fn) {
        var args = [].slice.call(arguments, 1)
        return function () {
            var newArgs = args.concat([].slice.call(arguments))
            return fn.apply(this, newArgs)
        }
    }
    
    function add(a, b) {
        return a + b;
    }
    
    var addCurry = curry(add, 1, 2);
    addCurry() // 3
    //或者
    var addCurry = curry(add, 1);
    addCurry(2) // 3
    //或者
    var addCurry = curry(add);
    addCurry(1, 2) // 3
    ```




## 常用DOM方法

### 访问/获取节点

```javascript
document.getElementById(id);　　　　　　　　 　　//返回对拥有指定id的第一个对象进行访问
document.getElementsByName(name);　　　　　　//返回带有指定名称的节点集合　　 注意拼写:Elements
document.getElementsByTagName(tagname); 　　//返回带有指定标签名的对象集合　  注意拼写：Elements
document.getElementsByClassName(classname);  //返回带有指定class名称的对象集合 注意拼写：Elements
```

### 创建节点/属性

```javascript
document.createElement(eName);　　//创建一个节点
document.createAttribute(attrName); //对某个节点创建属性
document.createTextNode(text);　　　//创建文本节点
```

### 添加节点

```javascript
document.insertBefore(newNode,referenceNode);　 //在某个节点前插入节点
parentNode.appendChild(newNode);　　　　　　　　//给某个节点添加子节点
```

### 复制节点

```javascript
document.cloneNode(true | false);　　//复制某个节点  参数：是否复制原节点的所有属性
```

### 删除节点

```javascript
parentNode.removeChild(node);　　//删除某个节点的子节点 node是要删除的节点
```

注意：为了保证兼容性，要判断元素节点的节点类型(nodeType)，若nodeType==1，再执行删除操作。通过这个方法，就可以在 IE和 Mozilla 完成正确的操作。

- 节点类型

| 元素类型     | 节点类型 |
| ------------ | -------- |
| 元素element  | 1        |
| 属性attr     | 2        |
| 文本text     | 3        |
| 注释comments | 8        |
| 文档document | 9        |



### offset

- 可见高度加滚动条

|                                                              |                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`HTMLElement.offsetHeight`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetHeight) 只读 | `double`                                                     | 元素自身可视高度加上上下border的宽度                         |
| [`HTMLElement.offsetLeft`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetLeft)只读 | `double`                                                     | 元素自己border左边距离父元素border左边或者body元素border左边的距离 |
| [`HTMLElement.offsetParent`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetParent)只读 | [`Element`](https://developer.mozilla.org/zh-CN/docs/Web/API/Element) | 元素的父元素，如果没有就是body元素                           |
| [`HTMLElement.offsetTop`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetTop)只读 | `double`                                                     | 元素自己border顶部距离父元素顶部或者body元素border顶部的距离 |
| [`HTMLElement.offsetWidth`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetWidth)只读 | `double`                                                     | 元素自身可视宽度加上左右border的宽度                         |



- 滚动高度，

|                              |      |                                                             |
| ---------------------------- | ---- | ----------------------------------------------------------- |
| Element.scrollHeight（只读） |      | 表示元素的滚动视图高度                                      |
| Element.scrollLeft（只读）   |      | 表示该元素横向滚动条距离最左的位移                          |
| Element.scrollTop（只读）    |      | 表示该元素纵向滚动条距离（开启了垂直滚动条的元素才会不为0） |
|                              |      |                                                             |
|                              |      |                                                             |

|                              |      |                                                              |
| ---------------------------- | ---- | ------------------------------------------------------------ |
| Element.clientHeight（只读） |      | 它是元素内部的高度(单位像素)，包含内边距，但不包括水平滚动条、边框和外边距。 |
| Element.clientLeft（只读）   |      | 表示一个元素的左边框的宽度，以像素表示。                     |
| Element.clientTop（只读）    |      | 一个元素顶部边框的宽度（以像素表示）。不包括顶部外边距或内边距。 |
|                              |      |                                                              |
|                              |      |                                                              |



### 获取元素是否进入视图（懒加载）

`getBoundingClientRect()`

- 懒加载

  ```js
   var imgs = document.querySelectorAll('img');
  
  //用来判断bound.top<=clientHeight的函数，返回一个bool值
  function isIn(el) {
      var bound = el.getBoundingClientRect();
      var clientHeight = window.innerHeight;
      console.log(bound.top, clientHeight)
      return bound.top <= clientHeight;
  } 
  //检查图片是否在可视区内，如果不在，则加载
  function check() {
      Array.from(imgs).forEach(function(el){
          if(isIn(el)){
              loadImg(el);
          }
      })
  }
  function loadImg(el) {
      if(!el.src){
          var source = el.dataset.src;
          el.src = source;
      }
  }
  window.onload = window.onscroll = function () { //onscroll()在滚动条滚动的时候触发
      check();
  }
  ```

  





## 正则表达式

### 正则表达式替换

```js
'bar foo'.replace(/(...) (...)/, '$2 $1')
```



### 匹配插值表达式

```js
let rkuohao = /\{\{(.+?)\}\}/g

/** 根据路径 访问对象成员 */
function getValueByPath(obj, path) {
  let paths = path.split('.'); // [ xxx, yyy, zzz ]
  let res = obj;
  let prop;
  while (prop = paths.shift()) {
    res = res[prop];
  }
  return res;
}

_value = _value.replace(rkuohao, function (_, g) {
  return getValueByPath(data, g.trim()); // 除了 get 读取器
});
```



### 贪婪、非贪婪匹配

| .    | （小数点）默认匹配除换行符之外的任何单个字符。<br>例如，`/.n/` 将会匹配 "nay, an apple is on the tree" 中的 'an' 和 'on'，但是不会匹配 'nay'。<br>如果 `s` ("dotAll") 标志位被设为 true，它也会匹配换行符。 |
| ---- | :----------------------------------------------------------- |
| ?    | 匹配前面一个表达式 0 次或者 1 次。等价于 `{0,1}`。<br>例如，`/e?le?/` 匹配 "angel" 中的 'el'、"angle" 中的 'le' 以及 "oslo' 中的 'l'。<br>如果**紧跟在任何量词 \*、 +、? 或 {} 的后面**，将会使量词变为**非贪婪**（匹配尽量少的字符），和缺省使用的**贪婪模式**（匹配尽可能多的字符）正好相反。例如，对 "123abc" 使用 `/\d+/` 将会匹配 "123"，而使用 `/\d+?/` 则只会匹配到 "1"。 |
| +    | 匹配前面一个表达式 1 次或者多次。等价于 `{1,}`。<br>例如，`/a+/` 会匹配 "candy" 中的 'a' 和 "caaaaaaandy" 中所有的 'a'，但是在 "cndy" 中不会匹配任何内容。 |
| *    | 匹配前一个表达式 0 次或多次。等价于 `{0,}`。<br>例如，`/bo*/` 会匹配 "A ghost boooooed" 中的 'booooo' 和 "A bird warbled" 中的 'b'，但是在 "A goat grunted" 中不会匹配任何内容。 |

```js
// 贪婪（尽可能匹配多的）
var re = /\{(.+)\}/g
var str = '{asd } sadf}'
str.match(re)[0]
// '{asd } sadf}'

// 非贪婪
var re = /\{(.+?)\}/g
var str = '{asd } sadf}'
str.match(re)[0]
// '{asd }'
```



## 迭代

### for ... in

遍历可枚举属性，包括原型链上的

```js
let arr = [0,1,2]
for(let i in arr) {
    console.log(i)
}
//0
//1
//2

let s = new Set([1,2,3])
for(let i in s) {
    console.log(i)
}
//不会有输出
let m = new Map()
m.set([1,2])
for(let i in m) {
    console.log(i)
}
//不会有输出
```

### for...of

遍历可迭代对象

```js
let arr = [0,1,2]
for(let i of arr) {
    console.log(i)
}
//0
//1
//2

let s = new Set([1,2,3])
for(let i of s) {
    console.log(i)
}
//0
//1
//2
let m = new Map()
m.set([1,2])
for(let i of m) {
    console.log(i)
}
//[1,2]
```



