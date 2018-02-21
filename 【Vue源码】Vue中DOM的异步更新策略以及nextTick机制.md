# 【Vue源码】Vue中DOM的异步更新策略以及nextTick机制

> 本篇文章主要是对`Vue`中的`DOM`异步更新策略和`nextTick`机制的解析，需要有一定的`Vue`使用经验和熟悉掌握JavaScript事件循环模型。

### 引入：DOM的异步更新

```html
<template>
  <div>
    <div ref="test">{{test}}</div>
    <button @click="handleClick">tet</button>
  </div>
</template>
```

```javascript
export default {
    data () {
        return {
            test: 'begin'
        };
    },
    methods () {
        handleClick () {
            this.test = 'end';
            console.log(this.$refs.test.innerText);//打印“begin”
        }
    }
}
```

打印的结果是`begin`而不是我们设置的`end`。这个结果足以说明`Vue`中`DOM`的更新并非同步。

在`Vue`官方文档中是这样说明的：

> 可能你还没有注意到，`Vue`异步执行`DOM`更新。只要观察到数据变化，`Vue`将开启一个队列，并缓冲在同一事件循环中发生的所有数据改变。如果同一个`watcher`被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和`DOM`操作上非常重要。然后，在下一个的事件循环“`tick`”中，`Vue`刷新队列并执行实际 (已去重的) 工作。

简而言之，就是在一个事件循环中发生的所有数据改变都会在下一个事件循环的`Tick`中来触发视图更新，这也是一个“批处理”的过程。（注意下一个事件循环的`Tick`有可能是在当前的`Tick`微任务执行阶段执行，也可能是在下一个`Tick`执行，主要取决于`nextTick`函数到底是使用`Promise/MutationObserver`还是`setTimeout`）

### Watcher队列

在`Watcher`的源码中，我们发现`watcher`的`update`其实是异步的。（注：`sync`属性默认为`false`，也就是异步）

```javascript
update () {
    /* istanbul ignore else */
    if (this.lazy) {
        this.dirty = true
    } else if (this.sync) {
        /*同步则执行run直接渲染视图*/
        this.run()
    } else {
        /*异步推送到观察者队列中，下一个tick时调用。*/
        queueWatcher(this)
    }
}
```

`queueWatcher(this)`函数的代码如下：

```javascript
 /*将一个观察者对象push进观察者队列，在队列中已经存在相同的id则该观察者对象将被跳过，除非它是在队列被刷新时推送*/
export function queueWatcher (watcher: Watcher) {
    /*获取watcher的id*/
    const id = watcher.id
    /*检验id是否存在，已经存在则直接跳过，不存在则标记哈希表has，用于下次检验*/
    if (has[id] == null) {
        has[id] = true
        if (!flushing) {
            /*如果没有flush掉，直接push到队列中即可*/
            queue.push(watcher)
        } else {
        ...
        }
        // queue the flush
        if (!waiting) {
            waiting = true
            nextTick(flushSchedulerQueue)
        }
    }
}
```

这段源码有几个需要注意的地方：

1. 首先需要知道的是`watcher`执行`update`的时候，默认情况下肯定是异步的，它会做以下的两件事：
    - 判断`has`数组中是否有这个`watcher`的`id`
    - 如果有的话是不需要把`watcher`加入`queue`中的，否则不做任何处理。

2. 这里面的`nextTick(flushSchedulerQueue)`中，`flushScheduleQueue`函数的作用主要是执行视图更新的操作，它会把`queue`中所有的`watcher`取出来并执行相应的视图更新。

3. 核心其实是`nextTick`函数了，下面我们具体看一下`nextTick`到底有什么用。

### nextTick

`nextTick`函数其实做了两件事情，一是生成一个`timerFunc`，把回调作为`microTask`或`macroTask`参与到事件循环中来。二是把回调函数放入一个`callbacks`队列，等待适当的时机执行。（这个时机和`timerFunc`不同的实现有关）

首先我们先来看它是怎么生成一个`timerFunc`把回调作为`microTask`或`macroTask`的。

