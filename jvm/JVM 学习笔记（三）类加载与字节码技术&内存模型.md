## 四、类加载与字节码技术

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4736f9f9ad3c91252ff22cfbf1c6bd06.png)

### 1、类文件结构

通过 java .c 类名.java 编译 java 文件后，会生成一个 .class 的文件  
以下是字节码文件：
```
0000000 ca fe ba be 00 00 00 34 00 23 0a 00 06 00 15 09 
0000020 00 16 00 17 08 00 18 0a 00 19 00 1a 07 00 1b 07 
......
0001100 00 12 00 00 00 05 01 00 10 00 00 00 01 00 13 00 
0001120 00 00 02 00 14
```

根据 JVM 规范，类文件结构如下：
```
u4 			   magic
u2             minor_version;    
u2             major_version;    
u2             constant_pool_count;    
cp_info        constant_pool[constant_pool_count-1];    
u2             access_flags;    
u2             this_class;    
u2             super_class;   
u2             interfaces_count;    
u2             interfaces[interfaces_count];   
u2             fields_count;    
field_info     fields[fields_count];   
u2             methods_count;    
method_info    methods[methods_count];    
u2             attributes_count;    
attribute_info attributes[attributes_count];
```
#### 1）魔数

u4 magic  
对应字节码文件的 0~3 个字节  
0000000 ca fe ba be 00 00 00 34 00 23 0a 00 06 00 15 09  
ca fe ba be ：意思是 .class 文件，不同的东西有不同的魔数，比如 jpg、png 图片等！

#### 2）版本

u2 minor_version;  
u2 major_version;  
0000000 ca fe ba be 00 00 00 34 00 23 0a 00 06 00 15 09  
00 00 00 34：34H（16进制） = 52（10进制），代表JDK8

#### 3）常量池

…  
参考文档  
[传送门](https://docs.oracle.com/javase/specs/jvms/se8/html/)

### 2、字节码指令

可参考：  
[字节码指令](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5)

#### 1）javap 工具

Java 中提供了 javap 工具来反编译 class 文件
```
javap -v D:Demo.class
```
#### 2）图解方法执行流程

**代码**
```
public class Demo3_1 {    
	public static void main(String[] args) {        
		int a = 10;        
		int b = Short.MAX_VALUE + 1;        
		int c = a + b;        
		System.out.println(c);   
    } 
}
```

**常量池载入运行时常量池**  
常量池也属于方法区，只不过这里单独提出来了  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6375e6b47e25207e01d584a36a67c3e3.png)  
**方法字节码载入方法区**  
（stack=2，locals=4） 对应操作数栈有 2 个空间（每个空间 4 个字节），局部变量表中有 4 个槽位。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cb5c808d5c00d0d5d7110fd9deac6511.png)  
**执行引擎开始执行字节码**  
**bipush 10**

