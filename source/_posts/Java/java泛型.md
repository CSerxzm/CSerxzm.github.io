---
title: java泛型
comments: false
date: 2021-12-15 20:56:52
categories:
  - java
tags:
  - 泛型
---

# 泛型

泛型就相当于 C++中的模板，来适应任何类型。使用泛型可以不必对类型进行强制转换，它是通过编译器对类型进行检查。

```java
//没有使用泛型的后果
ArrayList list = new ArrayList();
list.add("Hello");
// 获取到Object，必须强制转型为String，这样容易出现ClassCastException，即“误转型”
String first = (String) list.get(0);
```

同时，对于泛型，需要注意的是它的**向上转型**，如下面类型`ArrayList<T>`可以向上转型为`List<T>`。

要特别注意:不能把`ArrayList<Integer>`向上转型为`ArrayList<Number>`或`List<Number>`。

也就是说，**可以把`ArrayList<Integer>`向上转型为`List<Integer>`(`T`不能变!)，但不能把`ArrayList<Integer>`向上转型为`ArrayList<Number>`(`T`不能变成父类)**。

```java
public class ArrayList<T> implements List<T> {
    ...
}
List<String> list = new ArrayList<String>();
```

# 使用泛型

使用`ArrayList`时，如果不定义泛型类型时，泛型类型实际上就是`Object`，此时只能把`<T>`当作`Object`使用，使用的时候需要强制转换，没有发挥泛型的优势；当定义泛型类型`<String>`后,`List<T>`的泛型接口变为强类型`List<String>`。

```java
//编译器警告:
List list = new ArrayList();
list.add("Hello");
String first = (String) list.get(0);
//无编译器警告:
List<String> list = new ArrayList<String>();
list.add("Hello");
//无强制转型:
String first = list.get(0);
```

## 泛型接口

在接口中使用泛型，如`Arrays.sort(Object[])`可以对任意数组进行排序，但待排序的元素必须实现`Comparable<T>`这个泛型接口。可以直接对元素进行排序。

```java
class Person implements Comparable<Person> {
    String name;
    int score;
    public int compareTo(Person other) {
        return this.name.compareTo(other.name);
    }
}
public static void main(String[] args) {
        Person[] ps = new Person[] {
            new Person("Bob", 61),
            new Person("Alice", 88),
            new Person("Lily", 75),
        };
        Arrays.sort(ps);
        System.out.println(Arrays.toString(ps));
}
```

# 编写泛型

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() {
        return first;
    }
    public T getLast() {
        return last;
    }
}
```

在编写泛型类的时候，要特别注意，泛型类型`<T>`不能用于静态方法。

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() { ... }
    public T getLast() { ... }

    // 对静态方法使用<T>,但是这里可以发现导致编译错误，无法在静态方法create方法参数和返回类型上使用泛型类型T。
    public static Pair<T> create(T first, T last) {
        return new Pair<T>(first, last);
    }
}
```

静态方法不能使用泛型么？可以的。我们可以在 static 修饰符后面加一个`<T>`,编译就能通过，但实际上静态函数上的`<T>`和`Pair<T>`类型的`<T>`没有任何关系，一般会防止混淆，会将静态方法的泛型换成其他类型。

```java
public class Pair<T> {
    private T first;
    private T last;
    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() { ... }
    public T getLast() { ... }

    // 可以编译通过:
    public static <T> Pair<T> create(T first, T last) {
        return new Pair<T>(first, last);
    }
    // 静态泛型方法应该使用其他类型区分:
    public static <K> Pair<K> create(K first, K last) {
        return new Pair<K>(first, last);
    }
}
```

泛型也可以使用多个泛型，例如希望`Pair`不总是存储两个类型一样的对象，就可以使用类型`<T, K>`，在常见 Map 中使用的也就是这种方法：

```java
public class Pair<T, K> {
    private T first;
    private K last;
    public Pair(T first, K last) {
        this.first = first;
        this.last = last;
    }
    public T getFirst() { ... }
    public K getLast() { ... }
}
```

# 泛型继承

一个类可以继承自一个泛型类。如下面的示例，父类类型是`Pair<Integer>`，子类类型是`IntPair`，使用时，子类`IntPair`并没有泛型类型，正常使用即可。

```java
public class IntPair extends Pair<Integer> {
}
IntPair ip = new IntPair(1, 2);
```

