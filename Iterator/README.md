# Iterator

## 作用
+ 为各种数据结构，提供一个统一的、简便的访问接口
+ 使数据结构的成员能够按**某种次序**排列
+ ES6 创造了一种新的遍历命令for...of循环，Iterator 接口主要供for...of消费

## 本质：指针对象
第一次调用指针对象next，指针才会指向第一个成员；之后调用直到指向结束位置。

## 默认 Iterator 接口
如果一个数据结构具有`Symbol.iterator`属性，则认为是可遍历的。该属性本身是一个函数，用于生成遍历器。

> 一个简单的部署
```
var obj={
      data:['hello','world'],
      [Symbol.iterator]:function(){
        const self=this;
        let index=0;
        return {
          next:function(){
            if(index<self.data.length){
              return {
                value:self.data[index++],
                done:false
              };
            }else{
              return {
                value:undefined,
                done:true
              }
            }
          }
        };
      }
    };
    for(value of obj){
      console.log(value);//hello / world
    }
```


### 原生具有Iterator接口的数据结构：
+ Array
+ Map
+ Set
+ String
+ TypedArray
+ 函数 arguments对象
+ NodeList对象

##使用场合:默认调用Iterator接口
+ 解构赋值
+ 扩展运算符
  >利用这个原理可以将任何数据结构转换为数组
  ```
  console.log([...obj]);// ["hello", "world"]  引用上面obj对象
  ```
+ yield*
+ 其他场合
  >由于数组的遍历会调用遍历器接口，所以`任何接受数组作为参数`的场合，其实都调用了遍历器接口。下面是一些例子。
  + for...of
  + Array.from()
  + Map(), Set(), WeakMap(), WeakSet()
  + Promise.all()
  + Promise.race()

### 若要修改字符串上的默认Iterator接口，必须是通过其构造函数创建的字符串对象，否者失效。
for...of会正确识别32位UTF-16字符

### Iterator的最简单实现是使用Generator函数

## for...of

### Set与Map使用for...of遍历时，按被添加的顺序遍历，且Set返回一个值，Map返回一个数组分别表示键名、键值

### ES6数组、Map、Set的以下三个方法使用后返回迭代器对象：
+ `entries()`遍历\[键名、键值]组成的数组
+ `keys()` 遍历键名
+ `values()`遍历键值

### 对象可以使用Generator函数包装一下来使用for...of
```
let obj={
      name:1,
      age:18,
      hobby:'playgame',
      [Symbol.iterator]:function*(){
        for(let key of Object.keys(obj)){
          yield [key,obj[key]];
        }
      }
    };
    for(let [key,value] of obj){
      console.log(key,value);
      //name 1
      //age 18
      //hobby playgame
    }
```

使用for...in遍历时，先遍历出整数属性（integer properties，按照升序），然后其他属性按照创建时候的顺序遍历出来。
