先看callbacks的用法：

**1.1 once**
创建的 callbacks 对象只允许被 fireWith() 一次 [注意：方法fire() 是 fireWith() 的外观模式。

```js
var callbacks = $.Callbacks("once"); callbacks.add(function(){console.log("f1");}); callbacks.fire(); //输出 "f1" callbacks.fire(); //什么也不发生，在源码中已经禁用了 list.disable() 

```