- **将一个 byte 压入操作数栈**（其长度会补齐 4 个字节），类似的指令还有
    - sipush 将一个 short 压入操作数栈（其长度会补齐 4 个字节）
    - ldc 将一个 int 压入操作数栈
    - ldc2_w 将一个 long 压入操作数栈（**分两次压入**，因为 long 是 8 个字节）
    - 这里小的数字都是和字节码指令存在一起，**超过 short 范围的数字存入了常量池**

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/84935083c058c735b6e33914540dc81c.png)  
**istore 1**  
将操作数栈栈顶元素弹出，放入局部变量表的 slot 1 中  
对应代码中的 a = 10  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/42caf2f7078e4835976069697852bd9f.png)  
**ldc #3**  
读取运行时常量池中 #3 ，即 32768 (超过 short 最大值范围的数会被放到运行时常量池中)，将其加载到操作数栈中  
注意 Short.MAX_VALUE 是 32767，所以 32768 = Short.MAX_VALUE + 1 实际是在编译期间计算好的。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3616501ab0db38fe46ff30b985a1e722.png)  
**istore 2**  
将操作数栈中的元素弹出，放到局部变量表的 2 号位置  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5d68c9bf80b9153319c26ec853339214.png)  
**iload1 iload2**  
将局部变量表中 1 号位置和 2 号位置的元素放入操作数栈中。因为只能在操作数栈中执行运算操作  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d9650c22a246912f5cfe5fefb53773d0.png)  
**iadd**  
将操作数栈中的两个元素弹出栈并相加，结果在压入操作数栈中。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/415728aad9e8a1e49805ddf19d6ef8f5.png)  
**istore 3**  
将操作数栈中的元素弹出，放入局部变量表的3号位置。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/804b87e0a61c5be5b3eedd4b36782107.png)  
**getstatic #4**  
在运行时常量池中找到 #4 ，发现是一个对象，在堆内存中找到该对象，并将其引用放入操作数栈中  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8e1156a466ed884934765428431e3f70.png)  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/03d579cb62c2ef78fa6d5f01442f115e.png)  
**iload 3**  
将局部变量表中 3 号位置的元素压入操作数栈中。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0e229d9b0bf98f97fc9d64b3dffb857d.png)  
**invokevirtual #5**  
找到常量池 #5 项，定位到方法区 java/io/PrintStream.println:(I)V 方法  
生成新的栈帧（分配 locals、stack等）  
传递参数，执行新栈帧中的字节码  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5a0528691b05a8a505242f8b27ef72c1.png)  
执行完毕，弹出栈帧  
清除 main 操作数栈内容  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/462796df7cb5103ccf81024893825e25.png)  
**return**  
完成 main 方法调用，弹出 main 栈帧，程序结束

#### 3）通过字节码指令分析问题

代码
```
public class Code_11_ByteCodeTest {
    public static void main(String[] args) {

        int i = 0;
        int x = 0;
        while (i < 10) {
            x = x++;
            i++;
        }
        System.out.println(x); // 0
    }
}
```
为什么最终的 x 结果为 0 呢？ 通过分析字节码指令即可知晓
```
Code:
     stack=2, locals=3, args_size=1	// 操作数栈分配2个空间，局部变量表分配 3 个空间
        0: iconst_0	// 准备一个常数 0
        1: istore_1	// 将常数 0 放入局部变量表的 1 号槽位 i = 0
        2: iconst_0	// 准备一个常数 0
        3: istore_2	// 将常数 0 放入局部变量的 2 号槽位 x = 0	
        4: iload_1		// 将局部变量表 1 号槽位的数放入操作数栈中
        5: bipush        10	// 将数字 10 放入操作数栈中，此时操作数栈中有 2 个数
        7: if_icmpge     21	// 比较操作数栈中的两个数，如果下面的数大于上面的数，就跳转到 21 。这里的比较是将两个数做减法。因为涉及运算操作，所以会将两个数弹出操作数栈来进行运算。运算结束后操作数栈为空
       10: iload_2		// 将局部变量 2 号槽位的数放入操作数栈中，放入的值是 0 
       11: iinc          2, 1	// 将局部变量 2 号槽位的数加 1 ，自增后，槽位中的值为 1 
       14: istore_2	//将操作数栈中的数放入到局部变量表的 2 号槽位，2 号槽位的值又变为了0
       15: iinc          1, 1 // 1 号槽位的值自增 1 
       18: goto          4 // 跳转到第4条指令
       21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       24: iload_2
       25: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
       28: return

```
#### 4）构造方法

**cinit()V**
```
public class Code_12_CinitTest {
	static int i = 10;

	static {
		i = 20;
	}

	static {
		i = 30;
	}

	public static void main(String[] args) {
		System.out.println(i); // 30
	}
}
```
编译器会按从上至下的顺序，收集所有 static 静态代码块和静态成员赋值的代码，合并为一个特殊的方法 cinit()V ：

**init()V**

编译器会按从上至下的顺序，收集所有 {} 代码块和成员变量赋值的代码，形成新的构造方法，但原始构造方法内的代码总是在后.

#### 5）方法调用

不同方法在调用时，对应的虚拟机指令有所区别

