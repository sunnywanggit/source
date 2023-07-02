  

`jQuery` 曾经是前端里使用最广泛的库, 因为它有几个优点：

  

> TIP

> 1.浏览器兼容性好

> 2.封装了复杂, 不直观的 `DOM API`

> 3.封装 `Ajax`

  
  

好用是它的一个优点, 但深入下去会发现它的整体设计也是相当优秀, 尤其是 `API` 设计， `$()`, 链式调用. 我私下也实现了一个自己的 `jQuery`, 目前只封装了一些 `DOM API`,  在这里大概展开说一下

  

## `$()` 是选择器吗

  

首先需要说明的是，`$()`不是选择器, 事实上它是一个函数名为`$`的函数, 实际上我们是在把选择器或者元素当做参数传入这个函数当中, 比如`$('.el').`

  

## 链式调用是怎么实现的

  

比如要给一个`div`添加类名, `jQuery`会提供这样的写法: `$('div').addClass('xxx').` 。

  

这样的写法就是链式调用, 它的实现方法是怎样的?

  

之前说到`$()`是一个函数, 这个函数返回了一个对象, 对象里包含了各种功能函数。

  

`addClass()`就是其中一个， 除此之外, 我们需要获取到我们要操作的元素, 所以我们会在`$`函数里声明一个局部变量, 然后用传进的参数赋值作为我们要操作的元素.

  

```javascript

window.$ = function(stuff) {

  let elements;

  elements = document.querySelectorAll(stuff); // elements 是一个闭包

  return {

    addClass(className) {

      elements.forEach(el => {

        el.classList.add(className);

      });

      return this; // this 就是这个对象, 此段代码是为了可以继续使用对象进行下一个链式操作

    }

  };

};

```

  

## 获取元素

  

`$(stuff)`可以返回一个操作`stuff`的对象, 对象里包含各种各样的封装 `DOM` 函数。

  

目前`stuff`只支持传入 `CSS` 选择器和 `HTML` 标签, 在此之上, 我添加了一个功能, `$('<div>1</div>')`可以根据传入的 `HTML` 字符串来生成 `HTML` 元素.

  

```javascript

window.$ = function(stuff) {

  let elements;

  if (typeof elements === string) {

    if (stuff.indexOf("<") === 0) {

      const template = document.createElement("template");

      template.innerHTML = stuff.trim();

      elements = [template.content.firstChild];

    } else {

      elements = document.querySelectorAll(stuff);

    }

  }

};

```


## 原型链


如果每次生成的 `$` 实例里的方法都一样, 但还是重新开辟内存空间去容纳两个一样的对象，这会显得很愚蠢.

原型链也是为了解决这个问题而产生的. `$` 将实例所用到的函数放在`$.prototype`上, 每个实例只需保存一个对`$.prototype`的引用即可节省很大的内存。

```javascript

// 在window.$() 里

const api = Object.create($.prototype);

Object.assign(api, { elements, oldApi: stuff.oldApi });

  

// 在window.$() 外

$.prototype = {

  constructor: $,

  $: true

  // methods...

};

```


## `find`函数实现

  

`find`函数的功能是寻找出和传入参数一致的元素.

  

```javascript

find(selector) {

    let array = [];

    this.each(element => {

      array.push(...Array.from(element.querySelectorAll(selector)));

    });

    array.oldApi = this; // 与end函数实现有关

    return $(array);

  }

```

  

首先, 创建一个空数组, 遍历元素将合适的选择器们组成的 `NodeList` 转换为 `Array` 再一个个得 `push` 到空数组里, 这样就能得到一个符合需求的选择器数组。

  

那么, 为了使用链式操作, 我们会自然而然地`return this`, 但这样是不 `work` 的, 因为`this`指向的对象所操作的`elements`不是我们的选择器数组, 而是之前`$(stuff)`所生成的`elements`, 我们是操作不了选择器数组的. 可直接返回`array`又得不到我们的封装 `DOM` 函数, 这也行不通. 所以此时将数组传入`$()`里, 改写`elements`的赋值, 便能解决问题. 在此之前, 我们还要对`$()`内部进行修改

  

```javascript

window.$ = function(stuff) {

  let elements;

  if (typeof stuff === "string") {

    if (stuff.indexOf("<") === 0) {

      const template = document.createElement("template");

      template.innerHTML = stuff.trim();

      elements = [template.content.firstChild];

    } else {

      elements = document.querySelectorAll(stuff);

    }

  } else if (stuff instanceof Array) {

    elements = stuff;

  }

  const api = Object.create($.prototype);

  Object.assign(api, { elements, oldApi: stuff.oldApi });

  return api;

};

```

  

## `end`函数实现

  

`end`函数返回链式调用里的前一个`$`对象. 如果有这样一行代码: `$(".parent").find(".child").addClass('green').end().addClass("blue");`

  

它将在`.parent`里寻找`.child`并添加`green`类名, 然后返回到`.parent`并给它添加`blue`类名。

  

`end`函数做了些什么?

  

我们要搞清楚调用关系, 是谁调用了`end`函数, 在例子里可以知道是`find`函数返回了`$(array)`, 而`$(array)`返回了一个对象, 就是它调用了`end`函数. 再从例子里看, `end`函数执行后是返回了`$('.parent')`.

  

所以, 我们把`$(.parent)`保存了下来, 放在`find`函数的`array.oldApi`里, 我们在调用`$(array)`时, 它会生成`api`对象, `api`对象里有一个`oldApi`属性, 它便会读取`array.oldApi`的值作为`api.oldApi`的值, `end`函数就只需要返回`oldAPi`即可.

  

```javascript

end() {

    return this.oldApi;

  }

```