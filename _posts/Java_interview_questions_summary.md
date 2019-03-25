
# Java Interview Questions Summary

[一、java基础知识点](#java基础知识点)  
[二、java深入源码级的面试题](#java深入源码级的面试题)  
[三、数据结构](#数据结构)  
[四、线程、多线程和线程池](#多线程一)  
[五、JVM](#JVM)  
[六、数据库](#数据库)  
[七、多线程二](#多线程二)  
[案例](#案例)  
[末尾](#End)  
[ref](#ref)  


### java基础知识点

#### java中==和equals和hashCode的区别:
    
    ==是运算符，比较的是两个变量是否相等；
    而equals是object的方法，比较的是两个对象的内存地址是否相等，跟==一样, 因为equals内部也是通过==来实现的；
    hashcodey也是Object的方法，返回一个离散的int型整数,在集合操作类中使用,为了提高查询速度;
    从而在集合操作的时候有如下规则：
    将对象放入到集合中时，首先判断要放入对象的hashcode值与集合中的任意一个元素的hashcode值是否相等，如果不相等直接将该
    对象放入集合中。 如果hashcode值相等，然后再通过equals方法判断要放入对象与集合中的任意一个对象是否相等，如果equals
    判断不相等，直接将该元素放入到集合中，否则不放入。 回过来说get的时候，HashMap也先调key.hashCode()算出数组下标，
    然后看equals如果是true就是找到了，所以就涉及了equals
    
#### int、char、long各占多少字节数:
    int               4字节  
    short             2字节    
    long              8字节    
    byte              1字节    
    float             4字节   
    double            8字节      
    char              2字节    
    boolean           1字节 
    对也String来说，一个英文字符固定占1个字节，而中文字符占2个（GBK编码）或3个（UTF-8编码）字节
    
#### int与Integer的区别
    int和Integer最大的区别就是int是基本数据类型而Integer是包装类(复杂的数据类型)；
    int是基本数据类型直接存放数据，而Integer是一个对象，用一个引用指向该对象；
    
#### 探讨对java多态的理解
    

#### String、StringBuffer、StringBuilder区别
    String是一个字符串常量，Stringbuffer是一个字符串变量(线程安全的)，StringBuilder是字符串变量(非线程安全)
    String 类型和 StringBuffer 类型的主要性能区别其实在于 String 是不可变的对象, 因此在每次对 String 类型进
    行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String 对象；
    ref:https://www.imooc.com/article/22988

#### 什么是内部类？内部类的作用
    内部类：定义在另一个类中的类；  
    内部类可以访问该类定义躲在作用域中的数据，包括被private修饰的私有数据；  
    内部类可以实现java单继承的缺点；  
    当我们需要定义一个回调函数缺不想写大量代码的时候我们可以选择使用匿名内部类来实现；    
    https://juejin.im/post/5a903ef96fb9a063435ef0c8

#### 抽象类和接口区别
    抽象类是定义子类通用特性的，他是不能被实例化的，他只能作为子类的超类，抽象类是用来创建继承层里子类的魔板的；
    接口是抽象方法的集合，如果某个类实现了这个接口，那么这个类就要实现接口里的所有抽象方法；接口只是个形式；
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0eebokzv8j31390u0ju2.jpg)

   
#### 抽象类的意义
    为其子类提供一个公共的类型，封装子类中重复的内容，定义抽象方法，虽然子类的实现不一样，但是定义是一致的;
   
#### 接口的意义
    1,简单,规范性;  
    2.维护，拓展性；  
    3，安全，严密性;  
   
#### 抽象类与接口的应用场景
    抽象类：在需要统一的接口，又需要实例变量或者缺省的方法；
    接口:类与类之间需要特定的接口来协调而不需要在乎具体的实现;

#### 抽象类是否可以没有方法和属性？
    抽象类中可以没有抽象方法，但有抽象方法的一定是抽象类
    
* 泛型中extends和super的区别
* 父类的静态方法能否被子类重写
* 进程和线程的区别
* final，finally，finalize的区别
* 序列化的方式
* Serializable 和Parcelable 的区别
* 静态属性和静态方法是否可以被继承？是否可以被重写？以及原因？
* 静态内部类的设计意图
* 成员内部类、静态内部类、局部内部类和匿名内部类的理解，以及项目中的应用
* 谈谈对kotlin的理解
* 闭包和局部内部类的区别
* string 转换成 integer的方式及原理


### java深入源码级的面试题

#### 哪些情况下的对象会被垃圾回收机制处理掉？
    GC判断对象是否能被回收的方法:  
    1,引用计数法(会出现内存泄漏，例如出现对象的循环引用);    
    2,可达性分析法;
    ref:https://blog.csdn.net/justloveyou_/article/details/71216049 
        
#### 讲一下常见编码方式？
    编码是从一种形式或格式转换为另一种形式的过程也称为计算机编程语言的代码简称编码。
    ref:https://www.jianshu.com/p/54a3b6988368
        
#### utf-8编码中的中文占几个字节；int型几个字节？
    中文: 字节数 3; 编码 UTF-8
    int:  一个utf-8数字占1个字节

#### 静态代理和动态代理的区别，什么场景使用？
    静态代理的代理关系在编译的时候就已经确定了，而动态代理是在运行时才确定的，静态代理的实现比较简单，  
    适合做代理类较少且确定的情况，而动态代理比较灵活，例如日志输出，访问控制，或者是一些复杂的附加逻辑，  
    像spring的AOP等;  
    ref:http://www.importnew.com/27772.html  
        https://www.jianshu.com/u/5aff4598d8aa  

#### Java的异常体系
    Throwable是java异常体系的顶级父类，它有两个子类，分别是Error和Exception;   
    Error是JVM系统级错误，例如内存溢出，栈溢出，而Exception是程序运行时发送的各种不期望的事件，可以被
    java异常处理机制所使用;   
    总体上异常机制又分为两类:  
    非检查时异常:Error和Exception以及他们的子类,这类异常在javac编译时不会提示和发现这种异常，这种异常应该通过修正代码来解决
    自定义非检查时异常通过扩展RuntimeExcetion；
    检查时异常:除了Error和Exception之外的异常,javac强制要求程序员为这样的异常做预备处理(try,,catch,,finally或者throw),如：  
    SQLException，IOException,ClassNotFoundException等, 通过继承Exception类自定义的异常类属于检查时异常;   
    ref:http://www.importnew.com/26613.html  
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0l9v63spfj30ya0lwgn2.jpg)    
    
#### 谈谈你对解析与分派的认识。
    

#### 修改对象A的equals方法的签名，那么使用HashMap存放这个对象实例的时候，会调用哪个equals方法？
    会调用对象的equals方法。
    ref:https://www.jianshu.com/p/985534b21089
    
* Java中实现多态的机制是什么？

##### 如何将一个Java对象序列化到文件里？
    使用ObjectOutputStream将序列化后的对象写入文件，使用ObjectInputStream将序列化后的对象从文件中读取; 
    ref:https://github.com/xingxingt/centrecode/blob/master/src/main/java/serializable/SerializableInputOutDemo.java
    
#### 说说你对Java反射的理解
    反射可以让程序在运行的时候能够动态的加载任意类的内部信息(方法,变量,构造函数...)
    ref:http://www.importnew.com/23560.html
        https://github.com/xingxingt/centrecode/tree/master/src/main/java/reflectdemo
    
#### 说说你对Java注解的理解
    注解为我们代码添加信息提供一种形式化的方法，可以使我们在稍后某个时刻使用这些数据(利用反射);
    ref:http://www.importnew.com/23564.html  
        https://www.jianshu.com/p/51ffb289ccbe

* 说说你对依赖注入的理解
    ref:https://www.jianshu.com/p/506dcd94d4f9
* 说一下泛型原理，并举例说明
* Java中String的了解
* String为什么要设计成不可变的？
* Object类的equal和hashCode方法重写，为什么？


### 数据结构

#### 常用数据结构简介
    Collection:  
    List:LinkedList,ArrayList,Vector,Stack,Set,TreeSet 
    Map:HashMap,HashTable,WeakHashMap,TreeMap  
    
#### 并发集合了解哪些？
    concurrentHashMap  
    copyOnWriteArrayList  
    copyOnWriteHashSet  
    
#### 列举java的集合以及集合之间的继承关系
    例如Iterator的子类有Collection和Map，而Collection的子类有List和Set，Map的子类有HashMap和TreeMap...  
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0oh9on5hcj318a0najtt.jpg)

#### Java中的Iterator和Iterable 区别
    Ieterator是java.util包中迭代器类,而Ieterable是java.lang包中接口，好多对象都实现了Iterable接口，从而对象
    可以调用Iterator()方法;  
    ref:https://www.jianshu.com/p/b65ddde3acd8
    
#### 集合类/集合框架图

![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0pceuy5n3j31g90u0dhn.jpg)
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0pcexuvi2j31im0q00tw.jpg)

#### 集合类以及集合框架
    1,Set:HashSet的实现是利用HashMap来做的，LinkedHashSet是继承了HashSet底层用LinkedHashMap实现的,TreeSet底层使用二叉树树实现的，
      底层基于TreeMap实现的；  
    2,List:ArrayList底层是基于数组来实现的非线程安全擅长随机访问；LinkedList底层是用链表来实现的非线程安全的擅长插入和删除操作
      ,Vector与ArrayList相似底层也是数据实现的，但是Vector是线程安全的；Stack继承Vector是一个后进先出的栈；    
    3,Map: HashMap:散列表是底层基于数组存储数据，使用数组+链表+红黑树(Hash碰撞)来实现,LinkedHashMap继承HashMap底层使用链表来存储数据的,
      LinkedHashMap既可以按照它们插入的顺序排序，也可以按它们最后一次被访问的顺序排序；TreeMap有序散列表,基于红黑树来实现；
      WeakHashMap弱引用键实现的Map,跟GC有关;  
      1,强引用：普遍对象声明的引用，存在便不会GC  
      2,软引用：有用但并非必须，发生内存溢出前，二次回收  
      3,弱引用：只能生存到下次GC之前，无论是否内存足够    
      4,虚引用：唯一目的是在这个对象被GC时能收到一个系统通知  
    ref:https://www.jianshu.com/p/63e76826e852  
        http://alexyyek.github.io/2015/04/06/Collection/   
    code:https://github.com/xingxingt/centrecode/tree/master/src/main/java/collectionandmap  

#### 容器类介绍以及之间的区别（容器类估计很多人没听这个词，Java容器主要可以划分为4个部分：List列表、Set集合、Map映射、工具类（Iterator迭代器、Enumeration枚举类、Arrays和Collections），具体的可以看看这篇博文 [Java容器类]）
    ref:http://alexyyek.github.io/2015/04/06/Collection/
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0pd0tg95tj31hc0pwjuj.jpg)    

#### List,Set,Map的区别
     Map:一个保存键值对的对象，映射的Map中不能有重复的key；
     List:List存储的单个元素容器是个有序的可以索引到元素的容器，并且里面的元素可以重复；
     Set:Set里面和List最大的区别是Set里面的元素对象不可重复。
     
#### List和Map的实现方式以及存储方式
     List:List的实现由ArrayList,LinkedList；ArrayList的存储方式是数组，特点查询快;LinkedList存储方式是链表,  
          特点是删除，插入快;           
     Map: Map的实现由HashMap,LinkedHashMap,TreeMap,weekHashMap;HashMap的存储方式是散列表,特点快速查找键值；  
          LinkedHashMap存储方式是链表；TreeMap的存储方式是对键按序存放;  
         
#### HashMap的实现原理  
    ref:http://www.importnew.com/31278.html#comment-742217  
        http://wiki.jikexueyuan.com/project/java-collection/hashmap.html
        
#### HashMap源码理解
    https://github.com/xingxingt/xingxingt.github.io/blob/master/_posts/2018-08-01-HashMap%E8%AF%A6%E8%A7%A3.md
    
#### HashMap如何put数据（从HashMap源码角度讲解）？
    1，首先计算key的hashCode并计算数组的下标位置  
    2，判断数组table是否为空，如果为空则进行初始化，调用resize()  
    3，然后判断hashCode所对应的槽位是否有数据,如果没有数据就直接new一个新的Node存入table中;  
    4,如果发生hash碰撞则先判断key的hash值是否相同，如果key的hash值相同那就再用equals判断两个key是否相同，
      如果两个key相同，那就直接覆盖掉key对应的value值;  
    5,然后继续判断是节点是否是树，如果是树则直接挂载到树节点上    
    6,如果不是树节点，那就是是链表结构，将节点添加到链表的尾部，判断链表的长度是否是大于等于8，如果是则转为红黑树；  
    7，put之后判断数据量是否超过threshold，如果超过则进行resize()  

#### HashMap怎么手写实现？
    简易版:
    ref:https://github.com/xingxingt/centrecode/blob/master/src/main/java/dataStructure/MapImplDemo.java
    
#### ConcurrentHashMap的实现原理 
    ConcurrentHashMap和HashMap一样使用table存储Node，并用key的hashcode来确定table的index下标，处理hash碰撞时也是使用  
    链表和红黑树来解决;  
    在1.7之前ConcurrentHashMap使用的是segment锁分段的技术实现的高并发，而在1.8中ConcurrentHashMap是使用CAS+synchronized
    的方法来解决并发问题；
    
    https://www.jianshu.com/p/c0642afe03e0
    1.8ref:https://juejin.im/entry/59fc786d518825297f3fa968#comment
    1.6ref:https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/index.html  
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0wfah1argj31220kudjc.jpg)        

#### HashTable实现原理
    HashTable又叫散列表,它继承Dictionary并实现了Map接口，它的键值都是对象，HashTable底层是用Entry[]数组+链表的结构来实现的，    
    因为HashTable是一个同步容器，所以HashTable的方法基本上都用Synchronized修饰；    
    put：先检查put的key是否为空，如果为空则返回空指针异常，否则计算key的hashcode，然后根据key的hashcode计算table的下标，如果  
    table[index]有数据存在则进行遍历，如果遇到有相同个key则进行替换，如果不相同则直接添加；  
    get:根据key的hashcode计算table的index下标，然后遍历链表，找到了返回，找不到则返回null;  


#### HashMap和HashTable的区别
     1,HashMap的key和value都允许为空,非线程安全效率要比HashTable高，不能包含重复的键，但能包含重复的值，  
       HashMap是使用数组+链表+红黑树来实现的；  
     2,HashTable的key和value不可以为空，是线程安全的，HashTable是使用数组+链表来实现的；  
     3,HashMap的迭代器(Iterator)是fial-fast(快速失败);当其他线程改变了HashMap的结构(增加或者删除)，
       则抛出ConcurrentModificationException,而HashTable是enumerator迭代器，不会出现这样的情况；
       
#### HashMap与HashSet的区别
     1,HashSet实现了Set接口，它不允许有重复的值，存储的是单对象，HaseSetHashSet使用成员对象来计算hashcode值，  
       对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false  
     2,HashMap是实现了Map接口,存储的是key-value对，而且允许有空的key或者value，不允许有重复的key但是允许  
       有重复的value,HashMap使用的是使用key来计算hashcode；其实HashSet底层是使用HashMap实现的，利用HashMap的  
       key不允许重复的特性来实现的；
       
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0wub3q225j30ym0hsjsd.jpg)

#### HashSet与HashMap怎么判断集合元素重复？
     因为HashSet是基于HashMap来实现的，所以HashSet和HashMap都根据对象的hashCode和equals来判断的，如果对象和
     集合中元素的hashcode和equals都相同，则说明是重复元素;

* TreeMap具体实现
* 集合Set实现Hash怎么防止碰撞
* ArrayList和LinkedList的区别，以及应用场景
* 数组和链表的区别
* 二叉树的深度优先遍历和广度优先遍历的具体实现
* 堆的结构
* 堆和树的区别
* 堆和栈在内存中的区别是什么(解答提示：可以从数据结构方面以及实际实现方面两个方面去回答)？
* 什么是深拷贝和浅拷贝
* 手写链表逆序代码
* 讲一下对树，B+树的理解
* 讲一下对图的理解
* 判断单链表成环与否？
* 链表翻转（即：翻转一个单项链表）
* 合并多个单有序链表（假设都是递增的）


### 多线程一

#### 开启线程的三种方式？
    继承Thread: 定义一个继承Thead类的子类，子类重写Thread的run方法，然后实例化子类创建出线程对象，然后调用线程对象的start()开启线程;    
    实现Runnable: 定义一个实现Runnable的实现类，实现类重写Runnable的run方法，然后实例出子类即线程对象，然后调用线程对象的start()开启线程;    
    使用Callable和Future: 定义一个实现Runnable的实现类，实现类重写Callable的call()方法,call()方法可以有返回值,创建FutureTask对象，  
    实例出实现类，用FutureTask对象包装实例出的实现类，FutureTask封装了Callable的call()的返回值;通过new Thread(future).start的方式启动线程;
   

#### 线程和进程的区别？
    1,进程是系统进行资源分配和调度的一个独立单元；  
    2,而线程是进程的实体，是cpu资源分配和调度的最小单元，线程本身不拥有资源，它只拥有一些运行时所需要的资源（寄存器，栈，程序计数器等）,  
      它可与同属同一个进程的其他线程共享进程所拥有的所有资源;  
    3,进程和线程的主要区别在于它们的操作系统资源管理方式不同，进程有独立的地址空间，一个进程死掉后在保护模式下不会影响其他进程，   
      而线程只是进程的不同执行路径;  
    4,线程有自己独立的堆栈和局部变量，没有单独的地址空间，一个线程死掉后就等于整个进程死掉，所以多进程的程序  
    比多线程的程序健壮，但是进程之间的切换，耗费的资源更大效率要差，而且要同时对共享变量操作是只能用线程，不能用进程；   
    简述：  
    1，一个程序至少要有一个进程，而一个进程至少要有一个线程;  
    2,线程的划分尺度要小于进程，使得多线程程序的并发性高；  
    3，进程在执行过程中有独立的内存单元，而多个线程共享内存，从而极大的提高了程序的运行效率;  
    4,每个独立的线程有程序运行的入口，顺序执行的序列和程序运行的出口,但是线程不能独立运行必须依赖于应用程序，由应用程序提供多个线程的执行控制;  
    5,从逻辑的角度来看，多线程的意义在于在一个应用程序中有多个执行部分可以同时执行，但是操作系统并没有将多个线程看做是多个独立运行的应用，来
       实现进程的的调度管理和资源分配(进程和线程的重要区别);  
    ref:https://blog.csdn.net/mxsgoden/article/details/8821936

#### 为什么要有线程，而不是仅仅用进程？
    什么是进程？   
    应用程序不能直接执行，它需要装载到内存中，系统为它分配资源，才能运行，而这种执行的程序称为进程；  
    程序和进程的区别？  
    程序是运行指令的集合，他是进程运行的静态描述文件，而进程是程序的一次执行活动，是动态概念;   
    进程的缺点:  
    1,进程在一时间只能做一件事，如果想同时多多件事就无能为力了；  
    2，进程在执行过程中如果遇到阻塞，例如输入等待，一旦阻塞，那么其他操作也无法进行，导致整个进程都要挂起;  
    线程的优点:  
    1，线程可以提高进程的并发度，可以同时执行多个任务；  
    2，线程可以很好的利用计算机多处理器和多核的特性，从而提高进程的运行效率；  
    ref:https://blog.csdn.net/tongxinhaonan/article/details/42558561

#### run()和start()方法区别
    start():start方法是用来启动线程的，实现多线程的运行的，start()方法执行后，无需等待run()方法里面的代码体执行完毕而直接运行下面的代码，  
            通过Thread的start()方法启动一个线程，这时线程并没有运行，而是处于一个可运行状态，一旦得到cpu的时间片，就开始运行run方法里面  
            的代码体，run()方法包含了这个线程要运行的内容，run()方法执行完毕，这个线程也就终止了;    
    run():run()方法只是一个普通的方法而已，如果直接运行run()方法，那么还是只有主线程一个线程而已，程序要顺序执行，还是要等run()方法运行  
          完毕后在执行下面的代码，不过这样没达到多线程的意义;  
    ref:https://blog.csdn.net/dada360778512/article/details/6965790

#### 如何控制某个方法允许并发访问线程的个数？  
     使用semaphore(信号量来控制并发访问线程的个数),semaphore可以协调各个线程，以保证它们可以正确，合理的使用公共资源;
     ref code:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent/aqs

#### 在Java中wait,yield,sleep和join方法的不同；
    sleep:需要指定等待的时间,sleep方法可以让当前正在执行的线程暂停执行，进入阻塞状态，该方法可以让与该线程同优先级或者高优先级的线程  
          得到执行机会，也可以让低优先级的线程得到执行机会;但是sleep不会释放锁标志，如果多个线程同时访问synchronized同步代码块，那么  
          其他线程仍然无法访问共享数据;
    wait:wait()方法和notify(),notifyall()方法一起使用，这三个方法协调多线程对共享数据的存取，所以这三个方法要在synchronized代码块中使用,    
         也就是说调用wait()方法和notify(),notifyall()之前，线程必须获取锁,这三个方法是Object内的方法，不是Thread中的方法;    
         wait()和sleep()的区别在于该方法释放对象的锁标志,当某个对象调用wait方法后会暂停当前线程的执行，并将当前线程放入对象等待池中，  
         直到调用notify()方法后才将对象等待池中任意一个线程移出并放入锁标志等待池中，锁标志等待池中的线程随时准备竞争锁的拥有权，当对象  
         调用notifyAll()方法时，会将对象等待池中的所有对象都移入锁标志等待池中； 
         wait()，notify()及notifyAll()只能在synchronized语句中使用;      
    yield: yield和sleep相似,暂停线程之后不会释放锁标志，yield只会让当前线程重新回到可执行的状态，所以通过yield()方法，线程回到可执行状态后  
           又有可能进入可执行的状态后马上执行，所以说yield()是不可靠的；另外yeild()只能使同优先级或者高优先级的线程得到执行机会;  
    join: 会使当前线程等待调用join()方法的线程执行完毕后在执行,插队;  
       
    ref:https://blog.csdn.net/xiangwanpeng/article/details/54972952
        https://www.jianshu.com/p/25e959037eed
    
#### 谈谈wait/notify关键字的理解
    可以通过配合调用Object对象的wait（）方法和notify（）方法或notifyAll（）方法来实现线程间的通信;   
    具体看上题; 
    
#### 什么导致线程阻塞？
     线程阻塞的特点: 线程阻塞后该线程会让出cpu的使用权，暂停运行，只能等待阻塞的原因消除恢复正常执行，或者是该线程被中断，  
                   抛出InterruptedException异常，该线程也会退出阻塞状态;  
     导致线程阻塞的原因:   
                   1,该线程执行了Thread.sleep(time)方法,使该线程进入休眠并放弃cpu的使用权，等休眠时间到了再恢复cpu的使用权;  
                   2,线程执行同步代码块，未获取到相关同步锁，从而使该线程处于等待阻塞状态，等到获取到同步锁后，再恢复执行；  
                   3,线程执行了一个对象的wait()方法，使该线程处于阻塞状态，等到其他线程执行了notify()后者notifyAll()方法后才能恢复执行;  
                   4,线程执行一些IO操作，因为等待某些资源而进入等待阻塞状态，例如等待客户端的输入...  
      ref:https://blog.csdn.net/sinat_22013331/article/details/45740641
      
#### 线程如何关闭？
     1，使用标志位,例如定义一个用volatile修饰的状态为(为了保证线程之间的可见性)，然后在线程启动后定期检查这个状态位，如果这个状态位为True了，则  
        马上结束线程；这个方法的弊端就是如果线程处于阻塞状态，那么这个线程无法进行状态位的检查，从而无法停止线程;  
     2,使用java提供的中断机制,interupte(),isInterrupted(),interrupted();对线程调用interrupt()方法，不会真正中断正在运行的线程，  
       只是发出一个请求，由线程在合适时候结束自己。     
     3,使用Executor框架,ExecutorService扩展了Executor，ExecutorService.submit()之后返回一个future，可以使用future.cancle()来中断线程; 
     ref:https://www.jianshu.com/p/536b0df1fd55  
     code:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent/thread
        
#### 讲一下java中的同步的方法
     为什么要同步?    
     java允许多个线程并发控制，当多个线程对一个共享变量修改时会使共享资源变量不准确，所以通过同步锁的方法，使一个线程对共享变量进行操作的时候  
     禁止其他线程操作，从而保证了该变量的唯一性和准确性；  
     同步方法：  
     1，使用synchronized关键字修饰的方法；  
     2,使用synchronized关键字修饰代码块；  
     3,使用volatile修饰的变量实现线程同步，但是volatile修饰的变量不具备原子性和线程安全性;  
     4,使用锁机制来实现线程同步，例如：ReentrantLock  
     5,使用局部变量ThreadLocal实现线程同步；   
     6,使用阻塞队列实现线程同步LinkedBlockingQueue;  
     7,使用原子变量实现线程同步,util.concurrent.atomic包下的原子变量;  
     ref:https://www.cnblogs.com/XHJT/p/3897440.html
    
#### 数据一致性如何保证？
     1，CAS；  
     2，Final修饰不可变;  
     3,synchronized，修改代码块或者方法，通过线程互斥的方式实现;  
     4,volatile修饰,但是不能保证原子性;
     5,concurrent,lock,atomic包下的工具类来实现；
     ref:https://www.cnblogs.com/jiumao/p/7136631.html

#### 如何保证线程安全？
    1，互斥同步： 同步表示共享变量在被多个线程访问时，保证共享变量只被一个线程使用，互斥是方法，同步是目标，例如synchronized等锁机制；  
    2,非阻塞同步，非阻塞同步是先进行操作，如果没有其他线程争用共享数据，那操作就成功；如果数据有争用，产生了冲突，那就采取其他的补偿措施。  
    3，无同步方案；  对于一个方法本来就不涉及共享数据，那就自然无须同步措施来保证正确性。  
    ref:https://www.jianshu.com/p/fe7ed5b50933

#### 如何实现线程同步？
    1,互斥同步：同步是保证在多线程环境下并发的访问同一个共享变量时，同一时刻只有一个线程能够操作共享变量;  互斥是方法，同步是目的;  
      例如使用:synchronized关键字...   
    2,非阻塞同步： 互斥的话会带来线程间的阻塞和唤醒从而影响性能，非阻塞同步是真先进性操作共享数据，如果没有其他线程竞争共享资源，那就操作  
      成功，如果有线程竞争共享资源，产生了冲突，那么久采用其他的补救措施；  
    3，无同步方案：对于一段代码不涉及到共享资源，那么也就无需采用同步措施来保证正确性;     
    4,使用重入锁实现线程同步（ReenreantLock）；  
    5,使用局部变量实现线程同步 使用ThreadLocal管理变量;  
    6,使用特殊域变量(volatile)实现线程同步,但它是非原子性的；  
    7,使用阻塞队列实现线程同步,LinkedBlockingQueue;    
    8,使用原子变量实现线程同步util.concurrent.atomic包下的;  
    ref:https://www.cnblogs.com/XHJT/p/3897440.html

#### 两个进程同时要求写或者读，能不能实现？如何防止进程的同步？
    1,允许多个线程同时对文件进行读操作;  
    2,只允许一个写线程往文件中写操作;  
    3,任一写者在完成写操作之前不允许其他读者或写者工作;  
    4,写者执行写操作前，应让已有的读者和写者全部退出。   

#### 线程间操作List 
    多线层操作list会出现如下情况:  
    如果一个线程正在遍历list，而恰好另一个线程对该list做了remove操作，那么此时遍历list过程会出现ConcurrentModificationException；    
    解决办法:   
    1,使用vector，并在遍历该list的时候使用synchronized(list)来加锁，使在遍历list的时候不让remove操作执行；  
    2,使用CopyOnWriteArrayList,copyOnWrite也就是在写时进行copy，当对CopyOnWriteArrayList进行修改操作时，它首先会copy一份新的list  
      并在新的list上做修改，最后将原list的引用指向新的list；所以如果一个线程在遍历list的时候，另一个线程对该list做修改操作，其实是在新的list  
      上做的修改操作，并不会应该list的遍历操作;  
    3,使用线程安全的list.forEach，java8的新特性，例如vector的forEach方法，是在该forEach上加了synchronized关键字来控制线程安全的遍历;  
    ref:https://blog.csdn.net/xiao__gui/article/details/51050793 

#### java对象的生命周期  
    1,创建阶段，一旦某个对象被创建后，并分配给某个变量赋值，则这个对象就进入引用阶段；  
    2,应用阶段，对象至少被一个强引用持有着;  
    3,不可见阶段,程序本身不对该对象持有任何强引用，虽然该对象的这些引用还存在着，即程序的执行已经超出了该对象的作用域(这种情况编辑器就能检查的出来);  
    4,不可达阶段，指该对象不会被任何强引用所持有;  
    5,收集阶段,当对象已经进入了不可达阶段，并且垃圾收集器已经为该对象重新分配的内存空间，则该对象进入收集阶段，此阶段可以通过finalize()逃逸,
      从新获得引用;
    6,回收阶段,当对象仍然处于不可达阶段，并且已经执行了finalized()方法，则该对象进入回收阶段;  
    7,对象空间重新分配,垃圾收集器对该对象的内存空间进行回收或者重新分配，该对象彻底消失; 
    ref:https://blog.csdn.net/sodino/article/details/38387049
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g14bvg072ij30uo0eu43j.jpg)    

#### java对象的内存布局
    在HotSpot中对象的内存分布分为3个区域，对象头，实例数据和对齐填充;  
    对象头:有分为两部分存储: 
       1,一部分存储对象自身的运行时数据，包括：hash码，GC分代年龄,锁状态标志，线程持有的锁，偏向线层id，偏向时间戳等,这部分数据称为"Mark Word";  
       2,另一部分存储对象的类型指针,该指针指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例;    
    实例数据:存储的是对象真是的有效信息，也就是对象的代码中定义的各种类型的字段内容;   
    对齐填充:JVM要求对象大小必须是8的倍数，所以当对象的实例数据没有对齐时，就需要对齐填充来补全；  

#### 什么是线程的上下文切换
    上下文切换的过程:  
    CPU是能够处理多任务的，多任务是指cpu能同时处理两个或者两个以上的程序；在多任务的处理系统中，cpu需要处理所有程序的操作，当CPU来回切换任务  
    时需要记录这些任务执行到哪里了，这就是上下文的切换过程，允许cpu记录并恢复各种程序的运行状态，使它能够完成切换状态;  
    上下文切换:  
    操作系统通过时间片轮转的方法，cpu给每个任务都服务一定的时间，然后把当前任务的状态保存下来，然后加载下一个任务的状态，继续为下一个任务服务，  
    任务的保存和加载的过程就是线程的上下文切换
    ref:https://juejin.im/post/5b10e53b6fb9a01e5b10e9be

#### java的乐观锁和悲观锁
    乐观锁：乐观锁是一种乐观思想，即认为读多写少，遇到并发的可能性低，每次拿数据的时候都任务别人不会修改，所以不会上锁，但是更新的时候会判断  
           一下数据是否被别人修改过，即读取出当前的版本号，然后加锁，再对比旧的版本号，如果版本号一直则认为该数据没被修改然后更新，如果不  
           一致则重复读取-比较-更新;  java的乐观锁是通过CAS来实现的；
    悲观锁: 悲观锁则是一种悲观思想，总认为写多，遇到高并发的可能性高，每次去拿数据的时候总任务别人会修改数据，所以每次都会上锁，这样别人  
           去拿数据的时候总会被阻塞一直等到获取到锁，java中的悲观锁是Synchronized； AQS下的锁先用乐观锁CAS尝试获取锁，如果获取不到则转为  
           悲观锁，如ReetranLock;

#### 偏向锁，自旋锁，轻量级锁，重量级锁
    偏向锁:  
         偏向锁的使用场景是在无多线程竞争锁的情况；即在运行中只有一个线程运行无并发情况，就会给线程添加一个偏向锁; 如果在运行中遇到了其他线程  
         抢占锁，则持有偏向锁的线程会被挂起，jvm会把此偏向锁消除，转换为轻量级锁；   
         偏向锁的获取过程：  
         1，先判断对象的对象头部的Mark Word区域的锁标志是否为1，偏向锁标志是否为01;  
         2,如果开启了偏向锁，则判断对象头部记录的线程id是否是当前线程，如果是则进入同步代码块中，如果不是则进入下一步;  
         3,如果当前线程未获取到锁资源，则通过CAS竞争锁，如果获取成功则将Mark Work中的线程ID修改为当前线程的线程ID，然后执行同步代码块，  
           如果获取失败，则进入下一步;  
         4,如果通过CAS获取锁失败，则表示有线程竞争当前锁，当线程进入安全点(safePoint)的时候挂起当前线程，偏向锁升级为轻量级锁，然后被阻塞在  
           安全点的线程继续执行下面的同步代码块(因为这个时候肯定是已经获取到了锁在执行同步代码块的时候出现了其他线程竞争锁；注意:撤销偏向锁的
           时候会触发Stop the World）;  
                    
    自旋锁:
         如果获取锁的线程在短时间内可以释放锁资源，那么其他等待获取锁资源的线程就没必要进入阻塞状态，而是通过自旋的方式等待获取锁的锁资源，从而  
         避免了内核态和用户态的来回切换的消耗；自旋锁是需要消耗cpu的，利用cpu做无用功的方式来实现自旋；如果自旋时间超过了设定的最大时间，则停止  
         自旋进入阻塞状态(jdk1.7自旋锁的最大时间由JVM控制)；    
   
    轻量级锁:  
         轻量级锁也是在单线程环境下无锁竞争的情况下使用;  
         轻量级锁是由偏向锁转换而来的，当持有偏向锁的线程遇到有其他线程竞争锁资源的时,则会将偏向锁升级为轻量级锁;  
         加锁过程:  
         1,在线程进入同步代码块的时候，先判断对象头的锁状态，如果此时的锁状态位是01,是否为偏向锁状态为0，则在此线程的栈帧中建一个锁记录  
         (Lock Record)空间用于存储对象头的Mark Word；  
         2,将对象头的Mark Word拷贝到线程栈的Lock Record中；  
         3,copy成功后，使用CAS操作将对象的Mark Word指向Lock Record的指针，并将Lock Record中的Owner指向Object的Mark Word；  
         4,如果第三步更新成功了，则表示当前线程获取的带对象的锁，并将该对象的Mark Word中的锁标志置位00代表当前是轻量级锁；  
         5,如果第三部更新失败，则先检查对象的Mark Word指向的指针是否是当前线程栈，如果是则表明当前线程已经获取到了锁资源直接进入同步代码块即可；  
           如果不是当前线程则表示有多个线程在竞争锁资源，轻量级锁要膨胀为重量级锁，对象头的锁标志变为10(表示为重量级锁)，后面的线程则处于阻塞等  
           待状态，当前线程则自旋获取锁；  
         
     重量级锁：见下面：Synchronized👇    
     ref:https://blog.csdn.net/zqz_zqz/article/details/70233767  
         https://blog.csdn.net/u012465296/article/details/53022317  
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g162tj8p6rj30xe0ke75v.jpg)         

#### Synchronized用法
     Synchronied保证了在多线程的情况下方法或者代码块在同一时间只有一个线程能够进入(互斥性),同时它还能保证共享变量的可见性;  
     java的每个方法都可以作为锁：  
         1,作用于普通方法，锁是当前实例对象；  
         2,作用于静态方法，锁是当前类的class对象;当作用于静态方法时，锁住的是Class实例，又因为Class的相关数据存储在永久带PermGen（   
                       jdk1.8则是metaspace），永久带是全局共享的，因此静态方法锁相当于类的一个全局锁，会锁所有调用该方法的线程；      
         3,同步代码块，锁是括号中的对象;  
     ref:http://www.51gjie.com/java/727.html    
     refcode:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent/sync      

#### synchronize的原理
     首先Synchronized是重量级锁;    
     1,检测对象头中的Mark Word中存放的线程id是否为本线程，如果是，则表明该线程获取了偏向锁;  
     2,如果不是,则使用CAS将Mark Word中线程ID替换为本线程的ID,如果CAS操作成功则获取了偏向锁，进入同步代码块,否则下一步;  
     3,如果该线程的CAS操作失败，则表示有线程竞争该锁，则需要撤销偏向锁，升级为轻量级锁;  
     4,当前线程把Mark Word替换为锁记录指针，如果替换成功则当前线程获取锁;  
     5,如果获取失败则表示有其他线程竞争锁，则通过自旋的方式获取锁；  
     6，如果通过自旋获取锁成功则该线程依然持有轻量级锁；  
     7，如果自旋失败则升级为重量级锁;  
     这些操作都是由JVM内部实现，当使用Synchronized时，JVM会根据启用锁和当前线程的竞争情况来决定如何执行同步操作;    
     在所有的锁都启用的情况下线程进入临界区时会先去获取偏向锁，如果已经存在偏向锁了，则会尝试获取轻量级锁，启用自旋锁，如果自旋也没有获取到锁，
     则使用重量级锁，没有获取到锁的线程阻塞挂起，直到持有锁的线程执行完同步块唤醒他们；  
     ref:https://blog.csdn.net/zqz_zqz/article/details/70233767   
         https://www.jianshu.com/p/19f861ab749e  

#### 谈谈对Synchronized关键字，类锁，方法锁，重入锁的理解
    重入锁：  
    Synchronized是强制原子性的内置锁机制，当一个线程获取该对象锁之后再次获取该锁的时候可以再次得到该锁，不需要去竞争锁；也就是说一个synchronied  
    方法/块在调用被类内部的其他Synchronized方法时是永远可以拿到锁;  
    重入锁的实现原理:   
    在线程栈中有一个数据结构(monitor)用于存储锁的持有者(owner)以及锁的重入次数(Nest),当一个线程已经获取到了该锁那么Nest则被置为1,当该线程  
    再次想要获取该锁的时候，会判断该锁的持有者是否是该线程，如果是则将Nest+1，进入同步代码块；当线程退出同步代码块时，计数器会递减，如果计数器为0，
    则释放该锁； 
    重入锁ref:https://blog.csdn.net/aigoogle/article/details/29893667?utm_source=tuicool&utm_medium=referral  
             https://www.jianshu.com/p/19f861ab749e  
    
    refCode:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent
            /sync/SynchronizedMethodAndCodes.java  
    ref:https://blog.csdn.net/le_le_name/article/details/52348314    

#### static synchronized 方法的多线程访问和作用
    synchronizd是对类的当前实例进行加锁，而static synchronized是对类的所有实例进行加锁；
    看代码例子:  
     refCode:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent
            /sync/SynchronizedMethodAndCodes.java  
     ref:https://blog.csdn.net/wangtaomtk/article/details/52318634  
        
#### 同一个类里面两个synchronized方法，两个线程同时访问的问题
     ref:https://blog.csdn.net/aiyawalie/article/details/53261823
     refCode:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent
            /sync/SynchronizedMethodAndCodes.java  

#### java的内存模型
    引入:
    原子性：是指cpu的一个操作，要么执行完，要么不执行，不可中途暂停然后再调度;    
    可见性：指多个线程访问同一个变量，一个线程修改了该变量，其他线程应该立即能看的到修改的值;  
    有序性:指程序的执行顺序按照代码的顺序先后执行;  
    缓存一致性问题其实就是可见性问题。而处理器优化是可以导致原子性问题的。指令重排即会导致有序性问题。 
    为了保障共享内存的正确性(原子性,可见性,有序性),内存模型定义共享内存系统中多线程读写操作的规范； 
    java(JMM)内存模型:  
    java内存模型是一种规范，它规定了一个线程如何和核实被其他线程修改过的共享变量的值,以及如何同步的访问共享变量;    
    要求本地变量和对应栈存放在线程栈上，对象存放在堆上，线程之间的通信必须经过主内存；(Java内存模型规定了所有的     
    变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存中保存了该线程中是用到的变量的主内存副本拷贝，  
    线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存。不同的线程之间也无法直接访问对方工作内存中  
    的变量，线程间变量的传递均需要自己的工作内存和主存之间进行数据同步进行;)
    ref:https://www.hollischuang.com/archives/2550
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0szp0qn1vj30u40hsgn9.jpg)    
    

#### java内存屏障
    内存屏障(Memory Barrier)又称内存栅栏，是一个cpu指令，特点:  
    1,保证特定操作的执行顺序；  
    2，影响某些数据的可见性；  
    编译器和cup能够重排序指令，保证最后相同的结果，提高性能，但是如果插入一条Memory Barrier会告诉编译器和cpu  
    不管什么指令都不能和这条Memory Barrier进行重排序;  
    内存屏障还有一个重要的特点是可以强制刷出cpu的各种cache(主存)，比如读取屏障的操作，将会强制从cpu的cache总读取(主存);
    volatile的实现就是基于内存屏障来实现的;
    
#### volatile的原理
    volatile是通过加入内存屏障和禁止指令重排序优化来实现的；    
    volatile在写的操作，在写操作之后加入一个store屏障指令，将本地内存中的共享变量值刷新到主内存中;  
    volatile在读的操作，在读操作之前加一个load屏障指令，从主内存中读取共享变量;  
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g0sb3h62iij31180u077f.jpg)

