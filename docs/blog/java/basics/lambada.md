● lambada 表达式实质上是一个匿名方法，但该方法并非独立执行，而是用于实现由函数式接口定义的唯一抽象方法
● 使用 lambda 表达式时，会创建实现了函数式接口的一个匿名类实例
● 可以将 lambda 表达式视为一个对象，可以将其作为参数传递
1. 函数式接口
函数式接口是仅含一个抽象方法的接口，但可以指定 Object 定义的任何公有方法。
● 以下是一个函数式接口：
```java
@FunctionalInterface
public interface IFuntionSum<T extends Number> {
    T sum(List<T> numbers);      // 抽象方法
}
● 以下也是一个函数式接口：
@FunctionalInterface
public interface IFunctionMulti<T extends Number> {
    void multi(List<T> numbers); // 抽象方法
    
    boolean equals(Object obj);  // Object中的方法
}
● 但如果改为以下形式，则不是函数式接口：
@FunctionalInterface
public interface IFunctionMulti<T extends Number> extends IFuntionSum<T> {
    void multi(List<T> numbers);
    
    @Override
    boolean equals(Object obj);
}
```
// IFunctionMulti 接口继承了 IFuntionSum 接口，此时 IFunctionMulti 包含了2个抽象方法
tip 1: 可以用 @FunctionalInterface 标识函数式接口，非强制要求，但有助于编译器及时检查接口是否满足函数式接口定义
tip 2: 在 Java 8 之前，接口的所有方法都是抽象方法，在 Java 8 中新增了接口的默认方法
2. lambda 表达式
● lambda 表达式的2种形式
包含单独表达式 ：parameters -> an expression
list.forEach(item -> System.out.println(item));
包含代码块：parameters -> { expressions };
```java
list.forEach(item -> {
  int numA = item.getNumA();
  int numB = item.getNumB();
      System.out.println(numA + numB);
});
```
左侧指定 lambda 表达式需要的参数，右侧指定 lambda 方法体
● 上文提到，lambda 无法独立执行，它必须是实现一个函数式接口的唯一抽象方法。
每个 lambda 表达式背后必定有一个函数式接口，该表达式实现的是这个函数式接口内部的唯一抽象方法。
譬如以下 lambda 表达式：
list.forEach(item -> System.out.println(item));
我们看 ArrayList 中 foreach 方法：
```java
@Override
public void forEach(Consumer<? super E> action) {
  // 太长了，不看了~
}
其中 Consumer 是一个函数式接口：
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);    // lambda 表达式 item -> System.out.println(item) 实现了该方法

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```
Consumer 接口是 Java 8 中预先定义的函数式接口，java.util.function 包下都是些预定义的函数式接口
function 包下的部分接口使用了泛型，具有很强的通用性，在自定义函数式接口前，不妨去这个包下找找有没有能用的
● 在执行 lambda 表达式时，会自动创建一个实现了目标函数式接口的类实例，该类实例是一个匿名内部类。
同样以 list 的 foreach 方法为例：
```java
public class LambdaDemo {
  public static void main(String[] args) {
        List<String> list = Arrays.asList("item1", "item2");
        list.forEach(item -> System.out.println(item));
    }
}
```
用 Java VisualVM 追踪代码运行过程中的堆内存，发现会生成以下实例：


生成的实例类名为 LambdaDemo$$Lambda$1，根据匿名内部类的命名规则可知，这是 LamddaDemo的 一个匿名内部类。（普通匿名内部类 和 lambda匿名内部类的命名规则见下文）。
● 同样，由于 lambda 表达式在执行时会生成目标函数式接口的类实例，因此我们可以做以下操作：
// 有以下函数式接口
```java
@FunctionalInterface
public interface IFuntionSum<T extends Number> {
    T sum(List<T> numbers);
}

// 将一个lambda表达式赋值给函数式接口引用（类型须兼容）
IFuntionSum<Long> function = list -> {
  Long sum = 0L;
  for (Long item : list) {
      sum += item;
  }
  return sum;
};
function.sum(Arrays.asList(1L, 2L));   // 执行结果为3L
```
在开发过程中，我们可以将 lambda 表达式等同于一个对象使用，对其声明、引用、传递。
● 匿名内部类 和 lambda 表达式匿名内部类的命名规则
内部类的命名规则：外部类名 + $ + 内部类名
匿名类的命名规则：外部类名 + $ + (1, 2, 3，第几个匿名类就显示几)
lambada 匿名内部类的命名规则：外部类名 + $$ + Lambda + $ + (1, 2, 3，第几个lambda表达式就显示几)
假设外部类中用到了2个lambda 表达式，则生成的2个匿名内部类的命名分别为 ：
外部类名$$Lambda$1 和 外部类名$$Lambda$2
3. lambda 表达式规约
● lambda 表达式的参数可以通过上下文推断，如果需要显示声明一个参数的类型，则必须为所有的参数声明类型。
```java
@FunctionalInterface
public interface IFunctionMod {
  boolean (int n, int d);
}
  
IFunctionMod function = (n, d） -> (n % d) == 0          // 合理，n 和 d 的类型通过上下文推断
IFunctionMod function = (int n, int d) -> (n % d) == 0   // 合理，指定 n 和 d 的类型
IFunctionMod function = (int n, d) -> (n % d) == 0       // 不合理，须显示声明所有参数类型
```
● lambda 表达式中抛出的异常需要与目标函数式接口的抽象方法抛出的异常类型兼容：
以下是合理的：
```java
@FunctionalInterface
public interface IFunctionMod {
  boolean (int n, int d) throw Exception; 
}

IFunctionMod function = (n, d) -> {
    if (d == 0) {
      // IOException是EXception 的子类，通过类型转换，IOException 可转换为 Exception
      throw new IOException("test"); 
    } 
    return n % d == 0;
};
```
如果反一下，就不行了：
```java
@FunctionalInterface
public interface IFunctionMod {
  boolean (int n, int d) throw IOException; 
}

IFunctionMod function = (n, d) -> {
    if (d == 0) {
      // 父类不能通过自动类型转换转为子类，lambda 表达式抛出的异常类型与抽象方法抛出的异常类型不兼容
      throw new Exception("test");
    } 
    return n % d == 0;
};
```
● lambda 表达式中参数类型需要与目标函数式接口中抽象方法的参数类型兼容。
tip ：从接口与实现的角度，可以很容易理解抛出异常兼容 和 参数类型兼容 这2点。
4. 方法引用
可以引用已有方法构造 lambda 表达式，这里给一个例子，不做详细解释：
list.forEach(System.out::print)