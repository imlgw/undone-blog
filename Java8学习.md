---
title:
  Java8
tags:
  [Java8]
categories:
   [工具]
---

# Lambda

- Lambda表达式为Java添加了缺失的函数式编程特性，使我们能将函数当做一等公民看待

- 在将函数作为一等公民的语言中，Lambda表达式的类型是函数。**但在Java中，Lambda表达式是对象**，他们必须依附于一类特别的对象类型——函数式接口（functional interface）

## 函数式接口

如果一个接口只有一个抽象方法，那么这个接口就是一个函数式接口

如果我们在某个接口上声明了`FunctionalInterface`注解，那么编译器就会按照函数式接口来要求该接口

如果一个接口只有一个抽象方法，但是我们并没有给该接口声明`FunctionalInterface`注解，那么编译器也会将该接口看作是函数式接口（一般情况下还是加上比较好）

### 内部迭代

```java
public class Test1 {
    public static void main(String[] args) {
        List<Integer> list= Arrays.asList(1,2,3,4,5,6,7);
        for (int i=0;i<list.size();i++){
            System.out.println(list.get(i));
        }
        System.out.println("---------");
        for (Integer integer : list) {
            System.out.println(integer);
        }
        System.out.println("---------");
        list.forEach(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) {
                System.out.println(integer);
            }
        });

        System.out.println("---------");
        list.forEach(integer -> System.out.println(integer));
    }
}
```