#### happens-before
    如果线程A满足线程B的happens-before原则，那么线程A的执行动作的结果对线程B是可见的，如果两个线程未按照happens-before原则  
    ，则JVM会对他们的执行顺序重新排序;

#### 什么是CAS
    CAS即compare and swap(比较和交换);  
    CAS的思想: CAS会传入三个值，当前内存中的值V，旧的期望值A，即将要更新的值B,当且仅当期望值A和内存中的值V相同时，将内存值修改为B，
    并返回true，否则什么都不做，返回false;  
    CAS通过java的本地方法Unsafe来实现的，java方法无法操作底层内存中的数据，需要通过本地方法来访问,而Unsafe可以直接操作指定内存中数据;
    CAS的缺点:有ABA的问题，可以通过AtomicStampedReference来解决;
    ref:https://www.jianshu.com/p/fb6e91b013cc
    
#### 谈谈volatile关键字的用法
    volatile保证了不同线程对该变量操作的可见性;   
    禁止指令重排序;    
    所以volatile关键字只是保证了JMM中的可见性，有序性，并不能保证原子性；  
    ref:https://blog.csdn.net/hang1995/article/details/79417477
   
#### 谈谈volatile关键字的作用
    被volatile修饰的变量，在每次读取的时候都是读取主存中的变量值，例如多线程环境下，如果每个线程都会copy一份共享变量到线程栈中，如果  
    这个共享变量被volatile修饰，那么线程每次都会从主存中读取变量值，每次修改过后都会将修改后的变量值写回主存中去;  
    ref:https://www.cnblogs.com/aigongsi/archive/2012/04/01/2429166.html


