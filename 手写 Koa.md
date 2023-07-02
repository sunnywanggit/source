  Koa 是一个基于 Node.js的Web开发框架，特点是小而精，对比大而全的 Express，两者虽然由同一团队开发，但各有其更适合的应用场景：Express 适合开发较大的企业级应用，而Koa致力于成为Web开发中的基石，例如Egg.js就是基于Koa开发的。

  
```js

  const Koa = require('koa');

  const app = new Koa();

  

  app.use((ctx, next) => {

    ctx.body = 'Hello World';

  });

  

  app.listen(3000);
```

  

  原生的写法：

  

```js

  let http = require('http')

  

  let server = http.createServer((req, res) => {

    res.end('hello world')

  })

  

  server.listen(4000)

```

  

  简单来说 Koa 其实就是在原生的写法上面进行了一层封装，对比于原生的写法，Koa多了两个实例上的use、listen方法，和use回调中的ctx、next两个参数。这四个不同，几乎就是Koa的全部了，也是这四个不同让Koa如此强大。

  # 1. api 实现

  ## 1.1 listen

  简单！http的语法糖，实际上还是用了http.createServer()，然后监听了一个端口。

  ## 1.2 ctx

  比较简单！利用 **上下文(context)** 机制，将原来的req,res对象合二为一，并进行了大量拓展,使开发者可以方便的使用更多属性和方法，大大减少了处理字符串、提取信息的时间，免去了许多引入第三方包的过程。(例如ctx.query、ctx.path等)

  ## 1.3 use

  **重点**！Koa的核心 —— **中间件（middleware）**。解决了异步编程中回调地狱的问题，基于Promise，利用 **洋葱模型** 思想，使嵌套的、纠缠不清的代码变得清晰、明确，并且可拓展，可定制，借助许多第三方中间件，可以使精简的koa更加全能（例如koa-router，实现了路由）。其原理主要是一个极其精妙的 **compose** 函数。在使用时，用 **next()** 方法，从上一个中间件跳到下一个中间件。

  # 2. Koa 原理梳理：

  

  - Koa 的原理无非就是use的时候拿到一个回调函数，listen的时候执行这个函数

  - 此外，use回调函数的参数ctx拓展了很多功能，这个ctx其实就是原生的req、res经过一系列处理产生的

  - 其实，第一句不准确，use可以多次，所以是多个回调函数，用户第二个参数next()跳到下一个，把多个use的回调函数按照规则顺序执行。

  - 那么，看起来就很简单了，难点只有两个：一个是如何将原生req和res加工成ctx，另一个是如何实现中间件。

  - 第一个，ctx其实就是一个上下文对象，request和response两个文件用来拓展属性，context文件实现代理，我们会手写相关源码。

  - 第二个，源码中的中间件由一个中间件执行模块koa-compose实现，这里我们会手写一个。

  

  # 3. 中间件实现原理

  async await 是 promise 的语法糖，await 后面跟一个promise，所以代码可以写成：改成这样更好理解一些，所以 [流程控制](https://www.zhihu.com/search?q=%E6%B5%81%E7%A8%8B%E6%8E%A7%E5%88%B6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22article%22%2C%22sourceId%22%3A141890366%7D) 的核心在于 next 的实现。

  

```jsx

  const middleware = function (ctx, next) {

    console.log(1)

    next().then(() => {

      console.log(6)

    })

  }

  

  const middleware2 = function (ctx, next) {

    console.log(2)

    next().then(() => {

      console.log(5)

    })

  }

  

  const middleware3 = function (ctx, next) {

    console.log(3)

    next().then(() => {

      console.log(4)

    })

  }
```

  

  我们先看一下中间件的执行流程：

  

  - 第一步：让多个 use 的回调按照顺序排列成串。

  

  这里用到了数组和递归，每次use将当前函数存到一个数组中，最后按顺序执行。执行这一步用到一个**compose**函数，这个函数是重中之重。

  

  - 第二步，把每个回调包装成 Promise 以实现异步。

  - 最后一步，用 Promise.resolve 将每个回调包装成 Promise，并在调用时 then。

  

  next 要求调用队列中下一个 middleware，当达到最后一个的时候resolve。这样最后面的promise先resolve，一直到第一个，这样就是洋葱模型的顺序了。

  

  koa-compose 的实现是这样的：

  

```jsx

  function compose(middleware) {

    return function (context, next) {

      let index = -1

      return dispatch(0)

  

      function dispatch(i) {

        index = i

        let fn = middleware[i]

        if (i === middleware.length) fn = next

        if (!fn) return Promise.resolve()

        try {

          return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))

        } catch (err) {

          return Promise.reject(err)

        }

      }

    }

  }

```

  

  每次传入的next都是调用下一个middleware，这样是一个递归的过程，结束条件是最后一个middleware 的next是用户传入的。

  

  这里面有一些亮点：

  

  1. 这是一种尾递归的形式，尾递归的特点是最后返回的值是一个递归的函数调用，这样执行完就会在调用栈中销毁，不会占据调用栈.

  2. 返回的是一个 Promise.resolve 包装之后的调用，而不是同步的调用，所以这是一个异步递归，异步递归比同步递归的好处是可以被打断，如果中间有一些优先级更高的微任务，那么可以先执行别的微任务

  3. compose 是函数复合，把 n个 middleware复合成一个，参数依然是context和next，这种复合之后依然是一个middleware，还可以继续进行复合。

  

  # 4. 总结

  

  Koa 中间件的实现原理，也就是洋葱模型的实现原理，核心在于next的实现。next需要依次调用下一个middleware，当到最后一个的时候结束，这样后面middleware的promise先resolve，然后直到第一个，这样的流程也就是洋葱模型的流程了。

  

  实现的时候还有一些细节，一个是递归最好做成尾递归的形式，而是用异步递归而不是同步递归，第三就是形式上用函数复合的形式，这样复合之后的中间件还可以继续复合。


```javascript

  constructor () {

    super()

    // this.fn 改成：

    this.middlewares = [] // 需要一个数组将每个中间件按顺序存放起来

    this.context = context

    this.request = request

    this.response = response

  }

  

  use (fn) {

    // this.fn = fn 改成：

    this.middlewares.push(fn) // 每次use，把当前回调函数存进数组

  }

  

  compose(middlewares, ctx){

    function dispatch(index){

      if(index === middlewares.length) return Promise.resolve() // 若最后一个中间件，返回一个resolve的promise

      let middleware = middlewares[index]

      return Promise.resolve(middleware(ctx, () => dispatch(index + 1))) // 用Promise.resolve把中间件包起来

    }

    return dispatch(0)

  }

  

  handleRequest(req,res){

    res.statusCode = 404

    let ctx = this.createContext(req, res)

    let fn = this.compose(this.middlewares, ctx)

    fn.then(() => { // then了之后再进行判断

      if(typeof ctx.body == 'object'){

        res.setHeader('Content-Type', 'application/json;charset=utf8')

        res.end(JSON.stringify(ctx.body))

      } else if (ctx.body instanceof Stream){

        ctx.body.pipe(res)

      }

      else if (typeof ctx.body === 'string' || Buffer.isBuffer(ctx.body)) {

        res.setHeader('Content-Type', 'text/htmlcharset=utf8')

        res.end(ctx.body)

      } else {

        res.end('Not found')

      }

    }).catch(err => { // 监控错误发射error，用于app.on('error', (err) =>{})

      this.emit('error', err)

      res.statusCode = 500

      res.end('server error')

    })

  }
```

  

  参考文章：

  

  - [https://mp.weixin.qq.com/s/GyOHpz5bjM7kEmuNALGBeg](https://mp.weixin.qq.com/s/GyOHpz5bjM7kEmuNALGBeg)