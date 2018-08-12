## 1 this是什么

在JavaScript中，**this**总是一个逃不过的话题。**this**关键字是JavaScript中最复杂的机制之一，它是一个特别的关键字，被自动定义在所有函数的作用域中。

当一个函数被调用的时候，会创建一个活动记录（又叫执行上下文）。这个记录会包含函数在哪里被调0用（调用栈）、函数的调用方式、传入的参数等信息。而**this**就是这个记录的一个**属性**，会在函数执行的过程中用到。

由**this**的来源，我们不难发现，**this**是动态的，即它是在运行时绑定的，而不是在函数创建时绑定的，它的取值取决于函数在哪里被调用。

## 2 调用位置

上面我们提到，**this**的取值取决于函数在哪里被调用，那么查找函数的调用位置就至关重要。如何查找函数的调用位置——分析调用栈（即浏览器调试中的call stack），**this**的调用位置就在当前执行函数的前一个调用中。

``` javascript
// 例1. 查看调用栈
function foo() {
    // 当前调用栈是：(anonymous) → foo
    // 故当前调用位置是全局作用域Global
    console.log(this); // ① this = Window
    bar(); 
}
function bar() {
    // 当前调用栈是：(anonymous) → foo → bar
    // 故调用位置是外层函数作用域foo
    console.log(this); // ② this = Window
}
foo(); // 调用函数foo，当前调用栈是(anonymous)
```

运行上面的函数，我们发现一个奇怪的问题，`bar()`函数明明是在`foo()`中被调用了，那么此处**this**的调用位置应该是`foo`，怎么**this**还指向了Window呢？这便与**this**的绑定规则有关系了。

## 3 绑定规则

**this**有4条绑定规则：

1. 默认绑定（非严格模式下this会默认绑定到全局对象，严格模式下是undefined）

2. 隐式绑定

3. 显式绑定

4. new绑定

这4种绑定优先级有低到高：4 > 3 > 2 > 1。下面简单看一下这4种绑定：

### 3.1 默认绑定

当函数作为**独立函数调用**的时候，会执行默认绑定，这条规则可以看做是无法应用其他规则时的默认规则。那么什么是独立函数调用？也就是上文例1中的`foo()`和`bar()`，这两个函数都是直接***使用不带任何修饰的函数引用***进行调用的，所以只能使用默认绑定，无法引用其他规则，故会出现两个函数中的**this**均为Window的情况。

另外，严格模式下绑定的时候，this是undefined，只出现在函数在严格模式下运行的情况下。函数在严格模式下调用并不影响默认绑定：

``` javascript
// 例2. 严格模式影响默认绑定的情况
function foo() {
    'use strict';
    console.log(this.a);
}
var a = 2;
foo(); // TypeError: this is undefined

(function(){
    'use strict';
    foo(); // 2
})
```

### 3.2 隐式绑定

当调用位置有上下文对象的时候，或者说是被某个对象拥有的时候，**this**的默认绑定会让**this**指向调用位置的上下文对象。

``` javascript
// 例3. 隐式绑定&绑定丢失
function foo() {
    console.log(this.a);
}
var obj = {
    a: 2,
    foo: foo
};
var a = 'global a, wrong!';

obj.foo();			// 2

var bar = obj.foo;
bar();				// global a, wrong!

function doFoo(fn) {
    fn();
}
doFoo(obj.foo);		// global a, wrong!
```

由例3可知，除了直接执行`obj.foo()`之外，使用赋值之后再调用都会变成执行`foo()`的**独立函数调用**，而`foo`实际上被隐式绑定在`obj`上，使用`bar = obj.foo`的赋值操作，是将`bar`指向了`foo`在内存中的位置，`foo`前面的`obj`的上下文丢失了。此时则需要更加有效的绑定规则——显式绑定。

### 3.3 显式绑定

运用隐式绑定时，我们必须在一个对象内部包含一个指向函数的指针，并通过这个属性间接引用函数，从而把**this**间接（隐式）绑定到这个对象上。（注意：这句话也是**`call()`**函数实现思路的要点）。

在例3中使用`foo.call(obj);`也是可以输出期望答案2的，但是如果在调用函数的作用域中`obj`的值是undefined，此时调用`foo.call(obj)`同样会出现**绑定丢失**。

此处可以使用**硬绑定**来解决问题，硬绑定是显式的强制绑定，绑定后的函数不能再修改它的**this**。可以使用`Function.prototype.bind()`来执行硬绑定。

### 3.4 new绑定

通过new来调用函数，或者说发生构造函数调用时，会自动执行下面的操作：

1. 创建（或者说构造）一个全新的对象
2. 这个新对象对被执行[[Prototype]]连接
3. 这个新对象会绑定到函数调用的this
4. 如果函数没有返回其他对象，那么new表达式中的函数调用会自动返回这个新对象

``` javascript
// 例4. new绑定
function foo(a) {
    this.a = a;
}
var bar = new foo(2);
console.log(bar.a); // 2

// 例5. 模拟new函数执行过程
function imitateNew() {
    // 1.创建新对象
    var obj = new Object(); 
    // 2.为新对象执行[[Prototype]]连接
    var Context = [].shift.call(arguments);
    // 每个实例的__proto__都会指向构造函数的prototype
    obj.__proto__ = Context.prototype;
    // 3.新对象绑定函数调用的this(调用构造函数)
    var result = Context.apply(obj, arguments);
    // 4.判断是否返回其他对象
    return tyof result === 'object' ? ret : obj;
}
```

## 4 箭头函数

上面的四条规则已经可以包含所有正常的函数，但是ES6中的箭头函数不运用这些特殊的规则。

箭头函数并不是使用function关键字定义的，而是使用被称为“胖箭头”的操作符**`=>`**定义的。它是根据外层作用域来决定**this**的。

``` javascript
// 例6. 箭头函数&模拟箭头函数
function foo(){
    // 返回一个箭头函数
    return (a) => {
        // this继承自foo()
        console.log(this.a);
    };
}
// 上面箭头函数的等价替换
function foo1() {
    var self = this;	// 捕捉词法作用域上的this
    return function() {
        console.log(self.a);
    }
}
var obj1 = { a: 2 };
var obj2 = { a: 3 };
var bar = foo.call(obj1); // 2
bar.call(obj2);			 // 2，不是3
```


【参考链接】

[[译] JavaScript中的“this”是什么？](https://juejin.im/post/5b6676e6f265da0fa00a3a12?utm_source=gold_browser_extension)

《你不知道的JavaScript》（上卷）
