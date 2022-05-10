---
title: 重写equals和hasCode方法
date: 2020-6-23 14:24:00
comments: false
categories: 
- java
tags:
- 随笔
---

### 前言

>【强制】关于 hashCode 和 equals 的处理，遵循如下规则： 1） 只要覆写 equals，就必须覆写 hashCode。 2） 因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的对象必须覆写 这两种方法。 3） 如果自定义对象作为Map 的键，那么必须覆写hashCode 和 equals。
>说明：String 因为覆写了 hashCode 和 equals 方法，所以可以愉快地将 String 对象作为 key 来使用。
>
>————摘自阿里Java代码规范嵩山版

我们都知道，要比较两个对象是否相等时需要调用对象的equals()方法，即判断对象引用所指向的对象地址是否相等，对象地址相等时，那么与对象相关的对象句柄、对象头、对象实例数据、对象类型数据等也是完全一致的，所以我们可以通过比较对象的地址来判断是否相等。

### 实例

如对于一个班级的学生，我们默认姓名，年龄一样的话，那么不同的实例就是同一个对象（同一个人）。这就需要我们重写equals方法。

```java
public class Student {

    private String name;
    private Integer age;
    private String address;

    /**
     * 只要姓名，年龄两者相同，则是同一个人
     * @param obj
     * @return
     */
    @Override
    public boolean equals(Object obj) {
        if(!(obj instanceof Student)){
            return false;
        }
        Student student = (Student) obj;
        if(student == this){
            //地址相同
            return true;
        }
        if( (student.getName()!=null && !student.getName().equals(this.name)) ||
                (student.getAge()!=null && !student.getAge().equals(this.age)) ){
            return false;
        }
        return true;
    }
    //省略get/set以及toString方法
}
```

但是什么时候重写hasCode方法呢？一般将对象放入Set或者作为Map的key部分都需要重写hasCode方法，如果不写该方法，下面的代码及其运行结果就会有问题，如下。很容易看出，一个学生不会出现两次语文成绩。

```java
public static void main(String[] args) {
    Student student = new Student("A",23,"湖北省武汉市");
    Student student1 = new Student("A",23,"湖北武汉");
    System.out.println(student.equals(student1));

    Map<Student,Integer> mathScore = new HashMap<>();
    mathScore.put(student,99);
    mathScore.put(student1,100);
    for (Student key : mathScore.keySet()) {
        System.out.println("Key = " + key);
    }
}

运行结果如下：
true
Key = Student{name='A', age=23, address='湖北武汉'}
Key = Student{name='A', age=23, address='湖北省武汉市'}
```

所以需要我们重写hasCode方法，如下：

```java
@Override
public int hashCode() {
    return 13*name.hashCode()+17*age.hashCode();
}
```

运行以上代码结果则是：

```java
true
Key = Student{name='A', age=23, address='湖北省武汉市'}
```

### 总结

1） 按照规范性，只要覆写 equals，就必须覆写 hashCode。

2） 因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的对象必须覆写 这两种方法。

3） 如果自定义对象作为Map 的键，那么必须覆写hashCode 和 equals。