- 私有、构造、被 final 修饰的方法，在调用时都使用 invokespecial 指令
- 普通成员方法在调用时，使用 invokevirtual 指令。因为编译期间无法确定该方法的内容，只有在运行期间才能确定
- 静态方法在调用时使用 invokestatic 指令

- new 是创建【对象】，给对象分配堆内存，执行成功会将【对象引用】压入操作数栈
- dup 是赋值操作数栈栈顶的内容，本例即为【对象引用】，为什么需要两份引用呢，一个是要配合 invokespecial 调用该对象的构造方法 “init”: ()V （会消耗掉栈顶一个引用），另一个要 配合 astore_1 赋值给局部变量
- 终方法（ﬁnal），私有方法（private），构造方法都是由 invokespecial 指令来调用，属于静态绑定
- 普通成员方法是由 invokevirtual 调用，属于动态绑定，即支持多态 成员方法与静态方法调用的另一个区别是，执行方法前是否需要【对象引用】

#### 6）多态原理

因为普通成员方法需要在运行时才能确定具体的内容，所以虚拟机需要调用 invokevirtual 指令  
在执行 invokevirtual 指令时，经历了以下几个步骤

- 先通过栈帧中对象的引用找到对象
- 分析对象头，找到对象实际的 Class
- Class 结构中有 vtable
- 查询 vtable 找到方法的具体地址
- 执行方法的字节码

#### 7）异常处理

**try-catch**

对应字节码指令

- 可以看到多出来一个 Exception table 的结构，[from, to) 是前闭后开（也就是检测 2~4 行）的检测范围，一旦这个范围内的字节码执行出现异常，则通过 type 匹配异常类型，如果一致，进入 target 所指示行号
- 8 行的字节码指令 astore_2 是将异常对象引用存入局部变量表的 2 号位置（为 e ）

**多个 single-catch**

对应的字节码

- 因为异常出现时，只能进入 Exception table 中一个分支，所以局部变量表 slot 2 位置被共用

**finally**

对应字节码

可以看到 ﬁnally 中的代码被复制了 3 份，分别放入 try 流程，catch 流程以及 catch 剩余的异常类型流程  
注意：虽然从字节码指令看来，每个块中都有 finally 块，但是 finally 块中的代码只会被执行一次

**finally 中的 return**

对应字节码

- 由于 ﬁnally 中的 ireturn 被插入了所有可能的流程，因此返回结果肯定以ﬁnally的为准
- 至于字节码中第 2 行，似乎没啥用，且留个伏笔，看下个例子
- 跟上例中的 ﬁnally 相比，发现没有 athrow 了，这告诉我们：如果在 ﬁnally 中出现了 return，会吞掉异常
- 所以不要在finally中进行返回操作

**被吞掉的异常**

会发现打印结果为 20 ，并未抛出异常

**finally 不带 return**

对应字节码

#### 8）Synchronized

对应字节码

### 3、编译期处理

所谓的 **语法糖** ，其实就是指 java 编译器把 .java 源码编译为 .class 字节码的过程中，自动生成和转换的一些代码，主要是为了减轻程序员的负担，算是 java 编译器给我们的一个额外福利  
**注意**，以下代码的分析，借助了 javap 工具，idea 的反编译功能，idea 插件 jclasslib 等工具。另外， 编译器转换的**结果直接就是 class 字节码**，只是为了便于阅读，给出了 几乎等价 的 java 源码方式，并不是编译器还会转换出中间的 java 源码，切记。

#### 1）默认构造器

经过编译期优化后

#### 2）自动拆装箱

基本类型和其包装类型的相互转换过程，称为拆装箱  
在 JDK 5 以后，它们的转换可以在编译期自动完成

转换过程如下

#### 3）泛型集合 取值

泛型也是在 JDK 5 开始加入的特性，但 java 在编译泛型代码后会执行泛型擦除的动作，即泛型信息在编译为字节码之后就丢失了，实际的类型都当做了 Object 类型来处理：

对应字节码

