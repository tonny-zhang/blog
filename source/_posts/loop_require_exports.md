---
title: nodejs 循环依赖及引用中无法使用对象方法或属性
date: 2016-12-02 14:32:05
tags: blog, nodejs
---  
  相信很多使用 `nodejs` 有朋友都会遇到了循环依赖的问题，它的原理是什么？我们怎样在日常开发中规避掉这样的问题？
  
### 场景重现
one.js
```javascript
console.log('one init');
const two = require('./two');
var exp = module.exports = {};
console.log('one exports');
function start() {
    console.log('one start');
    two.start();
}
function load() {
    console.log('one load');
}
exp.start = start;
exp.load = load;
```
two.js
```javascript
console.log('two init');
const one = require('./one');
var exp = module.exports = {};

function start() {
    console.log('two start');
    one.load();
}

exp.start = start;
```
main.js
```javascript
const one = require('./one')

one.start();
```
> 输出结果
```
$ node main.js
one init
two init
one {}
one exports
one start
two start
E:\source\nodejs\test\loop_require\two.js:8
    one.load();
        ^

TypeError: one.load is not a function
    at Object.start (E:\source\nodejs\test\loop_require\two.js:8:9)
    at Object.start (E:\source\nodejs\test\loop_require\one.js:7:9)
    at Object.<anonymous> (E:\source\nodejs\test\loop_require\main.js:3:5)
    at Module._compile (module.js:570:32)
    at Object.Module._extensions..js (module.js:579:10)
    at Module.load (module.js:487:32)
    at tryModuleLoad (module.js:446:12)
    at Function.Module._load (module.js:438:3)
    at Module.runMain (module.js:604:10)
    at run (bootstrap_node.js:394:7)
```
上面的场景其实有一个循环依赖和 `module.exports` 的问题，首先我们肯定下造成上面的远行异常本身不是循环依赖造成的，而是我们代码中对 `exports` 使用异常造成的


## `module.exports` 是什么？
相信细心的朋友应该知道，如果你直接 `require` 一个空的模块的话，它是一个空对象，上面的场景中 `two.js` 引用 `one.js` 此时的对象为 `{}` 而 `one.js` 中对 `exports` 对象进行了重新指向，此时 `two.js` 中引用到的对象和 `one.js` 后续重新生成的 `exports` 对象已经不是一个对象了，所以在 `two.js` 中使用 `one.start()` 中的方法时提示 `TypeError: one.load is not a function` 

[官方文档中对 `exports` 的描述](https://nodejs.org/api/modules.html#modules_module_exports)

## 如何解决
### 1. 把模块的方法直接绑定到 `module.exports` 对象上（如果出现重复依赖时尤其要这样做）
```
module.exports.start = function() {
    // do something
}

exports.run = function() {
    // do something
}
```
### 2. 在方法使用时再引用（不建议这样使用）

上面场景中的 two.js 可以有如下写法:
```javascript
console.log('two init');
const one = require('./one');
var exp = module.exports = {};

function start() {
    console.log('two start');
    require('./one').load();
}

exp.start = start;
```


`nodejs` 本身对循环依赖的处理是很合理的，出现问题也都是我们逻辑处理上的问题，希望上面的分析和解决方案可以帮助到还被困扰中的你。