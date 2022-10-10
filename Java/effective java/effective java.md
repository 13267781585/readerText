# Effective java 读书笔记

### 第1条 采用静态方法代替构造方法

* 构造函数的缺点：

    * 没有函数名，若类中需要根据不同需求初始化类时表达功能不明确
    * 每次调用构造函数都会创建一个新的对象

* 采用静态方法

    * 优点：
        * 静态方法有函数名，函数功能可由函数名明确表示
        * 可以根据需求重复使用同一个对象（将该对象缓存起来重复使用），调用时可以返回现有的对象或者新创建新的对象
        * 可以返回 `返回类型` 的 `子类型`


* 例子：
    * 服务提供者框架

*** 

### 第4条 避免创建重复的对象

* 对于 `非可变对象`  
可以考虑多次重用

* 对于 `不可修改的可变对象`  
例如：
![4.1](图片\4.1.jpg)

每次执行 isBabyBooner() 函数时都会 创建 Calendar 和 两个 Date 对象， 这显然是不必要的，因为若每次传入的实参不变时，产生的结果也不会变化。改进后的代码如下：
![4.2](图片\4.2.jpg)
改进后的代码在类初始化时只创建了一次 isBabyBooner函数在 被调用时需要的对象，这样能显著地提高性能。

***

### 第7条 在改写equals方法时遵守的通用约定

* equals 方法的作用：比较 `引用` `值` 是否相同

* Object 类的 equals 方法  
比较的是对象的引用（两个对象是否为同一个）
```JAVA
  public boolean equals(Object obj) {
        return (this == obj);
    }
```


* 其他改写Object 类 equals 方法
比较的为对象的值是否逻辑相等（例如 String）
```JAVA
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

* 在改写 equals 方法时 需要遵守的通用规定  
    1. 自反性
    2. 对称性
    3. 传递性
    4. 一致性
        对于 x 和 y 引用，x.equals(y) 为 true 时， 若equals方法没有发生修改，那么 多次调用 equals 方法结果将不会发生改变。
    5. 对于 x.equals(null) ，结果一定返回 false

* Type1 instanceof Type2 若Type1 为 null 时， 无论 Type2 为何值，结果都返回 false

* == 比较的为 引用是否相等

* 改写 equals 方法的建议
    1. 不用将 equals 方法 改写得非常复杂
    2. 不要将使 equals 方法依赖于不可靠的资源
    3. 不要将 equals 方法中的 Object 参数类型 替换为 其他类型
    4. 比较类中的域是否相等时，应该从关键域开始比较，若相等再比较次关键域，当类中的域很多时，这样能显著提高性能。


***

### 第8条 改写 equals 方法时 总是要 改写 hashCode 方法

* hashCode 方法时通过对象实例中的数据，提取出一个32 位的整数作为标识该实例的唯一性

* Object 中的 hashCode 方法（native 说明该方法不是由 Java 实现，具体的实现方法在外部）

```java
  public native int hashCode(); 
```


* 为什么?

    * Object 规范中关于 hashCode 方法的通用约定：  
        1. 在同一个程序应用执行期间，如果关于 equals 方法比较的数据没有发生改变，那么多次调用该对象的 hashCode 方法将返回同一个值。

        2. 在同一个应用程序多次执行过程中，调用 hashCode 方法不一定返回相同的整数。

        3. 如果 两个对象通过 equals 方法判定为相同， 那调用 hashCode 方法必然返回相同的整数

        4. 反之不成立， 如果两个实例通过equals 方法判定为不相同，则 调用 hashCode 方法不一定要返回不一样的整数

    * 在修改了 equals 方法的类中必须要修改 hashCode 方法，因为不这样这 会违反 Object规范 中关于 hashCode 方法的约定的第二条（Object 中的 hashCode 方法 对于同一个类的不同实例返回的整数也是不相同的）， 这样导致该类无法结合基于所有 散列值（hash） 的集合类（hashMap、hashSet、hashtable）一起工作。
    
* 例子：

```java
//自定义类 
//改写了 equals 方法hash
//并没有改写 hashCode 方法
  public class A {
        private int value;
        A(){value = 0;}
        A(int value){this.value = value;}

        public int getValue(){return value;}
        public boolean equals(Object object){
            if(this == object)
                return true;
            if(!(object instanceof A))
                return false;
            if(((A) object).getValue() == value)
                return true;
            return false;
        }
    }

