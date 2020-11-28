# 1. java 基础

*  变量：final 表示常量，习惯上，常量名使用全大写。

* 运算符：除了常用运算外，Math 类提供了一些方便的运算，如 Math.pow, Math.sqrt 等。

* 强制转换：向下转换会损失精度，此外，不要将 boolean 转换成其他数值类型，如果少数情况实在需要，可以用 b ? 1 : 0。

* String 类型：**String 类型是不可变字符串，比如想要修改结尾位置，需要用 substring 来做拼接。虽然不可变导致了修改的效率问题，但是这样可以方便的共享同一个字符串变量，在这方面提升效率**

  * 子串：string 实例的 substring(begin, end) 方法将返回一个新的子串实例
  * 拼接：可以用+进行拼接，如果将字符串与非字符串进行拼接(非字符串在右侧)，则会转换为字符串拼接
  * 比较：equals 方法可以和另一个字符串进行比较，equalsIgnoreCase 可以无视大小写比较。切记，不能用==进行比较，java 中==只用来比较两个引用是不是存储在相同的位置(也就是指针指向地址是否相同)
  * 空串和 null：空串是长度为 0 的实际对象，也就是 str.length() == 0。null 表示该变量目前没有任何对象与之关联
  * 构建：有时候需要不断拼接，来构成字符串，每次拼接效率过于低下，可以用 StringBuilder 类的 append 方法来构建，构建完调用 toString 方法转变为String 类型。

* 输入输出

  * 输出采用 System.out 相关操作即可输出，如 format、println 等函数

  * 输入：输入较为复杂，具体如下代码：

    ```java
    Scanner in = new Scanner(System.in); // 创建一个 Scanner 类，并和标准输入关联
    in.nextLine(); // 读入一行数据 
    in.next(); // 以空格分隔读入一个数据
    in.nextInt(); // 读入一个整数
    ```

* 文件

  `Scanner in = new Scanner(Paths.get("myfile.txt"), "UTF-8"); // 用 file 对象来创建一个 scanner` 可以看到，使用文件和标准 io 一样方便，但是，文件相关操作可能会引发一场，因此需要在 main 中用 `throw IOException`表示可能会抛出异常。

* switch

  case 的标签可以说 char byte short int 的常量表达式、枚举常量、字符串字面量。

# 2. 对象与类

* 构造器

  ```java
  public ClassName(parameters...) {
  	statements;
  }
  ```

  构造器形式如上，采用无返回值的和类同名的函数，且只能和 new 操作符一起使用，构造后不可再调用。(**不要在构造函数中，定义与实例域同名的局部变量，书上说会导致无法设置，但是测试发现用 this 指针还是可行的**)

  * 可以重载多个
  * 参数可以任意个，包括0 个
  * 没有返回值，总是和 new 一起使用

* this 指针：java 和 cpp 一样，成员函数都有默认的 this 指针作为隐式参数
* java 所有成员函数都必须写在类内，但并不代表所有的都是内联的
* 返回可变对象：如果需要返回一个数据域可变对象，那么由于 java 会让返回的引用和数据域引用相同的位置，导致出错，正确的方式是返回一个可变对象的拷贝。
* 访问权限：方法可以访问对象的私有数据，**一个方法可以访问所属类对象的所有对象实例的私有数据！！！cpp 也是这样**
* 可以用关键字 final 声明常量，这种成员必须在构造中被赋值
* 析构：由于 java 采用 gc，所以并不用维护内存，因此不支持析构函数。但是当一些类使用了其他系统资源，可以用 finalize 方法，这个方法会在 gc 清楚对象前被调用。(**不要依赖 finalize 方法回收任何短缺的资源，因为并不知道什么时候会被 gc**)
* 包：java 用包来对类进行组织，确保类名的唯一性
  * 嵌套的包之间没有任何关系，例如 java.util 和 java.util.jar 没有关系
  * 一个包可以访问所属包的所有类，以及其他包的共有类，可以通过`包名.类名`的形式访问，更简单的方式是 import 包名，或者直接 import 到一个特定的类。（如果遇到冲突，比如两个包都有 Date 类，则需要在调用的时候加上完整的包名）
  * 想要把类放入包中，需要在源文件起始添加`package 包名`，如果没有，则全部类都会被添加到一个默认的包中，默认包是一个没有名字的包
  * 包作用域：private 限制只能被定义他的对象访问，public 则可以被所有对象访问。如果没有加限制符，那么可以则作用域是整个包内可以访问。(通常来说，对于某个类，对包可见是合理的，但是对于变量，通常都是不合理的)

