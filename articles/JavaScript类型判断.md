# JavaScript类型判断

> 本文主要讲解如何准确判断JavaScript中出现的各种类型和对象。（基本类型、`Object`类、`Window`对象、纯对象`plainObject`、类数组）其中部分参考了jQuery的函数实现。

## typeof

`typeof`只能用来判断基本类型，对于引用类型一律返回`object`。

`ES6`前基本类型有六个`Null`、`Undefined`、`Boolean`、`Number`、`String`、`Object`。

```javascript
typeof undefined // "undefined"
typeof null // "object"
typeof true // "boolean"
typeof 1 // "number"
typeof "s" // "string"
typeof {} // "object"
```

注意`typeof null === 'object'`其实是一个`bug`。

`typeof` 能够识别出`function`。

其中返回的字符串首字母都是小写的。

## Object.prototype.toString()

当`toString`方法被调用的时候，下面的步骤会被执行：

- 如果`this`值是`undefined`，就返回`[object Undefined]`
- 如果`this`的值是`null`，就返回`[object Null]`
- 让`O`成为`ToObject(this)`的结果
- 让`class`成为`O`的内部属性`[[Class]]`的值
- 最后返回由`"[object "`和`class`和`"]"`三个部分组成的字符串

该方法至少可以识别14种类型。

```javascript
// 以下是11种：
var number = 1;          // [object Number]
var string = '123';      // [object String]
var boolean = true;      // [object Boolean]
var und = undefined;     // [object Undefined]
var nul = null;          // [object Null]
var obj = {a: 1}         // [object Object]
var array = [1, 2, 3];   // [object Array]
var date = new Date();   // [object Date]
var error = new Error(); // [object Error]
var reg = /a/g;          // [object RegExp]
var func = function a(){}; // [object Function]

function checkType() {
    for (var i = 0; i < arguments.length; i++) {
        console.log(Object.prototype.toString.call(arguments[i]))
    }
}

checkType(number, string, boolean, und, nul, obj, array, date, error, reg, func)

// 还有不常见的Math、JSON
console.log(Object.prototype.toString.call(Math)); // [object Math]
console.log(Object.prototype.toString.call(JSON)); // [object JSON]

// 还有一个arguments
function a() {
    console.log(Object.prototype.toString.call(arguments)); // [object Arguments]
}
a();
```

## type API

> 结合上面我们可以写一个`type`函数，其中基本类型值走`typeof`，引用类型值走`toString`。

```javascript
var class2type = {};

// 生成class2type映射
"Boolean Number String Function Array Date RegExp Object Error".split(" ").map(function(item, index) {
    class2type["[object " + item + "]"] = item.toLowerCase();
})

function type(obj) {
    // 一箭双雕
    if (obj == null) {
        return obj + "";
    }
    return typeof obj === "object" || typeof obj === "function" ?
        class2type[Object.prototype.toString.call(obj)] || "object" :
        typeof obj;
}
```
通过`toLowerCase()`小写化和`typeof`的结果是小写一致。

注意IE6中`toString()`会把`Undefined`和`Null`都识别为`[object Object]`，所以加了一个判断，直接调用`+`来隐式`toString()-> "null"`。

这里之所以`class2type[Object.prototype.toString.call(obj)] || "object"`是考虑到`ES6`新增的`Symbol`、`Map`、`Set`在集合中没有，直接把他们识别成`object`。

**这个`type`其实就是`jQuery`中的`type`。**

## isFunction

之后可以直接封装：
```javascript
function isFunction(obj) {
    return type(obj) === "function";
}
```

## 数组

```javascript
var isArray = Array.isArray || function( obj ) {
    return type(obj) === "array";
}
```

jQuery3.0中已经完全使用`Array.isArray()`。

## plainObject

`plainObject`翻译为中文即为纯对象，所谓的纯对象，就是该对象是通过`{}`或`new Object()`创建的。

判断是否为“纯对象”，是为了和其他对象区分开比如说`null`、数组以及宿主对象（所有的`DOM`和`BOM`都是数组对象）等。

jQuery中有提供了该方法的实现，除了规定该对象是通过`{}`或`new Object()`创建的，且对象含有零个或者多个键值对外，一个没有原型(`__proto__`)的对象也是一个纯对象。

```javascript
console.log($.isPlainObject({})) // true

console.log($.isPlainObject(new Object)) // true

console.log($.isPlainObject(Object.create(null))); // true
```

jQuery3.0版本的`plainObject`实现如下：

