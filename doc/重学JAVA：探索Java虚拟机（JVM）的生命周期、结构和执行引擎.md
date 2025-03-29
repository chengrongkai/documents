

### JVM实例和JVM执行引擎实例
 1. JVM实例对应了一个独立运行的java程序，它是进程级别。
 2. JVM执行引擎实例则对应了属于用户运行程序的线程，它是线程级别的。

### JVM的生命周期
 
 1. JVM实例的诞生：当启动一个Java程序时，一个JVM实例就产生了，任何一个拥有public static void main(String[] args)函数的class都可以作为JVM实例运行的起点。 
 2. JVM实例的运行 main()作为该程序初始线程的起点，任何其他线程均由该线程启动。JVM内部有两种线程：守护线程和非守护线程，main()属于非守护线程，守护线程通常由JVM自己使用，java程序也可以标明自己创建的线程是守护线程。 
 3. JVM实例的消亡：当程序中的所有非守护线程都终止时，JVM才退出；若安全管理器允许，程序也可以使用Runtime类或者System.exit()来退出。

### JVM的体系结构

JVM的内部体系结构分为三部分

1. 类装载器（ClassLoader）子系统
作用: 用来装载.class文件
2. 执行引擎
作用:执行字节码，或者执行本地方法
3. 运行时数据区
方法区，堆，java栈，PC寄存器，本地方法栈
JVM类加载器

### 类加载过程

JVM将整个类加载过程划分为了三个步骤：
1. 装载
 装载过程负责找到二进制字节码并加载至JVM中，JVM通过类名、类所在的包名通过ClassLoader来完成类的加载，同样，也采用以上三个元素来标识一个被加载了的类：类名+包名+ClassLoader实例ID。
2. 链接
 链接过程负责对二进制字节码的格式进行校验、初始化装载类中的静态变量以及解析类中调用的接口、类。在完成了校验后，JVM初始化类中的静态变量，并将其值赋为默认值。最后一步为对类中的所有属性、方法进行验证，以确保其需要调用的属性、方法存在，以及具备应的权限（例如public、private域权限等），会造成NoSuchMethodError、NoSuchFieldError等错误信息。
3. 初始化
 初始化过程即为执行类中的静态初始化代码、构造器代码以及静态属性的初始化，在四种情况下初始化过程会被触发执行：调用了new；反射调用了类中的方法；子类调用了初始化；JVM启动过程中指定的初始化类。

### 类加载器

JVM两种类装载器包括：启动类装载器和用户自定义类装载器
 启动类装载器是JVM实现的一部分，用户自定义类装载器则是Java程序的一部分，必须是ClassLoader类的子类。
 主要分为以下几类：
 1. Bootstrap ClassLoader
 这是JVM的根ClassLoader，它是用C++实现的，JVM启动时初始化此ClassLoader，并由此ClassLoader完成$JAVA_HOME中jre/lib/rt.jar（Sun JDK的实现）中所有class文件的加载，这个jar中包含了java规范定义的所有接口以及实现。
 2. Extension ClassLoader
 JVM用此classloader来加载扩展功能的一些jar包
 3. System ClassLoader
 JVM用此classloader来加载启动参数中指定的Classpath中的jar包以及目录，在Sun JDK中ClassLoader对应的类名为AppClassLoader。
 4. User-Defined ClassLoader
 User-DefinedClassLoader是Java开发人员继承ClassLoader抽象类自行实现的ClassLoader，基于自定义的ClassLoader可用于加载非Classpath中的jar以及目录

### 关键方法

ClassLoader抽象类提供了几个关键的方法：
1. loadClass
 此方法负责加载指定名字的类，ClassLoader的实现方法为先从已经加载的类中寻找，如没有则继续从parent ClassLoader中寻找，如仍然没找到，则从System ClassLoader中寻找，最后再调用findClass方法来寻找，如要改变类的加载顺序，则可覆盖此方法
