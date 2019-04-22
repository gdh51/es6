# async
Generator函数的一种语法糖，将`*`替换为`async`，`yield`替换为`await`。

## 改进Generator
1. 内置执行器
   像普通函数一样调用，调用后会自动执行。
   ```js
   const asyncFn=async function () {
    await 1;
    await 2;
  }
  console.log(asyncFn());//Promise{...}
   ```
2. 语义明确
3. 更广泛的适用性
   `await`后面可以是 Promise 对象和原始类型的值（数值、字符串和布尔值，但这时会自动转成立即 `resolved` 的 Promise 对象）
4. 返回值是Promise
   原本Generator函数返回Iterator对象，进一步说明`async`函数是多个异步操作包装成一个Promise对象，而`await`则是内部`then`命令的语法糖

## 返回Promise对象
`async`函数内部return语句返回值，会成为`then`回调函数的参数
```js
async function f() {
      return 'hello world';
    }
    f().then(function (e) {
      console.log(e);//hello world
    });
```
同理`async`内部错误会导致`Promise`变为`reject`状态，并抛出错误同时被`catch`语句捕获

### Promise对象的状态变换
`async`函数返回的`Promise`对象，必须等到内部所有`await`命令后面的`Promise`对象执行完，才会发生状态改变，除非遇到`return`语句或者抛出错误。即`async`函数体内所有异步操作执行完后，才会执行`then`里面的回调函数

## await
*正常情况*下，await后跟一个Promise对象，返回该对象的结果。如果不是Promise对象，就直接返回对应的值。
```js
async function f() {
 return await 1;
}
f().then(function (e) {
  console.log(e);//1
});
```
*特殊情况*,await后跟一个定义了then方法的对象，则await会将其等同于Promise对象
```js
class Sleep {
  constructor(timeout) {
    this.timeout = timeout;
  }
  then(resolve, reject) {
    const startTime = Date.now();
    setTimeout(
      () => resolve(Date.now() - startTime),
      this.timeout
    );
  }
}

(async () => {
  const actualTime = await new Sleep(1000);
  console.log(actualTime);//一秒后显示1000
})();
```
`await`后的Promise对象如果变为`reject`状态，则`reject`的参数会被`catch`方法的回调函数接收。
```js
(async () => {
  await Promise.reject('error');
  console.log(actualTime);
})().catch(err=>{
  console.log(err);//error
});
```
以上没有通过`return`来向`catch`语句传递参数，而是通过reject的参数来传递；通过`return await Promise.reject('error');`也能达到同样的效果。
**任何一个Promise变为reject状态，那么整个async函数都会中断执行**
如果想其中一个出错而不中断执行，则有以下两种方式：
+ 将可能出错的await语句放在try...catch块中
+ 将可能出错的await语句后跟的Promise对象调用catch()方法进行单独捕获

### 用Generator函数保存堆栈
```js
const a = () => {
  b().then(() => c());
};
```
上面函数`a`运行时，运行到`b`函数，函数`a`会继续向下执行，当`b`运行结束时，`a`可能早已运行结束，这导致`b`、`c`如果报错则错误堆栈不包括`a`
```js
async a= () => {
  await b();
  c();
}
```

## async原理
自动执行的Generator函数
模拟async ↓
```js
function pro () {
  return new Promise((re)=>{
    setTimeout(() => {
      re('done');
    }, 2000);
  }).then(e=>{
    console.log(e);
  });
}
function simAsync(fnG) {
  return packingAndrun(fnG);
  function packingAndrun(fnG) {
    //包装为Promise对象并返回最后一个状态值
    return new Promise((resolve, reject) => {
      const Gen = fnG();//存储Generator函数
      step(() => Gen.next(undefined));
      function step(nextF) {
        let next;
        try {
          next = nextF();//yield返回值
        } catch (e) {
          return reject(e);
        }
        if (next.done) {
          //Generator函数执行完毕，返回return的值
          return resolve(next.value);
        }
        //重复调用step直到yield返回值done为true
        Promise.resolve(next.value).then(v => {
          step(() => Gen.next(v));
        }, e => {
          step(() => Gen.throw(e));
        });
      }
    });
  }
}
simAsync(function *() {
  yield pro();//done
  yield pro();//done
  return 'finish';
}).then((e)=>{
  console.log(e);//finish
});//按顺序隔一秒输出done
```

## 异步遍历器(asyncIterator)

### 特点：每次调用next()返回Promise对象
这个对象状态变为resolve后，之后调用的then()的回调函数参数为一个具有value与done属性的对象(跟遍历器对象返回值一样)

一个对象的同步遍历器接口在Symbol.iterator属性上，异步遍历器接口在Symbol.asyncIterator上，如果定义该属性就表示对其进行异步遍历。
```js
[].__proto__[Symbol.asyncIterator]=async function*() {
  for(let value of this){
    yield value;
  }
}
const asyncIterable = ['a', 'b'];
const asyncIterator = asyncIterable[Symbol.asyncIterator]();
asyncIterator
  .next()
  .then(iterResult1 => {
    console.log(iterResult1); // { value: 'a', done: false }
    return asyncIterator.next();
  })
  .then(iterResult2 => {
    console.log(iterResult2); // { value: 'b', done: false }
    return asyncIterator.next();
  })
  .then(iterResult3 => {
    console.log(iterResult3); // { value: undefined, done: true }
  });
```
异步遍历器的next()可以连续调用，不必等待上一步Promise对象resolve后调用。这种情况下，next()方法会累积，自动按照每一步的顺序运行下去

### for await...of
用于遍历异步Iterator接口
```js
var asyncIterable = {
  [Symbol.asyncIterator]() {
    return {
      i: 0,
      next() {
        if (this.i < 3) {
          return Promise.resolve({ value: this.i++, done: false });
        }
        return Promise.resolve({ done: true });
      }
    };
  }
};
(async function () {
  for await (let num of asyncIterable) {
    console.log(num);
  }
})();// 0 1 2
```
每当Promise变更为resolve状态时，就会把值传入for...of循环

## 异步Generator函数
语法上,异步 Generator 函数就是async函数与 Generator 函数的结合
```js
async function* gen(){
  yiled 'done'
}
```
异步 Generator 函数内部，能够同时使用await和yield命令。可以这样理解，await命令用于将外部操作产生的值输入函数内部，yield命令用于将函数内部的值输出。