```javascript
var toString = Object.prototype.toString;

var hasOwn = Object.prototype.hasOwnProperty;

function isPlainObject(obj) {
    var proto, Ctor;

    // 排除掉明显不是obj的以及一些宿主对象如Window
    if (!obj || toString.call(obj) !== "[object Object]") {
        return false;
    }

    /**
     * getPrototypeOf es5 方法，获取 obj 的原型
     * 以 new Object 创建的对象为例的话
     * obj.__proto__ === Object.prototype
     */
    proto = Object.getPrototypeOf(obj);

    // 没有原型的对象是纯粹的，Object.create(null) 就在这里返回 true
    if (!proto) {
        return true;
    }

    /**
     * 以下判断通过 new Object 方式创建的对象
     * 判断 proto 是否有 constructor 属性，如果有就让 Ctor 的值为 proto.constructor
     * 如果是 Object 函数创建的对象，Ctor 在这里就等于 Object 构造函数
     */
    Ctor = hasOwn.call(proto, "constructor") && proto.constructor;

    // 在这里判断 Ctor 构造函数是不是 Object 构造函数，用于区分自定义构造函数和 Object 构造函数
    return typeof Ctor === "function" && hasOwn.toString.call(Ctor) === hasOwn.toString.call(Object);
}
```

注意最后这一句非常的重要：
```javascript
hasOwn.toString.call(Ctor) === hasOwn.toString.call(Object)
```
`hasOwn.toString`调用的其实是`Function.prototype.toString()`而不是`Object.prototype.toString()`，因为`hasOwnProperty`是一个函数，它的原型是`Function`，于是`Function.prototype.toString`覆盖了`Object.prototype.toString`。

`Function.prototype.toString()`会把整个函数体转换成一个字符串。如果该函数是内置函数的话，会返回一个表示函数源代码的字符串。比如说：

`Function.prototype.toString(Object) === function Object() { [native code] }`

所以如果此时对象不是由内置构造函数生成的对象，这个`hasOwn.toString.call(Ctor) === hasOwn.toString.call(Object)`为`false`。

```javascript
function Person(name) {
    this.name = name;
}
var person = new Person("Devin");
plainObject(person) === false; // true
// 其实就是`hasOwn.toString.call(Ctor) === "function Person(name) { this.name = name; }"
```

## Window对象

`Window`对象有一个特性：`Window.window`指向自身。

```javascript
Window.window === Window; //true
```

## 类数组对象

常见的类数组有函数的`arguments`和`NodeList`对象。

### 判断

**对于类数组对象，只要该对象中存在`length`属性并且`length`为非负整数且在有限范围之内即可判断为类数组对象。**

JavaScript权威指南中提供了方法：
```javascript
function isArrayLike(o) {
    if (o && // o is not null, undefined, etc
        // o is an object
        typeof o === "object" &&
        // o.length is a finite number
        isFinite(o.length) &&
        // o.length is non-negative
        o.length >= 0 &&
        // o.length is an integer
        o.length === Math.floor(o.length) &&
        // o.length < 2^32
        o.length < 4294967296) //数组的上限值
      	return true;
    else 
      	return false;
}
```

以上的判断无论是真的数组对象或是类数组对象都会返回`true`，那我们如何区分到底是真的数组对象还是类数组对象？

其实只需要先判断是否为数组对象即可。

```javascript
function utilArray(o) {
    if (Array.isArray(o)) {
        return 'array';
    }
    if (isArrayLike(o)) {
        return 'arrayLike';
    } else {
        return 'neither array nor arrayLike';
    }
}
```

### 类数组对象的特征

**类数组对象并不关心除了数字索引和`length`以外的东西。**

比如说：
```javascript
var a = {"1": "a", "2": "b", "4": "c", "abc": "abc", length: 5};
Array.prototype.join.call(a, "+"); // +a+b++c
```

其中，`'0'`和`'3'`没有直接省略为两个`undefined`，同样的`abc`被忽略为`undefined`。

如果`length`多出实际的位数会补`undefined`（空位也补充`undefined`），少位则截断后面的数组成员。

```javascript
var a = {"1": "a", "2": "b", "4": "c", "abc": "abc", length: 6};
Array.from(a); // [undefined, "a", "b", undefined, "c", undefined]

var a = {"1": "a", "2": "b", "4": "c", "abc": "abc", length: 5};
Array.from(a); // [undefined, "a", "b", undefined, "c"]

var a = {"1": "a", "2": "b", "4": "c", "abc": "abc", length: 4};
Array.from(a); // [undefined, "a", "b", undefined]
```

### 类数组对象的转换

`Array.from`

该方法从一个类似数组或可迭代对象中创建一个新的数组实例。

```javascript
Array.from('foo');
// ["f", "o", "o"]
```

`Array.prototype.slice`

该方法返回一个从开始到结束（不包括结束）选择的数组的一部分**浅拷贝**到一个新数组对象。

```javascript
var a = {"0":"a", "1":"b", "2":"c", length: 3};
Array.prototype.slice.call(a, 0); // ["a", "b", "c"]
```

ES6扩展运算符

```javascript
var a = "hello";
[...a]; //["h", "e", "l", "l", "o"]
```

--- 

参考链接：

https://github.com/mqyqingfeng/Blog/issues/28

https://github.com/mqyqingfeng/Blog/issues/30