#### synchronized 和volatile 关键字的区别
    synchronized主要作用于执行控制，该线程获取了锁其他线程也要获取该锁的话就会被阻塞，这样就保护了被synchronized修饰的代码块不被其他线程访问，  
    同时Synchronized会建立一个内存屏障，内存屏障指令保证了cpu的所有操作结果都会刷新到主存中，这样就保证了共享资源的可见性；   
    volatile主要的作用是内存的可见行，它保证了通过volatile修饰的变量读写都会刷新到主存中；  
    区别:  
    1,被volatile修饰的变量会告诉jvm该值存在于线程的工作内存中是不可靠的，需要从主存中读取，而Synchronized主要是锁住当前变量，只有当前线程  
      可以访问，其他线程被阻塞；  
    2,volatile只能修饰变量，而Synchronized可以修饰变量，方法，类；  
    3,volatile修饰的变量只能保证多线程的可见性，而Synchronized修饰的可以保证多线程的可见性和原子性；  
    4,volatile不会造成线程阻塞，而Synchronized会造成线程阻塞;   
    5,volatile修饰的变量不会被编译器优化(禁止指令重排序)，而Synchronized会被编译器优化；  
    ref:https://blog.csdn.net/suifeng3051/article/details/52611233

#### synchronized与Lock的区别
    ref:http://www.cnblogs.com/dolphin0520/p/3923167.html
        https://blog.csdn.net/u012403290/article/details/64910926?locationNum=11&fps=1
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g17b75r93vj31hk0l875j.jpg)


