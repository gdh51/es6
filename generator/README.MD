# Generator函数

## 特征
+ `function`关键字与函数名之间有一个`*`
+ 函数体内部使用`yield`定义不同内部状态
```js
function *generator(){
  yield 1;
  yield 2;
  return 3;
}
var start = generator();//返回一个遍历器对象(Iterator)
start.next();//{value:1,done:false}
start.next();//{value: 2, done: false}
start.next();//{value: 3, done: true}
```
每次调用`next`，指针都会指向下一个`yield`或`return`，并执行指针移动间的代码。

## yield表达式

yield表达式后跟的表达式只有在当前指针指向时，才会进行计算求值
```js
function *generator(){
  yield 1+2;//若指针未指向此处便不会对1+2进行运算
}
```

### Generator中next()运行逻辑
1. 初次调用时：遇到`yield`表达式便停止操作，返回`yield`后面表达式值作为返回对象的value属性
2. 每次调用`next()`方法重复执行 1 步骤
3. 如果遇到`return`便返回后跟的值作为返回对象value属性并结束函数运行。
4. 如果没遇到`return`便运行到函数结束，并将 `undefined`作为返回对象的value属性

### Generator作为暂缓执行函数
当函数为`Generator`函数时，只有当调用`next()`方法时才会执行
```js
function* generator() {
  console.log('running...');
}
var start = generator();//调用该方法时，没有任何反应
start.next();//控制台打印 running...
```

### yield表达式存在的位置
+ 只能存在于`Generator`函数中(*必须是最顶层作用域*)
  ```js
  function* generator() {
    let arr=[1,2,3];
    arr.forEach((value)=>{
      yield value;
    });
  }
  //Uncaught SyntaxError: Unexpected identifier
  ```
+ 处于表达式中时，必须要用`()`包裹
  ```js
  function* generator() {
    console.log('running...');
    console.log('Hello'+yield 1);
  }
  //above:Uncaught SyntaxError: Unexpected identifier
  function* generator() {
    console.log('running...');
    console.log('Hello'+(yield 1));
  }
  var start = generator();
  start.next();//running...
  start.next();//Helloundefined
  ```
+ 作为函数参数或赋值表达式右操作数时，可以不加括号

## Generator函数与Iterator的接口
`Generator`函数执行后，返回一个遍历器对象，该对象的`Symbol.iterator`属性执行后会返回自身
```js
var start = generator();
start[Symbol.iterator]()==start;//true
start[Symbol.iterator]()==generator();//false
```

### next()方法参数
在之前 > yield表达式存在的位置 中，`console.log('Hello'+(yield 1))`打印的值为`undefined`。因为`yield`表达式本身没有返回值(`undefined`)。这时可以通过调用`next()`方法时，传入一个参数作为上一个`yiled`的返回值:
```js
function* generator() {
    console.log('running...');
    console.log('Hello'+(yield 1));
  }
  var start = generator();
  start.next();//running...
  start.next(' world');//Hello world
```
**next()方法的参数会作为上一个yiled表达式的返回值**
所以第一次调用`next()`方法传入参数时，会无效

### for...of循环
当返回对象的`done`为`true`时，`for...of`循环就会终止

## prototype属性上的方法

### throw
用于Generator函数返回的对象，在函数体外抛出错误，但在函数体内进行捕获:
```js
function* fibonacci() {
      try {
        let [prev, curr] = [0, 1];
        for (; ;) {
          yield curr;
          [prev, curr] = [curr, curr + prev];
        }
      } catch (e) {
        console.log('内部', e);
      }
    }
    var g = fibonacci();
    g.next();
    try {
      g.throw('出错');//内部 出错
      g.throw('出错');//外部 出错
    } catch (e) {
      console.log('外部', e);
    }
```
在次抛出时由于已经使用过则在外部进行捕获
**该方法接受一个参数，传递给catch语句作为第一个参数**
该方法抛出的错误要被捕获必须先调用至少一次next()方法且内部没有设置`try...catch`语句时，会被外部的`try...catch`语句捕获。

