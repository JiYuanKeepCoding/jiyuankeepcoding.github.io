---
layout:     post
title:      【翻译】javascript模块化入门
date:       2019-04-13
author:     JY
header-img: img/cat1_jsModule.jpg
catalog: true
tags:
    - 前端
    - javascript模块化
---
原文地址：https://medium.freecodecamp.org/javascript-modules-a-beginner-s-guide-783f7d7a5fcc
# 前言
如果你是一个javascript新手，上来就接触到一些术语诸如“module bundlers vs. module loaders,” “Webpack vs. Browserify” and “AMD vs. CommonJS”总是会感到很懵
虽然这些属于听着很让人望而却步，但是掌握它们对于javascript开发者来说还是很重要的。
这篇文章用简单的文字和代码示例介绍刚刚提到的一些术语，希望对大家有帮助
为了简单，本文会分成两部分。1.模块是啥，为啥我们要用到2.模块打包意味着什么，有哪些不同的方式


## part1
### 为啥用模块
1.易于维护
2.命名空间，不污染全局变量
3.可重用性
### 几个javascript模块的例子
模块范式是用来模仿class的概念的(javascript天生不支持class),所以我们能够像Java一样在一个对象里面存变量，公有方法，私有方法。
有很多方法可以让我们实现刚才所说的。第一个例子是一个匿名闭包
#### 例子1
```
(function () {
  // We keep these variables private inside this closure scope
  
  var myGrades = [93, 95, 88, 0, 55, 91];
  
  var average = function() {
    var total = myGrades.reduce(function(accumulator, item) {
      return accumulator + item}, 0);
    
      return 'Your average grade is ' + total / myGrades.length + '.';
  }

  var failing = function(){
    var failingGrades = myGrades.filter(function(item) {
      return item < 70;});
      
    return 'You failed ' + failingGrades.length + ' times.';
  }

  console.log(failing());

}());
```
这样一来所有执行过程中产生的变量都在这个匿名的闭包里面了，就不会污染全局变量，这就是刚刚提到的第二点。
值得注意的是，在刚刚的例子中最外面的括号是必须有的。因为如果没有括号的话这个函数就属于函数声明（必须有名字），但是加了括号就代表这是一个函数表达式，不需要名字。
#### 例子2
我们还可以给模块传递一个全局变量，这样模块会把公有方法放在全局变量里面，Jquery就是用这样的方式
```
(function (globalVariable) {

  // Keep this variables private inside this closure scope
  var privateFunction = function() {
    console.log('Shhhh, this is private!');
  }

  // Expose the below methods via the globalVariable interface while
  // hiding the implementation of the method within the 
  // function() block

  globalVariable.each = function(collection, iterator) {
    if (Array.isArray(collection)) {
      for (var i = 0; i < collection.length; i++) {
        iterator(collection[i], i, collection);
      }
    } else {
      for (var key in collection) {
        iterator(collection[key], key, collection);
      }
    }
  };

  globalVariable.filter = function(collection, test) {
    var filtered = [];
    globalVariable.each(collection, function(item) {
      if (test(item)) {
        filtered.push(item);
      }
    });
    return filtered;
  };

  globalVariable.map = function(collection, iterator) {
    var mapped = [];
    globalUtils.each(collection, function(value, key, collection) {
      mapped.push(iterator(value));
    });
    return mapped;
  };

  globalVariable.reduce = function(collection, iterator, accumulator) {
    var startingValueMissing = accumulator === undefined;

    globalVariable.each(collection, function(item) {
      if(startingValueMissing) {
        accumulator = item;
        startingValueMissing = false;
      } else {
        accumulator = iterator(accumulator, item);
      }
    });

    return accumulator;

  };

 }(globalVariable));
```
#### 例子3
对象接口,从这个例子里面可以看出myGrade是私有变量，average,failing是共有方法
```
var myGradesCalculate = (function () {
    
  // Keep this variable private inside this closure scope
  var myGrades = [93, 95, 88, 0, 55, 91];

  // Expose these functions via an interface while hiding
  // the implementation of the module within the function() block

  return {
    average: function() {
      var total = myGrades.reduce(function(accumulator, item) {
        return accumulator + item;
        }, 0);
        
      return'Your average grade is ' + total / myGrades.length + '.';
    },

    failing: function() {
      var failingGrades = myGrades.filter(function(item) {
          return item < 70;
        });

      return 'You failed ' + failingGrades.length + ' times.';
    }
  }
})();

myGradesCalculate.failing(); // 'You failed 2 times.' 
myGradesCalculate.average(); // 'Your average grade is 70.33333333333333.'
```
### CommonJS和AMD
刚刚的例子有一个共同之处:用一个全局的function创建了一个闭包，在闭包里面使用私有的命名空间。
但是这些例子中都有一个缺陷。
作为一个开发者，你需要知道正确的依赖顺序来加载你的文件。比如，你在项目中用到了Backbone，所以用<script>引用了它的源码。
然而，Backbone强依赖Underscore.js。那你就需要把对Underscore的应用加在Backbone前面。一旦依赖变多了，这样的事情会变得很复杂
还有一个问题这样仍然有可能导致全局变量冲突，比如例子3中其他模块也有叫myGradesCalculate的.
那有没有办法啊可以让这些模块不在全局作用域里面呢？答案是肯定的。有两种关于这个的著名的实现CommonJS, AMD
#### CommonJS
CommonJs是设计和实现Javascript模块声明API的一个志愿者工作组
CommonJs模块其实是一个可重用的一坨javascript，这坨javascript会export一些特定对象。这样就能被其他模块require进来。写过Node.js的朋友应该很熟悉。
下面是一个定义CommonJs模块的例子
```
function myModule() {
  this.hello = function() {
    return 'hello!';
  }

  this.goodbye = function() {
    return 'goodbye!';
  }
}

module.exports = myModule;
```
其他模块如果下昂要引用的话就像这样
```
var myModule = require('myModule');

var myModuleInstance = new myModule();
myModuleInstance.hello(); // 'hello!'
myModuleInstance.goodbye(); // 'goodbye!'
```
这种方式很好的解决了刚才剔除的缺陷。
1.由于用到的模块都需要require()所以依赖关系就变得清晰
2.每个模块不需要写全局变量
这里需要注意是require()是阻塞的。而javascript又是单线程的，这在server端通常没什么问题因为require的时候只是去读磁盘中的文件然后加载。但是在浏览器中这意味着一个网络请求，唯一的线程会一直等到require的模块文件返回load完毕。
#### AMD
接着刚才的问题说。如果在浏览器端能够异步加载模块这样不就好了吗。AMD正式解决这样一个问题的技术，全称是Asynchronous Module Definition。
下面给出一个AMD的例子
```
define(['myModule', 'myOtherModule'], function(myModule, myOtherModule) {
  console.log(myModule.hello());
});
```
define函数的第一个参数是所需要加载的模块。第二个参数是回调函数，刚刚加载好的模块就会作为args传进回调函数。
简单来说AMD主要是适用于浏览器。
#### UMD
有些项目需要你同时支持AMD和CommonJS功能，那么就可以用UMD(Universal Module Definition)
UMD支持刚刚说到的两种模块和定义在全局变量的模块。并且即可以在服务端也可以在浏览器端执行。
下面是一个UMD的例子:
```
(function (root, factory) {
  if (typeof define === 'function' && define.amd) {
      // AMD
    define(['myModule', 'myOtherModule'], factory);
  } else if (typeof exports === 'object') {
      // CommonJS
    module.exports = factory(require('myModule'), require('myOtherModule'));
  } else {
    // Browser globals (Note: root is window)
    root.returnExports = factory(root.myModule, root.myOtherModule);
  }
}(this, function (myModule, myOtherModule) {
  // Methods
  function notHelloOrGoodbye(){}; // A private method
  function hello(){}; // A public method because it's returned (see below)
  function goodbye(){}; // A public method because it's returned (see below)

  // Exposed public methods
  return {
      hello: hello,
      goodbye: goodbye
  }
}));
```
#### ES6
刚才我们说的所有的打包方式都不是javascript原生支持的。在ES6对语法和语义的定义中，引入了模块的定义。
ES6中的模块汲取了CommonJS和AMD的优点，声明式的语法和异步加载，和对循环依赖更好的支持。
还有很厉害的一点是ES6模块的引入是对导出内容的实时只读（这句比较难翻，需要看下面代码理解）。在CommonJS中，import只是对export的拷贝。这是CommonJs的例子
```
// lib/counter.js
var counter = 1;

function increment() {
  counter++;
}

function decrement() {
  counter--;
}

module.exports = {
  counter: counter,
  increment: increment,
  decrement: decrement
};


// src/main.js

var counter = require('../../lib/counter');

counter.increment();
console.log(counter.counter); // 1
```
为啥输出结果不是2呢。因为increment()被调用的时候被改写的counter是在counter.js里面的变量。而在main.js require过来的counter是counter.js里面的变量的拷贝，所以我们无法通过调用increment()改变require过来的count
ES6很好的解决了这个问题
```
// lib/counter.js
export let counter = 1;

export function increment() {
  counter++;
}

export function decrement() {
  counter--;
}


// src/main.js
import * as counter from '../../counter';

console.log(counter.counter); // 1
counter.increment();
console.log(counter.counter); // 2
```
也就是说ES6的import并不是对export建立拷贝

