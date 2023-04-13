---
layout: post
categories: JavaScript
description: none
keywords: JavaScript
---
# JavaScript模块
随着ES6的出现，js模块已经成为正式的标准了。

## 模块历史
曾经为了解决js模块问题而发展起来的民间秘籍，requireJs(AMD)、SeaJs(CMD)、Node(CommonJs)，已经或者不久的将来会成为历史。了解历史也是很重要的，因为正式标准就是以民间秘籍为基础而发展起来的，有些规范仍然被广泛应用于开发中（CommonJS）。

ES6的class是面向对象的语法糖，升级了ES5的构造函数的原型链继承的写法，并没有解决模块化问题。Module功能则是为了解决这一问题而提出的。

在ES6之前，社区制定了一些模块加载方案，最主要的有CommonJS和AMD两种。CommonJS用于服务器，AMD用于浏览器。

ES6模块的设计思想是尽量静态化，使得编译时就能确定模块的依赖关系，以及输入和输出的变量。

ES6模块不是对象，而是通过export命令显式指定输出的代码，输入时也采用静态命令的形式。

## 什么是模块
一个模块（module）就是一个文件。一个脚本就是一个模块。就这么简单。

模块可以相互加载，并可以使用特殊的指令 export 和 import 来交换功能，从另一个模块调用一个模块的函数：
- export 关键字标记了可以从当前模块外部访问的变量和函数。
- import 关键字允许从其他模块导入功能。

```shell
import { Fun1, Fun2, Fun3 } from 'file';
```
以上代码的实质是从file模块加载3个方法Fun1、Fun2和Fun3，其他方法不加载。这种加载方式称为编译时加载，即ES6能在编译时就完成模块编译。

一个模块就是一个独立的文件。该文件内部的所有变量，外部无法获取。如果想要外接获取模块内部的某个变量或方法，就必须使用export关键字输出该变量。
```shell
// profile.js
export var firstName = "John";
export var lastName = "Jackson";
export var year = 1999;
```
上述代码表示在profile.js文件（模块）中，输出了3个变量。

这种写法等价于：
```shell
var firstName = "John";
var lastName = "Jackson";
var year = 1999;
export { firstName, lastName, year };
```

在export命令后面使用大括号指定了所要输出的一组变量，等价于在每个变量前面加export。

export命令除了可以输出变量，也可以输出函数和类，写法相同。

export输出的变量就是本来的名字，但是可以使用as关键字重命名。
```shell
function f1() {}
function f2() {}
export { v1 as Fun1, v2 as Fun2, v2 as Foo };
```

可以使用as关键字对同一个变量或方法重命名两次，使其在引入模块中，可以使用不同的名字进行引用。

export命令可以出现在模块的任意顶层作用域的位置，不能出现在块级作用域内。

export语句输出的值是动态绑定的，绑定其所在的模块。

## import命令
使用export命令定义了模块的对外接口后，其他JS文件就可以通过import命令加载这个模块的接口。
```shell
// main.js
import { firstName, lastName, year } from './profile';
```
import命令接受一个对象，里面指定了要从其他模块导入的变量名。对象中的变量名必须和要加载的模块导出的接口变量名一致（如果接口没有使用as关键字，就使用原始变量名，如果使用as关键字，则使用重命名后的接口名）。

同理，如果要对引入的变量名进行重命名，可以在import命令中使用as关键字，将输入的变量重命名。
```shell
// main.js
import { firstName as surname } from './profile';
```
上述写法中，需要书写每个接口的变量名，如果是要对整个模块进行整体加载，可以使用星号（*）指定一个对象，将输出模块的所有接口都加载到这个对象上。
```shell

// main.js
import * as person from './profile';
```
import命令具有提升效果，会提升到整个模块的顶部首先执行。

## export default命令
import命令在加载变量名或函数名时，需要事先知道导出的模块的接口名，否则无法加载。可以使用export default命令指定模块的默认输出接口。
```shell
// profile.js
export default function () {
    console.log("my name is John Jackson, I was born in 1999");
}
```
上述代码中，profile模块默认输出的是一个函数。这样，导入模块就可以不用指定要加载的接口名了。

```
// main.js
import myName from './profile';
myName(); // "my name is John Jackson, I was born in 1999"
```

在main.js文件中，myName指代的就是profile文件输出的默认接口。这意味着import命令可以用任意名称指向profile文件输出的默认接口，而不需要知道接口名。

export default命令用在非匿名函数前也是可以的，此时函数名在模块外部是无效的，加载时视同匿名函数。
```
// profile.js
export default function sayName () {
    console.log("my name is John Jackson, I was born in 1999");
}
 
// main.js
import myName from './profile';
myName(); // "my name is John Jackson, I was born in 1999"
```
一个模块只能有一个默认输出，因此export default在一个模块中只能使用一次。所以，对应的import命令可以不加大括号。

如果要在一条import命令中同时引入默认方法和其他变量，可以写成以下这样：
```shell
import customName, { otherMethod } from './module-name';

```
customName指代默认接口的命名，otherMethod指代其他接口。

## 模块的继承
模块之间可以继承。假设有个Circle模块继承了Shape模块。
```shell

// cicle.js
export * from 'Shape';
export var pi = 3.14159265359;
export default function area(r) {
    return pi * r * r;
}
```
export * 表示输出模块Shape的所有属性和方法，但不会输出Shape的默认方法。

## module命令
如果要整体加载模块，可以使用module命令代替上述的import * as命令。module命令不会加载模块的默认方法。需要额外使用import命令加载模块的默认方法。
```shell
// main.js
module person from './profile';
```

## ES6模块加载的实质
ES6模块输出的是值的引用。

ES6模块遇到模块加载命令import时不会去执行模块，只会生成一个动态的只读引用。等到真的需要用到时，再到模块中取值。

ES6的输入有点像UNIX系统的“符号链接”，原始值变了，输入值也会跟着变。因此，ES6模块是动态引用，并且不会缓存值，模块里面的变量绑定其所在的模块。
```shell
// lib.js
export let count = 3;
export function add() { count++; }
 
// main.js
import { count, add } from './lib';
console.log(count); // 3
add();
console.log(count); // 4
```
由于ES6输入的模块变量只是一个“符号链接”，所以这个变量是只读的，对它进行重新赋值会报TypeError异常。

```shell
// lib.js
export let obj = {};
 
// main.js
import { obj } from './lib';
obj.prop = 123; // OK
obj = {}; // TypeError
```
obj指向的地址是只读的，无法为其重新赋值。