//测试类
  @Test
    public void test4() {
        A a = new A();
        A a1 = new A(1);
        A a2 = new A(1);
        System.out.println(a.equals(a1));
        System.out.println(a1.equals(a2));
        System.out.println(a.hashCode());
        System.out.println(a1.hashCode());
        System.out.println(a2.hashCode());

        Map map = new HashMap();
        map.put(a,"a");
        map.put(a1,"a1");
        map.put(a2,"a2");

        String temp = (String) map.get(new A());
        String temp1 = (String)map.get(a);
        System.out.println(temp);
        System.out.println(temp1);
    }

```

```java
//输出结果
false
true
1883919084
1860513229
1150538133
null
a
```

可以看出 同一个类 的不同实例 ，虽然对应数据相同，equals 方法为true ， 但是没有 hashCode 方法返回的整数 不相同，因为 hashMap 集合类是基于 散列值进行 值 的区分，因此 当我们 取值时传入的参数不是我们 调用 put() 函数时 传入的 键 原实例时，并不能 顺利返回 对应的 值。

* 书中提供一种改写 hashCode 方法(该方法不是很完善)：  
    1. 把某个 非零 的整数值保存在 result 的int变量中

    2. 对于 equals 方法中考虑的每一个对象域 f，实行一下操作：  
        a. boolean类型， 计算（ f ? 0 : 1 )
        b. byte 、 char 、 short 、 int 执行 （int）f
        c. long 计算 （int）（ f ^ ( f >>> 32 ) )
        d. float 计算 Float.floatToIntBits(f)
        e. double 计算 Double.doubleToLongBits(f),执行 c
        f. 对象引用，如果该类 equals 方法采用 递归 比较， 则 递归调用hashCode 方法 ，null 返回 0
        g. 数组 递归计算 每一个 元素的 hash， 采用3中的方法将hash结合起来

    3. 代入下列算法计算返回
```java
//选择 37 是因为 37 是 奇素数 ，如果为偶数，在数值溢出时会发生信息丢失，因为 2 相乘 等价于 移位运算，采用 素数的好处不明显，但习惯上这么用？？？？

    result = 37 * result + c;
```
    


以下为上例中改写的 hashCode 方法 和结果：

```java
public int hashCode(){
            return 17 * 37 + value;
        }

```

```java
false
true
629
630
630
a
a
```
***

### 第 10 条 谨慎地改写 Clone

* Object 中 clone 方法的源代码

```java
protected native Object clone() throws CloneNotSupportedException;
```

* Cloneable 接口中没有任何 需要重写的方法

```java
public interface Cloneable {
}

```

* 改写 Object 类中 的 clone 方法需要实现 Cloneable 接口，实现 接口不需要重写任何方法，它改变了 超类（Object） 的 Clone 方法，使他返回 该对象的逐域拷贝，否则抛出 CloneNotSupportException 异常。（接口不常用方法）   

* 这里有一个疑问，Cloneable 接口并没有重新任何方法，是如何改变 Object 类中 clone 方法的行为的？  因为 clone 方法 标识了 native ，说明不是java语言实现的，需要调用 外部 的方法实现，只不过在调用时需要判定类是否实现了 Cloneable 接口。

* 调用 clone 方法 返回 新实例 的过程 不需要 调用 构造函数。

* 为什么说谨慎地调用 clone 方法，这里主要 涉及到 “浅拷贝” 和 “深拷贝” 的问题。  
    1. clone 方法 拷贝的仅仅是 各个 域 的值，他并不关心 该 域 的性质，若 该域 为简单的数据类型（int、 long、 double），则没有什么影响，若 该域 为 对象引用 ，则 会产生 “两个 对象引用 指向用一个 对象 ” 的问题，若该域是 标识了 fianl，在短时间可以 正常工作，若 无标识，可能会产生 难以发现的错误。

    2. “浅拷贝” 只考虑 复制 域 的值，却不为 对象 开辟新的内存空间。


***

### 第 12 条 使类和成员的可访问能力最小化

* 在设计一个模块时应该做好充分地数据封装，只提供必要的 API 供他人调用，调用 API 的人不需要知道该模块的 具体 实现细节，只需要知道 API 的 作用，这样有利于 分工合作、共同开发，同时 降低 模块之间的 耦合度，便于后期的 维护和错误排查。

* 尽可能地使每一个 类 和 成员 不被外界访问，必要时可根据 具体 工程需求 编写 方法接口。

* 成员域 四种 可能 访问级别：
    1. 私有的（private）---只有声明该域的类内部可以使用该域
    2. 包级私有的（package-private）--- 声明该域的包内所有类都可以方法该域（没有说明访问级别时默认为包级私有的，也称为 “ 默认访问级别 ”
    3. 公有的（public）--- 任何地方都可以访问该域
    4. 受保护的（protected）--- 声明该域的 类 和 子类 都可以 访问该域，并且声明该域的包内的任何类都可以访问该域


* 改写超类方法时子类改写的方法访问级别不能低于超类中的方法，这样可使子类方法应用于任何超类使用的地方。

* 公有类不应该包含公有域。  如果一个公有类包含有公有域，外界可以随意改变该域的值，导致了该类失去了对该域的控制，会导致很多问题。

例外：通过 公有的 静态 final 暴露 常量是常见操作，这里的 域 一般为 原语类型 或者 对象引用，需要注意的是，当 该域指向 对象引用 时，要求该引用 也为被 final 限制，否则虽然指向 的对象不变，但是 对象中的数据却可以随意改变，这样没有任何意义。 

```java
public class A{
    public static final int VALUE = 1;
    public static final B b = new B();
}

