## 1 CommonJS规范
在ES6之前，javascript规范中没有模块化机制，CommonJS规范的提出弥补了这个这个缺陷，是javascript不仅能开发客户端应用，还可以开发服务端、桌面程序、混合应用。
node借鉴了CommonJS的module规范实现了一套模块化系统，使用npm包（模块）管理工具来管理应用中使用到的模块。

### 1.1 CommonJS模块规范
CommonJS对模块的定义十分简单，主要分为模块引用、模块定义和模块标识3个部分：
```javascript
// 模块引用
let math = require('math');

// 模块定义
exports.add = () => {
    // do something
}

// 模块标识，模块标识就是传递给require方法的参数
let math = require('math');                 // 可以是这样的字符串
let math = require('./utils');              // 相对路径
let math = require('/config/data.json');    // 绝对路径
```
在使用CommonJS规范的模块定义时请注意exports关键字和module.exports的区别:
错误实例
```javascript
exports = {
    aaa: function() {...},
    bbb: function() {...}
}
```

正确实例
```javascript
mudule.exports = {
    aaa: function() {...},
    bbb: function() {...}
}
```

正确实例
```javascript
exports.aaa = function() {...};
exports.bbb = function() {...};
```

正确实例
```javascript
exports = module.exports = {
    aaa: function() {...},
    bbb: function() {...}
}
```
使用CommonJS规范导出模块时，每个模块都会初始化一个空对象mudule.exports = {},而<code>exports</code>关键字只是对这个对象的引用，在导出模块时真正导出的是mudule.exports对象，一般情况下mudule.exports == exports，但是一旦重新为exports对象赋值时，其引用就不再指向mudule.exports（错误实例）。


### 1.2 node模块的实现
在node中引入模块，需要经历3个步骤：
(1) 路径分析
(2) 文件定位
(3) 编译执行
node中的模块分为两类：node提供的核心模块和用户自己编写的文件模块。
+ 核心模块在node编译过程中编译进了二进制执行文件，因此在node进程启动时，核心模块就被直接加载进内存中，因此引入核心模块时不需要经历文件定位和编译两个步骤，并且在路径分析是会优先判断，所以其加载速度也是最快的。
+ 文件模块则是在运行时动态加载的，需要经历完整的3个步骤，加载速度比核心模块慢。
#### 1.2.1 优先从缓存中加载
node加载模块也使用了缓存机制，不管是核心模块还是文件模块，node加载时都相同模块的二次加载一律采用缓存优先的方式。

#### 1.2.2 路径分析和文件定位
1. 路径分析：由于标识符的不同，模块的路径有不同程度上的差异。
    + 核心模块：核心模块的优先级仅次于缓存加载，因为它在node的源代码编译过程中已经被编译为二进制代码，因此其加载过程是最快的。
    + 文件模块：在加载用户编写的文件模块时，首先会将路径转换为真实的路径，找到对应文件后再进行编译执行，并将编译后的二进制代码存放在缓存中便于二次加载，文件模块的加载速度慢于核心模块。
2. 文件定位：文件定位就是通过分析得到的路径去查找文件，在这个过程中需要注意一些细节。
    + 文件扩展名分析：当出现在标识符中不包含扩展名时，node会按照.js、.node、.json的次序补足扩展名依次尝试，在尝试过程中需要调用fs模块同步阻塞式的判断文件是否存在，因此建议在传递标识符时尽量带上扩展名。
    + 目录分析和包：require()在分析文件扩展名之后可能没有查找到对应的文件，但是却找到一个目录，此时node会将此目录当做模块来处理，在这个过程中node首先在该目录下查找并解析package.json文件，取出其中mian属性指定的文件名进行定位，如果解析package.json出错或根本就没有package.json，node则会依次查找index.js、index.node、index.json。

## 模块编译
在node中，每个文件模块都是一个对象，其定义如下：
```javascript
function Module(id, parent) {
    this.id = id;
    this.exports = {};
    this.parent = parent;
    if(parent && parent.children) {
        parent.children.push(this);
    }

    this.filename = null;
    this.loaded = false;
    this.children = [];
}
```
1. javascript模块的编译
我们知道每个模块文件中都存在require、exports、module这三个变量，但是在模块文件中并没有定义，它们是从何而来的呢？
事实上在编译过程中，node对获取到的javascript文件内容进行了头尾包装，在头部添加了<code>(function (exports, require, module, _filename, _dirname){\n</code>，在尾部添加了<code>\n});</code>所以一个正常的javascript文件会被包装成如下的样子：
```javascript
(function(exports, require, module, _filename, _dirname) {
    let math = require('math');
    
    exports.area = function(radius) {
        return Math.PI * radius * radius;
    };
});
```
上面代码这样把每个模块包装起来，每个模块之间的作用域都相互隔离，避免了全局变量的污染。被包装后的模块通过编译之后，会将模块的exports属性返回给调用者，exports上的所有属性都能被外部访问。

