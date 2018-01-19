---
title: thinking-in-java-notes-1
tags:
---

最近在读《Thinking in Java》，为了避免忘记一些（我认为）重要的知识点，就开了这个系列。博客的结构和《Java编程思想（英文版 第4版）》（机械工业出版社）的目录结构一致，以便快速找到出处。

## Everything is an Object

### You must create all the objects

#### Special case: primitive types

*  布尔值类型`boolean`的大小是不确定的，只要能容纳两个值即可，由虚拟机决定大小。
*  `char`类型的大小和C++的一个字节不同，是两个字节的，存储的是Unicode编号0x0000到0xFFFF的值。从[这篇SO帖子](https://stackoverflow.com/questions/1366068/whats-the-complete-range-for-chinese-characters-in-unicode)可以发现，常见中文字的范围是在这个范围内的，根本不虚……

#### Arrays in Java

*  即使是基本类型数组，也会被初始化为0.


### You never need to destroy an object

#### Scoping

*  Java使用花括号指定变量作用域，但是不允许如下的定义：

   ```java
   {
     int x = 1;
     {
       int x = 2; // Illegal
     }
   }
   ```

## Operators

### Casting operators

#### Promotion

*  除`boolean`外，小于`int`的类型（`short`, `byte`, `char`）在参与运算之前会被自动升格为`int`后再运算（自然也会得到`int`类型的结果）。

## Reusing Classes

### Initialization and class loading

#### Initialization with inheritance

*  当一个类或其子类的静态成员被访问（静态方法、静态变量**和构造方法**）的时候，该类会被加载。

*  类加载中的语句执行顺序：层级向下、文本向下初始化静态变量。其中“层级向下”指的是先执行基类，然后向下依次执行各个子类；“文本向下”指的是根据静态变量在类中定义的先后顺序执行；下同。

*  类对象构造方法被调用时的语句执行顺序：

   1. 调用上一层级的构造方法，如果构造方法中显式指定了`super`，则调用该构造方法；
   2. 文本向下初始化实例变量；
   3. 执行构造方法中`super`之后（如果有）的部分。

   其中需要注意的是，该类的成员变量的初始化在基类构造之后，在该类构造方法之前。

## Interfaces

### Interfaces

*  实现接口的方法时，实现方法必须显式修饰为`public`的，否则就失去了接口方法原有的可访问性。

### Complete decoupling

*  关于协变返回类型和Liskov替换原则。

   **以下内容有错误，需要仔细研究！！！**

   书中提到，重写方法的时候，返回类型可以是被重写的方法的返回类型的子类，即协变返回类型（covariant return type），谷歌之后查到一篇看起来不错的文章：[Java中的逆变与协变 - Treant - 博客园](http://www.cnblogs.com/en-heng/p/5041124.html)。文章我还没有通读，但其中提到了*Liskov替换原则*：

   *  子类完全拥有父类的方法，且具体子类必须实现父类的抽象方法。
   *  子类中可以增加自己的方法。
   *  当子类覆盖或实现父类的方法时，方法的形参要比父类方法的更为宽松。
   *  当子类覆盖或实现父类的方法时，方法的返回值要比父类更严格。

   其中第四点就是指协变返回类型了。对后两条，我的理解是：避免需要显式指定的向下类型转换。

   假设`Dog`和`Cat`均继承`Animal`，那么下面代码是合法的：

   ```java
   class Base {
       Animal someMethod(Cat arg) {
           // ...
       }
   }

   class Derived {
       Cat someMethod(Animal arg) {
           // ...
       }
   }
   ```

   对于第三条，我们考虑下面的情况：

   ```java
   class Base {
       Animal someMethod(Animal arg) {
           // ...
       }
   }

   class Derived {
       Cat someMethod(Cat arg) {
           // ...
       }
   }

   class Main {
       void anotherMethod() {
           Base base = new Base();
           Animal animal = new Dog();
           base.someMethod(animal);
       }
   }
   ```

   可以看到，在`anotherMethod`中看起来很合理的参数传递（毕竟对`Base.someMethod`而言他的确可以接受一个`Animal`类型的参数）实际上会导致出错，因为会执行被重写的方法里的语句，而Dog和Cat类型不相容。如此可以反证第三条，第四条也是同样的道理，就不赘述了。

### "Multiple inheritance" in Java

*  如果一个类实现了接口I，但没有实现I的某个方法F，且该类的某个父类存在和F签名一致的方法，那么这是合法的，即：只要一个类包含接口I的所有方法的定义（`definition`，即实现），那么这个类就被认为合法地实现了接口I。

## Type Information

### The **Class** object

#### Class literals

*  一个class的准备过程为：

   1. 加载。Class loader加载类的字节码并借此创建Class对象。
   2. 链接。校验类的字节码，为static域分配存储空间，并处理由该类引用到的所有其他的类。
   3. 初始化。若存在基类则初始化基类，然后初始化static代码段（参见*Reusing Classes - Initialization and class loading - Initialization with inheritance*）。

   而在通过`SomeClass.class`的形式获取某个类的Class对象的时候，上述第3步并不会被执行，而是被延迟到对静态方法或非常量静态域（即类似如下的域：`static final int NUMBER = 10`，因为这样的编译期常量不需要初始化类也能获得其值）的首次引用时，即延迟到不得不初始化的时候。与之相对的是，通过`Class.forName("SomeClass")`的方法获取类对象则会立刻完成初始化。

#### Generic class references

*  对于确定的Class对象，`newInstance()`会返回具体的类型而非Object类型；而类的泛型语法则只能返回Object类型：

   ```java
   Class<FancyToy> ftClass = FancyToy.class;
   FancyToy ft = ftClass.newInstance();
   Class<? super FancyToy> tClass = ftClass.getSuperclass();
   Object o = fClass.newInstance();
   ```

   另外，尽管编译期就能确定`getSuperclass()`返回的Class对象是哪个类的，但其返回值仍然只能是“某个被FancyToy类继承的类的Class对象”（从文档也可以看出：[Class (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getSuperclass--)），因此如下的语法是无法编译的：

   ```java
   Class<Toy> tClass = ftClass.getSuperclass();
   ```

   ​