class B{
    public final int YEAR = 12;
}

```

依照上面的规则，凡是 类中 有 公有的静态 final 数组时，总是错误的。

```java
     public static final int[] array={1};

    @Test
    public void test7(){
        System.out.println(array[0]);
        array[0] = 0;
        System.out.println(array[0]);
    }

    

```

```java
运行结果：
1
0

```

解决的办法：   

1. 将 公有数组变为私有，并转化为 公有的非可变列表

```java
 public static final List VALUE = Collections.unmodifiableList(Arrays.asList(1,2,3,4));

```



2. 将 公有数组变为私有，编写 公有方法 返回 该数组的拷贝
    
```java
private static final int[] array = {1,2,3,4};

public static final int[] values(){
        return array.clone();
    }

```

***

### 第 17 条 接口只是被用于定义类型

* 二进制兼容性
    1. 现在的软件越来越依赖 其他 厂商、作者开发的组件，二进制兼容性是指类在更新后能替换原来的版本，而不使 依赖 该类的其他组件和软件发生 错误。

    2. 例如：程序a 引用了类库 b，类库引用了类库 c，这是 类库c发布了新的版本，新老版本之间 二进制不兼容，新的版本不能替换旧的版本，这是依赖 类库c 的类库b 和 程序a 便不能正常工作。

    3. 在C++中，对域（数据成员或实例变量）的访问被编译成相对于对象起始位置的偏移量，在编译时就确定，如果类加入了新的域并重新编译，偏移量随之改变，原先编译的使用老版本类的代码就不能正常执行；虚拟方法调用也存在同样的问题。

    4. 引用博客地址：https://www.cnblogs.com/extjs4/p/9035449.html

***

### 第 26 条 谨慎地使用 重载

* 重载：一个类（编译多态性）
    1. 方法名必须一致
    2. 参数必须不同，可以是类型，也可以是顺序
    3. 不规定返回值类型必须一样   


    4. 缺点：容易造成隐蔽的错误   
        * 例子：


```java
class A{
    public static void a(Set set){
        System.out.println("this is a set");
    }

    public static void a(List list){
        System.out.println("this is a list");
    }

    public static void a(Collection collection){
        System.out.println("unknown");
    }

//测试类

 @Test
    public void test(){
        Collection[] collections = new Collection[]{new HashSet(),new ArrayList(),new HashMap().values()};
        A.a(collections[0]);
        A.a(collections[1]);
        A.a(collections[2]);
    }

/*
按照我们的思维 测试打印应该为 
this is a set
this is a list
unknown
*/

//实际的测试结果

unknown
unknown
unknown


```
        * 原因：





* 重写: 父类和子类 中（运行多态性）
    1. 相同的 方法名 
    2. 相同 参数类型 和 个数
    3. 相同 返回值类型
    4. 重写函数不能抛出 新异常 或 更加 宽泛的检查型异常
    5. 子类的 访问限定符 必须 大于 父类