## 2 核心模块
node的核心模块在被编译成可执行文件的过程中被编译进了二进制文件，。核心模块可以分为C/C++编写的和javascript编写的两部分。

### 2.1 javascript核心模块编译过程
在编译所有C/C++文件之前，编译程序需要将所有的javascript模块文件编译为C/C++代码。node采用了v8附带的js2c.py工具将所有内置的javascript代码转换成C++里的数组，生成node_natives.h头文件。
在此过程中javascript代码以字符串的形式存储在node命名空间中，是不可执行的，在启动node进程时，javascript代码直接加载进内存中。

### 2.2 C/C++核心模块的编译过程
在node中，Buffer、crypto、fs、os等核心模块都是通过C/C++编写的，在编译时可以直接编译成二进制文件，node开始执行时，它们被直接加载进内存中，无需再次做标识定位、文件定位、编译等过程，直接就可以执行。

## 3 包与npm
node组织了自身的核心模块，也使得第三方文件模块可以有序的编写和使用。但是第三方模块中，模块与模块之间依然没有任何联系，相互之间不能直接引用，在模块之外，包和npm就是将模块联系起来的一种机制。

### 3.1 包结构
包实际上就是一个存档文件，即一个目录直接打包为.zip或tar.gz格式的文件，安装后解压还原为目录。完全符合CommonJS规范的包目录应该包含如下文件：
+ package.json：包描述文件
+ bin：用于存放可执行二进制文件的目录
+ lib：用于存放javascript代码的目录
+ doc：用于存放文档的目录
+ test：用于存放单元测试用例的代码

### 3.2 npm常用功能
CommonJS包规范是理论，npm是其中的一种实践。借助npm使我们可以快速的安装管理项目依赖包。npm功能主要有以下几点：
+ 安装项目依赖包
    ```javascript
    npm install express --save
    ```
+ 使用npm钩子函数
    ```javascript
    "scripts": {
        "preinstall": "preinstall.js",
        "install": "install.js",
        "test": "test.js",
        "uninstall": "uninstall.js"
    }
    ```
    在以上字段中执行<code>npm install &lt;package&gt;</code>时，preinstall指向的脚本文件会被加载执行，然后install指向的脚本文件会被执行。同样的在执行<code>npm uninstall &lt;package&gt;</code>时，uninstall指向的脚本文件也会相应执行。
+ 发布包
    任何一个开发者都可以发布自己的npm包供别人使用，发布一个包主要有以下步骤：
    (1) 编写模块代码
    (2) 使用<code>npm init</code>命令初始化包描述文件，即创建package.json文件
    (3) 注册包仓库账号，为了维护包，npm必须使用仓库账号才允许将包发布到仓库中。注册仓库账号的命令是<code>npm adduser</code>
    (4) 上传包，上传包的命令是<code>npm publish &lt;folder&gt;</code>。在刚刚创建的package.json文件所在目录下，执行<code>npm publish .</code>命令开始上传包。

## 4 AMD规范
AMD规范是CommonJS模块规范的一个延伸，它的模块定义如下：
```javascript
define(id?, dependencies?, factory);
```
它的模块id和依赖是可选的，与node模块相似的地方在于factory的内容就是实际代码的内容。下面的代码定义了一个简单的模块
```javascript
define(function() {
    let exports = {};
    exports.sayHello = function() {
        // do something
    }

    return exports;
});
```
与node模块不同的是，AMD模块需要用define来明确定义一个模块，而在node实现中是隐式包装的。

## 5 CMD规范
CMD规范是由国内的玉伯（阿里）提出的，与AMD规范主要的区别在于定义模块和依赖引入部分。AMD需要在声明模块的时候指定所有的依赖，通过形参传递依赖到模块内容中：
```javascript
define(['dep1', 'dep2'], function(dep1, dep2) {
    return function() {
        // do something
    };
});
```
与AMD模块规范相比，CMD模块规范更接近于node对ComonJS规范的定义：
```javascript
define(factory);
```
在依赖部分，CMD支持动态引入，require、exports、module通过形参传递给模块：
```javascript
define(function(require, exports, module) {
    // 模块代码
})
```