虽然无法获取`Pair<T>`的`T`类型，即给定一个变量`Pair<Integer> p`，无法从`p`中获取到`Integer`类型。

但在父类是泛型类型的情况下，编译器就必须把类型`T`（对`IntPair`来说，也就是`Integer`类型）保存到子类的 class 文件中，不然编译器就不知道`IntPair`只能存取`Integer`这种类型。

# extends 和 super

## 擦拭法

**擦拭法，是指虚拟机对泛型其实一无所知，所有的工作都是编译器做的**，这就会导致编译器把类型`<T>`视为 Object，以及编译器根据`<T>`实现安全的强制类型装换。

但是，这样会导致泛型会有一些局限性：

1. `<T>`不能是基本类型，例如 int，double，因为实际类型是 Object，Object 类型无法持有基本类型。
2. 无法获取带泛型的 Class，如对于`Pair<String>`获得的 class 其实是 Pair.class。简而言之，所有泛型实例，无论`T`的类型是什么，`getClass()`返回同一个`Class`实例，因为编译后它们全部都是`Pair<Object>`。
3. 无法判断带泛型的类型，编译器会抛出异常。
4. 不能实例化`<T>`类型，如需要实例化只能借助额外的`Class<T>`参数。

```java
//局限1
Pair<int> p = new Pair<>(1, 2); // compile error!
//局限2，下面的示例得到的对象是Pair.class，所以无论是泛型对象为String或者Integer，它们都相等。
Pair<String> p1 = new Pair<>("Hello", "world");
Pair<Integer> p2 = new Pair<>(123, 456);
Class c1 = p1.getClass();
Class c2 = p2.getClass();
System.out.println(c1==c2); // true
System.out.println(c1==Pair.class); // true
//局限3
Pair<Integer> p = new Pair<>(123, 456);
if (p instanceof Pair<String>) { // Compile error:
}
//局限4
public class Pair<T> {
    private T first;
    private T last;
    public Pair(Class<T> clazz) {
        first = new T();// Compile error:
        //使用这样的方式则可
        first = clazz.newInstance();
        last = clazz.newInstance();
    }
}
```

## extends 通配符

针对`Pair<Number>`写个静态方法，他接受的参数类型是`Pair<Number>`。通过调用`int sum = PairHelper.add(new Pair<Number>(1, 2));`是可以正常编译的，但是实际上变量 1 和 2 也是 Integer 类型，当直接传入`<Integer>`类型,就会编译出错，因为`Pair<Integer>`不是`Pair<Number>`的子类，故`add(Pair<Number>)`不接受参数类型为`Pair<Integer>`。

```java
public class PairHelper {
    static int add(Pair<Number> p) {
        Number first = p.getFirst();
        Number last = p.getLast();
        return first.intValue() + last.intValue();
    }
}
//incompatible types: Pair<Integer> cannot be converted to Pair<Number>
Pair<Integer> p = new Pair<>(123, 456);
int n = add(p);
```

有没有办法使得方法参数接受`Pair<Integer>`？使用`Pair<? extends Number>`使得方法接收所有泛型类型为 Number 或 Number 子类的 Pair 类型可以达到我们的目的。

```java
static int add(Pair<? extends Number> p) {
    Number first = p.getFirst();
    Number last = p.getLast();
    return first.intValue() + last.intValue();
}
```

通过这样的修改，给方法传入`Pair<Integer>`类型时，它符合参数`Pair<? extends Number>`类型。这种使用`<? extends Number>`的泛型定义称之为上界通配符（Upper Bounds Wildcards），即把泛型类型`T`的上界限定在`Number`。除了可以传入`Pair<Integer>`类型，我们还可以传入`Pair<Double>`类型，`Pair<BigDecimal>`类型等等，因为 Double 和 BigDecimal 都是 Number 的子类。

但是，对于下面的`int add(Pair<? extends Number> p)`方法，这样依然会出错，原因`p.setFirst(...)`在于擦拭法。如传入的`p`是`Pair<Double>`，显然它满足参数定义`Pair<? extends Number>`，然而，`Pair<Double>`的`setFirst()`显然无法接受 Integer 类型，但可以接受 null，这里就需要用到 super 通配符。

```java
static int add(Pair<? extends Number> p) {
    Number first = p.getFirst();
    Number last = p.getLast();
    //编译错误
    p.setFirst(new Integer(first.intValue() + 100));
    p.setLast(new Integer(last.intValue() + 100));
    return p.getFirst().intValue() + p.getFirst().intValue();
}
```