* 类设计技巧(和 cpp 基本一致)：
  * 数据私有
  * 数据一定要初始化（虽然类的实例域不像局部变量，编译器会初始化，但是最好不要依赖系统的默认值）
  * 不要在类中使用过多基本类型
  * 不是所有数据域都需要独立的域访问器和更改器（有些成员初始化后，就不该被更改）
  * 类需要合适的进行分解、设计
  * 类名和方法要体现他们的用处

# 3. 继承

## 3.1 类、超类、子类

* 继承

  ```java
  class Manager extends Employee {
  	...
  }
  ```

  java 继承关键字为 extends，且只有公有继承，且 java 不支持多继承！！！

* 覆盖：重写基类(也叫超类)的方法

  如果发生重写方法需要调用超类的同名方法，可以用 super.methodName 的方法（等价于 cpp 的 baseName::methodName）。super 除了可以调用基类的方法，还可以在构造函数中，调用基类的构造函数。覆盖方法的返回值，需要是元返回类型，或者是他的子类型。

* 动态绑定：java 默认是动态绑定的，只有 private、static 和 final 是静态绑定的

* 强制转换：只能在继承层次内进行类型转换，将超类转换成子类之前，应该用 instanceof 进行检查，否则如果不能，将会抛出 ClassCastException 异常。

* 抽象类和方法：

  ```java
  public abstract class Person {
    private String name;
    public Person(String name) {
      this.name = name;
    }
    public abstract String getDecription();
  }
  ```

  * 抽象关键字是 abstract，**只要包含至少一个抽象类，必须把这个类声明成抽象的，而且抽象类可以包含具体数据和方法，但是即使包含了具体数据抽象类不能被实例化**

  * 派生类在继承的时候，如果只实现了一部分抽象方法，那么他还是一个抽象类，如果实现了全部，那么就可以不是抽象的了
  * 一个类假使没有抽象方法，也可以被声明成抽象的，可以用来禁止类被实例化。但是抽象类虽然不能被实例化，但是可以定义抽象类的引用，用来指向派生类

## 3.2 Object 所有类的超类

* Object 类是所有类的始祖，Java 中的每个类都是由它扩展出来的，但是并不需要显示声明，只要没有明确的指出超类，Object 就将作为这个类的超类。也因此，熟悉 Object 提供的服务特别重要
* equals 方法：由于`==` 是比较两个对象是否有相同的引用，但是常常需要检测两个对象状态的相等性，那么就可以实习 equls。
  * 在实现子类 equals 时，可能需要比较成员变量的引用的值是否相等，比如两者的 `String name`是否相等，为了避免出现 null 的情况，需要调用`Objects.equals(a, b)`，这个函数当二者都不为 null 的时候，会调用 a.equals(b)，而且会处理任一为 null 的情况
  *  在实现子类的时候，首先调用超类的 equals 方法（不匹配，肯定不同），再实现子类的
* hashCode 方法：返回一个对象的 hash 值（必须与 equals 结果一致，也就是值相等的两个对象，hashcode 也相等）。
* toString 方法：返回表示对象值的字符串。通常遵循这样的格式：类名[变量名=变量值, ...]。之前说的+号相连，会把对象变成字符串，就是调用的这个方法。

## 3.3 泛型数组列表

动态数组，ArrayList<ClassName> staff = new ArrayList<>(); 支持 for each 循环。