#### ReentrantLock 、synchronized和volatile比较
     synchronized:是内置锁，锁的管理交给底层的JVM，又称为互斥锁，即操作互斥;  
     ReentrantLock:可重入锁,需要自己去管理锁，特点如下：  
             1,ReentrantLock有tryLock()方法，允许尝试获取锁，如果获取到锁则返回true，否则返回false；  
             2,ReentrantLock有公平锁和非公平锁功能，在创建的时候可以指定，默认是非公平锁,可抢占;    
             3,ReentrantReadWriteLock,可实现读写分离锁,可用于读多写少的情景，读的时候不加锁，写的时候加锁从而大大提高性能；  
     volatile:修饰于变量,主要是内存可见性和禁止指令重排序，但是非线程安全的，也不是原子性的；         
     ref:https://www.cnblogs.com/dennyzhangdd/p/6020566.html  
     
#### ReentrantLock的内部实现
    ReentrantLock是可重入锁，实现主要有两种，公平锁和非公平锁，公平锁就是多个线程竞争锁的时候，每个线程按照抢占锁的顺序依次获取锁，  
    而非公平锁如果线程A获取了锁，线程B处于锁等待被挂起状态，当A执行完代码释放锁后，B开始尝试获取锁，在B尝试获取锁的时候C线程也来尝试  
    获取锁并且获取锁成功，那B线程则继续处于等待状态，这就是非公平锁的抢占机制;   
    ReentrantLock内部有一个计数器state，当线程A获取了锁，那么state的值原子性的+1,而在线程A获取锁的时候，线程A又尝试获取锁，此时  
    线程A无需排队等待，只需将计数器state的值原子性的+1即可,然后访问同步代码块(+1操作是用CAS来完成的);  
    另外还有条件变量Condition:条件变量很大一个程度上是为了解决Object.wait/notify/notifyAll难以使用的问题. 在Synchronized中  
    所有线程都在一个object条件队列中等待，而ReentrantLock一个lock可以有多个condition条件队列;   
    ref:https://blog.csdn.net/yanyan19880509/article/details/52345422   
        https://www.jianshu.com/p/4358b1466ec9  

