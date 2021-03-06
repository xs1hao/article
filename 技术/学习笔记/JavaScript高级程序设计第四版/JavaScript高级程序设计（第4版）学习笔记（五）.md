---
name: JavaScript高级程序设计（第4版）学习笔记（五）
title: JavaScript高级程序设计（第4版）学习笔记（五）
tags: ["技术","学习笔记","JavaScript高级程序设计第四版"]
categories: 学习笔记
info: "十年磨一剑，红宝书：7. 迭代器与生成器"
time: 2020/10/19
desc: 'javascirpt高级程序设计, 红宝书, 学习笔记'
keywords: ['javascirpt高级程序设计第四版', '前端', '红宝书第四版', '学习笔记']


---

# JavaScript高级程序设计（第4版）学习笔记（五）

## 第 7 章 迭代器与生成器

ES6 新增（照抄其他语言）的高级特性：迭代器和生成器能够更清晰、高效、方便地实现迭代。

### 7.1 理解迭代

最简单的迭代就是 for 循环迭代，但这种方法只适用于数组。还有一些像 forEach 这样的迭代函数，虽然比较常用，但却没有办法标识迭代何时终止，且回调结构也比较笨拙。

在此基础上，ES6 和其他语言一样支持了迭代器模式。

### 7.2 迭代器模式

可迭代对象（iterable）是一种抽象说法，任何实现`Iterable`接口的数据结构都可以被实现`Iterator`接口的结构“消费”（consume）（又是一段看不懂的神翻译.....擦）。

#### 7.2.1 可迭代协议 Iterable && 7.2.2 迭代器协议

（PS：这两小节得一块看才比较好理解，而且又是不该翻译的乱翻译......）

可迭代协议要求同时具备两种能力：

- 支持迭代的自我识别（？？）能力
- 创建实现 Iterator 接口的对象的能力

这意味着必须暴露一个特殊的`Symbol.iterator`作为 key，这个默认迭代器属性必须引用一个工厂函数（生成器），调用该函数会返回一个新的迭代器。

迭代器是一种一次性使用的对象，用于迭代与其关联的可迭代对象。迭代器 API 使用`next()`方法在可迭代对象中遍历数据，每次成功调用，都会返回一个`IteratorResult`对象。（我真是谢谢作者没翻译这个名词，不然险些又看不懂...）

```javascript
// 一个 IteratorResult 类型的对象包含两个属性：done 和 value
// done 代表是否还可以通过调用 next 取得下一个值，value 包含可迭代对象的当前值

// eg. 可迭代对象
let arr = ['foo', 'bar']
// 调用生成器生成
let iter = arr[Symbol.iterator]()
console.log(iter.next()) // {value: "foo", done: false}
console.log(iter.next()) // {value: "bar", done: false}
console.log(iter.next()) // {value: undefined, done: true}
```

迭代器并不知道 怎么从可迭代对象中取得下一个值，也不知道可迭代对象有多大。只要迭代器到达 done: true 状态，后续调用 next 就一直返回同样的值了。

不同迭代器的实例相互之间没有联系，只会独立地遍历可迭代对象。

#### 7.2.3 自定义迭代器

利用这个原理，**就可以自定义迭代器来迭代任何一个自定义对象了**：

```javascript
var a = {}
a[Symbol.iterator] = function () {
  var count = 1
  return {
    next () {
      return {done: count >= 3, value: count++}
    }
  }
}
for (let i of a) {console.log(i)} // 1 2

// 在某些奇葩的情况下，可能会出现要求每个对象只能迭代一次的需求
// 此时可以这样，确保每次生成器生成的都是一样的迭代器
var b = {}
b.count = 1
b.next = function () {
  return {done: this.count >= 3, value: this.count++}
}
b[Symbol.iterator] = function () { return this }
for (let i of b) {console.log(i)} // 1 2
for (let i of b) {console.log(i)} // undefinded
```

#### 7.2.4 提前终止迭代器

在某些情况下，如需要提前关闭迭代的时候，比如`break`或者`throw`的情况下，可以使用`return()`方法执行提前关闭时候的逻辑。（不当人的翻译...简单来说就是个停止迭代时候的钩子函数）

`return()`方法必须返回一个有效的 IteratorResult 对象。简单情况下，可以只返回`{ done: true }`。 

```javascript
var a = {}
a[Symbol.iterator] = function () {
  var count = 1
  return {
    next () {
      return {done: count >= 3, value: count++}
    },
    return () {
      console.log('尚未迭代完成，提前终止')
      return { done: true }
    }
  }
}
for (let i of a) {
  if (i === 2) break
  console.log(i)
} // 1 尚未迭代完成
```

> 因为 return() 方法是可选的，所以并非所有迭代器都是可关闭的。要知道某个迭代器是否可关闭， 可以测试这个迭代器实例的 return 属性是不是函数对象。不过，仅仅给一个不可关闭的迭代器增加这 个方法并不能让它变成可关闭的。这是因为调用 return()不会强制迭代器进入关闭状态。即便如此， return() 方法还是会被调用。

下次如果重新调用这个迭代器，迭代会从暂停的地方重新开始（PS：但这个好像只能在数组的迭代器使用，自定义的不行，**初步猜测也许是上面的自定义 iterator 不是生成器函数的原因**，具体原因暂时不明）。

