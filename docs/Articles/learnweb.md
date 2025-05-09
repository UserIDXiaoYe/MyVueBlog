## 学习笔记

# 事件循环

## 浏览器的进程模型

### 何为进程？

程序运行需要有它自己专属的内存空间，可以把这块内存空间简单的理解为进程

<img src="http://mdrs.yuanjin.tech/img/202208092057573.png" alt="image-20220809205743532" style="zoom:50%;" />

每个应用至少有一个进程，进程之间相互独立，即使要通信，也需要双方同意。

### 何为线程？

有了进程后，就可以运行程序的代码了。

运行代码的「人」称之为「线程」。

一个进程至少有一个线程，所以在进程开启后会自动创建一个线程来运行代码，该线程称之为主线程。

如果程序需要同时执行多块代码，主线程就会启动更多的线程来执行代码，所以一个进程中可以包含多个线程。

![image-20220809210859457](http://mdrs.yuanjin.tech/img/202208092108499.png)

### 浏览器有哪些进程和线程？

**浏览器是一个多进程多线程的应用程序**

浏览器内部工作极其复杂。

为了避免相互影响，为了减少连环崩溃的几率，当启动浏览器后，它会自动启动多个进程。

![image-20220809213152371](http://mdrs.yuanjin.tech/img/202208092131410.png)

> 可以在浏览器的任务管理器中查看当前的所有进程

其中，最主要的进程有：

1. 浏览器进程

   主要负责界面显示、用户交互、子进程管理等。浏览器进程内部会启动多个线程处理不同的任务。

2. 网络进程

   负责加载网络资源。网络进程内部会启动多个线程来处理不同的网络任务。

3. **渲染进程**（本节课重点讲解的进程）

   渲染进程启动后，会开启一个**渲染主线程**，主线程负责执行 HTML、CSS、JS 代码。

   默认情况下，浏览器会为每个标签页开启一个新的渲染进程，以保证不同的标签页之间不相互影响。

   > 将来该默认模式可能会有所改变，有兴趣的同学可参见[chrome官方说明文档](https://chromium.googlesource.com/chromium/src/+/main/docs/process_model_and_site_isolation.md#Modes-and-Availability)

## 渲染主线程是如何工作的？

渲染主线程是浏览器中最繁忙的线程，需要它处理的任务包括但不限于：

- 解析 HTML
- 解析 CSS
- 计算样式
- 布局
- 处理图层
- 每秒把页面画 60 次
- 执行全局 JS 代码
- 执行事件处理函数
- 执行计时器的回调函数
- ......

> 思考题：为什么渲染进程不适用多个线程来处理这些事情？

要处理这么多的任务，主线程遇到了一个前所未有的难题：如何调度任务？

比如：

- 我正在执行一个 JS 函数，执行到一半的时候用户点击了按钮，我该立即去执行点击事件的处理函数吗？
- 我正在执行一个 JS 函数，执行到一半的时候某个计时器到达了时间，我该立即去执行它的回调吗？
- 浏览器进程通知我“用户点击了按钮”，与此同时，某个计时器也到达了时间，我应该处理哪一个呢？
- ......

渲染主线程想出了一个绝妙的主意来处理这个问题：排队

![image-20220809223027806](http://mdrs.yuanjin.tech/img/202208092230847.png)

1. 在最开始的时候，渲染主线程会进入一个无限循环
2. 每一次循环会检查消息队列中是否有任务存在。如果有，就取出第一个任务执行，执行完一个后进入下一次循环；如果没有，则进入休眠状态。
3. 其他所有线程（包括其他进程的线程）可以随时向消息队列添加任务。新任务会加到消息队列的末尾。在添加新任务时，如果主线程是休眠状态，则会将其唤醒以继续循环拿取任务

这样一来，就可以让每个任务有条不紊的、持续的进行下去了。

**整个过程，被称之为事件循环（消息循环）**

## 若干解释

### 何为异步？

代码在执行过程中，会遇到一些无法立即处理的任务，比如：

- 计时完成后需要执行的任务 —— `setTimeout`、`setInterval`
- 网络通信完成后需要执行的任务 -- `XHR`、`Fetch`
- 用户操作后需要执行的任务 -- `addEventListener`

如果让渲染主线程等待这些任务的时机达到，就会导致主线程长期处于「阻塞」的状态，从而导致浏览器「卡死」

![image-20220810104344296](http://mdrs.yuanjin.tech/img/202208101043348.png)

**渲染主线程承担着极其重要的工作，无论如何都不能阻塞！**

因此，浏览器选择**异步**来解决这个问题

![image-20220810104858857](http://mdrs.yuanjin.tech/img/202208101048899.png)

使用异步的方式，**渲染主线程永不阻塞**

> 面试题：如何理解 JS 的异步？
>
> 参考答案：
>
> JS是一门单线程的语言，这是因为它**运行在浏览器的渲染主线程中，而渲染主线程只有一个**。
>
> 而渲染主线程承担着诸多的工作，渲染页面、执行 JS 都在其中运行。
>
> 如果使用同步的方式，就极有可能导致主线程产生阻塞，从而导致消息队列中的很多其他任务无法得到执行。这样一来，一方面会导致繁忙的主线程白白的消耗时间，另一方面导致页面无法及时更新，给用户造成卡死现象。
>
> 所以浏览器采用异步的方式来避免。具体做法是当某些任务发生时，比如计时器、网络、事件监听，主线程将任务交给其他线程去处理，自身立即结束任务的执行，转而执行后续代码。当其他线程完成时，**将事先传递的回调函数包装成任务**，加入到消息队列的末尾排队，等待主线程调度执行。
>
> 在这种异步模式下，浏览器永不阻塞，从而最大限度的保证了单线程的流畅运行。

### JS为何会阻碍渲染？

先看代码

```html
<h1>Mr.Yuan is awesome!</h1>
<button>change</button>
<script>
  var h1 = document.querySelector('h1');
  var btn = document.querySelector('button');

  // 死循环指定的时间
  function delay(duration) {
    var start = Date.now();
    while (Date.now() - start < duration) {}
  }

  btn.onclick = function () {
    h1.textContent = '袁老师很帅！';
    delay(3000);
  };
</script>
```

点击按钮后，会发生什么呢？

渲染主线程将最初的页面渲染完成后，等待消息队列任务，而消息队列为空，此时，交互线程监听到按钮点击，点击执行后执行 fn ，将 fn 放进消息队列，再进入渲染主线程，但是修改了h1之后没有立即渲染，而是等待死循环结束后，消息队列的渲染任务才进入渲染主线程。

js和渲染都在渲染主线程上，处理js代码时，渲染任务处于等待状态。

### 任务有优先级吗？

任务没有优先级，在消息队列中先进先出

但**消息队列是有优先级的**

根据 W3C 的最新解释:

- 每个任务都有一个任务类型，同一个类型的任务必须在一个队列，不同类型的任务可以分属于不同的队列。
  在一次事件循环中，浏览器可以根据实际情况从不同的队列中取出任务执行。
- 浏览器必须准备好一个微队列，微队列中的任务优先所有其他任务执行
  https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint

> 随着浏览器的复杂度急剧提升，W3C 不再使用宏队列的说法

在目前 chrome 的实现中，至少包含了下面的队列：

- 延时队列：用于存放计时器到达后的回调任务，优先级「中」
- 交互队列：用于存放用户操作后产生的事件处理任务，优先级「高」
- 微队列：用户存放需要最快执行的任务，优先级「最高」

> 添加任务到微队列的主要方式主要是使用 Promise、MutationObserver
>
> 例如：
>
> ```js
> // 立即把一个函数添加到微队列
> Promise.resolve().then(函数)
> ```

> 浏览器还有很多其他的队列，由于和我们开发关系不大，不作考虑

> 面试题：阐述一下 JS 的事件循环
>
> 参考答案：
>
> 事件循环又叫做消息循环，是浏览器渲染主线程的工作方式。
>
> 在 Chrome 的源码中，它开启一个不会结束的 for 循环，每次循环从消息队列中取出第一个任务执行，而其他线程只需要在合适的时候将任务加入到队列末尾即可。
>
> 过去把消息队列简单分为宏队列和微队列，这种说法目前已无法满足复杂的浏览器环境，取而代之的是一种更加灵活多变的处理方式。
>
> 根据 W3C 官方的解释，每个任务有不同的类型，同类型的任务必须在同一个队列，不同的任务可以属于不同的队列。不同任务队列有不同的优先级，在一次事件循环中，由浏览器自行决定取哪一个队列的任务。但浏览器必须有一个微队列，微队列的任务一定具有最高的优先级，必须优先调度执行。

> 面试题：JS 中的计时器能做到精确计时吗？为什么？
>
> 参考答案：
>
> 不行，因为：
>
> 1. 计算机硬件没有原子钟，无法做到精确计时
> 2. 操作系统的计时函数本身就有少量偏差，由于 JS 的计时器最终调用的是操作系统的函数，也就携带了这些偏差
> 3. 按照 W3C 的标准，浏览器实现计时器时，如果嵌套层级超过 5 层，则会带有 4 毫秒的最少时间，这样在计时时间少于 4 毫秒时又带来了偏差
> 4. 受事件循环的影响，计时器的回调函数只能在主线程空闲时运行，因此又带来了偏差

### github删除仓库文件

github现在可以直接在网页端操作删除文件夹<br><br>
![选中“Delete directory”直接删除文件夹](./images/image.png)

### JavaScript数组方法

参考自CSDN：[青松pine](https://blog.csdn.net/qq_43223007)：[JavaScript中的数组方法总结+详解](https://blog.csdn.net/qq_43223007/article/details/110201463)

数组方法总结：

| 顺序 | 方法名        | 功能                                                         | 返回值                                             | 是否改变原数组 | 版本 |
| ---- | ------------- | ------------------------------------------------------------ | -------------------------------------------------- | -------------- | ---- |
| 1    | push()        | (在结尾)向数组添加一或多个元素                               | 返回新数组长度                                     | Y              | ES5- |
| 2    | unshift()     | （在开头)向数组添加一或多个元素                              | 返回新数组长度                                     | Y              | ES5- |
| 3    | pop()         | 删除数组的最后一位                                           | 返回被删除的数据                                   | Y              | ES5- |
| 4    | shift()       | 移除数组的第一项                                             | 返回被删除的数据                                   | Y              | ES5- |
| 5    | reverse()     | 反转数组中的元素                                             | 返回反转后数组                                     | Y              | ES5- |
| 6    | sort()        | 以字母顺序(字符串Unicode码点)对数组进行排序                  | 返回新数组                                         | Y              | ES5- |
| 7    | splice()      | 在指定位置删除指定个数元素再增加任意个数元素 （实现数组任意位置的增删改) | 返回删除的数据所组成的数组                         | Y              | ES5- |
| 8    | concat()      | 通过合并（连接）现有数组来创建一个新数组                     | 返回合并之后的数组                                 | N              | ES5- |
| 9    | join()        | 用特定的字符,将数组拼接形成字符串 (默认",")                  | 返回拼接后的字符串                                 | N              | ES5- |
| 10   | slice()       | 裁切指定位置的数组                                           | 被裁切的元素形成的数组                             | N              | ES5- |
| 11   | toString()    | 将数组转换为字符串                                           | 字符串                                             | N              | ES5- |
| 12   | valueOf()     | 查询数组原始值                                               | 数组的原始值                                       | N              | ES5- |
| 13   | indexOf()     | 查询某个元素在数组中第一次出现的位置                         | 存在该元素,返回下标,不存在 返回 -1                 | N              | ES5- |
| 14   | lastIndexOf() | 反向查询数组某个元素在数组中第一次出现的位置                 | 存在该元素,返回下标,不存在 返回 -1                 | N              | ES5- |
| 15   | forEach()     | (迭代) 遍历数组,每次循环中执行传入的回调函数                 | 无/(undefined)                                     | N              | ES5- |
| 16   | map()         | (迭代) 遍历数组, 每次循环时执行传入的回调函数,根据回调函数的返回值,生成一个新的数组 | 有/自定义                                          | N              | ES5- |
| 17   | filter()      | (迭代) 遍历数组, 每次循环时执行传入的回调函数,回调函数返回一个条件,把满足条件的元素筛选出来放到新数组中 | 满足条件的元素组成的新数组                         | N              | ES5- |
| 18   | every()       | (迭代) 判断数组中所有的元素是否满足某个条件                  | 全都满足返回true 只要有一个不满足 返回false        | N              | ES5- |
| 19   | some()        | (迭代) 判断数组中是否存在,满足某个条件的元素                 | 只要有一个元素满足条件就返回true,都不满足返回false | N              | ES5- |
| 20   | reduce()      | (归并)遍历数组, 每次循环时执行传入的回调函数,回调函数会返回一个值,将该值作为初始值prev,传入到下一次函数中 | 最终操作的结果                                     | N              | ES5- |
| 21   | reduceRight() | (归并)用法同reduce,只不过是从右向左                          | 同reduce                                           | N              | ES5- |
| 22   | includes()    | 判断一个数组是否包含一个指定的值.                            | 是返回 true，否则false                             | N              | ES6  |
| 23   | Array.from()  | 接收伪数组,返回对应的真数组                                  | 对应的真数组                                       | N              | ES6  |
| 24   | find()        | 遍历数组,执行回调函数,回调函数执行一个条件,返回满足条件的第一个元素,不存在返回undefined | 满足条件第一个元素/否则返回undefined               | N              | ES6  |
| 25   | findIndex()   | 遍历数组,执行回调函数,回调函数接受一个条件,返回满足条件的第一个元素下标,不存在返回-1 | 满足条件第一个元素下标,不存在=>-1                  | N              | ES6  |
| 26   | fill()        | 用给定值填充一个数组                                         | 新数组                                             | Y              | ES6  |
| 27   | flat()        | 用于将嵌套的数组“拉平”，变成一维的数组。                     | 返回一个新数组                                     | N              | ES6  |
| 28   | flatMap()     | flat()和map()的组合版 , 先通过map()返回一个新数组,再将数组拉平( 只能拉平一次 ) | 返回新数组                                         | N              | ES6  |

1. **Arr.push() ：在数组最后一位添加一个或多个元素,并返回新数组的长度,改变原数组.(添加多个元素用逗号隔开)**

```        javascript
const arr = [1, 2, "c"];
console.log(arr);
const rel = arr.push("A", "B");//返回的是数组长度
console.log(arr); // [1, 2, "c", "A", "B"]
console.log(rel); //  5  (数组长度)
```
2. **Arr.unshift() ：在数组第一位添加一个或多个元素，并返回新数组的长度，改变原数组。(添加多个元素用逗号隔开)**

```javascript
const arr = [1, 2, "c"];
const rel = arr.unshift("A", "B");
console.log(arr); // [ "A", "B",1, 2, "c"]
console.log(rel); //  5  (数组长度)
```
3. **Arr.pop() ：删除数组的最后一位，并且返回删除的数据，会改变原来的数组。(该方法不接受参数,且每次只能删除最后一个)**

```javascript
const arr = [1, 2, "c"];
const rel = arr.pop();
console.log(arr); // [1, 2]
console.log(rel); // c

```
4. **Arr.shift() ：删除数组的第一位数据，并且返回被删除的数据，会改变原来的数组。(该方法不接受参数,且每次只能删除数组第一个)**
```javascript
const arr = ["a","b", "c"];
const rel = arr.shift();
console.log(arr); // ['b', "c"]
console.log(rel); // a

```
5. **Arr.reverse() 将数组的数据进行反转，并且返回反转后的数组，会改变原数组**
```javascript
const arr = [1, 2, 3, "a", "b", "c"];
const rel = arr.reverse();
console.log(arr); //    ["c", "b", "a", 3, 2, 1]
console.log(rel); //    ["c", "b", "a", 3, 2, 1]

```
6. **Arr.sort() <strong>相当重要</strong>,对原数组进行操作，会改变原来的数组**

如果单纯使用sort()不加任何参数：
```javascript
const arr1 = [10, 1, 5, 2, 3];
arr1.sort();
console.log(arr1);//1，10，2，3，5
```
得到的结果并不是有序的（按照unicode排列），因此经常配合参数及语法使用：
```javascript
//正序排列
const arr = [10, 1, 5, 2, 3];
arr.sort(function (a, b) {
return a - b;
});
console.log(arr);//1,2,3,5,10
```
```javascript
//倒序排列
const arr = [10, 1, 5, 2, 3];
arr.sort(function (a, b) {
return b - a;
});
console.log(arr);//1,2,3,5,10
```
特殊的，可以按照某个属性进行排列：
```javascript
const arr1 = [
{ name: "老八", age: "38" },
{ name: "赵日天", age: "28" },
{ name: "龙傲天", age: "48" },
];
arr1.sort(function (a, b) {
console.log(a, b);
return b.age - a.age;//按照age属性倒序即从大到小排列
});
console.log(arr1);
```
7. **Arr.splice(index,howmany,item1,…,itemX) ：向数组中添加，或从数组删除，或替换数组中的元素，然后返回被删除/替换的元素所组成的数组。可以实现数组的增删改。**

参数：

index：必须的，且为整数（可以为负数。负数即为从数组尾部往前数），规定添加/删除元素的位置（下标位置）；

howmany：必须的，可以为从0开始的整数，规定要删除的项目数量（为0不会删除）；

item1, …, itemX：可选，向数组添加的新项目。
```javascript
const arr = ["a", "b", "c", 2, 3, 6];
const rel = arr.splice(2, 1, "add1", "add2");//删除"c",添加"add1","add2"
console.log(arr);   //改变后的原数组   
console.log(rel);	//拆下来的新数组 "c" 
```
8. **Arr.concat() ：拼接数组：如果拼接的是数组，则将数组展开,之后将数组中的每一个元素放到新数组中；如果是其他类型, 直接放到新数组中；如果不给该方法任何参数，将返回一个和原数组一样的数组（复制数组）**

```javascript
const arr1 = [1, 2, 3];
const arr2 = ["a", "b", "c"];
const arr3 = ["A", "B", "C"];
const rel = arr1.concat(arr2, arr3);
console.log(arr1); //原数组 [1, 2, 3]
console.log(rel); //新数组 [1, 2, 3 , "a", "b", "c" ,"A", "B", "C" ]

```

9. **Arr.join() ：用特定的字符,将数组拼接形成字符串 (默认",")**

```   javascript
const list = ["a", "b", "c", "d"];
let result = list.join("-");     //"a-b-c-d"
console.log(result);
result = list.join("/");     //"a/b/c/d"
console.log(result);
result = list.join("");      //"abcd"
console.log(result);
result = list.join();        //  "a,b,c,d"
console.log(result);    
```

10. **Arr.slice()：裁切指定位置的数组，返回值为被裁切的元素形成的新数组 ，<u>不改变原数组</u>**
    **同concat() 方法 slice() 如果不传参数,会使用默认值,得到一个与原数组元素相同的新数组 (复制数组)**

```javascript
const list = ["a", "b", "c", "d"];
const result = list.slice(1, 3);
console.log(result);  // ["b", "c"]
console.log(list); //['a', 'b', 'c', 'd']
```

11. **Arr.toString()：直接将数组转换为字符串,返回值是String,不改变原数组，类似于Arr.join()不添加任何参数，得到的是引号包裹的字符串**

```javascript
const list = ["a", "b", "c", "d"];
const rel = list.toString();
console.log(rel);   // "a,b,c,d"   (字符串类型，不是数组)
console.log(list);  // ['a', 'b', 'c', 'd']
```

12. **Arr.valueOf()：返回数组的原始值（一般情况下其实就是数组自身）**

```javascript
const list = [1, 2, 3, 4];
const rel = list.valueOf();
console.log(list); // [1, 2, 3, 4]
console.log(rel); // [1, 2, 3, 4]
```

[关于toString()和valueOf()](https://segmentfault.com/a/1190000025126691)

13. **Arr.lastIndexOf()：查询某个元素在数组中最后一次出现的位置 (或者理解为反向查询第一次出现的位置) 存在该元素,返回下标,不存在 返回 -1 (可以通过返回值 变相的判断是否存在该元素)**

```javascript
const list = [1, 2, 3, 4];
let index = list.lastIndexOf(4); //3
console.log(index);
index = list.lastIndexOf("4"); //-1
console.log(index);
```

14. **Arr.indexOf()：查询某个元素在数组中第一次出现的位置 存在该元素,返回下标,不存在 返回 -1 (可以通过返回值 变相的判断是否存在该元素)**

```javascript
const list = [1, 2, 3, 4];
let index = list.indexOf(4); //3
console.log(index);
index = list.indexOf("4"); //-1
console.log(index);
```

15. **Arr.forEach()：遍历数组,每次循环中执行传入的回调函数 。没有返回值,或理解为返回值为undefined,不改变原数组。**

语法:

```javascript
arr[].forEach(function(value,index,array){
　 　//do something
})
```

```javascript
const list = [32, 93, 77, 53, 38, 87];
const res = list.forEach(function (item, index, array) {
  console.log(item, index, array);
});
console.log(res);
// 32 0 [32, 93, 77, 53, 38, 87]
// 93 1 [32, 93, 77, 53, 38, 87]
// 77 2 [32, 93, 77, 53, 38, 87]
// 53 3 [32, 93, 77, 53, 38, 87]
// 38 4 [32, 93, 77, 53, 38, 87]
// 87 5 [32, 93, 77, 53, 38, 87]
```

16. **Arr.map()：遍历数组, 每次循环时执行传入的回调函数,根据回调函数的返回值,生成一个新的数组 ,**
    **同forEach() 方法,但是map()方法有返回值,可以return出来**

语法：

```javascript
arr[].map(function(item,index,array){
　　//do something
　　return XXX
})
```

用法：

```javascript
const list = [32, 93, 77, 53, 38, 87];
const res = list.map(function (item, index, array) {
    return item + 5 * 2;
});
console.log("原数组", list); //[32, 93, 77, 53, 38, 87]
console.log("新数组", res); //[42, 103, 87, 63, 48, 97]
```

17. **Arr.filter()：遍历数组, 每次循环时执行传入的回调函数,回调函数返回一个条件,把满足条件的元素筛选出来放到新数组中。**

语法：

```javascript
arr[].filter(function(item,index,array){
　　//do something
　　return XXX //条件
})
```

参数： item:每次循环的当前元素, index:当前项的索引, array:原始数组

用法：

```javascript
const list = [32, 93, 77, 53, 38, 87];
const resList = list.filter(function (item, index, array) {
    return item >= 60; // true || false
});
console.log(resList);//[93, 77, 87]
```

18. **Arr.every()：遍历数组, 每次循环时执行传入的回调函数,回调函数返回一个条件,全都满足返回true 只要有一个不满足 返回false => <u>判断数组中所有的元素是否满足某个条件</u>**

```javascript
function isBigEnough(element, index, array) {
  return element >= 10;
}
[12, 5, 8, 130, 44].every(isBigEnough); // false
[12, 54, 18, 130, 44].every(isBigEnough); // true
```

19. **Arr.some()：遍历数组, 每次循环时执行传入的回调函数,回调函数返回一个条件,只要有一个元素满足条件就返回true,都不满足返回false => <u>判断数组中是否存在满足某个条件的元素</u>**

```javascript
function isBiggerThan10(element, index, array) {
  return element > 10;
}

[2, 5, 8, 1, 4].some(isBiggerThan10); // false
[12, 5, 8, 1, 4].some(isBiggerThan10); // true
```

20. **Arr.reduce()：遍历数组, 每次循环时执行传入的回调函数,回调函数会返回一个值,将该值作为初始值prev,传入到下一次函数中, 返回最终操作的结果**

语法：

```javascript
arr.reduce( function(prev,item,index,array){
    
} , initVal)
```

用法1：不设置初始值的累加（不设置初始值跳过第一次循环，prev默认等于第一个值）

```javascript
const arr = [2, 3, 4, 5];
let sum = arr.reduce(function (prev, item, index, array) {
    console.log(prev, item, index, array);
    return prev + item;
});
console.log(arr, sum);
```

用法2：设置初始值的累加

```javascript
const arr = [2, 3, 4, 5];
let sum = arr.reduce(function (prev, item, index, array) {
  console.log(prev, item, index, array);
  return prev + item;
}, 0);//设置0为初始值，最终结果相同，但多循环一次
console.log(arr, sum);
```

21. **Arr.reduceRight()：用法和reduce()相同，但从右向左累加**
22. **Arr.includes()：用来判断一个数组是否包含一个指定的值，如果是返回 true，否则false。**

```javascript
const site = ['runoob', 'google', 'taobao'];
site.includes('runoob'); // true 
site.includes('baidu'); // false
```

23. **Arr.from()：将一个类数组对象或者可遍历对象转换成一个真正的数组**

将一个类数组对象转换为一个真正的数组，必须具备以下条件：

（1）该 伪数组 / 类数组 对象必须具有length属性，用于指定数组的长度。如果没有length属性，那么转换后的数组是一个空数组。
（2）该 伪数组 / 类数组 对象的属性名必须为数值型或字符串型的数字

```javascript
let all = {
  0: "张飞",
  1: "28",
  2: "男",
  3: ["率土", "鸿图", "三战"],
  length: 4,
};
let list = Array.from(all);
console.log(all);
console.log(list, Array.isArray(list));
```

24. **Arr.find()：遍历数组 每次循环 执行回调函数,回调函数接受一个条件 返回满足条件的第一个元素,不存在则返回undefined**

参数：
item:必须 , 循环当前元素
index:可选 , 循环当前下标
array:可选 , 当前元素所属的数组对象

用法：

```javascript
const list = [55, 66, 77, 88, 99, 100];
let res= list.find(function (item, index, array) {
  return item > 60;
});
console.log(res); //66
```

```javascript
  //快速查找对象数组满足条件的项
  let arr = [{ id: 1, name: 'coco' }, { id: 2, name: 'dudu' }]
  let res = arr.find(item => item.id == 1)//箭头函数简写
  console.log('res', res)  //res {id: 1, name: "coco"}
```

相较于filter：find返回第一个匹配的单个元素；filter返回所有匹配元素组成的数组。

25. **Arr.findIndex()：遍历数组,执行回调函数,回调函数接受一个条件,返回满足条件的第一个元素下标,不存在则返回-1**

参数：
item:必须 , 循环当前元素
index:可选 , 循环当前下标
array:可选 , 当前元素所属的数组对象

```javascript
const list = [55, 66, 77, 88, 99, 100];
let index = list.findIndex(function (item, index, array) {
  console.log(item, index, array);
  return item > 60;
});
console.log(index); // 1
```

```javascript
//快速查找对象数组满足条件的索引，indexOf不支持
let arr = [{ id: 1, name: 'coco' }, { id: 2, name: 'dudu' }]
let res = arr.findIndex(item => item.id == 1)
console.log('res', res)  //res 0
```



相较于indexOf：indexOf是传入一个值.找到了也是返回索引,没有找到也是返回-1 ,属于ES5；findIndex是传入一个测试条件,也就是函数,找到了返回当前项索引,没有找到返回-1. 属于ES6

26. **Arr.fill()：用给定值填充一个数组**

参数：
value 必需。填充的值。
start 可选。开始填充位置。
end 可选。停止填充位置 (默认为 array.length)

```javascript
 const result = ["a", "b", "c"].fill("填充", 1, 2);//["a", "填充", "c"]
```

27. **Arr.flat()：用于将嵌套的数组"拉平",变成一维的数组。该方法返回一个新数组，对原数据没有影响。**

```javascript
const list = [1, 2, [3, 4, [5]]];
const arr = list.flat(); // 默认拉平一次
console.log("拉平一次", arr);//[1, 2, 3, 4, [5]]

const arr = list.flat(2); // 拉平2次
console.log("拉平两次", arr);//[1, 2, 3, 4, 5]
```

28. **Arr.flatMap()： flat()和map()的组合版 , 先通过map()返回一个新数组,再将数组拉平( 只能拉平一次 )**

```javascript
const list = [55, 66, 77, 88, 99, 100];
const newArr = list.map(function (item, index) {
    return [item, index];
});
console.log("Map方法:", newArr);
const newArr2 = list.flatMap(function (item, index) {
    return [item, index];
});
console.log("flatMap方法:", newArr2);
```

输出结果：

```javascript
Map方法: [
  [55, 0], [66, 1], [77, 2], 
  [88, 3], [99, 4], [100, 5]
]
flatMap方法: [
  55, 0, 66, 1, 77, 2, 
  88, 3, 99, 4, 100, 5
]
```

### pinia学习记录

#### pinia引入

1. 使用包管理器下载pinia

```
npm install pinia
```

2. 在vue项目中引入

main.ts中引入、创建、安装

```javascript
//引入pinia
import { createPinia } from 'pinia'

//创建pinia
const pinia = createPinia()

//安装pinia
app.use(pinia)
```

2. pinia使用——存储数据

创建store文件夹、创建ts文件存放数据

```javascript
// ./store/count.ts
import { defineStore } from "pinia";
export const useCountStore = defineStore('count',{
    //真正存储数据的地方
    state(){//state写成一个函数 返回一个对象
        return{
            sum:6
        }
    }
})
```

可以在组件中直接使用

```javascript
// ./components/Count.vue
// 引入
import {useCountStore} from '@/store/count'
//使用useCountStore 得到一个专门保存count相关的store
const countStore = useCountStore()
console.log(countStore.sum);//6
```

3. pinia使用——修改数据

pinia是“符合直觉的”状态管理库，拿到的数据可以直接修改，pinia也提供$patch批量修改，也可以借助action封装方法

```javascript
function add(){
    //第一种直接修改
    countStore.sum += 1
    //第二种批量修改 借助$patch分发碎片
    countStore.$patch({
        sum:888,
        //多个参数需要修改时适用
    })
    //第三种 action
    //逻辑复杂时适用
    countStore.increment(n.value)
}
```

使用action方法，对应的store里面要定义action

```javascript
//新增action
export const useCountStore = defineStore('count',{
    //放置方法 用于响应“动作”
    actions:{
        increment(value:number){
            this.sum += value
        }
    },
    //真正存储数据的地方
    state(){//state写成一个函数 返回一个对象
        return{
            sum:6
        }
    }
})
```