#### lock原理
    底层AQS分析  
    https://blog.csdn.net/endlu/article/details/51249156  
    https://www.jianshu.com/p/d8eeb31bee5c  

#### 死锁的四个必要条件:  
    1,互斥条件：一个资源每次只能被一个进程使用。  
    2,请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。  
    3,不剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。   
    4,循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。  
        
#### 怎么避免死锁？
     1,线程按照一定的顺序加锁;   
     2,给线程添加加锁时限，当一个线程请求获取锁被阻塞时,会判断等待的时长，如果超过设定的时限，则自动放弃请求锁；  
     3，死锁检测;  
     ref:https://blog.csdn.net/ls5718/article/details/51896159
     
##### 对象锁和类锁是否会互相影响？
    对象锁和类锁是不同的锁，对象锁类的实例锁，而类锁则是一个类的class对象锁,java类可能有很多个对象，但只有一个class类；  
    对于对象锁来说，各自对象持有各自的对象锁，互补干扰，不存在竞争锁的情况；但是对于类锁则不同，例如该类的多个对象都持有类锁，  
    那么这些对象所持有的是同一把锁，所以会出现锁竞争的情况；  
    1个线程访问静态synchronized的时候，允许另一个线程访问对象的实例synchronized方法。反过来也是成立的，因为他们需要的锁是不同的。  
    ref:https://blog.csdn.net/codeharvest/article/details/70649375  
        https://github.com/xingxingt/concurrent/blob/master/src/main/java/com/concurrent/concurrent/
        sync/SynchronizedMethodAndCodes.java