所以调用 get 函数取值时，有一个类型转换的操作。

如果要将返回结果赋值给一个 int 类型的变量，则还有自动拆箱的操作

使用反射可以得到，参数的类型以及泛型类型。泛型反射代码如下：

输出：

#### 4）可变参数

可变参数也是 JDK 5 开始加入的新特性： 例如：

可变参数 String… args 其实是一个 String[] args ，从代码中的赋值语句中就可以看出来。 同 样 java 编译器会在编译期间将上述代码变换为：

注意，如果调用的是 foo() ，即未传递参数时，等价代码为 foo(new String[]{}) ，创建了一个空数组，而不是直接传递的 null .

#### 5）foreach 循环

仍是 JDK 5 开始引入的语法糖，数组的循环：

编译器会帮我们转换为

如果是集合使用 foreach

集合要使用 foreach ，需要该集合类实现了 Iterable 接口，因为集合的遍历需要用到迭代器 Iterator.

#### 6）switch 字符串

从 JDK 7 开始，switch 可以作用于字符串和枚举类，这个功能其实也是语法糖，例如：

在编译器中执行的操作

过程说明：

- 在编译期间，单个的 switch 被分为了两个
    - 第一个用来匹配字符串，并给 x 赋值
        - 字符串的匹配用到了字符串的 hashCode ，还用到了 equals 方法
        - 使用 hashCode 是为了提高比较效率，使用 equals 是防止有 hashCode 冲突（如 BM 和 C .）
    - 第二个用来根据x的值来决定输出语句

#### 7）switch 枚举

编译器中执行的代码如下

#### 8）枚举类

JDK 7 新增了枚举类，以前面的性别枚举为例：

转换后的代码

#### 9）try-with-resources

JDK 7 开始新增了对需要关闭的资源处理的特殊语法，‘try-with-resources’

其中资源对象需要实现 AutoCloseable 接口，例如 InputStream 、 OutputStream 、 Connection 、 Statement 、 ResultSet 等接口都实现了 AutoCloseable ，使用 try-with- resources 可以不用写 finally 语句块，编译器会帮助生成关闭资源代码，例如：

会被转换为：

为什么要设计一个 addSuppressed(Throwable e) （添加被压制异常）的方法呢？是为了防止异常信息的丢失（想想 try-with-resources 生成的 fianlly 中如果抛出了异常）：

输出：

#### 10）方法重写时的桥接方法

我们都知道，方法重写时对返回值分两种情况：  
- 父子类的返回值完全一致  
- 子类返回值可以是父类返回值的子类（比较绕口，见下面的例子）

对于子类，java 编译器会做如下处理：

其中桥接方法比较特殊，仅对 java 虚拟机可见，并且与原来的 public Integer m() 没有命名冲突，可以  
用下面反射代码来验证：

结果：

#### 11）匿名内部类

转换后的代码

引用局部变量的匿名内部类，源代码：

转换后代码：

注意：这同时解释了为什么匿名内部类引用局部变量时，局部变量必须是 final 的：因为在创建 Candy11$1 对象时，将 x 的值赋值给了 Candy11$1 对象的 值后，如果不是 final 声明的 x 值发生了改变，匿名内部类则值不一致。

### 4、类加载阶段

#### 1）加载

- 将类的字节码载入方法区（1.8后为元空间，在本地内存中）中，内部采用 C++ 的 instanceKlass 描述 java 类，它的重要 ﬁeld 有：
    - _java_mirror 即 java 的类镜像，例如对 String 来说，它的镜像类就是 String.class，作用是把 klass 暴露给 java 使用
    - _super 即父类
    - _ﬁelds 即成员变量
    - _methods 即方法
    - _constants 即常量池
    - _class_loader 即类加载器
    - _vtable 虚方法表
    - _itable 接口方法
- 如果这个类还有父类没有加载，先加载父类
- 加载和链接可能是交替运行的  
    ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2946c0ce3f66277be06d5fc1ee2fb869.png)