```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    /*使用Promise*/
    var p = Promise.resolve()
    var logError = err => { console.error(err) }
    timerFunc = () => {
        p.then(nextTickHandler).catch(logError)
        // in problematic UIWebViews, Promise.then doesn't completely break, but
        // it can get stuck in a weird state where callbacks are pushed into the
        // microTask queue but the queue isn't being flushed, until the browser
        // needs to do some other work, e.g. handle a timer. Therefore we can
        // "force" the microTask queue to be flushed by adding an empty timer.
        if (isIOS) setTimeout(noop)
    }
} else if (typeof MutationObserver !== 'undefined' && (
    isNative(MutationObserver) ||
    // PhantomJS and iOS 7.x
    MutationObserver.toString() === '[object MutationObserverConstructor]'
    )) {
    // use MutationObserver where native Promise is not available,
    // e.g. PhantomJS IE11, iOS7, Android 4.4
    /*新建一个textNode的DOM对象，用MutationObserver绑定该DOM并指定回调函数，在DOM变化的时候则会触发回调,该回调会进入主线程（比任务队列优先执行），即textNode.data = String(counter)时便会触发回调*/
    var counter = 1
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(String(counter))
    observer.observe(textNode, {
        characterData: true
    })
    timerFunc = () => {
        counter = (counter + 1) % 2
        textNode.data = String(counter)
    }
} else {
    // fallback to setTimeout
    /* istanbul ignore next */
    /*使用setTimeout将回调推入任务队列尾部*/
    timerFunc = () => {
        setTimeout(nextTickHandler, 0)
    }
}
```

值得注意的是，它会按照`Promise`、`MutationObserver`、`setTimeout`优先级去调用传入的回调函数。前两者会生成一个`microTask`任务，而后者会生成一个`macroTask`。（微任务和宏任务）

之所以会设置这样的优先级，主要是考虑到浏览器之间的兼容性（`IE`没有内置`Promise`）。另外，设置`Promise`最优先是因为`Promise.resolve().then`回调函数属于一个**微任务**，浏览器在一个`Tick`中执行完`macroTask`后会清空当前`Tick`所有的`microTask`再进行`UI`渲染，把`DOM`更新的操作放在`Tick`执行`microTask`的阶段来完成，相比使用`setTimeout`生成的一个`macroTask`会少一次`UI`的渲染。

而`nextTickHandler`函数，其实才是我们真正要执行的函数。

```javascript
function nextTickHandler () {
    pending = false
    /*执行所有callback*/
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}
```

这里的`callbacks`变量供`nextTickHandler`消费。而前面我们所说的`nextTick`函数第二点功能中“等待适当的时机执行”，其实就是因为`timerFunc`的实现方式有差异，如果是`Promise\MutationObserver`则`nextTickHandler`回调是一个`microTask`，它会在当前`Tick`的末尾来执行。如果是`setTiemout`则`nextTickHandler`回调是一个`macroTask`，它会在下一个`Tick`来执行。

还有就是`callbacks`中的成员是如何被`push`进来的？从源码中我们可以知道，`nextTick`是一个自执行的函数，一旦执行是`return`了一个`queueNextTick`，所以我们在调用`nextTick`其实就是在调用`queueNextTick`这个函数。它的源代码如下：

```javascript
return function queueNextTick (cb?: Function, ctx?: Object) {
    let _resolve
    /*cb存到callbacks中*/
    callbacks.push(() => {
        if (cb) {
            try {
            cb.call(ctx)
            } catch (e) {
            handleError(e, ctx, 'nextTick')
            }
        } else if (_resolve) {
            _resolve(ctx)
        }
    })
    if (!pending) {
        pending = true
        timerFunc()
    }
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise((resolve, reject) => {
            _resolve = resolve
        })
    }
}
```

可以看到，一旦调用`nextTick`函数时候，传入的`function`就会被存放到`callbacks`闭包中，然后这个`callbacks`由`nextTickHandler`消费，而`nextTickHandler`的执行时间又是由`timerFunc`来决定。

我们再回来看`Watcher`中的一段代码：

```javascript
 /*将一个观察者对象push进观察者队列，在队列中已经存在相同的id则该观察者对象将被跳过，除非它是在队列被刷新时推送*/
export function queueWatcher (watcher: Watcher) {
  /*获取watcher的id*/
  const id = watcher.id
  /*检验id是否存在，已经存在则直接跳过，不存在则标记哈希表has，用于下次检验*/
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
        /*如果没有flush掉，直接push到队列中即可*/
        queue.push(watcher)
    } else {
      ...
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```
这里面的`nextTick(flushSchedulerQueue)`中的`flushSchedulerQueue`函数其实就是`watcher`的视图更新。每次调用的时候会把它`push`到`callbacks`中来异步执行。