```javascript
let a = [1, 2, 3, 4, 5]
let iter = a[Symbol.iterator]()
iter.return = function () {
  console.log('提前退出')
  return { done: true }
}
for (let i of iter) {
  console.log(i)
  if (i === 2) break
}
// 1 2 提前退出
for (let i of iter) {
  console.log(i)
}
// 3 4 5
```

### 7.3 生成器

生成器是 ES6 新增的一个极为灵活的结构，拥有一个在函数块内暂停和恢复代码执行的能力。这种能力可以使得开发者方便的自定义迭代器和实现协程。

#### 7.3.1 生成器基础

函数名称前面加一个星号（*）表示它是一个生成器，只要是可以定义函数的地方，就可以定义生成器。

```javascript
function* a () {}
let a = function* () {}
let foo = {
  * a() {}
}
class Foo {
  * a() {}
}
class Bar {
  static * a(){}
}
```

以上都是能成功声明的生成器。

调用生成器函数会产生一个生成器对象，生成器对象一开始处于暂停执行（suspended）状态，与迭代器类似，生成器对象也实现了`Iterator`接口，具有`next()`方法。该方法的返回值也类似于迭代器有一个`done`属性和`value`属性。函数体为空的生成器函数中间不会停留，调用一次 next() 就会让生成器到达 done: true 状态。

生成器函数只会在初次调用`next()`方法后才开始执行。

```javascript
function* a () {
  console.log('a')
}
var iter = a()
iter.next() // a
```

value 属性是生成器函数的返回值，默认为 undefined，可以通过生成器函数的返回值指定。

#### 7.3.2 通过 yield 中断执行

`yield`关键字可以让生成器停止和开始执行，生成器函数在遇到`yield`以后执行会停止，函数作用域的状态会被保留，停止执行的生成器函数只能通过在生成器对象上调用`next()`方法来恢复执行。

通过`yield`关键字退出的生成器函数会处于`done: false`状态，通过 return 关键字退出的生成器函数会处于 done: true 状态。

用途：

1. 生成器对象作为可迭代对象

   显式调用 next 方法的用处并不大，但如果把生成器对象当成可迭代对象，使用起来就会很方便。

   ```javascript
   function* generatorFn () {
     yield 1
     yield 2
     yield 3
   }
   for (const x of generatorFn()) {console.log(x)} // 1 2 3
   ```

   当需要一个固定次数的可迭代对象时，可以使用一个简单的循环来实现一个生成器：

   ```javascript
   function* nTimes(n) {
     while (n--) {
       yield
     }
   }
   for (let i of nTimes(3)) {console.log('foo')} // foo foo foo
   ```

2. 使用`yield`实现输入和输出

   ```javascript
   function* generatorFn(initial){
     console.log(initial)
     console.log(yield)
     console.log(yield)
   }
   var generatorObj = generatorFn('foo')
   generatorObj.next('bar') // foo
   generatorObj.next('baz') // baz
   generatorObj.next('qux') // qux
   ```

   因为函数必须对整个表达式求值才能确定要返回的值，所以它在遇到 yield 关键字时暂停执行并计算出要产生的值："foo"。下一次调用 next()传入了"bar"，作为交给同一个 yield 的值。然后这个值被确定为本次生成器函数要返回的值。

   还可以同时用于输入和输出。

   ```javascript
   function* generatorFn() {
     return yield 'foo'
   }
   var generatorObj = generatorFn()
   generatorObj.next() // { done: false, value: 'foo' }
   generatorObj.next('bar') // { done: true, value: 'baz' }
   ```

3. 产生可迭代对象

   `yield *`可以用于将一个可迭代对象序列化为一连串可以单独产出的值。

   ```javascript
   function* generatorFn() {
     yield* [1, 2, 3]
   }
   ```

   该语法最有用的是实现递归操作，此时生成器可以产生自身。

   ```javascript
   function* nTimes(n) {
     if (n > 0) {
       yield* nTimes(n - 1)
       yield n - 1
     }
   }
   for (const x of nTimes(3)) {console.log(x)} // 0 1 2
   ```

#### 7.3.3 生成器作为默认迭代器

```javascript
var foo = { values: [1, 2, 3, 4] }
foo[Symbol.iterator] = function* () {
  yield* this.values
}
for (const x of foo) {console.log(x)} // 1 2 3 4
```

#### 7.3.4 提前终止生成器

三种方法关闭生成器：

- next()

- return()

  ```javascript
  function* generatorFn() {
    yield *[1, 2, 3]
  }
  var g = generatorFn()
  console.log(g.next()) // { done: false, value: 1 }
  console.log(g.return(4)) // { done: true, value: 4 }
  ```

- throw()

  ```javascript
  function* generatorFn() {
    try {
      yield *[1, 2, 3]
    } catch (e) {
      console.log(e)
    }
  }
  var g = generatorFn()
  console.log(g.next()) // {value: 1, done: false}
  // log err
  console.log(g.throw('foo')) // { done: true, value: undefined }
  console.log(g.next()) // { done: true, value: undefined }
  ```