- instanceKlass保存在方法区。JDK 8以后，方法区位于元空间中，而元空间又位于本地内存中
- _java_mirror则是保存在堆内存中
- InstanceKlass和*.class(JAVA镜像类)互相保存了对方的地址
- 类的对象在对象头中保存了*.class的地址。让对象可以通过其找到方法区中的instanceKlass，从而获取类的各种信息

**注意**

- instanceKlass 这样的【元数据】是存储在方法区（1.8 后的元空间内），但 _java_mirror 是存储在堆中
- 可以通过前面介绍的 HSDB 工具查看

#### 2）连接

**验证**  
验证类是否符合 JVM规范，安全性检查  
用 UE 等支持二进制的编辑器修改 HelloWorld.class 的魔数，在控制台运行  
**准备**  
为 static 变量分配空间，设置默认值

- static 变量在 JDK 7 之前存储于 instanceKlass 末尾，从 JDK 7 开始，存储于 _java_mirror 末尾
- static 变量分配空间和赋值是两个步骤，分配空间在准备阶段完成，赋值在初始化阶段完成
- 如果 static 变量是 final 的基本类型，以及字符串常量，那么编译阶段值就确定了，赋值在准备阶段完成
- 如果 static 变量是 final 的，但属于引用类型，那么赋值也会在初始化阶段完成将常量池中的符号引用解析为直接引用

#### 3）初始化

##### <cinit>()v 方法

初始化即调用 <cinit>()V ，虚拟机会保证这个类的『构造方法』的线程安全

##### 发生的时机

概括得说，类初始化是【懒惰的】

- main 方法所在的类，总会被首先初始化
- 首次访问这个类的静态变量或静态方法时
- 子类初始化，如果父类还没初始化，会引发
- 子类访问父类的静态变量，只会触发父类的初始化
- Class.forName
- new 会导致初始化

不会导致类初始化的情况

- 访问类的 static final 静态常量（基本类型和字符串）不会触发初始化
- 类对象.class 不会触发初始化
- 创建该类的数组不会触发初始化

#### 4）练习

从字节码分析，使用 a，b，c 这三个常量是否会导致 E 初始化

典型应用 - 完成懒惰初始化单例模式

以上的实现特点是：

- 懒惰实例化
- 初始化时的线程安全是有保障的

### 5、类加载器

类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远超类加载阶段  
对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在 Java 虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个 Class 文件，被同一个 Java 虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等！  
以JDK 8为例

|名称|加载的类|说明|
|---|---|---|
|Bootstrap ClassLoader（启动类加载器）|JAVA_HOME/jre/lib|无法直接访问|
|Extension ClassLoader(拓展类加载器)|JAVA_HOME/jre/lib/ext|上级为Bootstrap，显示为null|
|Application ClassLoader(应用程序类加载器)|classpath|上级为Extension|
|自定义类加载器|自定义|上级为Application|

#### 1）启动类的加载器

可通过在控制台输入指令，使得类被启动类加器加载

#### 2）扩展类的加载器

如果 classpath 和 JAVA_HOME/jre/lib/ext 下有同名类，加载时会使用拓展类加载器加载。当应用程序类加载器发现拓展类加载器已将该同名类加载过了，则不会再次加载。

#### 3）双亲委派模式

双亲委派模式，即调用类加载器ClassLoader 的 loadClass 方法时，查找类的规则。  
loadClass源码

#### 4）自定义类加载器

**使用场景**

- 想加载非 classpath 随意路径中的类文件
- 通过接口来使用实现，希望解耦时，常用在框架设计
- 这些类希望予以隔离，不同应用的同名类都可以加载，不冲突，常见于 tomcat 容器

**步骤**

- 继承 ClassLoader 父类
- 要遵从双亲委派机制，重写 ﬁndClass 方法  
    不是重写 loadClass 方法，否则不会走双亲委派机制
- 读取类文件的字节码
- 调用父类的 deﬁneClass 方法来加载类
- 使用者调用该类加载器的 loadClass 方法

**破坏双亲委派模式**

