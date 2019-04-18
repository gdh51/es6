# ES6 Class

## 要点

### 定义原型方法时不能用'，'否则报错
```
class Potin{
  constructor(x,y){
    this.x=x;
    this.y=y;
  }

  toString(){
    return '('+this.x+', '+this.y+')';
  }

  valueOf(){

  }
}
```
### 定义在Class中的方法不可枚举

### 若未定义constructor函数，则默认会使用一个空的constructor函数

### 创建实例时必须用new操作符，否则报错

### 存储set、get属性描述符，会存储在该属性的属性描述符Descripter 上

### Class内部默认为严格模式，所以内部this指向undefined,且Class的定义同let相同，只在定义阶段提升
```
var Point=1;//变量定义提升
  {
    console.log(Point);//error:not defined
    class Point{
      constructor(x,y){
        this.x=x;
        this.y=y;
      }

      sayName(){
        console.log(this.x,this.y);
      }
}
  }
```
### 可以通过直接在Class上定义属性，来在实例中定义属性
````
class Logger {
      _config=1;

      static classMethod(){
        return 'hello';
      }
    }
    //结果为：Logger {_config: 1}
````
### new.target返回命令作用于的那个构造函数。当不同过new或Reflect.construcor调用时为undefined；Class内部该属性返回该Class；子类继承父类时，返回子类Class

### 使用extends继承父类时，如果子类自己定义constructor函数，则必须在其中调用super()方法,在没有定义时，实际上默认调用了该方法，且super()必须最先调用，否则报错

### super()作为父类构造函数调用时，返回的是子类的实例，即super()中this指向子类。且super()作为函数时，只能在构造函数中使用。

### super作为对象时，在普通方法中指向父类的原型对象；在静态方法中指向父类。在子类普通方法中通过super调用父类方法时，方法内部的this指向当前的子类实例

### 可以通过extends继承内置构造函数，ES5之前不行
