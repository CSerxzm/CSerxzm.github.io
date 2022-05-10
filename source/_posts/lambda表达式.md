---
title: lambda表达式
date: 2020-11-19 22:13:00
comments: false
categories: 
- java
tags:
- lambda
- 函数式接口
---

### 前言

Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。对于lambad其最主要的就是减少了匿名内部类的使用，让代码更加简洁。

lambda实现的关键需要是`函数式接口`，即该类只包含一个抽象的方法。如下面的函数，我们就可以使用lambda来创建该接口的对象。

```java
public interface Runnable{
    public abstract void run();
}
```

### 演示过程

1.定义一个函数式的接口

```java
interface Like{
    void LikeSomething();
}
```

2.实现该接口，并在主类中测试。

```java
class MyLike implements Like{
    @Override
    public void LikeSomething() {
        System.out.println("I like xiang");
    }
}
//测试
public class Test07 {
    public static void main(String[] args) {
        Like like = new MyLike();
        like.LikeSomething();
    }
}
//运行结果
I like xiang
```

3. 改进1：使用静态内部类

```java
public class Test07 {
    static class MyLike implements Like{
        @Override
        public void LikeSomething() {
            System.out.println("I like xiang");
        }
    }
    public static void main(String[] args) {
        Like like = new MyLike();
        like.LikeSomething();
    }
}
```

4. 改进2：局部内部类。

```java
public class Test07 {
    public static void main(String[] args) {
        class MyLike implements Like{
            @Override
            public void LikeSomething() {
                System.out.println("I like xiang");
            }
        }
        Like like = new MyLike();
        like.LikeSomething();
    }
}
```

5. 改进3：匿名内部类。

```java
public class Test07 {
    public static void main(String[] args) {
		Like like = new Like() {
            @Override
            public void LikeSomething() {
                System.out.println("I like xiang");
            }
        };
        like.LikeSomething();
    }
} 
```

### lambda表达式

对于上面的改进我们可以进一步。

```java
public class Test07 {
    public static void main(String[] args) {
        Like like=()->{
            System.out.println("I like xiang");
        };
        like.LikeSomething();
    }
}
```

上面的运行，其运行结果一致，对于lambda表达式的还可以进一步的简化，以下面的接口为示例。

```java
interface Math{
    int add(int x,int y);
}

public class Test06 {
    public static void main(String[] args) {
        //1.写成lambda表达式
        Math math = (int x,int y)->{
            return x+y;
        };
        //打印结果为3；
        System.out.println(math.add(1, 2));
        //进一步简化，省略参数类型
        math = (x,y)->{
            return x+y;
        };
        //进一步简化，如果只有一个参数，可以省略参数外的括号
        //如果函数中只有一句，那么可以简化花括号。
        math = (x,y)-> x+y+1;
        System.out.println(math.add(1,2));
        //打印结果为4，可见只有一句的时候，return都可以省略
    }
}
```

### 使用

对于项目的开发中，我们可以发现lambda表达式在很多地方都可以使用，如以下地方：

```java
public class Test05 {
    public static void main(String[] args) {
        new Thread(()->{
            System.out.println("用于多线程");
        }).start();
        
        Map<String,String> map = new HashMap<>();
        map.put("key","value");
        map.forEach((k,v)->{
            System.out.println("key:"+k+"--->value:"+v);
        });
        
        List<Integer> list = new ArrayList<>();
        list.forEach(i-> System.out.println("value:"+i));
    }
}
```

总之，只要我们的接口只有一个抽象的方法，我们就叫它为函数式接口，就可以实现lambda编程。