- 双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK1.2面世以前的“远古”时代
    - 建议用户重写findClass()方法，在类加载器中的loadClass()方法中也会调用该方法
- 双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷导致的
    - 如果有基础类型又要调用回用户的代码，此时也会破坏双亲委派模式
- 双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的
    - 这里所说的“动态性”指的是一些非常“热”门的名词：代码热替换（Hot Swap）、模块热部署（Hot Deployment）等

### 6、运行期优化

#### 1）即时编译

**分层编译**  
JVM 将执行状态分成了 5 个层次：

- 0层：解释执行，用解释器将字节码翻译为机器码
- 1层：使用 C1 即时编译器编译执行（不带 proﬁling）
- 2层：使用 C1 即时编译器编译执行（带基本的profiling）
- 3层：使用 C1 即时编译器编译执行（带完全的profiling）
- 4层：使用 C2 即时编译器编译执行

proﬁling 是指在运行过程中收集一些程序执行状态的数据，例如【方法的调用次数】，【循环的 回边次数】等

即时编译器（JIT）与解释器的区别

- 解释器
    - 将字节码解释为机器码，下次即使遇到相同的字节码，仍会执行重复的解释
    - 是将字节码解释为针对所有平台都通用的机器码
- 即时编译器
    - 将一些字节码编译为机器码，并存入 Code Cache，下次遇到相同的代码，直接执行，无需再编译
    - 根据平台类型，生成平台特定的机器码

对于大部分的不常用的代码，我们无需耗费时间将其编译成机器码，而是采取解释执行的方式运行；另一方面，对于仅占据小部分的热点代码，我们则可以将其编译成机器码，以达到理想的运行速度。 执行效率上简单比较一下 Interpreter < C1 < C2，总的目标是发现热点代码（hotspot名称的由 来），并优化这些热点代码。  
**逃逸分析**  
逃逸分析（Escape Analysis）简单来讲就是，Java Hotspot 虚拟机可以分析新创建对象的使用范围，并决定是否在 Java 堆上分配内存的一项技术

逃逸分析的 JVM 参数如下：

- 开启逃逸分析：-XX:+DoEscapeAnalysis
- 关闭逃逸分析：-XX:-DoEscapeAnalysis
- 显示分析结果：-XX:+PrintEscapeAnalysis

逃逸分析技术在 Java SE 6u23+ 开始支持，并默认设置为启用状态，可以不用额外加这个参数

对象逃逸状态

全局逃逸（GlobalEscape）

- 即一个对象的作用范围逃出了当前方法或者当前线程，有以下几种场景：
    - 对象是一个静态变量
    - 对象是一个已经发生逃逸的对象
    - 对象作为当前方法的返回值

参数逃逸（ArgEscape）

- 即一个对象被作为方法参数传递或者被参数引用，但在调用过程中不会发生全局逃逸，这个状态是通过被调方法的字节码确定的

没有逃逸

- 即方法中的对象没有发生逃逸

**逃逸分析优化**  
针对上面第三点，当一个对象没有逃逸时，可以得到以下几个虚拟机的优化

**锁消除**  
我们知道线程同步锁是非常牺牲性能的，当编译器确定当前对象只有当前线程使用，那么就会移除该对象的同步锁  
例如，StringBuffer 和 Vector 都是用 synchronized 修饰线程安全的，但大部分情况下，它们都只是在当前线程中用到，这样编译器就会优化移除掉这些锁操作  
锁消除的 JVM 参数如下：

- 开启锁消除：-XX:+EliminateLocks
- 关闭锁消除：-XX:-EliminateLocks

锁消除在 JDK8 中都是默认开启的，并且锁消除都要建立在逃逸分析的基础上

**标量替换**  
首先要明白标量和聚合量，基础类型和对象的引用可以理解为标量，它们不能被进一步分解。而能被进一步分解的量就是聚合量，比如：对象  
对象是聚合量，它又可以被进一步分解成标量，将其成员变量分解为分散的变量，这就叫做标量替换。

