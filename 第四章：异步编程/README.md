## 1 异步编程的优劣
node最大的特性莫过于基于事件驱动的非阻塞IO模型，非阻塞IO可以使CPU与IO之间不用相互依赖等待，让资源得到更好的利用。对于网络应用而言，并行带来的好处不言而喻，在分布式系统和云计算等领域受到广泛的应用。
另一方面异步IO也为开发者带来了一些麻烦，下面我们梳理一下异步编程的几个难点：
#### 1. 异常处理
对于同步代码，我们进行异常处理十分简单，使用try/catch/final语句来进行异常捕获：
```javascript
try {
    JSON.parse(json);
} catch(e) {
    // do something
}
```
以上代码对于异步编程并不适用，前面我们提到异步IO的实现主要包括两个阶段：提交请求和处理结果，提交请求之后立即返回返回，可能异常并不发生在这个阶段，导致try/catch语句不会执行。
所以在node的处理异常上形成了一种约定，即将异常作为回调函数的第一个实参传回，如果是空值则表明没有异常发生。在我们自行编写的异步方法上，也需要遵循这样的原则：
+ 必须执行调用者传入的回调函数
+ 正确传递回异常供调用者判断
```javascript
const async = functin(callback) {
    process.nextTick(function() {
        let result = something;

        if(error) {
            return callback(error);
        }
        callback(null, result)
    });
};
```
#### 2. 嵌套过深
在某些场景中，经常是一个异步请求需要依赖另一个异步请求的结果，例如文件一个遍历目录的操作，如果采用嵌套的写法其代码如下：
```javascript
fs.readdir(path.join(__dirname, '...'), function(err, files) {
    files.forEach(function(filename, index) {
        fs.readFile(filename, 'utf-8', function(err, file) {
            // do something
        });
    });
});
```
上面的代码仅仅只是两次嵌套，看起来代码并不是特别糟糕，但是如果还要根据文件的内容发起另外的异步操作，就需要再继续嵌套代码，一旦嵌套过深代码将会变的非常糟糕，这样的代码被"回调地狱"。

#### 3. 多线程编程
前面介绍到node是单线程执行的，为了支持多线程编程，node借鉴了浏览器的Web Workers，提供了child_process创建子进程的API，cluster模块时更深次的应用。详细内容参考[第九章：node进程](../第九章：node进程/README.md)

## 2 异步编程解决方案
javascript发展到现在，已经出现了很多的异步编程的解决方案，详细的异步编程方案可浏览阮一峰的文章[《深入掌握 ECMAScript 6 异步编程》](http://www.ruanyifeng.com/blog/2015/04/generator.html)，这里不再做具体的讲解。

### 2.1 事件发布/订阅模式
事件监听器模式是一种广泛的异步编程模式，利用了回调函数的事件化，又被称为发布/订阅模式。node自身提供的[events模块](http://nodejs.cn/api/events.html)是发布订阅模式的一个简单实现。它具有addListener/on()、once()、removeListener()、removeAllListeners()和emit()等基本的事件监听模式方法的实现。简单的发布/订阅模式代码如下：
```javascript
const EventEmitter = require('events');

class getDataEmitter extends EventEmitter {}

const getNewsEmitter = new getDataEmitter();

// 订阅event事件
getNewsEmitter.on('event', () => {
  console.log('触发了一个事件！');
});

// 发布event事件
getNewsEmitter.emit('event');
```
#### 利用事件队列解决缓存雪崩问题
缓存雪崩是由于原有缓存失效(过期)，新缓存未到期间。所有请求都去查询数据库，而对数据库CPU和内存造成巨大压力，严重的会造成数据库宕机。从而形成一系列连锁反应，造成整个系统崩溃。更多关于雪崩问题的详细解答可浏览文章[《关于缓存雪崩和缓存穿透等问题》](https://blog.csdn.net/csdn265/article/details/56012271)。
这里我们只介绍利用事件监听的once()方法解决雪崩问题，该方法利用了once()方法只执行一次的原理。当有大量的请求时，将所有的请求的回调都压入到事件队列中，利用once()方法只执行一次的原理，所有的相同调用只会执行一次，新到的相同调用只需要在队列中等待新数据就绪即可，一旦查询结束得到的结果将会被这些调用共同使用。
```javascript
const proxy = new events.EventEmitter();

let status = 'ready';
let select = function(callback) {
    // 只订阅一次selected事件
    proxy.once('selected', callback);
    if(status === 'ready') {
        status = 'pending';
        // 只需要进行一次数据库查询即可
        db.select('SQL', function(result) {
            proxy.emit('selected', result);
            status = 'ready';
        })
    }
}
```

## 3 异步并发控制
node中大量采用了异步编程，其解决的问题无外乎保持异步的性能优势，提升编程体验，但是这里存在一个过犹不及的问题：如果并发量过大，我们底层的服务器将会吃不消，如果是对文件系统进行大量的并发调用，操作系统的文件描述符将会被瞬间用光，抛出错误。
因此虽然在node中并发容易实现，但是我们依然需要进行并发控制，给予一定的过载保护。