2. findLoadedClass
 此方法负责从当前ClassLoader实例对象的缓存中寻找已加载的类，调用的为native的方法。
3.  findClass
 此方法直接抛出ClassNotFoundException，因此需要通过覆盖loadClass或此方法来以自定义的方式加载相应的类。
4. findSystemClass
 此方法负责从System ClassLoader中寻找类，如未找到，则继续从Bootstrap ClassLoader中寻找，如仍然为找到，则返回null。
5. defineClass 
 此方法负责将二进制的字节码转换为Class对象
6. resolveClass
 此方法负责完成Class对象的链接，如已链接过，则会直接返回。

### 类加载示例

```
/*
* 重写ClassLoader类的findClass方法，将一个字节数组转换为 Class 类的实例
*/
public Class findClass(String name) throws ClassNotFoundException {
    byte[] b = null;
    try {
        b = loadClassData(AutoClassLoader.FormatClassName(name));
    } catch (Exception e) {
    e.printStackTrace();
    }
    return defineClass(name, b, 0, b.length);
}
/*
* 将指定路径的.class文件转换成字节数组
*/
private byte[] loadClassData(String filepath) throws Exception {
    int n =0;
    BufferedInputStream br = new BufferedInputStream(new FileInputStream(new File(filepath)));
    ByteArrayOutputStream bos= new ByteArrayOutputStream();
    while((n=br.read())!=-1){
    bos.write(n);
    }
    br.close();
    return bos.toByteArray();
}
/*
* 格式化文件所对应的路径
*/
public static String FormatClassName(String name){
    FILEPATH= DEAFAULTDIR + name+".class";
    return FILEPATH;
}
  
/*
* main方法测试
*/
public static void main(String[] args) throws Exception {
    AutoClassLoader acl = new AutoClassLoader();
    Class c = acl.findClass("testClass");
    Object obj = c.newInstance();
    Method m = c.getMethod("getName",new Class[]{String.class ,int.class});
    m.invoke(obj,"你好",123);
    System.out.println(c.getName());
    System.out.println(c.getClassLoader());
    System.out.println(c.getClassLoader().getParent());
}
```

### JVM执行引擎

JVM通过执行引擎来完成字节码的执行，在执行过程中JVM采用的是自己的一套指令系统
 每个线程在创建后，都会产生一个程序计数器（pc）和栈（Stack），其中程序计数器中存放了下一条将要执行的指令。
 
 Stack中存放Stack Frame栈帧，表示的为当前正在执行的方法，每个方法的执行都会产生Stack Frame，Stack Frame中存放了传递给方法的参数、方法内的局部变量以及操作数栈。
 
 操作数栈用于存放指令运算的中间结果，指令负责从操作数栈中弹出参与运算的操作数，指令执行完毕后再将计算结果压回到操作数栈，当方法执行完毕后则从Stack中弹出，继续其他方法的执行。
 
 在执行方法时JVM提供了invokestatic、invokevirtual、invokeinterface和invokespecial四种指令来执行
1. invokestatic：调用类的static方法
2.  invokevirtual： 调用对象实例的方法
3.  invokeinterface：将属性定义为接口来进行调用
4.  invokespecial： JVM对于初始化对象（Java构造器的方法为：）以及调用对象实例中的私有方法时。

### 反射机制

反射机制是Java的亮点之一，基于反射可动态调用某对象实例中对应的方法、访问查看对象的属性等
而无需在编写代码时就确定需要创建的对象，这使得Java可以实现很灵活的实现对象的调用，代码示例如下：

```
Class actionClass=Class.forName(外部实现类);
Method method=actionClass.getMethod(“execute”,null);
Object action=actionClass.newInstance();
method.invoke(action,null);
```
反射的关键：要实现动态的调用，最明显的方法就是动态的生成字节码，加载到JVM中并执行。

#### Class actionClass=Class.forName(外部实现类);

调用本地方法，使用调用者所在的ClassLoader来加载创建出Class对象；
 
