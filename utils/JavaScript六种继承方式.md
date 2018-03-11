### 借用原型链继承

核心是子类构造函数的prototype绑定到父类的实例上。

存在的问题是父类的引用类型属性会被子类的实例共享、传递参数给父类构造函数会影响子类的所有实例。

```javascript
function Super() {
    this.name = 'super';
    this.colors = ['red', 'blue', 'green'];
}

Super.prototype.sayName = function() {
    console.log(this.name);
};

function Sub() {
    this.name = 'Sub';
}

// 原型链绑定
Sub.prototype = new Super();
// 修正constructor
Sub.prototype.constructor = Sub;

Sub.prototype.sayHello = function() {
    console.log('sayHello');
};

var sub = new Sub();
sub.colors.push('yello');

var _sub = new Sub();
console.log(_sub.colors);
```

### 借用构造函数继承

核心是在子类的构造函数中对父类构造函数调用`call`。 

存在问题是没有进行原型链的连接，父类原型上的方法没有办法被继承。

优点是解决了原型链继承存在的父类引用型属性被共享以及无法传入参数给构造函数的缺点。

```javascript
function Super(name) {
    this.name = name;
    this.colors = ['red', 'blue', 'green'];
}

function Sub(name) {
    this.name = name;
    Super.call(this, 'Super');
}

var sub = new Sub('sub');
sub.colors.push('yellow');
console.log(sub.colors);

var _sub = new Sub('_sub');
console.log(_sub.colors);
```

### 组合继承

核心：结合构造函数和原型链继承，对于方法的继承使用原型链继承，对于父类的属性继承使用构造函数继承。

存在问题是父类的构造函数被调用两次，会发生属性屏蔽。

```javascript
function Super(name) {
    this.name = name;
    this.colors = ['red', 'blue', 'green'];
}

Super.prototype.sayName = function() {
    console.log(this.name);
};

function Sub(name) {
    Super.call(this);
    this.name = name;
}

Sub.prototype = new Super('super');
Sub.prototype.constructor = Sub;
Sub.prototype.sayHello = function() {
    console.log('Hello');
};

var sub = new Sub('sub');
sub.sayName();
// 发现父类上也有一个colors属性

var _sub = new Sub('_sub');
_sub.sayName();
```

### 原型继承

核心：实质就是__proto__连接，不使用父类构造函数，而是基于已有对象进行继承。

```javascript
var Super = {
    name: 'super',
    colors: ['red', 'blue', 'green']
};

function object(o) {
    function f() {}
    f.prototype = o;
    return new f();
}

var sub = object(Super);
sub.name = 'sub';
console.log(sub);
```

### 寄生式继承

 核心：就是在原型继承的基础上，对对象的增强。

```javascript
var Super = {
    name: 'super',
    colors: ['red', 'blue', 'green']
};

function clone(o) {
    var _o = object(o);
    _o.sayHi = function() {
        console.log('Hi');
    };
    return _o;
}

var sub = clone(Super);
console.log(sub);
```

### 寄生式组合继承

核心：子类的原型不绑定在父类构造函数的实例上，而是直接绑定在父类的原型上。然后在子类构造函数中调用call去继承父类构造函数的属性。

优点是解决寄生组合存在的属性屏蔽问题。 

注：  
原型链继承 `Sub.prototype = new Super(); // (new Sub()).__proto__.__proto__ === Super.prototype`  
原型继承 `Sub.prototype = Super.prototype; // (new Sub()).__proto__ === Super.prototype`
```javascript
function Super(name) {
    this.name = name;
    this.colors = ['red', 'blue', 'green'];
}
Super.prototype.sayName = function() {
    console.log(this.name);
};

function Sub(name) {
    // 借用构造函数继承
    Super.call(this);
    this.name = name;
}

// 原型继承
Sub.prototype = Super.prototype;
Sub.prototype.constructor = Sub;
// 上面两句代码相当于
// Sub.prototype = Object.create(Super.prototype);
// Sub.prototype.constructor = Sub

var sub = new Sub('sub');
console.log(sub);
```