在这里也就可以总结`<? extends Integer>`的限制，

- 允许调用`get()`方法获取 Integer 的引用
- 不允许调用`set(? extends Integer)`方法并传入任何 Integer 的引用（null 除外）

**在定义泛型类的时候，同样可以使用 extends 限定 T 的类型**。

```java
public class Pair<T extends Number> { ... }
Pair<Number> p1 = null;
Pair<Integer> p2 = new Pair<>(1, 2);
//报错
Pair<String> p1 = null; // compile error!
Pair<Object> p2 = null; // compile error!
```

## super 通配符

我们考虑以下方法，传入`Pair<Integer>`是允许的，但是传入`Pair<Number>`是不允许的。

和 extends 通配符相反，这次我们希望接受`Pair<Integer>`类型，以及`Pair<Number>`、`Pair<Object>`，因为 Number 和 Object 是 Integer 的父类，`setFirst(Number)`和`setFirst(Object)`实际上允许接受 Integer 类型

```java
void set(Pair<Integer> p, Integer first, Integer last) {
    p.setFirst(first);
    p.setLast(last);
}
//使用super解决该问题
void set(Pair<? super Integer> p, Integer first, Integer last) {
    p.setFirst(first);
    p.setLast(last);
}
```

注意，`Pair<? super Integer>`表示，方法参数接受所有泛型类型为`Integer`或`Integer`父类的`Pair`类型。

```java
Pair<Number> p1 = new Pair<>(12.3, 4.56);
Pair<Integer> p2 = new Pair<>(123, 456);
set(p1,100,200);
set(p2,200,200);
```

因此，使用`<? super Integer>`通配符表示：

- 允许调用`set(? super Integer)`方法传入 Integer 的引用；
- 不允许调用`get()`方法获得 Integer 的引用。

唯一例外是可以获取 Object 的引用：`Object o = p.getFirst()`。

## extends 和 super 对比

根据上面的描述，**作为方法参数**，`<? extends T>`类型和`<? super T>`类型的区别如下，也就是一个只读不写，一个只写不读：

- `<? extends T>`允许调用读方法`T get()`获取 T 的引用，但不允许调用写方法`set(T)`传入 T 的引用（传入 null 除外）；
- `<? super T>`允许调用写方法`set(T)`传入 T 的引用，但不允许调用读方法`T get()`获取 T 的引用（获取 Object 除外）。

```java
public class Collections {
    // 把src的每个元素复制到dest中:
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (int i=0; i<src.size(); i++) {
            T t = src.get(i);
            dest.add(t);
        }
    }
}
//这个copy()方法的另一个好处是可以安全地把一个List<Integer>添加到List<Number>，但是无法反过来添加。
// copy List<Integer> to List<Number> ok:
List<Number> numList = ...;
List<Integer> intList = ...;
Collections.copy(numList, intList);
// ERROR: cannot copy List<Number> to List<Integer>:
Collections.copy(intList, numList);
```

### PECS 原则

到底如何使用，为了便于记忆，我们可以用 PECS 原则：Producer Extends Consumer Super。即，如果需要返回 T，它是生产者(Producer)，要使用 extends 通配符；如果需要写入 T，它是消费者(Consumer)，要使用 super 通配符。

## 无限定通配符

实际上，Java 的泛型还允许使用无限定通配符（Unbounded Wildcard Type），即只定义一个`<?>`。

因为`<?>`通配符既没有 extends，也没有 super，因此：

- 不允许调用`set(T)`方法并传入引用（null 除外）；
- 不允许调用`T get()`方法并获取 T 引用（只能获取 Object 引用）

也就是不能读，也不能写，只能做一些 null 的判断，但是大多数情况下，可以引入泛型参数`<T>`消除`<?>`通配符，`<?>`通配符有一个独特的特点,安全的向上转型，也就是`Pair<?>`是所有`Pair<T>`的超类。

```java
static boolean isNull(Pair<?> p) {
    return p.getFirst() == null || p.getLast() == null;
}
// 安全地向上转型，定义的为Pair<T>
Pair<Integer> p = new Pair<>(123, 456);
Pair<?> p2 = p;
System.out.println(p2.getFirst() + ", " + p2.getLast());
```
