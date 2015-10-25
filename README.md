# underscore.js
Read and Comment on source code of underscore.js

受 [这篇文章](http://web.jobbole.com/83872/) 的启发,萌生阅读 underscore.js 源
码的念头,其中有许多不理解的地方,也是读了上述文章后才明白的.为了保持本文的完整
性,也尽量按自己的理解进行注释. 不再提及上述引用文章.

# 源码阅读

```js
(function(){
    ...
}())
```
underscore.js 中通过自执行函数来防止打乱已有的命令空间中的变量.这样函数中定义
的所有变量在外部都是不可见的.但是仍旧需要以某种方式来导出其中定义的变量.

```js
// Establish the root object, `window` (`self`) in the browser, `global`
// on the server, or `this` in some virtual machines. We use `self`
// instead of `window` for `WebWorker` support.
var root = typeof self == 'object' && self.self === self && self ||
        typeof global == 'object' && global.global === global && global ||
        this;
```

`root` 变量的作用是用来捕捉外部环境. 由于在自执行函数中,`this` 变量会被设置成
`Window` (浏览器中),所以我们可能通过为 `this` (即此处的`root`) 添加相应的变量
来导出函数. 如:

```js
(function() {
    this.exported_var = 10
}())

console.log(this.exported_var);
// => 10
```


```js
// Save bytes in the minified (but not gzipped) version:
var ArrayProto = Array.prototype, ObjProto = Object.prototype;
```

为了减少 JS 代码在网络传输中占用的流量,通常要对其进行压缩,以减少源代码的大
小.方法之一是替换现有的变量名.将 `Array.prototype` 赋值给新变量,就允许我们对
该变量进行重命名.例如: `ArrayProto.toString => a.toString` 而若使用诸如
`Array.prototype.toString => a.prototype.toString` 则找不到该函数.

```js
  // Create quick reference variables for speed access to core prototypes.
  var
    push = ArrayProto.push,
    slice = ArrayProto.slice,
    toString = ObjProto.toString,
    hasOwnProperty = ObjProto.hasOwnProperty;

  // All **ECMAScript 5** native function implementations that we hope to use
  // are declared here.
  var
    nativeIsArray = Array.isArray,
    nativeKeys = Object.keys,
    nativeCreate = Object.create;
```
以上同理.

```
  // Naked function reference for surrogate-prototype-swapping.
  var Ctor = function(){};
```

`Ctor` 函数只有一个用途,就是为了兼容老版本 JavaScript 的继承,即用来实现
`Object.create` 函数.

```
SubClass.prototype = Object.create(SuperClass.prototype)
// 等价于
var ctor = function () {}
ctor.prototype = SuperClass.prototype;
SubClass.prototype = new ctor();
```

`Ctor` 在之后的 `baseCreate` 函数中使用.

## 链式调用

因为涉及的内容较多,所以归成一节.

首先,我们要明白什么是链式调用.简单地说,链式调用是方便我们写代码的一个手段,看
下面的例子:

```
var x = obj.method_O();
var y = x.method_X();
var z = y.method_Y();
z.method_Z();
```

上述写法需要许多中间变量,由于对象 `obj` 的 `method_O` 方法正好返回一个类 `X`
的对象(这里指的是返回的变量 `x` 需要有 `method_X` 方法),所以可以直接调用 `X` 的方
法 `method_X()`. 以此类推.因此我们可以省略其中
的中间变量,写成:

```
obj.method_O().method_X().method_Y().method_Z();
```

要达到上述效果,我们便需要让 `method_O()` 方法在结束时返回一个类 `X` 的对象:

```js
function method_O() {
    ...
    var ret = new X(); // 创建一个 `X` 的对象返回
    一些逻辑处理
    ...

    return ret;
}
```

以此类推.上述方法是 Javascript 原生支持的.现在的问题在于,例如调用 `method_X`
方法返回了 `Y` 的对象,就再也无法使用类 `X` 中的方法了.例如:

```
var flattened_obj = _([[1,2]]).flatten();
flattened_obj.each(...) // 出错
```

上述代码中我们首先创建了一个 underscore.js 的对象 `_([[1,2]])` 目的是使用
underscore.js 为我们提供的丰富辅助函数.之后我们调用 underscore.js 中的
`flatten` 函数得到一个扁平化的数组: `[1,2]`. 之后我们想在其中调用
underscore.js 的 `each` 函数. 此时报错,提示没有该函数.故此时我们无法使用链式
调用:

```
_([[1,2]]).flatten().each(...) // 报错
```

故而 underscore.js 需要提供一些机制来包裹返回的对象,使之能访问 underscore.js
中的函数.

underscore.js 中通过 `_.chain(obj)` 来返回一个包裹的 `_` 对象;再对
underscore.js 中提供的所有函数做特殊的处理,使得:当调用函数的是包裹的对象时,返
回的结果也是一个 `_` 的对象,而由于 underscore.js 中的所有函数都存放在 `_`
中,所以调用链中的每一步都可以访问 underscore.js 中的函数.

例如:

```
_.chain([[1,2]]) instanceof _; // => true
_.chain([[1,2]]).flatten() instanceof _; // => true
_([[1,2]]).flatten() instanceof _; // => false
```

### 链式调用的实现

```
  // Create a safe reference to the Underscore object for use below.
  var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
  };

  // Add a "chain" function. Start chaining a wrapped Underscore object.
  _.chain = function(obj) {
    var instance = _(obj);
    instance._chain = true;
    return instance;
  };
```

从上面的函数可以看到 `_` 函数生成一个新的 `_` 对象,并将输入的 `obj` 置于
`this._wrapped` 中. 而 `_.chain` 函数则再设置 `this._chain = true` 的标志.

单凭上述两个函数并没有实际用途,因此需要一个辅助函数:

```
  // Helper function to continue chaining intermediate results.
  var chainResult = function(instance, obj) {
    return instance._chain ? _(obj).chain() : obj;
  };
```

该函数检查 `instacne` 本身是否设置了 `_chain` 标志,若是则将 `obj` 用 `chain()`
包裹,它的作用就是对调用链上函数返回的结果进行处理,如 `x.method()` 中,若
设置了 `_chain` 标志,则将 `x.method()` 的返回结果再用 `chain()` 包裹.这样调用
链中的每个函数返回的都是一个 `_` 的对象,因此也就能继续访问类 `_` 的方法了.

还有一个问题是,即使有以上函数, underscore.js 在定义新的函数时仍需手工调用
`chainResult` 函数,十分麻烦. 所以 underscore.js 又提供了另一个辅助函数,将所有
已有的函数进行包裹:

```
  // Add your own custom functions to the Underscore object.
  _.mixin = function(obj) {
    _.each(_.functions(obj), function(name) {
      var func = _[name] = obj[name];
      _.prototype[name] = function() {
        var args = [this._wrapped];
        push.apply(args, arguments);
        return chainResult(this, func.apply(_, args));
      };
    });
  };
  
  // Add all of the Underscore functions to the wrapper object.
  _.mixin(_);
```

该函数将 `obj` 中的所有函数替换成包裹后的函数.首先取出 `_` 对象中包裹的实际
值, `push.apply(args, arguments)` 将该值与现有的函数参数结合,最后对原函数的返
回值进行处理: `chainResult(this, func.apply(_, args))`.

还有一些函数单独作了处理,如 `pop`, `push`, `reverse`, 等等,此处不再详谈.