#### Method method=actionClass.getMethod(“execute”,null);

校验此Class是否为public类型的，以确定类的执行权限，如不是public类型的，则直接抛出SecurityException;调用privateGetDeclaredMethods来获取到此Class中所有的方法，在privateGetDeclaredMethods对此Class中所有的方法的集合做了缓存，在第一次时会调用本地方法去获取；
 
扫描方法集合列表中是否有相同方法名以及参数类型的方法，如有则复制生成一个新的Method对象返回；
如没有则继续扫描父类、父接口中是否有此方法，如仍然没找到方法则抛出NoSuchMethodException；

#### Object action=actionClass.newInstance();

第一步：校验此Class是否为public类型，如权限不足则直接抛出SecurityException；

第二步：如没有缓存的构造器对象，则调用本地方法获取到构造器，并复制生成一个新的构造器对象，放入缓存，如没有空构造器则抛出InstantiationException；

第三步：校验构造器对象的权限；

第四步：执行构造器对象的newInstance方法；构造器对象的newInstance方法判断是否有缓存的ConstructorAccessor对象，如果没有则调用sun.reflect.ReflectionFactory生成新的ConstructorAccessor对象；

第五步：sun.reflect.ReflectionFactory判断是否需要调用本地代码，可通过sun.reflect.noInflation=true来设置为不调用本地代码，在不调用本地代码的情况下，就转交给MethodAccessorGenerator来处理了；

第六步：MethodAccessorGenerator中的generate方法根据Java Class格式规范生成字节码，字节码中包括了ConstructorAccessor对象需要的newInstance方法，此newInstance方法对应的指令为invokespecial，所需的参数则从外部压入，生成的Constructor类的名字以：sun/reflect/GeneratedSerializationConstructorAccessor或sun/reflect/GeneratedConstructorAccessor开头，后面跟随一个累计创建的对象的次数；

第七步：在生成了字节码后将其加载到当前的ClassLoader中，并实例化，完成ConstructorAccessor对象的创建过程，并将此对象放入构造器对象的缓存中；
最后一步：执行获取的constructorAccessor.newInstance，这步和标准的方法调用没有任何区别。

#### method.invoke(action,null);

这步执行的过程和上一步基本类似，只是在生成字节码时生成的方法改为了invoke，其调用的目标改为了传入的对象的方法，同时生成的类名改为了：sun/reflect/GeneratedMethodAccessor。
注：但是getMethod是非常耗性能的，一方面是权限的校验，另外一方面所有方法的扫描以及Method对象的复制，因此在使用反射调用多的系统中应缓存getMethod返回的Method对象
 
### 类加载执行技术
主要的执行技术有:解释，即时编译，自适应优化、芯片级直接执行
1. 解释属于第一代JVM，
2. 即时编译JIT属于第二代JVM，
3. 自适应优化（目前Sun的HotspotJVM采用这种技术）则吸取第一代JVM和第二代JVM的经验，采用两者结合的方式
4. 自适应优化：开始对所有的代码都采取解释执行的方式，并监视代码执行情况，然后对那些经常调用的方法启动一个后台线程，将其编译为本地代码，并进行仔细优化。若方法不再频繁使用，则取消编译过的代码，仍对其进行解释执行。

### 总结

这篇文章全面探讨了 Java 虚拟机（JVM）的核心概念，包括 JVM 实例与执行引擎实例、生命周期、体系结构、类加载过程、类加载器、关键方法、反射机制以及类加载执行技术。文章详细解释了 JVM 内部的工作原理，以及类加载、执行引擎、反射等关键技术的实现方式和应用场景，为 Java 开发者提供了深入理解 JVM 的指南。
 
 

欢迎关注我的公众号“**毕知必会**”，原创技术文章第一时间推送。

<center>
    <img src="https://telegraph-image-aed.pages.dev/file/74ebf1fb389a9ab62228a.jpg" style="width: 100px;">
</center>