这样，如果一个对象没有发生逃逸，那压根就不用创建它，只会在栈或者寄存器上创建它用到的成员标量，节省了内存空间，也提升了应用程序性能  
标量替换的 JVM 参数如下：

- 开启标量替换：-XX:+EliminateAllocations
- 关闭标量替换：-XX:-EliminateAllocations
- 显示标量替换详情：-XX:+PrintEliminateAllocations

标量替换同样在 JDK8 中都是默认开启的，并且都要建立在逃逸分析的基础上

**栈上分配**  
当对象没有发生逃逸时，该对象就可以通过标量替换分解成成员标量分配在栈内存中，和方法的生命周期一致，随着栈帧出栈时销毁，减少了 GC 压力，提高了应用程序性能

**方法内联**  
**内联函数**  
内联函数就是在程序编译时，编译器将程序中出现的内联函数的调用表达式用内联函数的函数体来直接进行替换

**JVM内联函数**  
C++ 是否为内联函数由自己决定，Java 由编译器决定。Java 不支持直接声明为内联函数的，如果想让他内联，你只能够向编译器提出请求: 关键字 final 修饰 用来指明那个函数是希望被 JVM 内联的，如

总的来说，一般的函数都不会被当做内联函数，只有声明了final后，编译器才会考虑是不是要把你的函数变成内联函数

JVM内建有许多运行时优化。首先短方法更利于JVM推断。流程更明显，作用域更短，副作用也更明显。如果是长方法JVM可能直接就跪了。

第二个原因则更重要：方法内联

如果JVM监测到一些小方法被频繁的执行，它会把方法的调用替换成方法体本身，如：

方法调用被替换后

#### 2）反射优化

foo.invoke 前面 0 ~ 15 次调用使用的是 MethodAccessor 的 NativeMethodAccessorImpl 实现  
invoke 方法源码

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bb509bd9b351fc0193657649e4e8ead7.png)  
会由 DelegatingMehodAccessorImpl 去调用 NativeMethodAccessorImpl  
NativeMethodAccessorImpl 源码

- 一开始if条件不满足，就会调用本地方法 invoke0
- 随着 numInvocation 的增大，当它大于 ReflectionFactory.inflationThreshold 的值 16 时，就会本地方法访问器替换为一个运行时动态生成的访问器，来提高效率
    - 这时会从反射调用变为正常调用，即直接调用 Reflect1.foo()

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/191f81734faed8472d18b8321f8ccec6.png)

阿里开源工具：arthas-boot

## 五、内存模型

