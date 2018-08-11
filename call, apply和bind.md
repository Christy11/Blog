## 1 引入

我们都知道，在使用this 的时候，可能会出现绑定丢失，也可能会出现绑定的对象不是我们想要的绑定对象的情况。针对这些情况，我们可以通过使用`call()`和`apply()`方法在函数执行的时候绑定指定的对象。

在高程3中，我们可以知道，每个函数都包含两个**非继承**而来的方法：`apply()` 和 `call()`。每个函数创建的时候，都会有一个内嵌的Function原型对象，这个对象的初始值指向**Function.prototype**，这两个方法都是**Function.prototype**上面的方法。这两个方法的用途都是在特定的作用域中调用函数，实际上等于设置函数体内this对象的值。

下面分别讨论一下这两个方法。

## 2 Function.prototype.call()

| call(context, param1, param2 …) | 描述                                 | 备注                   |
| :-----------------------------: | :----------------------------------- | ---------------------- |
|          arguments[0]           | 需要绑定的对象                       | 可选                   |
|          arguments[1]           | 参数1                                | 可选                   |
|                …                | call()方法可接收多个参数（参数列表） | 每个参数必须以逐个传递 |

由上表可知，`call()`方法可以接收多个参数，第一个可选参数为需要绑定的对象，后面的参数均为函数需要传递的参数。举个🌰 ，在执行方法`foo.call(bar)`的时候，会先让`foo`方法中的**`this`**变成第一个参数值`bar`，然后再执行`foo()`函数，可以理解为只是修改了**`this`**的指向，函数执行仍旧是原函数的执行顺序。

我们来看一下call()函数的大致实现思路（参考[深入JS系列（一：call, apply, bind实现）](https://blog.csdn.net/u010377383/article/details/80646415)中的实现思路）：

``` javascript
// context表示需要被绑定的上下文
Function.prototype.myCall = function(context) {
    var context = context || window; // 将执行上下文设置为当前上下文，若无参数值，则设置为全局上下文
    context.func = this; // 使当前上下文可以执行函数
    var args = []; // call()方法必须逐个传参，此处将arguments转为数组放入函数中执行
    // arguments是类数组对象，可通过length获取长度，遍历从1开始是因为arguments[0] = context
    for(var i = 1; i < arguments.length; i ++) {
        args.push(arguments[i]);
    }
    var result = context.func(...args); // 基于this的上下文执行函数并返回结果
    delete context.func;
    return result;
}
```

上面代码的具体使用可以参见[模拟Function.prototype.call()](http://jsrun.net/FJgKp/edit)

## 3 Function.prototype.apply()

| apply(context, arr) | 描述                | 备注 |
| :-----------------: | ------------------- | ---- |
|    arguments[0]     | 需要绑定的对象      | 可选 |
|    arguments[1]     | 参数数组或arguments | 可选 |

由上表可知，`apply()`方法可接收两个参数，一个是需要绑定的执行上下文对象，一个是参数数组。其中第二个参数便是`apply`与`call`方法的**不同之处**：`call()`方法必须逐个传递参数，而`apply()`方法则允许传递数组，从**本质**上来说，`apply()`方法是`call()`方法的**语法糖**。

看一下apply()函数的大致实现思路：

``` javascript
// context表示需要被绑定的上下文
Function.prototype.myApply = function(context, arr) {
    var context = context || window; // 将执行上下文设置为当前上下文，若无参数值，则设置为全局上下文
    context.func = this; // 使当前上下文可以执行函数
    var result;
    if (!arr) { // 如果没有传值，直接执行函数
        result = context.func();
    } else {
        // 判断第二个参数是否是数组，若不是，则直接返回错误提示
        if (!(arr instanceof Array)) throw new Error('params must be an array!');
        result = context.fn(...arr);
    }
    delete context.func;
    return result;
}
```

## 4 Function.prototype.bind()

| bind(context, param1, param2…) | 描述                                 | 备注                   |
| :----------------------------: | ------------------------------------ | ---------------------- |
|          arguments[0]          | 需要绑定的对象                       |                        |
|          arguments[1]          | 参数1                                |                        |
|               …                | call()方法可接收多个参数（参数列表） | 每个参数必须以逐个传递 |

由上表可知，`bind()`方法可接收多个参数，第一个参数为执行上下文，后面可依次接收其他参数。

`bind()`方法会创建一个硬编码的新函数，把指定的参数设置为**`this`**的上下文并调用原始函数。

看一下bind()函数的大致实现思路：

``` javascript
Function.prototype.myBind = function(context) {
    if (typeof this !== 'function') throw new TypeEroor('Invalid invoked!');
    var self = this;
    var args = Array.prototype.slice.call(arguments, 1); // 获得除上下文之外的其他参数
    // ① 创建空函数
    var fNOP = function(){}
    const fBound = function() {
        // 获取函数的参数
        var bindArgs = Array.prototype.slice.call(arguments);
        // 判断函数时作为构造函数还是普通函数
        // 构造函数this instanceof fNOP返回true，将绑定函数的this指向该实例，可以让实例获得来自绑定函数的值
        // 当作为普通函数时，this指向window，此时结果为false，将绑定函数的this指向context
        return self.apply((this instanceof fNOP ? this : context), args.concat(bindArgs));
    }
    // ② fNOP函数的prototype为绑定函数的prototype
    fNOP.prototype = this.prototype;
    // ③ 返回函数的prototype等于fNOP函数的实例实现继承
    fBound.prototype = new fNOP();
    // 上面的①②③相当于Object.create(this.prototype)
    return fBound;
}
```

## 5 call、apply和bind的区别

首先说一下call和apply的区别与联系：

call和apply都是用于修改函数执行上下文的方法，两个方法都是**Function.prototype**上的原生方法。

二者的最大**区别**主要有两点：

1. 传递参数的类型不同，call()方法接收一个参数列表，对参数逐个处理；apply()方法的第二个参数可以是数组可以是类数组arguments；
2. 由于apply()本质是call()的语法糖，所以在执行上，call()的性能比apply()好，具体原因参见[为什么call比apply快](https://juejin.im/post/59c0e13b5188257e7a428a83)

`bind()`方法与`call()`和`apply()`最**不相同**的地方在于：`bind()`会返回一个函数，并且当使用bind绑定了上下文后，无法再使用call和apply重新绑定。

总结如下表

|                          | call()                   | apply()                   | bind()                                                 |
| ------------------------ | ------------------------ | ------------------------- | ------------------------------------------------------ |
| 参数                     | 除context外需逐个传值    | 除context外仅需传一个数组 | 除context外需逐个传值                                  |
| 性能                     | ☆☆☆☆☆                    | ☆☆☆☆                      | —                                                      |
| 返回值                   | 返回调用函数本身的返回值 | 返回调用函数本身的返回值  | 返回硬编码的新函数                                     |
| 绑定后是否可修改绑定对象 | 可以                     | 可以                      | bind()绑定不能被call()和apply()修改，**可以被new修改** |

【参考网址】

《JavaScript高级程序设计》P(116 - 118)

[JavaScript中的call、apply、bind深入理解](https://www.jianshu.com/p/00dc4ad9b83f)

[深入JS系列（一：call, apply, bind实现）](https://blog.csdn.net/u010377383/article/details/80646415)

[为什么 call 比 apply 快？](https://juejin.im/post/59c0e13b5188257e7a428a83)

[JS中的bind()方法](https://blog.csdn.net/kongjunchao159/article/details/59113129)