* add 方法：添加一个新的数据成员到数组。add(ele)表示尾部插入，add(n, ele)表示在第 n 个元素位置插入
* capacity：staff.ensureCapacity(100)来指定 capacity 为 100，或者在初始化的括号内指定 capacity
* size 方法：返回数组大小
* trimToSize 方法：将 cap 设置为 size，让 gc 可以回收多余的空间
* set 方法：staff.set(i, harry)，设置第 i 个元素为 harry（**java 没有重载，感觉好多时候繁琐的一逼。。。**）
* get 方法：ele = staff.get(i)，获取第 i 个元素
* toArray 方法：转换成 array
* remove 方法：remove(n)删除第 n 个元素

## 3.4 变参函数和枚举类

* 变参函数（**可变参数需要是参数列表的最后一个**）

  ```java
  public static double max(T... values); // Type + ... 表示类型为 T 的可变参数，编译器将 new T[]{para1, para2, ...}这样传递给函数。
  ```

* 枚举类

  * `public enum Size {SMALL, MEDIUM, LARGE};`
  * 必要的话，可以给没具体添加一些构造器，方法，和域。也可以给每个枚举值指定值

# 4. 接口、lambda 表达式与内部类

## 4.1 接口

* 简述

  ```java
  public interface Comparable<T> {
    int compareTo(T other);
  }
  ```

  * 接口中所有方法自动属于 public，所以不需要也不建议写上
  * 接口可以有多个方法，也可以有常量，但是绝对不能有实力域，java 8 之后可以在接口中提供简单方法
  * 一个类可以实现多个接口，用 implements 关键字

* 特性

  * 不能实例化，但是可以声明接口的变量，引用实现了这个接口的变量
  * 可以被扩展，和继承一样利用 extends 关键字
  * 可以包含常量，且接口内的变量默认为静态常量，也就是`double PI = 3.14`等价于`public static final double PI = 3.14`
  * 一个类只能继承一个超类，但是可以实现多个接口（**感觉这个就是接口出现的原因，否则采用抽象类也可以做到类似的事情**）
  * 静态方法过去通常会声明一个伴随类，但是 java 8 后可以直接放在接口内部了
  * 利用 default 关键字，可以给接口提供一个默认的实习（**某些情况下很有用，比如只关系接口的某几个方法，提供了默认方法就不需要事先其他不关心的接口。此外，如果某个类新添加了一个方法，且没有默认实现，那么之前实现这个方法的，则不能被编译**）

* 冲突解决：如果接口将一个方法定义为默认方法，另一个超类或者接口中，也定义了相同的方法，那么在继承 or 实现的时候，则会发生冲突

  * 超类优先于接口
  * 接口冲突的话，则必须通过覆盖来解决。（覆盖中，可以明确指定调用某一个方法）

## 4.2 lambda 表达式

lambda 出现的目的主要是简化传递代码段的任务，否则必须要专门声明一个类，然后再创建个实例进行传递。

* 形式

  ```java
  (para...)->statement //如果一个表达式放不下，则用大括号
  ```

  * 不需要指定返回类型
  * 对于函数式接口的参数，可以传递一个对应的 lambda 表达式。（**函数式接口：只有一个抽象方法的接口，例如 Comparator 接口**）

* 方法引用

  有时候某个类的方法，就可以满足我们的需求，此时可以直接传递这段代码

  ```java
  // 三种形式
  object::instanceMethod;
  Class::staticMethod;
  Class::instanceMethod;
  ```

  对于第三种情况，第一个参数，会成为方法的目标。

* 变量捕获

  java 和 cpp 一样可以在 lambda 内部捕获外部变量而形成闭包。但是为了并发安全考虑，java 只允许引用值不会改变的变量。（**这里需要注意！不论是在 lambda 内部，还是在外部其他地方有更改，都不行。意思就是，lambda 捕获的，必须实际上是最终变量，无论它是否被声明成 final 了**）

## 4.3 内部类