## part2
### 为啥要打包模块呢
当你把你的程序分成多个模块，你通常要把这些模块组织成不同的文件夹和文件。比如在你使用React的时候你有一组你所使用的库的模块
那你就需要在你的主html为每一个你所依赖的模块加<script>标签来把这些依赖引入进来。当用户打开你的页面的时候浏览器就会一个一个的单独加载这些<script>
这样页面加载速度就会很慢。

为了克服这个问题，我们捆绑或者说把这些文件混合进一个大文件(也可能是多个文件)为了减少http请求数。当我们听到其他开发者谈论构建阶段，构建过程。其实指的就是这些事。
还有一个加快打包速度的方法就是减少打包出来的代码量. 减少代码量的办法就是去除源代码中不必要的字符（比如空格，注解，换行符）这样就可以减少整体大小而不改变代码的功能。
数据量越小意味着浏览器能够更快的下载，加载这些文件。如果你见过文件里面带个min的后缀比如 “underscore-min.js”,你可能发现了min版本比起完全版小得多(虽然min版的代码不便阅读/调试)。
打包工具比如gulp，grunt让混合，压缩变得简单直接，并且保证了刻度的代码仍然可以让开发者看到而浏览器读到的是打包，压缩过的代码。

### 有哪些不同的打包模块的方式呢
只要你使用了一种标准的方式定义模块。就可以方便的混合，压缩你的javascript代码。
然而如果你使用了你的浏览器无法解释的非原生的模块系统，比如CommonJS, AMD (甚至是es6的原生模块语法)。你需要一个专门的工具来吧这些模块组织成能浏览器能够解释执行的的代码。这些就是Browserify, RequireJS, Webpack以及一些其他的模块打包工具做的事情。
除了打包加载你的模块之外，模块打包工具提供了很多额外的功能比如自动重新编译当你在修改代码和debug的时候。