#### 什么是线程池，如何使用?
    线程池：
        存放线程的对象池,池中线程的执行调度由池管理器来管理，当执行任务时从池中获取一条线程，任务执行完毕后再放回池中，这样可以
        避免反复创建对象带来的性能开销，节省系统资源；  
    线程池的作用:  
        1,线程的创建和销毁都伴随着系统的开销，过于频繁的创建和销毁线程会造成严重的影响执行效率；  
        2,可运用线程池来控制线程的并发数，从而可以避免线程任务过多而系统资源不足导致任务阻塞的情况;  
        3,对线程的管理,比如对线程定时循环执行或者延迟执行；  
    ref:https://www.jianshu.com/p/210eab345423  
    ref:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent/threadpool

#### 线程池的拒绝策略  
    java线程池的拒绝策略的实现:  
    1,ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。  
    2,ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。  
    3,ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）  
    4,ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务  
    ref:ref:https://github.com/xingxingt/concurrent/tree/master/src/main/java/com/concurrent/concurrent/
    threadpool/ThreadPoolRejectedDemo.java

#### java的BlockQueue
    ref:http://www.importnew.com/28053.html

#### Java的并发、多线程、线程模型
    ref:http://ifeve.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B/
    
* 谈谈对多线程的理解
* 多线程有什么要注意的问题？
* 谈谈你对并发编程的理解并举例说明
* 谈谈你对多线程同步机制的理解？
* 如何保证多线程读写文件的安全？
* 多线程断点续传原理
* 断点续传的实现
* 谈谈NIO的理解