* 内部类的用处主要有以下三点：

  * 内部类可以访问定义该类所在作用域的数据，包括私有数据

  * 内部类可以对包内其他类隐藏（**类的作用域只能 public 或者包内可见**）
  * 使用匿名内部类进行回调函数的设置（**java 8 中可以用 lambda 代替了**）

* 内部类既可以访问自身的数据域，也可以访问创建它的外围对象的数据域（**内部类有一个隐式引用，指向创建它的外部类对象，为 外部类名.this**）

* 内部类声明的所有静态域都必须是 final（不这么做，内部类的 static 域，有可能会有多个实例）

* 局部类：如果只在一个局部用到，可以在局部定义局部类，这样作用域只在这个局部内有效

* 匿名内部类：如果构造函数后面跟着一个大括号，正在定义的就是匿名内部类。这是过去注册回调的方式，现在还是建议使用 lambda。

  ```java
  new SuperType(construction paras) {
    inner class methods and data;
  }
  ```

  匿名类由于没有类名，所以不能有构造器，只能借由超类的构造器。

* 静态内部类：没有指向外围类的指针，相当于只起到隐藏到一个类内部的作用。（这部分跟 cpp 相似）

# 5. 异常、断言和日志

## 5.1 异常

<img src="/Users/jingyu/Documents/GitHub/book-note/assets/image-20201122111202357.png" alt="image-20201122111202357" style="zoom: 50%;" />

* 异常分类：异常主要分为两大类

  * Error：严重的错误，描述了 java 运行时系统内部错误和系统资源耗尽错误，程序一般对此无能为力，应用程序也不该抛出。典型的例子有：内存耗尽、无法加载 class、栈溢出等

  * Exception：运行时错误，可以被捕获并处理，也分为两大类

    * RuntimeException：程序错误引起的异常，如访问 null 指针
    * IOException：非程序错误引起的异常，如访问不存在的文件

    java 规定，Exception 及其子类，但不包括 RuntimeException 及其子类称为受查异常，必须被捕获，Error 和 RuntimeException 及其子类，可以不被捕获。

* 基本形式：

  ```java
  try {
    // 可能抛出异常的代码
  } catch (Exception name) {
    
  } catch ... {
    
  } finally { // 这部分无论是否有异常发生，都会被执行
    
  }
  ```

  方法在调用的时候，通过 throws ExceptionType 来表示会抛出异常，在被调用的时候，必须被捕获，或者同样声明抛出异常。

## 5.2 日志

跳

# 6. 集合

## 6.1 集合框架

java 集合类库采用了接口与实现分离的结构，以队列为例，首先实现一个队列的接口，然后再用具体类实现这个接口。（**这种做法，方便用户实现某个接口，然后自己实习自己定制的集合类**）

* iterator 接口(迭代器)：java 集合类也有这迭代器类，内容如下

  ```java
  public interface Iterator<E> {
    E next();
    boolean hasNext();
    void remove();
    default void forEachRemaining(Consumer<? super E> action);
  }
  ```

  通过迭代器和他的 next 方法，可以访问容器内的每个元素。for each 循环可以与任何实现了迭代器接口的对象一起工作，编译器会将其翻译为带有迭代器的循环。remove 方法会移除上一次调用 next 的那个元素的方法（remove 之前如果没有调用 next 是不合法的，因此不能连续调用两次 remove，想要删除连续的值，必须中间插入一条 next）。

  **java 中的 iterator 跟 cpp 很不同，可以看到，迭代器本身并不能取出自己的值，取出值的方法只有调用 next。可以将 java 中的 iterator 认为是在两个元素之间，每次调用 next，则越过一个元素，并且返回他的引用（对于基本类型，不成立）。**

* Collection：集合的基本接口，扩展了 iterator 接口

  * `boolean add(E element)`: 向集合添加元素，成功则返回 boolean
  * `Iterator<E> iterator()`: 返回一个迭代器

## 6.2 具体容器

