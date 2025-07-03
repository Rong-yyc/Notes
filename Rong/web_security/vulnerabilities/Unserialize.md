# 反序列化渗透与防御

​		序列化的作用：将对象的信息转换为可以存储或传输的形式的过程

## PHP反序列化渗透与防御

```php
例子：
# 空
serialize(null);    =>  N;
# 整数
serialize(123);     =>  i:123;
# 小数
serialize(123.3);   =>  d:123.3;
# 布尔类型
serialize(false);   =>  b:0;
# 字符串
serialize("rong");  =>  s:4:"rong";
# 数组
serialize(array("rong", "Rong", "RONG"));  => a:3:{i:0;s:4:"rong";i:1;s:4:"Rong";i:2;s:4:"RONG";}
```

```php
# 序列化一个对象
class demo{
    public $name = "rong";
}

serialize(new demo());   =>   O:4:"demo":1:{s:4:"name";s:4:"rong";}
```

```php
# 序列化一个对象
class demo{
    public $name = "rong";
    public $age;
}

serialize(new demo());   =>  O:4:"demo":2:{s:4:"name";s:4:"rong";s:3:"age";N;}
```

### 反序列化

​	将序列化得到的字符串转换为一个对象

​	**注：反序列化生成的对象的成员属性值由被反序列化的字符串决定，与原来类的预设值无关**

### 反序列化漏洞利用

​	在反序列化过程中，unserialize()接收的值可控

#### 魔术方法

​	预定好的，在特定情况下自动触发的行为方法

​	作用：在特定条件下自动调用相关方法，最终导致触发代码

| 魔术方法      | 描述                                  |
| ------------- | ------------------------------------- |
| __construct() | 类的构造函数，创建对象时触发          |
| __destruct()  | 类的析构函数，对象被销毁时触发        |
| __sleep()     | 执行serialize()方法时会先调用这个方法 |
| __wakeup()    | 执行unserialize()时先调用这个方法     |
| __toString()  | 把对象当字符串调用时触发              |
| __invoke()    | 把对象当作函数调用时触发              |

#### POP链构造思路

​	魔术方法触发的前提是：魔术方法所在的类或者对象被调用

​	POP链起点：魔术方法 -> .... -> eval等命令注入函数（POP链终点）

#### 绕过魔术方法

​	__wakeup()：传入属性个数值大于实际属性个数时，wakeup()就不会被触发了

#### 绕过正则表达式

​	`O:+4`：一般会匹配"O:\d"来拦截攻击

## Java 反序列化漏洞

![](./images/serialize.png)

使用场景：持久化内存数据、网络传输对象、远程方法调用

序列化：`java.io.ObjectOutputStream.writeObject()`

反序列化：`java.io.ObjectInputStream.readObject()`

利用思路：（1）利用自定义的 readObject() 方法执行代码

​				（2）寻找重写了 readObject() 方法的类

```
package: sun.reflect.annotation;  >  AnnotationInvocationHandler
package: javax.management;  >  BadAttributeValueExpException
```