[参考这篇文章！](https://blog.csdn.net/weixin_50280576/article/details/113532093)

收录于

![](https://i-blog.csdnimg.cn/blog_column_migrate/4c74aa6600e313d04e3ffe6bc8cf9912.png?x-oss-process=image/resize,m_fixed,h_224,w_224)JVM

![](https://i-blog.csdnimg.cn/devpress/blog/e2c59332157d4bf8a7b1ecc8c05427b8.png)2048 AI社区

标签

[#jvm](https://so.csdn.net/so/search/s.do?q=jvm&t=all&o=vip&s=&l=&f=&viparticle=&from_tracking_code=tag_word&from_code=app_blog_art)[#java](https://so.csdn.net/so/search/s.do?q=java&t=all&o=vip&s=&l=&f=&viparticle=&from_tracking_code=tag_word&from_code=app_blog_art)[#java字节码](https://so.csdn.net/so/search/s.do?q=java%E5%AD%97%E8%8A%82%E7%A0%81&t=all&o=vip&s=&l=&f=&viparticle=&from_tracking_code=tag_word&from_code=app_blog_art)

版权声明：本文为博主原创文章，遵循[CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/)版权协议，转载请附上原文出处链接和本声明。

[使用C# HslCommunication库实现_三_菱PLC通信（含完整代码示例）在工业自动化开发中，上位机_与_PLC的通信是关键环节。HslCommunication库是一个开源的.NET工业通信库，支持_三_菱、西门子、欧姆龙等多个品牌PLC的通信协议。本文将演示如何通过库实现C#程序_与__三_菱PLC的通信，并提供完整的代码示例。阅读详情](https://blog.csdn.net/cjh16606260986/article/details/145836283)

### 相关推荐

换一换

[

ArcGIS基于DEM提取微流域单元（附练习数据下载）

流域作为一个相对独立的地理单元，_与_行政区的划分有着本质上的区别。流域的划分更注重自然因素，流域作为一个综合的自然整体，在其内部形成了一个天然的物质_与_能量流动的场所，并通过水流的作用，_与_周围的区域进行物质和能量的交换。流域的精确划分作为一个研究的基础工作，那么接下来给大家说一下利用DEM提取小流域的案例教程——以贵州省遵义市正安县为例。 1 数据获取 地理空间数据云（http://www.gscloud.cn/），数据分辨率为30m。也可直接点击这篇直接下载→全国30m90mDEM下载。 若想要获得更高精度.

qq_43173805的博客4827

](https://blog.csdn.net/qq_43173805/article/details/120113922)[

黑马程序员_jvm_笔记（五）--_类加载_部分心得

11._类加载_阶段 第一阶段：加载 将类的_字节码_载入 方法区 （1.8后为元空间，在本地内存中）中，内部采用 C++ 的 instanceKlass 描述 _java_ 类，它的重要 ﬁeld 有： __java__mirror 即 _java_ 的类镜像，例如对 String 来说，它的镜像类就是 String.class，作用是把 klass 暴露给 _java_ 使用 _super 即父类 _ﬁelds 即成员变量 _methods 即方法 _constants 即常量池 _class_loader 即_类加载_器

abaidaye的博客7807

](https://blog.csdn.net/abaidaye/article/details/123022817)[

STM32F4_DHT11数字温湿度传感器

开机的时候先检测是否有DHT11存在，如果没有，则提示错误。DHT11和DS18B20相同，传感器输出的都是未编码的二进制数字。本节我们将利用STM32F4来读取DHT11的温湿度传感器，获取环境的温度和湿度值。因此：温湿度的读取方式就是整数部分的二进制转换成十进制放在整数部分；小数部分的二进制转换成十进制放在小数部分。DHT11和MCU的一次通讯时间最大3ms，主机连续向DHT11采样的间隔建议不小于100ms。DHT11是广州奥松有限公司生产的一款温湿度一体化的数字传感器。数据分小数部分和整数部分。

light_2025的博客3426

](https://blog.csdn.net/light_2025/article/details/129074144)[

_JVM_详解

定义： _Java_ Virtual Machine - _java_ 程序的运行环境（_java_二进制_字节码_的运行环境）好处：一次编写，到处运行1.我们编写的_Java_源码，编译后会生成一种 .class 文件，称为_字节码_文件2._Java_虚拟机_JVM_就是负责将_字节码_文件翻译成特定平台下的机器码然后运行。也就是说，只要在不同平台上安装对应的_JVM_，就可以运行_字节码_文件，运行我们编写的_Java_程序。注意：跨平台的是_Java_程序，不是_JVM_。_JVM_是用C/C++开发的,不同平台下需要安装不同版本的_JVM_。

qq_62354865的博客5074

](https://blog.csdn.net/qq_62354865/article/details/132004199)[

SAP S4 HANA 新资产会计中的平行分类账估值

在资产会计中，我们可以设置使用折旧区域的平行分类账会计 我们需要根据涉及的会计原则定义折旧区域，在我们的方案中，公司 FR01 是一家国际集团的当地分支机构，公司 FR01 需要按照集团会计准则 IFRS 和当地 GAAP 会计准则报告财务报表。过账到 LG 会计原则（分类帐 2L）-->分类账 2L的 150 欧元损失 -->在 IFRS 折旧区域，我们有 100 欧元的收益，在 LG 折旧区域，我们有 150 欧元的损失，使用平行分类账 0L 和 2L，我们能够根据不同的会计原则进行平行估值。