### JVM
#### 说一下jvm的主要组成部分？及其作用？
    堆:  
    java堆是java虚拟机所管理的内存中最大的一块。java堆是被所有线程共享的一块内存区域，在虚拟机启动的时候创建，此内存区域的  
    唯一目的就是存放对象实例;  几乎所有的对象实例都在这里分配内存;也是java垃圾收集器管理的主要区域;Java堆可使用-Xms -Xmx进行内存控制;  
    方法区:  
    方法区和堆一样也是线程共享的内存区域，它用于存储已被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据；方法区在JDK1.7
    版本及以前被称为永久代(用-XX:MaxPermSize来控制大小)，从JDK1.8永久代被移除，从JDK1.7版本之后，运行时常量池从方法区移到了堆上；   
    虚拟机栈:  
    虚拟机栈是线程私有的，它的生命周期和线程相同，虚拟机栈描述的是java方法执行的内存模型,每个方法在执行的时候都会创建一个栈帧来存储局部变量表，  
    操作数栈，动态链接，方法的出口等信息;每一个方法的调用和执行完成的过程都对应着一个栈帧在虚拟机栈中的入栈和出栈的过程；  
    本地方法栈:  
    本地方法栈和虚拟机栈的作用相似，只不过本地方法栈服务于虚拟机使用到的本地方法，而虚拟机栈服务于虚拟机执行的java方法;  
    程序计数器:  
    当前线程所执行的字节码的行号指示器，指示Java虚拟机下一条需要执行的字节码指令;  
    
#### 说一下jvm运行时数据区？
    线程私有区域:  
       程序计数器,虚拟机栈，本地方法栈;  
    共有区域:  
       堆，方法区；  

#### 说一下堆栈的区别？
    栈内存:  
    栈内存区域主要是存储局部变量，即凡是定义在方法内的变量都是局部变量，方法外的是全局变量(for循环里面的也是局部变量);局部变量的定义  
    一定是先有方法的入栈，每个变量都有自己的作用域，一旦该变量离开了作用域则立即被释放，栈内存的更新速度更快，因为局部变量的生命周期都很短；  
    堆内存：  
    堆内存区域主要是存放对象实体(数组/对象),即new建立的都存放于堆中,堆中存放的是实体，而实体用于封装多种数据，如果实体的数据消失了，那么该  
    实体还存在，还可以再次使用所以堆内存中的实体不会立即释放;但是栈中存放的是单个变量，变量消失了那就没了，堆中的实体释放依靠JVM的垃圾收集器  
    不定时的进行回收；  
    栈内存和堆内存的区别：  
    1，栈存储的是局部变量而堆存储的是实体；  
    2，栈内存的更新速度比堆内存快，因为局部变量的生命周期很短;  
    3，栈内存的局部变量的生命周期结束后立马会被释放，而堆内存的实体生命周期结束后由JVM的垃圾收集器不定时的进行回收;  
    ref:https://blog.csdn.net/pt666/article/details/70876410
    
#### 队列和栈是什么？有什么区别？
    栈:是一种后进者先出，先进者后出的数据结构，栈是一种操作受限的数据结构，它只允许在一段插入或者删除数据，入栈push()出栈pop()；  
    队列：是一种先进者先出的数据结构,入队列enqueue()将一个数据插入尾部,出队列dequeue()在头部取数据；  
    
#### 什么是双亲委派模型？
    类加载器: 根据类的全限定名称来将class文件加载到JVM虚拟机中;  
    双亲委派模型： 
    某个类加载器再收到加载类的请求时，它首先不会去尝试加载类，而是将将加载任务委托给父类加载器，依次类推，如果在父类加载器中可以完成加载任务，  
    则返回成功，如果父类加载器在搜索范围内找不到该类，即ClassNotFoundException，子加载器才去尝试加载任务;    
   
    如何实现一个双亲委派模型:  
    继承ClassLoader，ClassLoader中有三个方法：  
    1,loadClass()该方法是ClassLoader类默认实现的，在该方法中首先会判断指定的类是否已经被加载，如果被加载了就无需在此加载；如果还没有加载 
      则先判断是否有父类加载器，如果有则使用父类加载器parent.loadClass(name, false)或者使用findBootstrapClassOrNull(name);  
    2，findClass(),如果第一步中的父类加载器和bootstrap加载器都没有加载到该类，则使用当前类加载的findClass()来完成类加载;    
    3,defineClass(),如果第二步中读取一个指定的名称的类为字节数组的话，可以使用defineClass()该方法将字节数组转为Class对象；    
    ref:https://blog.csdn.net/huachao1001/article/details/52297075  
        https://blog.csdn.net/tang9140/article/details/42738433 
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g1dmghb1z6j31gi0qugme.jpg)   


