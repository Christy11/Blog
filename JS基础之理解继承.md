## 1 继承分类

在《JavaScript高级程序设计》的第6章，把继承分成了以下几种类型：原型链继承、构造函数继承、组合继承、原型式继承、寄生式继承和寄生组合继承。

以上几种继承的类型，可以有两种分类维度：

1. 是否与prototype相关
2. 是否使用Object.create()函数创建

其中第二种分类维度在博客-[一篇文章理解JS继承](https://segmentfault.com/a/1190000015727237)中，有详细解释说明，简单来看可以看下图：


![clipboard.png](https://segmentfault.com/img/bVbd9t9?w=2162&h=990) 

本文主要以第一种分类维度来讨论继承的实现。

## 2 构造函数的属性

在讨论继承之前，我们先看一下构造函数中的属性性质，以及在函数设计中希望其达到的目的。通过理解属性的性质，可以加深对各种继承模式的设计意图的理解。

``` javascript
function Parent(name) {
    this.name = name;	// 实例基本属性（强调私有，不共享）
    this.family = ['Alice', 'Bob'];	// 实例引用属性（强调私有，不共享）
    this.sayName = function() {	// 实例引用属性（强调复用，需要共享）
        console.log('Hello', name);
    }
}
```

由上面函数可知，在构造函数中实例有<u>基本属性</u>和<u>引用属性</u>。其中，数组和函数都属于实例引用属性，单数组强调私有不共享，函数强调复用。也正是针对实例引用是否需要复用，才会衍生出各种不同的继承方式。

## 3 继承方式

上面我们讨论到，继承可以按照是否与prototype相关来分类，在这样的分类方式下，与prototype不相关的其实只有构造函数继承这一种，其他的继承方式均与prototype相关。

### 3.1 借用构造函数

#### 3.1.1 概念

构造函数继承实际上使用的是借用构造函数的技术，借用构造函数的本质是**[复制]**。

> 核心思想：子类型构造函数的内部**调用**父类型构造函数。
>
> **核心代码： Parent.call(Child);**
>
> 优点：父类引用属性不会被共享，子类构造函数中可以向父类传参
>
> 缺点：没有函数复用（每次调用都是新建一个对象，**所有属性均不共享**）

 #### 3.1.2 实例分析

```javascript
function Parent(name) {
    this.name = name;	// 实例基本属性（强调私有，不共享）
    this.family = ['Alice', 'Bob'];	// 实例引用属性（强调私有，不共享）
    this.sayName = function() {	// 实例引用属性（强调复用，需要共享）
        console.log('Hello, ', name);
    }
}
function Child(nameC) {
    Parent.call(this, nameC);	// 继承Parent
    this.age = 22;	// 实例基本属性
}
var instance = new Child('Tony');
console.log(instance.name);	// Tony
console.log(instance.age);	// 22
```

#### 3.1.3 原型链分析

![借用构造函数](https://github.com/Christy11/Blog/blob/master/images/inheritance/1.png)

从图中可知，Parent和Child两个构造函数之间并没有直接在原型链上有所关联，但是由于Child构造函数内部调用了Parent的构造函数，故Child的实例中会有Parent中的属性，即传入的属性值name，而实例上则会把Parent的所有属性都继承下来。

### 3.2 原型链继承

#### 3.2.1 概念

构造函数、原型、实例的关系：每个构造函数都有一个原型对象`Object.prototype`，原型对象都包含一个指向构造函数的指针`Object.prototype.constructor`，实例都包含一个指向原型对象的内部指针`[[prototype]]`。原型链继承的原理便是，重写原型对象，代之以一个新类型的实例。原型链继承的本质是共享父类的方法和属性。

> 核心思想：利用**原型链**让子类型继承父类型的属性和方法。
>
> **核心代码： Child.prototype = new Parent(); Child.prototype.constructor = Child;**
>
> 优点：父类方法得以复用
>
> 缺点：父类属性被所有子类实例共享（**所有属性均共享**），子类构建实例无法向父类传参

注：核心代码第二句有修改子类的构造函数指向的行为，此处会在后面 **[原型对象修复构造函数]**
中解释。

#### 3.2.2 实例分析

``` javascript
function Parent() {
    this.name = 'Tony';	// 实例基本属性（强调私有，不共享）
    this.family = ['Alice', 'Bob'];	// 实例引用属性（强调私有，不共享）
    this.sayName = function() {	// 实例引用属性（强调复用，需要共享）
        console.log('Hello, ', this.name);
    }
}
function Child() {
    this.age = 22;	// 实例基本属性
}
Child.prototype = new Parent();
Child.prototype.constructor = Child;

var instance1 = new Child();
console.log(instance1.name);	// Tony
console.log(instance1.age);	// 22
var instance2 = new Child();
instance1.family.push('Cloud');
console.log(instance1.family);	// ["Alice", "Bob", "Cloud"]
console.log(instance2.family);	// ["Alice", "Bob", "Cloud"]，此处由于原型共享也被修改
```

#### 3.2.3 原型链分析

![原型链继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/2.png)

图中红色的部分，是原型链相较构造函数继承变化的部分。不难发现，由于Child.prototype = new Parent()，Child.prototype对象成为了Parent的实例，故Child.prototype的内部原型属性指向构造函数Parent的原型对象，即Child.prototype.__ proto__ = Parent.prototype，同时，所有的属性都会被共享，基本属性可以通过实例新建同名属性覆盖，而引用属性则是所有实例共享，然而并不是是所有的引用属性都希望被共享，所以单纯的原型链继承使用得很少。

### 3.3 原型式继承

#### 3.3.1 概念

别看原型链继承和原型式继承只有一字之差，两者之间的实现方式还是有很大差异的。之所以把原型式继承写在组合继承前面，是因为我认为借用构造函数、原型链继承、原型式继承是其他继承方式的基础版，现在来看一下原型式继承：

> 核心思想：object()对传入的参数对象进行浅复制。
>
> **核心代码： **
>
> ```javascript
> function object(o) {
>     function F () {}
>     F.prototype = o;
>     return new F();
> }
> // ES5中可以直接使用Object.create()方法来创建函数
> ```
>
> 优点：父类方法可以复用
>
> 缺点：父类属性被所有子类实例共享（**所有属性均共享**），子类构建实例无法向父类传参

仅从优缺点来看，原型链继承与原型式继承的优缺点好像是一样的，那么差别在哪里呢？原型链继承本身是构造函数之间的实例继承，即把一个构造函数的实例指向另一个构造函数的原型对象，构造函数是本来就存在的；而原型式继承的意义在于两个对象之间本没有原型链连接，我通过自己写的函数创建了两个函数的原型链，整体来说这种方式会更加灵活而没有那么多限制。

#### 3.3.2 实例分析

``` javascript
function object(o) {
    function F () {}
    F.prototype = o;
    return new F();
}
var Parent = {
    name: 'Tony',
    family: ['Alice', 'Bob']
}
var instance = object(Parent);
console.log(instance.name);	// Tony
```

不知道你有没有发现，这边没有用`Function Parent()`，而是变量声明`var Parent`，通过`object()`函数，使用临时构造函数`F`，将`Parent`对象指定为了`instance`的原型对象。之所以这样设计，是因为我们**无法直接在两个对象之间创建原型链**，必须要通过一个构造函数，并且指定构造函数的原型，才能将两个对象用原型链连接在一起。

#### 3.3.3 原型链分析

![原型式继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/3.png)

图中灰色的地方便是临时构造函数F()在原型链中的位置。

不难发现，通过临时构造函数F，我们建立起了instance和Parent之间的原型链，通过object函数创建的instance对象本身不会继承父类的任何属性，所有属性访问都是通过在原型链访问的。

### 3.4 组合继承

#### 3.4.1 概念

组合继承指的是将**原型链**和**借用构造函数**的技术混合到一块，从而发挥两者之长的一种继承模式。
> 核心思想：用原型链实现对原型属性和方法的继承，用借用构造函数实现对实例属性的继承。
>
> 优点：父类方法可以复用，父类引用属性不会被共享，子类构建实例时可以向父类传参
>
> 缺点：父类的构造函数被两次调用，第二次调用会覆盖第一次调用的同名参数，造成性能浪费

#### 3.4.2 实例分析

``` javascript
function Parent() {
    this.name = 'Tony';	// 实例基本属性（强调私有，不共享）
    this.family = ['Alice', 'Bob'];	// 实例引用属性（强调私有，不共享）
}
// 原型链上添加可复用的方法
Parent.prototype.sayName = function() {	// 实例引用属性（强调复用，需要共享）
        console.log('Hello, ', this.name);
}
function Child() {
    Parent.call(this);	// 构造函数继承，第二次调用
    this.age = 22;	// 实例基本属性
}
Child.prototype = new Parent();	// 第一次调用
Child.prototype.constructor = Child;

var instance = new Child();
var instance1 = new Child();
console.log(instance1.name);	// Tony
console.log(instance1.age);	// 22
var instance2 = new Child();
instance1.family.push('Cloud');
console.log(instance1.family);	// ["Alice", "Bob", "Cloud"]
console.log(instance2.family);	// ["Alice", "Bob"]，此处引用属性不会被修改
```

#### 3.4.3 原型链分析

![组合继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/4.png)

组合继承相对于原型链继承来说，可以共享原型链上面的方法，而每个实例又有自己独立的属性。是一种比较理想的优化。

### 3.5 寄生式继承

#### 3.5.1 概念

寄生式继承是原型式继承的改良版，它的思路与寄生构造函数和工厂模式类似，即创建一个仅用于封装继承过程的函数，该函数在内部以某种方式来增强对象，然后返回该对象。
> 核心思想：使用原型式继承获得一个浅复制的目标对象，然后增强这个浅复制的能力。
>
> 优点：仅是为寄生组合继承提供一种思路，不讨论优点
>
> 缺点：无法做到函数复用，因为增强功能会给每个实例创建同名的方法（该方法理论上希望共享）

#### 3.5.2 实例分析

``` javascript
function object(o) {
    function F () {}
    F.prototype = o;
    return new F();
}
function enhance(target) {
    var clone = object(target);
    clone.sayName = function() {
        console.log('Hello', clone.name);
    }
    return clone;
}
var Parent = {
    name: 'Tony',
    family: ['Alice', 'Bob']
}
var instance = enhance(Parent);
console.log(instance.sayName());	// Hello, Tony
```

#### 3.5.3 原型链分析

![寄生式继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/5.png)

这个例子的代码基于Parent返回了一个新对象instance。新对象不仅可以访问Parent的所有属性和方法，还有自己的sayNme()函数。

### 3.6 寄生组合式继承

#### 3.6.1 概念

组合继承是JavaScript中最常用的继承模式，但是前面也说过，组合继承会有一个两次调用构造函数的问题，寄生组合式继承便解决了这个问题。
> 核心思想：使用寄生式继承来继承父类的原型，再将结果指定给子类型的原型。
>
> 优点：是实现基于类型继承的最有效方式
>
> 缺点：如果实现起来相对来说比较复杂可以算是缺点←_←

#### 3.6.2 实例分析

``` javascript
function object(o) {
    function F () {}
    F.prototype = o;
    return new F();
}
// 重写subType.prototype，使subType.prototype.__proto__ === superType.prototype
function inheritPrototype(subType, superType) {
    var prototype = object(superType.prototype);	// 创建对象
    prototype.constructor = subType;	// 增强对象
    subType.prototype = prototype;	// 指定对象
}
function Parent(name) {
    this.name = name;	// 实例基本属性（强调私有，不共享）
    this.family = ['Alice', 'Bob'];	// 实例引用属性（强调私有，不共享）
}
// 原型链上添加可复用的方法
Parent.prototype.sayName = function() {	// 实例引用属性（强调复用，需要共享）
      console.log('Hello, ', this.name);
}
function Child(name, age) {
    Parent.call(this, name);	// 构造函数继承，唯一一次调用
    this.age = age;	// 实例基本属性
}
// 因为是对父类原型的复制，所以不包含父类的构造函数，也就不会调用两次父类的构造函数造成浪费
inheritPrototype(Child, Parent);
Child.prototype.sayAge = function() {
    console.log(this.age);
}
var instance = new Child('Nerd', 18);
instance.sayName();	// Hello, Nerd
instance.sayAge();	// 18
```

#### 3.6.3 原型链分析

![寄生组合式继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/6.png)

由图中可以发现，通过原型式继承，将`Child.prototype`和`Parent.prototype`两个对象构建成原型链，再通过`Parent.call(this, name)`的行为，执行一次`Parent`的构造函数。优化了组合继承中，构造函数被两次调用的问题。

总体来看，寄生组合继承模式实现了所有属性相关的需求：每个实例都有自己的不被共享的实例属性，而希望共享的方法也都在原型链上实现。是最理想的继承实现方式。

## 其他
#### 原型对象修复构造函数

在原型链继承中，我们常常会看到这样的代码：

``` javascript
function Parent() {}
Parent.prototype.name = 'Tony';
Parent.prototype.age = 22;
Parent.prototype.sayName = function() {
    console.log(this.name);
}
```

这样每添加一个属性和方法就要敲一遍Parent.prototype看起来很冗余。为减少不必要的输入，也为了从视觉上更好地封装原型的功能，更常见的做法是用一个包含所有属性和方法的对象字面量来重写整个原型对象：

``` javascript
function Parent() {}
Parent.prototype = {
    name: 'Tony',
    age: 22,
    sayName: function() {
       console.log(this.name);
    } 
}
```

这样看起来似乎简洁方便，但是也出现了一个问题，对象原型上的constructor没有了……

![寄生组合式继承](https://github.com/Christy11/Blog/blob/master/images/inheritance/7.png)

我们知道，每创建一个函数，就会同时创建它的prototype对象，这个对象也会自动获得constructor属性。而我们使用后面对象字面量的语法，本质上完全重写了默认的prototype对象（我认为这里可以理解为指向内存另一个由Object创造的新对象），原型对象应有的constructor属性则需要重新赋值，才能够指向到用户期望的函数对象。

故此处还需要重新为constructor赋值：

``` javascript
// 重新设置构造函数
Object.defineProperty(Parent.prototype, 'constructor', {
    enumerable: false,
    value: Parent
});
```


##### [参考书目] 

[一篇文章理解JS继承](https://segmentfault.com/a/1190000015727237)

[js继承、构造函数继承、原型链继承、组合继承、组合继承优化、寄生组合继承](https://segmentfault.com/a/1190000015216289)

《JavaScript高级程序设计》

《你不知道的JavaScript》（上卷）