### 打包CommonJS
CommonJs使用同步的方式加载模块，这样虽然很好但是对浏览器来说却是不现实的。比如我们有个main.js应用了一个模块来计算平均数:
```
var myDependency = require(‘myDependency’);

var myGrades = [93, 95, 88, 0, 91];

var myAverageGrade = myDependency.average(myGrades);
```
这个例子里面，我们的代码依赖了myDependency模块。在以下命令里面。浏览器递归的打包了所有依赖的模块，最终生成了bundle.js
```
browserify main.js -o bundle.js
```
Browserify 会先根据当前代码生成一个抽象语法树，这样就可以遍历整个项目依赖图。理清楚了项目的依赖结果之后，就可以按照顺序打包所有的依赖到一个文件里面了。这个时候你就只需要引入单个<script>进你的html代码。
接下来你可以用Minify-JS去压缩你的bundle.js。完成!

### 打包AMD
如果你用AMD，你会用诸如RequireJs或者Curl这样的AMD加载器。一个模块加载器动态的加载了你的程序所需的模块
用AMD的话其实不需要build这个步骤来打包你的模块到一个文件里面因为是异步加载的模块。这些模块只有在用到的时候才会去下载而不是一开始就下载全部模块。
事实上每个用户行为所产生的大量的网络请求无法满足生产环境对性能的要求。大多数web开发者仍然在用打包工具来压缩他们的AMD模块为了提升性能，比如RequireJS optimizer, r.js（2016年的文章）。
总得来说，AMD和CommonJs打包的区别是，在开发的时候AMD没有build步骤

### Webpack
Webpack是比较新的打包工具。它的设计初衷是不管你用CommonJs还是AMD还是ES6
你可能会想既然有了browserfy和reactJs我们为什么要webpack。其中一点是webpack提供了诸如code splitting这样的功能 - 可以把代码分出"chunks"用来按需加载

### ES6模块
ES6模块和刚才几种模块最大的区别就是在设计上会注意静态分析。比如说你在代码里面import了一个module你没有用到，在编译期间就可以把它从依赖里面移除，这样对性能会有很大的提升。
ES6和其他一些去除无用代码的工具诸如UglifyJS的区别:
ES6用的方式叫做tree shaking。他做的事情是只include你所需的模块而不是去除掉你不需要的模块。看下面的例子