throw()方法被捕获后，会附带执行下一句yield表达式，相当于同时执行了next()方法
```js
function* fibonacci() {
      try {
        let [prev, curr] = [0, 1];
        for (; ;) {
          yield curr;
          [prev, curr] = [curr, curr + prev];
        }
      } catch (e) {
        console.log('内部', e);
      }
      yield 'next';
      yield 'finish';
    }
    var g = fibonacci();
    g.next();//{value:1,done:false}
    g.throw('出错');//{value:'next',done:false} 打印 内部 出错
    g.next();//{value:'finish',done:false}
```
Generator内部错误被外部捕获
```js
function* fibonacci() {
      let x=yield 3;
      yield x.toUpperCase();
    }
    var g = fibonacci();
    try{
      g.next();
      g.next(1);//TypeError: x.toUpperCase is not a function
    }catch(e){
      console.log(e);
    }
```
**总结：错误捕获**
+ 内部设置`try...catch`，外部无：内部捕获
+ 内部无，外部设置`try...catch`：外部捕获
+ 内部外部同时设置`try...catch`：优先内部捕获，其次外部
+ 都未设置捕获：未捕获的错误，此时内部出错将会停止运行

### return
返回给定的值，并停止`Generator`函数
```js
function* gen() {
  yield 1;
  console.log('1');//在return时并未执行
  yield 2;
  yield 3;
}
var g = gen();
g.next();//{value:1,done:false}
g.return(4);//{value:4,done:true}
g.next();//{value:undefined,done:true}
```
当`Generator`函数中有`try...finally`语句时，会先执行完`finally`块中代码在执行`return()`方法(*必须要执行try块中代码才有效*)：
```js
function* gen() {
  try{
    yield 1;
  }finally{
    yield 5;
  }
  yield 2;
  yield 3;
}
var g = gen();
g.next();//{value:1,done:false}
g.return(4);//{value:5,done:false}
g.next();//{value:4,done:true}
```

### next、throw、return三者共同点
+ 都让Generator恢复执行
  + `next`用其参数替换上一个`yield`表达式的值
  + `throw`将`yield`替换为一个`throw`语句
  + `return`将`yield`替换为一个`return`语句

## yield*表达式
### yield *fn()
在Generator函数中调用 `yield *fn()`，可以在一个Generator函数中执行另一个Generator函数
```js
function* gen() {
  yield 1;
  yield *an();
  yield 3;
}
function *an() {
  yield 2;
}
var g = gen();
console.log(g.next());//{value: 1, done: false}
console.log(g.next());//{value: 2, done: false}
console.log(g.next());//{value: 3, done: false}
//等同于
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}
//等同于
function *gen() {
  yield 1;
  for(let value of an()){
    yield value;
  }
  yield 3;
}
```

当作为`yield *fn()`的函数使用return时，会作为当前yiled的返回值
```js
function* gen() {
    yield 1;
    let value=yield* an();
    console.log(value);//foo
    yield 3;
  }
  function* an() {
    yield 2;
    return 'foo';
  }
  //1,2,3
```

### yield * + 有iterator接口的数据结构
```JS
function* gen() {
  yield 1;
  yield *[1,2,3];
  yield 3;
}
// 1,1,2,3,3
```

## Generator的this
Generator函数总是返回一个遍历器对象，该对象是Generator函数的实例，继承了`Generator.prototype`的方法。
不能通过`new`来生成实例，也不能在内部自定义`this`属性。(`this`指向`window`)

可以通过call、apply来构造一个既可以定义属性，又可以调用next的对象实例
```js
  function* gen() {
    this.a=1;
    yield this.b=2;
    yield this.c=3;
  }
  var obj={};
  var f=gen.call(obj);
  f.next();//obj{a:1,b:2}
  f.next();//obj {a:1,b:2,c:3}
```
利用Generator函数的prototype来使所有实例继承this属性
```js
function Gen() {
    return gen.call(gen.prototype);
    function* gen() {
      this.a = 1;
      yield this.b = 2;
      yield this.c = 3;
    }
  }

var gen=new Gen();
gen.next();//gen.a=1
```