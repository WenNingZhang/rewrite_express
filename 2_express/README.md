## 重写express(二)

### 目标：实现路由中间件(以get方法为例)

app.get('/test', (req, res) => res.send('Hello World!'));

### 预备知识

#### 1. Array.prototype.slice.call(arguments);

把类数组对象转化为数组，并且返回转化后的数组。

```
function foo () {
      console.log(arguments)
      console.log(Array.prototype.slice.call(arguments))
      console.log(Array.prototype.slice.call(arguments, 1))
      console.log(arguments);
}


The slice() method returns a shallow copy of a portion of an array into a new array object selected from begin to end (end not included). The original array will not be modified.

foo ('hello', 'world')
{ '0': 'hello', '1': 'world' }
[ 'hello', 'world' ]
[ 'world' ]
{ '0': 'hello', '1': 'world' }
```

#### 2. require模块时，Node对模块进行缓存，第二次require时，是不会重复开销的。

#### 3. router、route、layer 区别：
+ router 相当于一个中间件容器，每个应用只会创建一个router。
+ 每个路由中间件会对应一个layer对象，而判断路由中间件和普通中间件的区别是判断layer.route是否为空。

### 流程

```
let express = require('./express');
let app = express();

app.get('/test', (req, res) => res.send('Hello World!'));
app.listen(3001, function () {
    console.log('the server is listening:', 3001);
});
```

路由get的实现分为三板斧

```
1、express()   //会在app对象中添加应用处理方法
2、app.get('/test', (req, res) => {})  // 执行构建前端请求和后端处理程序的关联
3、createServer(app)  // 请求方法的执行，请求到来后，调用函数app()
```

### 具体实现

#### 1. express()

当`require('./express')`时，返回一个`createApplication`函数，再执行express()，实例化一个app对象，并且把`application.js`中的原型对象合并到app对象上。这里当执行`require('./application')`时，node已经对模块进行缓存，`express()`时，直接从缓存中拿。对应预备知识2

#### 2. app.get('/test', (req, res) => {})

app.get()会调用`application.js`中构建的请求方法`get`,具体代码如下。

```
methods.forEach(function (method) {
    app[method] = function (path) {
        this.lazyrouter();  //新建一个`router`对象
        let route = this._router.route(path); //新建一个route并添加到刚刚建立的router的stack中，
        这里实际上是调用的`router`函数
        route[method].apply(route, slice.call(arguments, 1)); //为route添加stack
        return this;
    }
});

```
this.lazyrouter();
这个方法会创建router对象，而这个对象会一直绑定到应用的_router属性上，创建的router对象在每个应用中只有一个,作用是处理每一个路由请求。<br>
注: `express.js`中的app对象和这里创建的router对象是结构极为类似(router对象本身是个函数，并且添加了一些属性)。

let route = this._router.route(path);


```
proto.route = function route(path) {
    var route = new Route();
    var layer = new Layer(path, {}, route.dispatch.bind(route));
    layer.route = route;
    this.stack.push(layer);
    return route;
}
```
先创建一个Route对象，创建layer对象，把layer对象的route属性指向Route对象，把新创建的layer对象放到router的stack中，返回route对象，这里的layer中handle属性是route.dispatch函数，这个函数的作用是通过next()获取stack中的每一个layer来执行对应的中间件，这样可以保证定义在路由上的中间件可以按照顺序依次执行。

let handle = slice.call(arguments, 1);
获取处理函数(数组的形式)

route[method].apply(route, handle);
获取处理函数，再次实例化layer对象，并且设置layer.method = get,然后把新创建的layer放到route对象的stack中。

这时路由和后端程序已经关联好了

最后总结一下:

router中的stack数组是有layer组成的，而route中的stack也是由layer组成的，区别是前者layer的fn是dispatch函数，用于遍历程序中的整个路由中间件，后者fn是真正的处理函数。

![总结](http://wx2.sinaimg.cn/large/e8616f3dgy1fmkrlwfyxoj20kx0b6dga.jpg)

#### 3. createServer(app)

请求来到后，实际上调用的是`application.js`中的handle函数(获取应用的_router属性，实际上是router对象)
----> 调用router.handle() ---->
根据请求的url和第二步中注册的路由路径判断是否是满足，
满足的话，调用相应layer对象中的handle对象进行调用

运行`npm start`，结果如图:

![图中的](../pictures/2_express01.png)