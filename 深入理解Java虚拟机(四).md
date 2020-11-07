---
title:
  深入理解Java虚拟机(四)
tags:
  [虚拟机执行引擎,JVM]
categories:
	[JVM]
---



> 这一篇主要介绍虚拟机的执行引擎

# 运行时栈帧结构

> 栈帧（Stack Frame）是用于支持虚拟机进行方法执行的数据结构，封装了方法的`局部变量表`，`操作栈`，`动态链接`信息和`返回地址`和一些额外的附加信息

## 局部变量表

这个听名字就知道是干啥的了，主要用于存放`方法参数`和方法内部定义的`局部变量`，在上一篇关于[字节码](http://imlgw.top/2019/09/05/shen-ru-li-jie-java-xu-ni-ji-san/#Code%E5%B1%9E%E6%80%A7)的文章中我们分析了Class字节码的结构，可以得知在Javac编译的时候其实就在方法的Code属性中的max_locals确定了局部变量表的最大容量，局部变量表的容量以变量槽（Slot）为最小单位

值得注意的地方就是局部变量和`类变量`或者`实例变量`不同，局部变量使用前必须赋初始值，否则无法编译，即使编译也无法运行，也就是说局部变量不会被自动赋予默认初始值，需要程序员手动的初始化

## 操作数栈

Java虚拟机基于栈的执行引擎，其中所指的栈就是操作数栈，栈这种数据结构相信大家还是比较熟悉，一种后入先出（LIFO）的数据结构，它的大小同样在编译成字节码的时候就已经写入到Code属性的max_stacks值中，操作数栈的每一个元素可以是任意的Java数据类型，包括long 和double，32位数据类型所占的栈容量为1，64位数据类型所占的栈容量为2，在方法执行的任何时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值

## 动态连接

这里应该指的是当前栈帧所属方法的直接引用？

## 方法返回地址

当一个方法开始执行后，只有两种方式可以退出这个方法

1. 第一种方式是执行引擎遇到任意一个方法返回的字节码指令，这时候可能会有返回值传递给上层的方法调用者（调用当前方法的方法称为调用者），是否有返回值和返回值的类型将根据遇到何种方法返回指令来决定，这种退出方法的方式称为**正常完成出口**（Normal Method Invocation Completion）。
2. 另外一种退出方式是，在方法执行过程中遇到了异常，并且这个异常没有在方法体内得到处理，无论是Java虚拟机内部产生的异常，还是代码中使用athrow字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种退出方法的方式称为**异常完成出口**（Abrupt Method Invocation-Completion）。**一个方法使用异常完成出口的方式退出，是不会给它的上层调用者产生任何返回值的**。

无论采用何种退出方式，在方法退出之后，都需要返回到方法被调用的位置，程序才能继续执行，方法返回时可能需要在栈帧中保存一些信息，用来帮助恢复它的上层方法的执行状态。一般来说，方法正常退出时，调用者的PC计数器的值可以作为返回地址，栈帧中很可能会保存这个计数器值。

而方法异常退出时，返回地址是要通过异常处理器表来确定的，栈帧中一般不会保存这部分信息
方法退出的过程实际上就等同于**把当前栈帧出栈**，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值（如果有的话）压入调用者栈帧的操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令等

## 附加信息

虚拟机规范允许其体的虚拟机实现增加一些规范里没有描述的信息到栈帧之中，例址与调试相关的信息，这部分信息完全取决于具体的虚拟机实现，这里不再详述。在实际开发中，一般会把动态连接、方法返回地址与其他附加信息全部归为一类，称为栈帧信息

# 方法调用

Java虚拟机提供了5条方法调用字节码指令，分别如下

- **invokestatic**：调用静态方法
- **invokespecial**：调用实例构造器`<init>`方法，私有方法，父类方法
- **invokeinterface**：调用接口方法，会在运行的时候再确定一个实现此接口的对象
- **invokevirtual**：调用所有的虚方法
- **invokedynamic**：先在运行时动态的解析出调用点限定符所引用的方法，然后再执行该方法

方法调用阶段的唯一任务就是确定被调用方法的版本（调用哪一个方法），并不涉及到方法内部的具体运行过程，在编译期间所有方法调用的目标方法在Class文件中都是一个常量池中的符号引用，并不是真正的内存中的方法入口（直接引用），所以Java方法的调用需要在类加载阶段，甚至运行期间才能确定目标方法的直接引用，举个很简单的例子

## 解析

**静态解析**对应就是类加载过程中的**解析**阶段，也就是在类加载阶段确定方法的直接引用的

这个阶段会将一部分方法的符号引用解析为直接引用，这类方法的特点就是：方法在程序运行之前就已经有一个可确定的版本了，并且这个方法在运行期间是不可改变的，简单来说就是**这个方法在编译的时候就确定了**，而其它的方法则会在第一次被调用的时候才会转换为直接引用

所以说**解析调用**一定是个静态的过程，在编译期间就可以完全确定，在类加载的解析阶段就会将涉及到的符号引用转换为可以确定的直接引用，不会延迟到运行期间再去完成

`invokestatic` 和`invokespecial`指令调用的方法，都可以在**类加载的解析阶段中**就直接将符号引用解析为直接引用，从而确定唯一的调用版本，符合这个条件的方法有

- 静态方法
- 私有方法（无法被重写）
- 实例构造器
- 父类方法

这些方法被称为非虚方法，对应的其他方法（final方法除外）的就称为虚方法（Java语言层面是没有这个概念的，这是C++里面的概念）

> final方法(非static)虽然是用的`invokevirtual`来调用的，但是它也是非虚方法，其实根据上面的规则来看很好理解，final方法无法被覆盖，不会有其他版本，编译期间就可以确定

```java
public class Test1 {
    public static void test(){
        System.out.println("static test() invoke");
    }

    public static void main(String[] args) {
        test();
    }
}
```

用`jclasslib`反编译看一下main方法的字节码

```java
0 invokestatic #5 <jvmstudy/class_executor/Test1.test>
3 return
```
可以看到确实是通过`invokestatic`指令调用的，参数是常量池第5项常量也就是test方法

## 分派

分派其实对应的就是Java语言**多态**的体现，在Java语言层面上对应的就是**重载**和**重写** ，而分派也有两种，一种是静态分派，一种是动态分派，分别对应的就是重载和重写

### 静态分派

静态分派只会涉及重载（Oveload），而重载是在编译期间确定的，那么静态分派自然是一个静态的过程（因为还没有涉及到Java虚拟机）。静态分派的最直接的解释是在重载的时候是通过参数的静态类型而不是实际类型作为判断依据的。因此在编译阶段，Javac编译器会根据参数的静态类型决定使用哪个重载版本。

> 说实话，这里我一直感觉挺别扭，这里按我的理解，静态分派发生在编译阶段，既然编译期间就确定了方法调用的版本为什么不在解析阶段就直接转换为直接引用？
>
> 突然有点想通了，重载的方法也是可能被子类重写的，虽然你在编译期间就确定了调用那一个重载的方法，但是这个方法同时也可能被重写了，所以也是不能直接解析的，也需要等到运行的时候再去根据虚方法表解析

```java
/**
 * @author imlgw.top
 * @date 2019/9/7 14:19
 */
public class StaticDispatch {
    public void eat(Animal animal){
        System.out.println("Animal eat food");
    }

    public void eat(Cat cat) {
        System.out.println("Cat eat fish");
    }

    public void eat(Dog dog) {
        System.out.println("Cat eat fish");
    }

    public static void main(String[] args) {
        StaticDispatch staticDispatch = new StaticDispatch();
        Animal cat= new Cat();
        Animal dog= new Dog();

        staticDispatch.eat(cat);
        staticDispatch.eat(dog);
    }
}

class Animal {

}

class Cat extends Animal{

}

class Dog extends Animal{

}
```

这里最后两次调用打印的结果都是 `"Animal eat food"` ，这一点其实你在IDE里面写出来的话就知道了，另外两个方法下面会有波浪线，提示你这两个方法没有被使用(说明被调用的方法编译期间就确定了)

其实这段代码还是挺有迷惑性的，如果对**方法重载**不够熟悉的话可能就会搞错

所以为什么会选择`Animal` 类型的参数重载呢？这两个对象的实际类型不是Cat和Dog么？

首先我们搞清楚两个概念 `Animal cat=new Cat();` 

这行代码里的`Animal`称为变量`cat`的**静态类型**，或者比较直白的叫做**外观类型**，后面的`Cat` 被称为变量`cat`的**实际类型**

静态类型和实际类型在在程序中都可以发生变化，并不是说静态类型就一定不会变化，只不过静态类型的变化只会发生在**使用这个变量的时候**，而这个**变量本身的静态类型是不会变的**，并且不管如何变化在编译期间都是可以确定的，看起来很绕，看个例子就懂了

```java
//静态类型变化
staticDispatch.eat((Cat)cat);
staticDispatch.eat((Dog)dog);
```
对变量进行了向下的类型转换，这样一来打印的结果就是各自对应的重载方法了，而`cat`和`dog`本身的静态类型还是没变，仍然是`Animal`所以**静态类型是在编译的时候就可以确定的**

**实例类型**的变化也很好理解了

```java
Animal cat=new Dog();
cat=new Cat();
```

这样的变化在编译期间其实是无法确定的，必须要在运行的时候才能确定（如果还有疑惑为啥不能确定可以看下面动态分派例子）

再回到代码中，**编译器**在重载的时候是通过参数的静态类型而不是实际类型作为判定依据的，并且静态类型是编译期可知的，因此，Javac在编译期间会根据参数的静态类型决定使用那个重载版本，所以最后选择的是静态类型Animal参数的方法并且将这个方法的符号引用作为main方法中`invokevirtual` 指令的参数

**main方法反编译的Code**

```
Code:
stack=2, locals=4, args_size=1
0: new           #6                 // class jvmstudy/class_executor/StaticDispatch
3: dup
4: invokespecial #7            // Method "<init>":()V
7: astore_1
8: new           #8            // class jvmstudy/class_executor/Cat
11: dup
12: invokespecial #9            // Method jvmstudy/class_executor/Cat."<init>":()V
15: astore_2
16: new           #10           // class jvmstudy/class_executor/Dog
19: dup
20: invokespecial #11           // Method jvmstudy/class_executor/Dog."<init>":()V
23: astore_3
24: aload_1
25: aload_2
26: invokevirtual #12           // Method eat:(Ljvmstudy/class_executor/Animal;)V
29: aload_1
30: aload_3
31: invokevirtual #12           // Method eat:(Ljvmstudy/class_executor/Animal;)V
34: return
```

**总结**

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派，对应的就是Java中的重载，**静态分派发生在编译阶段**，因此确定静态分派的动作实际上并不是由虚拟机执行的而是由**编译器**执行的

 🎯 编译器虽然能确定出方法的重载版本但是这个重载的版本并**不一定是唯一的**，看个Demo就明白了

```java
/**
 * @author imlgw.top
 * @date 2019/9/7 16:44
 */
public class OverloadTest {

    public void test(char a) {
        System.out.println("char");
    }

    public void test(int a) {
        System.out.println("int");
    }

    public void test(long a) {
        System.out.println("long");
    }

    public void test(float a) {
        System.out.println("float");
    }

    public void test(double a) {
        System.out.println("double");
    }

    public void test(Character a) {
        System.out.println("Character");
    }

    public void test(Serializable serializable) {
        System.out.println("serializable");
    }

    public void test(char... a) {
        System.out.println("char[]");
    }

    //不会执行
    public void test(Double a) {
        System.out.println("Double");
    }

    public static void main(String[] args) {
        OverloadTest overloadTest = new OverloadTest();
        overloadTest.test('a');
    }
}
```

从上到下依次的注释test方法，除了最后一个都可以正常的执行，而这些方法明显参数也并不一样，主要的原因还是由于**字面量**是不需要定义的，他没有显式的静态类型

所以编译器在确定方法重载的版本的时候并不是选择的唯一的一个，而是当前最合适的一个，其实还可以写很多但是意义并不大，具体选择的规则可以参考 [Java语言规范15.12.2.5](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.12.2.5)

### 动态分派

动态分派在**运行期**根据**实际类型**确定方法执行版本，其实开始我也有点疑惑为啥一定要在运行期间才能确定方法调用的版本，直到我看到了下面的例子

```java
Scanner in = new Scanner(System.in);
Person person = null;
if (in.nextLine().equals("chinese")) {
    person = new Chinese();
} else {
    person = new English();
}
person.sayHello();
```

这个例子可能比较特殊，但是即使不用if-else单纯的只new 一个对象出来，编译期间也是无法确定的，比如**动态代理**之类的技术就可以在运行时修改字节码文件，所以无论如何在**编译期间都是无法确定变量的实际类型的**

动态分派对应的就是Java中的**重写(Override)** ，看一个具体的Demo

```java
public class DynamicDispatch {
    public static void main(String[] args) {
        Human man= new Man();
        Human woman= new Woman();
        man.eat();
        woman.eat();
    }
}

class Human {
    public void eat(){
        System.out.println("Human eat");
    }
}

class Woman extends Human{
    public void eat(){
        System.out.println("Woman eat");
    }
}

class Man extends Human{
    public void eat(){
        System.out.println("Man eat");
    }
}
```

结果不用我再多说了，问题是Java虚拟机是怎么知道要调用那个方法的？我们看一下反编译后的字节码

```java
stack=2, locals=3, args_size=1
0: new           #2            // class jvmstudy/class_executor/Man
3: dup                         // 复制栈顶并重新入栈
4: invokespecial #3            // Man的构造方法 Method jvmstudy/class_executor/Man."<init>":()V
7: astore_1                    // 将栈顶出栈并存入局部变量表第一个Slot槽位 man
8: new           #4            // class jvmstudy/class_executor/Woman
11: dup						   // 复制栈顶并重新入栈	
12: invokespecial #5           // Woman构造方法 Method jvmstudy/class_executor/Woman."<init>":()V
15: astore_2                   // 将栈顶出栈并存入局部变量表第二个Slot槽位 woman
16: aload_1                    // 将局部变量槽第一个变量man加载到操作栈栈顶
17: invokevirtual #6           // Method jvmstudy/class_executor/Human.eat:()V
20: aload_2                    // 将局部变量槽第二个变量woman加载到操作栈栈顶
21: invokevirtual #6           // Method jvmstudy/class_executor/Human.eat:()V
24: new           #4           // class jvmstudy/class_executor/Woman
27: dup                        // 复制栈顶并重新入栈
28: invokespecial #5           // Method jvmstudy/class_executor/Woman."<init>":()V
31: astore_1                   // 将栈顶出栈并存入局部变量表第一个Slot槽位 woman
32: aload_1                    // 将局部变量槽第一个变量woman加载到操作栈栈顶
33: invokevirtual #6           // Method jvmstudy/class_executor/Human.eat:()V
36: return
```
可以看到 `invokevirtual` 后面的参数仍然是`Human.eat()`，所以可以确定在编译期是无法确定的，那么问题肯定出在 `invokevirtual` 的执行流程上，`invokevirtual`执行一般分下面几个步骤

1. 找到操作数栈顶的元素所指对象的**实际类型**，假设为`Man`类型

2.  如果在`Man`类型中找到了和指令后面所跟参数（`Human.eat()`方法）的**描述符(方法参数和返回值) **和**简单名称(方法名)** 都一致的方法，则进行访问权限验证，如果通过校验就直接返回这个方法的直接引用，查找过程结束，如果检验不通过就`throw IllegaAccessError`异常
3.  如果第二步中没有找到就按照继承关系从下往上，重复第二步
4.  如果始终找不到，则`throw AbstractMethodError`

#### 动态分派实现

由于动态分派是非常频繁的动作，而且这个动作是在**运行期间**在类的方法元数据中搜索合适的目标方法，因此大部分的实现都不会进行如此频繁的搜索，其中一种实现就是利用**虚方法表**，为类在**方法区**中建立一个虚方法表(vtable)，针对于`invokeinterface`指令来说，虚拟机会建立一个叫做接口方法表的数据结构（itable），使用虚方法表索引来代替元数据查找以提高性能

> 关于虚方法表的创建过程，找到了一篇很硬核的文章  [从jvm虚拟机角度看Java多态](<https://bbs.pediy.com/thread-225413.htm>) 

**总结**

- 虚方法表一般在`类加载的连接阶段`进行初始化，准备了类的变量初始值后，虚拟机也会将该类的方法表也初始化完毕

- vtable 分配在 instanceKlass对象实例的内存末尾 (方法区中)
- 其实vtable可以看作是一个数组，数组中的每一项成员元素都是一个**指针**，指针指向 Java 方法在 JVM 内部所对应的 method 实例对象的内存首地址 
- vtable是 Java 实现面向对象的多态性的机制，如果一个 Java 方法可以被继承和重写， 则最终通过 put_method_at函数将方法地址替换,完成 Java 方法的动态绑定
- Java 子类会继承父类的 vtable，Java 中所有类都继承自 `java.lang.Object`，Object 中有 5 个虚方法（可被继承和重写）：  void finalize() ， boolean equals(Object) ，String toString()，int hashCode()，Object clone() 因此，如果一个 Java 类中不声明任何方法，则其 vtable 的长度默认为 5 
- Java 类中不是每一个 Java 方法的内存地址都会保存到 vtable 表中，只有当 Java子类中声明的 Java 方法是 public 或者 protected 的，且没有 final 、 static 修饰，并且 Java 子类中的方法并非对父类方法的重写时， JVM 才会在 vtable 表中为该方法增加一个引用 
- 如果 Java 子类某个方法重写了父类方法，则子类的vtable 中原本对父类方法的指针会被替换成子类对应的方法指针，调用put_method_at函数替换vtable中对应的方法指针
- 当使用 invokeinterface 来调用方法时，由于不同的类可以实现同一 interface，我们无法确定在某个类中的 inteface 中的方法处在哪个位置。于是，也就无法解析 CONSTANT_intefaceMethodref_info 为直接索引，而必须每次都执行一次在 methodtable 中的搜索了，所以，在这种实现中，通过 invokeinterface 访问方法比通过 invokevirtua﻿﻿l 访问明显慢很多

## 动态调用

_未完待续_



# 基于栈和基于寄存器

现代JVM在执行Java代码的时候，通常都会将`解释执行`与`编译执行`二者结合起来进行。所谓解释执行，就是通过解释器来读取字节码，遇到相应的指令就去执行该指令。所谓编译执行，就是通过即时编译器（Just In Time，JIT）将字节码转换为本地机器码来执行，现代JVM会根据代码热点来生成相应的本地机器码

基于栈的指令集与基于寄存器的指令集之间的关系：

1. JVM执行指令时所采取的方式是基于栈的指令集
2. 基于栈的指令集主要的操作有入栈与出栈两种
3. 基于栈的指令集的优势在于它可以在不同平台之间移植，而基于寄存器的指令集是与硬件架构紧密关联的，无法做到可移植
4. 基于栈的指令集的缺点在于完成相同的操作，指令数量通常要比基于寄存器的指令渠数量要多；基于栈的指令集是在内存中完成操作的，而基于寄存器的指令集是直接由CPU来执行的，它是在高速缓冲区中进执行的，速度要快很多。虽然虚拟机可以采用一些优化手段，但总体来说，基于栈的指令集的执行速度要慢一些

## 实例分析

```java
public class JVMStack {
    public static int calculate(){
        int a=1;
        int b=2;
        int c=3;
        int d=4;
        int result=(a+b-c)*d;
        return  result;
    }
}
```

很简单的代码，我们深入的分析一下底层的字节码

```java
Code:
stack=2, locals=5, args_size=0
 0 iconst_1   //常量1入栈
 1 istore_0   //将栈顶的元素1出栈并且保存到局部变量表Slot 0的位置中(静态方法，没有this)
 2 iconst_2   //下面的都是和前两条一样的，都是给局部变量赋值就不赘述了
 3 istore_1   //.....
 4 iconst_3
 5 istore_2
 6 iconst_4
 7 istore_3   //到这里局部变量表为 [1，2，3，4] 栈为空【】
 8 iload_0    //和istore_0相对应，将局部变量表Slot 0位置的元素1，加载到栈顶
 9 iload_1    //将局部变量表Slot 1位置的元素2，加载到栈顶，栈状态为【2，1】
10 iadd       //将栈顶的两个元素2，1出栈，并将它们的和3压入栈，栈状态【3】
11 iload_2    //将局部变量表Slot 2位置的元素3，加载到栈顶，栈状态【3，3】
12 isub       //将栈顶的两个元素3，3出栈，并将他们的差0压入栈，栈状态【0】
13 iload_3    //将局部变量表Slot 3位置的元素4，加载到栈顶，栈状态【4，0】
14 imul       //将栈顶两个元素4，0相乘，结果重新压入栈，栈状态【0】
15 istore 4   //将栈顶的元素0出栈并保存到局部变量表Slot 4的位置，局部变量表状态为[1，2，3，4，0]
17 iload 4    //将局部变量表Slot 4位置的元素加载到栈顶，栈状态【0】，这里如果直接返回就没有这两步
19 ireturn    //返回栈顶元素
```

通过分析其实也可以验证上面 `stack=2, locals=5, args_size=0`的正确性，最大栈深度为2，最大局部变量表所需Slot为5，参数个数为0