另外，关于`waiting`变量，这是很重要的一个标志位，它保证`flushSchedulerQueue`回调只允许被置入`callbacks`一次。

**所以，也就是说`DOM`确实是异步更新，但是具体是在下一个`Tick`更新还是在当前`Tick`执行`microTask`的时候更新，具体要看`nextTcik`的实现方式，也就是具体跑的是`Promise/MutationObserver`还是`setTimeout`。**

附：[`nextTick`源码带注释]((https://github.com/answershuto/learnVue/blob/master/docs/Vue.js%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0DOM%E7%AD%96%E7%95%A5%E5%8F%8AnextTick.MarkDown#nexttick))，有兴趣可以观摩一下。

这里面使用`Promise`和`setTimeout`来执行异步任务的方式都很好理解，比较巧妙的地方是利用`MutationObserver`执行异步任务。`MutationObserver`是`H5`的新特性，它能够监听指定范围内的`DOM`变化并执行其回调，它的回调会被当作`microTask`来执行，具体参考[`MDN`](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)，。

```javascript
var counter = 1
var observer = new MutationObserver(nextTickHandler)
var textNode = document.createTextNode(String(counter))
observer.observe(textNode, {
    characterData: true
})
timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
}
```

可以看到，通过借用`MutationObserver`来创建一个`microTask`。`nextTickHandler`作为回调传入`MutationObserver`中。  
这里面创建了一个`textNode`作为观测的对象，当`timerFunc`执行的时候，`textNode.data`会发生改变，便会触发`MutatinObservers`的回调函数，而这个函数才是我们真正要执行的任务，它是一个`microTask`。

注：`2.5+`版本的`Vue`对`nextTick`进行了修改，具体参考下面“版本升级”一节。

### 关于Vue暴露的全局`nextTick`

继续来看下面的这段代码：

```html
<div id="example">
    <div ref="test">{{test}}</div>
    <button @click="handleClick">tet</button>
</div>
```
```javascript
var vm = new Vue({
    el: '#example',
    data: {
        test: 'begin',
    },
    methods: {
        handleClick() {
            this.test = 'end';
            console.log('1')
            setTimeout(() => { // macroTask
                console.log('3')
            }, 0);
            Promise.resolve().then(function() { //microTask
                console.log('promise!')
            })
            this.$nextTick(function () {
                console.log('2')
            })
        }
    }
})
```

在`Chrome`下，这段代码执行的顺序的`1、2、promise、3`。

可能有同学会以为这是`1、promise、2、3`，其实是忽略了一个标志位`pending`。

我们回到`nextTick`函数`return`的`queueNextTick`可以发现：
```javascript
return function queueNextTick (cb?: Function, ctx?: Object) {
    let _resolve
    /*cb存到callbacks中*/
    callbacks.push(() => {
        if (cb) {
        try {
            cb.call(ctx)
        } catch (e) {
            handleError(e, ctx, 'nextTick')
        }
        } else if (_resolve) {
        _resolve(ctx)
        }
    })
    if (!pending) {
        pending = true
        timerFunc()
    }
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise((resolve, reject) => {
        _resolve = resolve
        })
    }
}
```
这里面通过对`pending`的判断来检测是否已经有`timerFunc`这个函数在事件循环的任务队列等待被执行。如果存在的话，那么是不会再重复执行的。

最后异步执行`nextTickHandler`时又会把`pending`置为`false`。

```javascript
function nextTickHandler () {
    pending = false
    /*执行所有callback*/
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}
```

所以回到我们的例子：
```javascript
handleClick() {
    this.test = 'end';
    console.log('1')
    setTimeout(() => { // macroTask
        console.log('3')
    }, 0);
    Promise.resolve().then(function() { //microTask
        console.log('promise!')
    })
    this.$nextTick(function () {
        console.log('2')
    })
}
```

代码中，`this.test = 'end'`必然会触发`watcher`进行视图的重新渲染，而我们在文章的`Watcher`一节中就已经有提到会调用`nextTick`函数，一开始`pending`变量肯定就是`false`，因此它会被修改为`true`并且执行`timerFunc`。之后执行`this.$nextTick`其实还是调用的`nextTick`函数，只不过此时的`pending`为`true`说明`timerFunc`已经被生成，所以`this.$nextTick(fn)`只是把传入的`fn`置入`callbacks`之中。此时的`callbacks`有两个`function`成员，一个是`flushSchedulerQueue`，另外一个就是`this.$nextTick()`的回调。

因此，上面这段代码中，在`Chrome`下，有一个`macroTask`和两个`microTask`。一个`macroTask`就是`setTimeout`，两个`microTask`：分别是`Vue`的`timerFunc`（其中先后执行`flushSchedulerQueue`和`function() {console.log('2')}`）、代码中的`Promise.resolve().then()`。

### 版本升级带来的变化

上面讨论的`nextTick`实现是`2.4`版本以下的实现，`2.5`以上版本对于`nextTick`的内部实现进行了大量的修改。

#### 独立文件

首先是从`Vue 2.5+`开始，抽出来了一个单独的文件[`next-tick.js`](https://github.com/vuejs/vue/blob/dev/src/core/util/next-tick.js)来执行它。

#### microTask与macroTask

在代码中，有这么一段注释：

![](http://oxybu3xjd.bkt.clouddn.com/18-2-21/50449591.jpg)

其大概的意思就是：在`Vue 2.4`之前的版本中，`nextTick`几乎都是基于`microTask`实现的（具体可以看文章`nextTick`一节），但是由于`microTask`的执行优先级非常高，在某些场景之下它甚至要比事件冒泡还要快，就会导致一些诡异的问题；但是如果全部都改成`macroTask`，对一些有重绘和动画的场景也会有性能的影响。**所以最终`nextTick`采取的策略是默认走`microTask`，对于一些`DOM`的交互事件，如`v-on`绑定的事件回调处理函数的处理，会强制走`macroTask`。**

具体做法就是，在`Vue`执行绑定的`DOM`事件时，默认会给回调的`handler`函数调用`withMacroTask`方法做一层包装，它保证整个回调函数的执行过程中，遇到数据状态的改变，这些改变而导致的视图更新（`DOM`更新）的任务都会被推到`macroTask`。

```javascript
function add$1 (event, handler, once$$1, capture, passive) {
    handler = withMacroTask(handler);
    if (once$$1) { handler = createOnceHandler(handler, event, capture); }
    target$1.addEventListener(
        event,
        handler,
        supportsPassive
        ? { capture: capture, passive: passive }
        : capture
    );
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a Task instead of a MicroTask.
 */
function withMacroTask (fn) {
    return fn._withTask || (fn._withTask = function () {
        useMacroTask = true;
        var res = fn.apply(null, arguments);
        useMacroTask = false;
        return res
    })
}
```

而对于`macroTask`的执行，`Vue`优先检测是否支持原生`setImmediate`（高版本IE和Edge支持），不支持的话再去检测是否支持原生`MessageChannel`，如果还不支持的话为`setTimeout(fn, 0)`。

最后，写一段demo来测试一下：

```html
<div id="example">
    <span>{{test}}</span>
    <button @click="handleClick">change</button>
</div>
```
```javascript
var vm = new Vue({
    el: '#example',
    data: {
        test: 'begin',
    },
    methods: {
        handleClick: function() {
            this.test = end;
            console.log('script')
            this.$nextTick(function () { 
                console.log('nextTick')
            })
            Promise.resolve().then(function () {
                console.log('promise')
            })
        }
    }
})
```

在`Vue 2.5+`中，这段代码的输出顺序是`script`、`promise`、`nextTick`，而`Vue 2.4`输出`script`、`nextTick`、`promise`。`nextTick`执行顺序的差异正好说明了上面的改变。

#### MessageChannel

在`Vue 2.4`版本以前使用的`MutationObserver`来模拟异步任务。而`Vue 2.5`版本以后，由于兼容性弃用了`MutationObserver`。

`Vue 2.5+`版本使用了[`MessageChannel`](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageChannel)来模拟`macroTask`。除了`IE`以外，`messageChannel`的兼容性还是比较可观的。

```javascript
const channel = new MessageChannel()
const port = channel.port2
channel.port1.onmessage = flushCallbacks
macroTimerFunc = () => {
port.postMessage(1)
}
```

可见，新建一个`MessageChannel`对象，该对象通过`port1`来检测信息，`port2`发送信息。通过`port2`的主动`postMessage`来触发`port1`的`onmessage`事件，进而把回调函数`flushCallbacks`作为`macroTask`参与事件循环。

#### MessageChannel VS setTimeout

为什么要优先`MessageChannel`创建`macroTask`而不是`setTimeout`？

`HTML5`中规定`setTimeout`的最小时间延迟是`4ms`，也就是说理想环境下异步回调最快也是`4ms`才能触发。

`Vue`使用这么多函数来模拟异步任务，其目的只有一个，就是**让回调异步且尽早调用**。而`MessageChannel`的延迟明显是小于`setTimeout`的。[对比传送门](https://stackoverflow.com/questions/18826570/settimeout0-vs-window-postmessage-vs-messageport-postmessage)



### 为什么要异步更新视图

看下面的代码：

```html
<template>
  <div>
    <div>{{test}}</div>
  </div>
</template>
```

```javascript
export default {
    data () {
        return {
            test: 0
        };
    },
    mounted () {
      for(let i = 0; i < 1000; i++) {
        this.test++;
      }
    }
}
```

现在有这样的一种情况，`mounted`的时候`test`的值会被`++`循环执行`1000`次。 每次`++`时，都会根据响应式触发`setter->Dep->Watcher->update->run`。 如果这时候没有异步更新视图，那么每次`++`都会直接操作`DOM`更新视图，这是非常消耗性能的。 所以`Vue`实现了一个`queue`队列，在下一个`Tick`（或者是当前`Tick`的微任务阶段）的时候会统一执行`queue`中`Watcher`的`run`。同时，拥有相同`id`的`Watcher`不会被重复加入到该`queue`中去，所以不会执行`1000`次`Watcher`的`run`。最终更新视图只会直接将`test`对应的`DOM`的`0`变成`1000`。 保证更新视图操作`DOM`的动作是在当前栈执行完以后下一个`Tick`（或者是当前`Tick`的微任务阶段）的时候调用，大大优化了性能。

### 应用场景

在操作`DOM`节点无效的时候，就要考虑操作的实际`DOM`节点是否存在，或者相应的`DOM`是否被更新完毕。

比如说，在`created`钩子中涉及`DOM`节点的操作肯定是无效的，因为此时还没有完成相关`DOM`的挂载。解决的方法就是在`nextTick`函数中去处理`DOM`，这样才能保证`DOM`被成功挂载而有效操作。

还有就是在数据变化之后要执行某个操作，而这个操作需要使用随数据改变而改变的`DOM`时，这个操作应该放进`Vue.nextTick`。

之前在做慕课网音乐`webApp`的时候关于播放器内核的开发就涉及到了这个问题。下面我把问题简化：

现在我们要实现一个需求是点击按钮变换`audio`标签的`src`属性来实现切换歌曲。

```html
<div id="example">
    <audio ref="audio"
           :src="url"></audio>
    <span ref="url"></span>
    <button @click="changeUrl">click me</button>
</div>
```
```javascript
const musicList = [
    'http://sc1.111ttt.cn:8282/2017/1/11m/11/304112003137.m4a?tflag=1519095601&pin=6cd414115fdb9a950d827487b16b5f97#.mp3',
    'http://sc1.111ttt.cn:8282/2017/1/11m/11/304112002493.m4a?tflag=1519095601&pin=6cd414115fdb9a950d827487b16b5f97#.mp3',
    'http://sc1.111ttt.cn:8282/2017/1/11m/11/304112004168.m4a?tflag=1519095601&pin=6cd414115fdb9a950d827487b16b5f97#.mp3'
];
var vm = new Vue({
    el: '#example',
    data: {
        index: 0,
        url: ''
    },
    methods: {
        changeUrl() {
            this.index = (this.index + 1) % musicList.length
            this.url = musicList[this.index];
            this.$refs.audio.play();
        }
    }
});
```

毫无悬念，这样肯定是会报错的：
```javascript
Uncaught (in promise) DOMException: The element has no supported sources.
```
原因就在于`audio.play()`是同步的，而这个时候`DOM`更新是异步的，`src`属性还没有被更新，结果播放的时候`src`属性为空，就报错了。

解决办法就是在`play`的操作加上`this.$nextTick()`。

```javascript
this.$nextTick(function() {
    this.$refs.audio.play();
});
```
---

参考链接

https://github.com/youngwind/blog/issues/88

https://github.com/answershuto/learnVue/blob/master/docs/Vue.js%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0DOM%E7%AD%96%E7%95%A5%E5%8F%8AnextTick.MarkDown

https://juejin.im/post/5a1af88f5188254a701ec230