#### 说一下类加载的执行过程？
    类加载过程:  

* 怎么判断对象是否可以被回收？
* java 中都有哪些引用类型？
* 说一下 jvm 有哪些垃圾回收算法？
* 说一下 jvm 有哪些垃圾回收器？
* 详细介绍一下 CMS 垃圾回收器？
* 新生代垃圾回收器和老生代垃圾回收器都有哪些？有什么区别？
* 简述分代垃圾回收器是怎么工作的？
* 说一下 jvm 调优的工具？
#### 常用的 jvm 调优的参数都有哪些？
https://mp.weixin.qq.com/s/tfyHwbsNCTjvMGTrfQ0qwQ

https://juejin.im/post/5c875fb96fb9a049a82029b4




### 数据库
* 数据库的三范式是什么？
* 一张自增表里面总共有 7 条数据，删除了最后 2 条数据，重启 mysql 数据库，又插入了一条数据，此时 id 是几？
* 如何获取当前数据库版本？
* 说一下 ACID 是什么？
* char 和 varchar 的区别是什么？
* float 和 double 的区别是什么？
* mysql 的内连接、左连接、右连接有什么区别？
* mysql 索引是怎么实现的？
* 怎么验证 mysql 的索引是否满足需求？
* 说一下数据库的事务隔离？
* 说一下 mysql 常用的引擎？
* 说一下 mysql 的行锁和表锁？
* 说一下乐观锁和悲观锁？
* mysql 问题排查都有哪些手段？
* 如何做 mysql 的性能优化


### 多线程二
* CPU核心数、线程数的关系
* 在CPU时间片轮转机制中设置多少毫秒是合理的？
* 什么是进程？什么是线程？一个进程最多可以创建多少个线程？
* 用户单一进程同时可打开文件数量是多少？
* 什么是并行，什么是并发？
* 什么是同步，什么是异步，什么是堵塞，什么是非堵塞？
* 实现线程的三种方式？
* 线程的生命周期是什么？线程池的初始化的时候，池里面的线程处于生命周期的那个阶段？
* 什么是线程组？其左右是什么？
* ThreadLocal是用来解决共享资源的多线程访问的问题吗？

* 每次使用完ThreadLocal，都调用它的remove()方法，为什么呢？
* volatile的作用？
* run方法是否可以抛出异常？如果抛出异常，线程的状态如何？
* 什么是隐式锁？什么是显式锁？什么是无锁？
* 多线程之间是如何通信的？
* Java的内存模型是什么？
* 什么是原子操作？生成对象的过程是不是原子操作？
* CopyOnWrite机制是什么？
* 什么是CAS?
* 什么是AQS?

* Fail-Fast机制是多线程原因造成的吗？
* 为什么要用线程池？常见的线程池有哪些？
* 阻塞队列的常用方法？
* 为什么数组比链表随机访问速度会快很多呢？
* 什么时候用定时器，什么时候用延时队列？
* 堵塞队列的add，offer，put的区别？
* 线程的阻塞与挂起有什么区别？
* sleep的时候，是否会释放已经获得到锁？
* yield的作用是什么？
* join的作用？

* sleep方法和yield方法的区别？
* 什么时候会发生InterruptedException异常？
* 如何设计一个利用无锁来实现线程的安全？
* 隐式锁什么情况下会释放锁？
* 描述一下可重入的实现机制？
* 什么是内存可见性？什么是寄存器可见性？
* 什么是自旋？举例说明一下。自旋的后果是什么呢？
* notifyAll之后所有的线程都会在次抢夺锁，如果抢夺失败怎么办？
* 什么是内存栅栏？
* 什么是before-happen？

* 常见的限流算法有哪些？
* synchronized锁的范围有哪些？
* 为什么使用线程池技术？
* 常见的创建线程池的三种方式是什么？各有什么特点？
* 可缓存的线程池中多少秒未使用的线程将被移除？
* 线程池内部的核心队列什么？
* 线程池中控制线程创建数目的参数是什么？
* 线程池在什么情况下需要丢弃处理？
* 线程池任务拒绝策略有哪些？
* 创建线程池常用的堵塞队列有哪些？
* Future的主要功能是什么？
* FutureTask的结构关系？FutureTask如何使用呢？


### 案例
#### 案例一
* 自我介绍及项目
* java的内存分区
* java对象的回收方式,回收算法
* CMS和G1了解吗，CMS解决什么问题，说一下回收的过程
* CMS回收停顿了几次？为什么要停顿两次.
* java栈什么时候回发生内存溢出，java堆呢？说一种场景（集合类持有对象）,集合类如何解决这个问题呢，用软引用和弱引用，再讲下这两个引用的区别
* java里的锁了解哪些，Lock和synchronized，他们的使用方式和实现原理有什么区别？
* synchronized升级的过程，偏向锁到轻量级锁再到重量级锁，他们是如何实现的，解决了哪些问题，什么时候发生锁升级
* tomcat了解吗,说下类加载结构
* 关于spring,spring 如何让A和B两个Bean按顺序加载
* 10亿数据去重

#### 案例二 
* 1.jvm内存模型，
* 2.Oom和stackOverFlow的差别，for循环一直创建线程并且run起来会报什么错，
* 3.线程安全，锁，
* 4.gc，
* 5.ConcurrentHashMap1.8，
* 6.Zookeeper，
* 7.Mysql引擎和索引，
* 8.分页查询效率问题，
* 9.还有现场coding，写一个函数，输入n，表示n对括号，返回复合规则的括号组合的数量，比如输入1，组合有()，那么返回就是1，输入是2，
    组合有()()、(())，那   么返回2，输入3，能有这些组合()()()，(())()，()(())，((()))，(()())，所以输出为5


----

### 并发编程有关知识点:

**学习的参考资料如下：**

> Java 内存模型

* [java线程安全总结](http://www.iteye.com/topic/806990)
* [深入理解java内存模型系列文章](http://ifeve.com/java-memory-model-0/)

> 线程状态：

* [一张图让你看懂JAVA线程间的状态转换](https://my.oschina.net/mingdongcheng/blog/139263)

> 锁：

* [锁机制：synchronized、Lock、Condition](http://blog.csdn.net/vking_wang/article/details/9952063)
* [Java 中的锁](http://wiki.jikexueyuan.com/project/java-concurrent/locks-in-java.html)

> 并发编程：

* [Java并发编程：Thread类的使用](http://www.cnblogs.com/dolphin0520/p/3920357.html)
* [Java多线程编程总结](http://blog.51cto.com/lavasoft/27069)
* [Java并发编程的总结与思考](https://www.jianshu.com/p/053943a425c3#)
* [Java并发编程实战-----synchronized](http://www.cnblogs.com/chenssy/p/4701027.html)
* [深入分析ConcurrentHashMap](http://www.infoq.com/cn/articles/ConcurrentHashMap#)





### ref
ref:https://juejin.im/entry/578d938079bc44005ff26aec  
    https://www.jianshu.com/p/511fa4fbf3d5
    https://juejin.im/post/5c788d986fb9a049f154e479

https://zhuanlan.zhihu.com/p/23713529
### End


[跳转至文章头部](#Java Interview Questions Summary)  