* 链表 `LinkedList<Integer> lists`
  * Lists.add/remove: 在链表尾添加/删除一个元素
  * ListIterator: 能够在当前 iterator 处，添加元素。而且支持双向遍历，除了 next 方法向后，还有 previous 方法向前，且 remove 方法依赖于迭代器的上一个状态。set 方法，更改上一次返回的元素。
  * 迭代器失效：不允许一个迭代器修改集合，另一个迭代器对其遍历，如果发生，会抛出异常。（**没有修改结构，只改变值的话，没事。实现上，迭代器记录修改的当前修改元素的次数，当遍历时跟容器内部修改次数进行比较，不一致就抛出异常**）
  * Lists.get(index): 获取第 index 个元素
* 动态数组 ArrayList：实现了类似接口，但是底层是数组，因此 get 和 set 的效率很高。
* 散列集 HashSet：利用 hashCode来产生哈希值，对于自定义类，需要自己负责这个类的 hashcode 实现。
  * Java8 采用数组+链表的形式来解决冲突，当桶内元素达到一定值时，会将链表转换为 BST，来提高桶内的效率
  * add 添加元素，contains 检查是否在容器内，remove(object o)删除元素 o，如果存在的话
* 有序集合 TreeSet ：利用红黑树组织，相当于 cpp 的 set。因此传入的类必须实现 Comparable 接口，或者构造时，传入一个 Comparator。
  * first/last:返回最大、最小的元素
  * Higher/lower：大于或小于 value 的最接近的那个元素，没有返回 null
  * ceiling/floor: 返回大于等于或者小于等于 value 的最接近元素，没有返回 null
  * pollFirst/pollLast：删除集合中最大元素或最小元素，为空返回 null
* 队列 Queue：java 和 cpp 不一样，java 中的 Queue 是一个借口，常用的实现类是 LinkedList
  * Add：添加到队尾，成功返回 true，队列满了(有的实现，并不是所有都会满)，抛出异常。
  * offer：跟 add 类似，失败返回 false，不抛异常
  * remove(): 队首出队列，为空抛出异常
  * pool：同 remove，为空返回 null
  * Element：返回队首，为空抛出异常
  * peek：同 element，为空返回 null

## 6.3 映射（k v 对）

提供了 HashMap 和 TreeMap 两类，都实现了 Map 接口

* Map 接口

  * put(key, value)：放入一个键，如果之前存在，则会覆盖，并且返回上一个值
  * get(key)：返回键对应的 value，不存在返回 null
  * getOrDefault(key, defaultValue)：返回 key 对应的 value，不存在，则设为 defaultValue 添加进集合
  * remove(key)：删除指定 key
  * Set<K> keySet(): 返回一个 key 的 set
  * Collection<V> values: 返回 values 的集合
  * Set<Map.Entry<K, V>> entrySet()：返回一个键值对的集合

  上面三种视图，可以用来删除元素，但是不能增加元素。

# 7. 并发

## 7.1 线程

* Runnable 接口：是一个函数接口，因此可以用 lambda 创建一个实例

  ```java
  public interface Runnable {
    void run();
  }
  // 由 Runnable 创建一个 Thread 对象，并用 start 启动线程
  Tread t = new Tread(r);
  t.start();
  ```

  **不用调用 Tread 类或者 Runnable 对象的 run 方法，那样只会在当前线程执行函数**

  线程直到 return，或者出现了没有捕获的异常，才会终止

* 终止相关：线程有一个中断标志位，每个线程都应该不时检测这个标志，以判断是否被中断

  * Tread.currentTread().isInterrupted(): 判断是否被中断，不会改变中断状态
  * Tread.currentTread().interrupted(): 判断是否被终端，会清除中断状态
  * interrupt(): 向线程发送中断请求，如果目标线程被 sleep 等阻塞，会抛出异常

* 线程状态：利用 join() 可以等待指定线程终止，getState() 可以获取线程状态

  * New：新创建
  * Runnable：可运行，调用 start 方法之后，处于的状态，线程可能被调度运行，也有可能还没
  * Blocked：被阻塞，获取已经被占有的资源（如锁）造成等待时的状态
  * Waiting：等待，等待其他线程达到某些条件，主动放弃时间片的状态
  * Timed Waiting：计时等待，调用了带有超时参数的一些调用
  * Terminated：被终止

