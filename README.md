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