## 7.2 同步

* 重入锁 ReentrantLock：跟互斥锁类似，用于维护临界区。**如果有 catch 语句，需要把解锁放在 finally 中，让锁正确的释放**

  * ReentrantLock()：获取一个可重入锁
  * ReentrantLock(boolean fair)：构建一个带有公平策略的锁，这个锁会偏爱等待时间最长的线程，但是会大大降低性能。
  * lock()：获取这个锁，可重入，有一个 state 记录锁的次数，每次重入都必须 unlock
  * unlock()：释放这个锁

* 条件变量 Condition：调用 ReentrantLock::newCondition() 方法可以获得一个跟该互斥锁相关的条件变量，一个互斥锁可以有多个相关联的条件变量

  * await()：将线程放入等待集中，放入前需要加锁，获取后需要解锁，且跟 cpp 一样，采用 while 循环等待条件
  * signalAll(): 解除等待集中所有线程的等待状态
  * signal(): 从等待集合随机选择一个线程，接触其阻塞状态

* synchronized 关键字：为了提供一个粗粒度、简单的并发控制，java 在每个类内，都添加了一个内部锁，如果一个方法用 synchronize 声明，那么对象锁将会保护整个方法。并且改锁有一个内部条件变量

  * wait(): 等待
  * notify()/notifyAll(): 唤醒一个或者多个

  ```java
  public synchronized void method() {
    ...
  }
  // 等价于
  public void method() {
    this.intrinsiclock.lock();
    try {
      ...
    } finally {
      this.intrinsiclock.unlock();
    }
  }
  ```

* volatile：因为多线程由于缓存的问题，不能保证数据读到的是最新的，且编译器有可能会重排指令，使用 volatile 关键字可以避免这两个问题，但是需要注意，**volatile 不能保证原子性**，通常用于一个更新，其他多个线程读取的情况，不出现并发问题的同时，保证其他线程能读到最新的值

* 原子性：java 的原子操作

  * compareAndSet(oldValue, newValue):：一些简单的操作，比如自增，读取，可以直接利用原子操作，但是做一些复杂的操作，不能简单的利用一些原子操作的结合，因为多个原子操作的结合不是原子的
  
    ```java
    // 一个简单的，更新最大值的例子
    AtomicLong largest = new AtomicLong();
    largest.set(Math.max(largest.get(), observed)); // wrong！！操作非原子
    
    // true answer
    do {
      oldValue = largest.get();
      newValue = Math.max(oldValue, observed);
    } while (!largest.compareAndSet(oldValue, newValue));
    ```
  
  * updateAndGet()：传入一个 lambda，自动完成更新
  
  * accumulateAndGet()：利用一个二院操作符，来合并原子值和锁提供的参数，如 `accumulateAndGet(observed, Math::max)`
  
* 线程局部变量 TreadLocal<T> 用来每个线程创建一个实例

  * TreadLocal.withInitial()：为每个线程构造一个实例，且可以传入构造函数
  * get()：调用局部变量的 get 方法，可以获得该线程的实例

* 读写锁 ReentrantReadWriteLock 

* 阻塞队列：可以利用阻塞队列来简单的解决生产者-消费者模型中的并发问题。当生产者存入队列且队列满了，或者消费者消费队列时队列为空，会阻塞当前线程。

* 线程安全的集合：ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet、ConcurrentLinkedQueue。

* Callable 与 Future：Runnable 接口相当于没有参数和返回值，也不能捕获异常的一个异步方法。Callable 与 Runnable 类似，但是有返回值，且可以抛出异常

  ```java
  public interface Callable<V> {
    V call() throws Exception;
  }
  ```

  Futrue 可以保存异步计算的结果，可以启动一个计算，将 Future 交给某个线程，之后用 Future 对象来获取计算结果等。
