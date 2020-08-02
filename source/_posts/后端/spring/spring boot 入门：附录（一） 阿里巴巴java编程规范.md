---
title: spring boot 入门：附录（一） 阿里巴巴java编程规范
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 97f48634
date: 2019-03-17 16:30:00
updated: 2019-03-17 16:30:00
---

这里只截取笔者认为重要、或有影响、或未理解的内容，详情可戳 [这里](https://github.com/alibaba/p3c/tree/master/p3c-gitbook/%E7%BC%96%E7%A8%8B%E8%A7%84%E7%BA%A6)。

### 命名
* 【推荐】如果模块、接口、类、方法使用了设计模式，在命名时体现出具体模式。
* 【推荐】接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的Javadoc注释。
* 接口和实现类的命名有两套规则：
1）【强制】对于Service和DAO类，基于SOA的理念，暴露出来的服务一定是接口，内部的实现类用Impl的后缀与接口区别。
正例：CacheServiceImpl实现CacheService接口。
2）【推荐】 如果是形容能力的接口名称，取对应的形容词为接口名（通常是–able的形式）。
正例：AbstractTranslator实现 Translatable。
* 参考】各层命名规约：
A) Service/DAO层方法命名规约
1） 获取单个对象的方法用get作前缀。
2） 获取多个对象的方法用list作前缀。
3） 获取统计值的方法用count作前缀。
4） 插入的方法用save/insert作前缀。
5） 删除的方法用remove/delete作前缀。
6） 修改的方法用update作前缀。
B) 领域模型命名规约
1） 数据对象：xxxDO，xxx即为数据表名。
2） 数据传输对象：xxxDTO，xxx为业务领域相关的名称。
3） 展示对象：xxxVO，xxx一般为网页名称。
4） POJO是DO/DTO/BO/VO的统称，禁止命名成xxxPOJO

### oop规约

* 【强制】Object的equals方法容易抛空指针异常，应使用常量或确定有值的对象来调用equals。如 "test".equals(object);
* 【强制】所有的相同类型的包装类对象之间值的比较，全部使用equals方法比较。说明：对于Integer var = ? 在-128至127范围内的赋值，Integer对象是在IntegerCache.cache产生，会复用已有对象，这个区间内的Integer值可以直接使用==进行判断，但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象，这是一个大坑，推荐使用equals方法进行判断。
* 关于基本数据类型与包装数据类型的使用标准如下：
1） 【强制】所有的POJO类属性必须使用包装数据类型。
2） 【强制】RPC方法的返回值和参数必须使用包装数据类型。
3） 【推荐】所有的局部变量使用基本数据类型。
说明：POJO类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何NPE问题，或者入库检查，都由使用者来保证。
正例：数据库的查询结果可能是null，因为自动拆箱，用基本数据类型接收有NPE风险。
反例：比如显示成交总额涨跌情况，即正负x%，x为基本数据类型，调用的RPC服务，调用不成功时，返回的是默认值，页面显示为0%，这是不合理的，应该显示成中划线。所以包装数据类型的null值，能够表示额外的信息，如：远程调用失败，异常退出。
* 【强制】序列化类新增属性时，请不要修改serialVersionUID字段，避免反序列失败；如果完全不兼容升级，避免反序列化混乱，那么请修改serialVersionUID值。
* 【强制】POJO类必须写toString方法。使用IDE中的工具：source> generate toString时，如果继承了另一个POJO类，注意在前面加一下super.toString。说明：在方法执行抛出异常时，可以直接调用POJO的toString()方法打印其属性值，便于排查问题。
* 【推荐】使用索引访问用String的split方法得到的数组时，需做最后一个分隔符后有无内容的检查，否则会有抛IndexOutOfBoundsException的风险。如 "a,b,c,,"经split处理后，得到的数组长度仅为3，按索引取第四位就可能报错。
* 【推荐】 类内方法定义的顺序依次是：公有方法或保护方法 > 私有方法 > getter/setter方法。说明：公有方法是类的调用者和维护者最关心的方法，首屏展示最好；保护方法虽然只是子类关心，也可能是“模板设计模式”下的核心方法；而私有方法外部一般不需要特别关心，是一个黑盒实现；因为承载的信息价值较低，所有Service和DAO的getter/setter方法放在类体最后。
* 【推荐】循环体内，字符串的连接方式，使用StringBuilder的append方法进行扩展。说明：反编译出的字节码文件显示每次循环都会new出一个StringBuilder对象，然后进行append操作，最后通过toString方法返回String对象，造成内存资源浪费。

### 集合处理

* 【强制】关于hashCode和equals的处理，遵循如下规则：
1） 只要重写equals，就必须重写hashCode。
2） 因为Set存储的是不重复的对象，依据hashCode和equals进行判断，所以Set存储的对象必须重写这两个方法。
3） 如果自定义对象作为Map的键，那么必须重写hashCode和equals。
说明：String重写了hashCode和equals方法，所以我们可以非常愉快地使用String对象作为key来使用。
* 【强制】 ArrayList的subList结果不可强转成ArrayList，否则会抛出ClassCastException异常，即java.util.RandomAccessSubList cannot be cast to java.util.ArrayList.
说明：subList 返回的是 ArrayList 的内部类 SubList，并不是 ArrayList ，而是 ArrayList 的一个视图，对于SubList子列表的所有操作最终会反映到原列表上。
* 【强制】在subList场景中，高度注意对原集合元素个数的修改，会导致子列表的遍历、增加、删除均会产生ConcurrentModificationException 异常。
* 【强制】使用集合转数组的方法，必须使用集合的toArray(T[] array)，传入的是类型完全一样的数组，大小就是list.size()。说明：使用toArray带参方法，入参分配的数组空间不够大时，toArray方法内部将重新分配内存空间，并返回新数组地址；如果数组元素个数大于实际所需，下标为[ list.size() ]的数组元素将被置为null，其它数组元素保持原值，因此最好将方法入参数组大小定义与集合元素个数一致。直接使用toArray无参方法存在问题，此方法返回值只能是Object[]类，若强转其它类型数组将出现ClassCastException错误。 
* 【强制】使用工具类Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法，它的add/remove/clear方法会抛出UnsupportedOperationException异常。说明：asList的返回对象是一个Arrays内部类，并没有实现集合的修改方法。Arrays.asList体现的是适配器模式，只是转换接口，后台的数据仍是数组。
* 【强制】泛型通配符<? extends T>来接收返回的数据，此写法的泛型集合不能使用add方法，而<? super T>不能使用get方法，作为接口调用赋值时易出错。说明：扩展说一下PECS(Producer Extends Consumer Super)原则：第一、频繁往外读取内容的，适合用<? extends T>。第二、经常往里插入的，适合用<? super T>。 
* 【强制】不要在foreach循环里进行元素的remove/add操作。remove元素请使用Iterator方式，如果并发操作，需要对Iterator对象加锁。
* 【强制】 在JDK7版本及以上，Comparator要满足如下三个条件，不然Arrays.sort，Collections.sort会报IllegalArgumentException异常。
说明：三个条件如下：
1） x，y的比较结果和y，x的比较结果相反。
2） x>y，y>z，则x>z。
3） x=y，则x，z比较结果和y，z比较结果相同。
反例：下例中没有处理相等的情况，实际使用中可能会出现异常：
```java
new Comparator<Student>() {           
      @Override          
      public int compare(Student o1, Student o2) {              
        return o1.getId() > o2.getId() ? 1 : -1;       
      }  
};  
```
* 【推荐】集合初始化时，指定集合初始值大小。如 HashMap 指定初始容量避免其重建hash表，影响性能。初始化HashMap(int initialCapacity)时，initialCapacity = (需要存储的元素个数 / 负载因子) + 1。负载因子（即loader factor）默认为0.75，如果暂时无法确定初始值大小，请设置为16（即默认值）。
* 【推荐】使用entrySet遍历Map类集合KV，而不是keySet方式进行遍历。说明：keySet其实是遍历了2次，一次是转为Iterator对象，另一次是从hashMap中取出key所对应的value。
* 【推荐】高度注意Map类集合K/V能不能存储null值的情况，如下表格：
| 集合类            | Key          | Value        | Super       | 说明                   |
|-------------------|--------------|--------------|-------------|------------------------|
| Hashtable         | 不允许为null | 不允许为null | Dictionary  | 线程安全               |
| ConcurrentHashMap | 不允许为null | 不允许为null | AbstractMap | 锁分段技术（JDK8:CAS）  |
| TreeMap           | 不允许为null | 允许为null   | AbstractMap | 线程不安全             |
| HashMap           | 允许为null   | 允许为null   | AbstractMap | 线程不安全             |
反例： 由于HashMap的干扰，很多人认为ConcurrentHashMap是可以置入null值，而事实上，存储null值时会抛出NPE异常。
* 【参考】合理利用好集合的有序性(sort)和稳定性(order)，避免集合的无序性(unsort)和不稳定性(unorder)带来的负面影响。说明：有序性是指遍历的结果是按某种比较规则依次排列的。稳定性指集合每次遍历的元素次序是一定的。如：ArrayList是order/unsort；HashMap是unorder/unsort；TreeSet是order/sort。
* 【参考】利用Set元素唯一的特性，可以快速对一个集合进行去重操作，避免使用List的contains方法进行遍历、对比、去重操作。

### 并发处理

* 【强制】获取单例对象需要保证线程安全，其中的方法也要保证线程安全。说明：资源驱动类、工具类、单例工厂类都需要注意。
* 【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。说明：使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。
* 【强制】线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。说明：Executors返回的线程池对象的弊端如下： 1）FixedThreadPool和SingleThreadPool: 允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。 2）CachedThreadPool和ScheduledThreadPool: 允许的创建线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。
* 【强制】SimpleDateFormat 是线程不安全的类，一般不要定义为static变量，如果定义为static，必须加锁，或者使用DateUtils工具类。
正例：注意线程安全，使用DateUtils。亦推荐如下处理：
```java
private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {        
    Override        
    protected DateFormat initialValue() {         
        return new SimpleDateFormat("yyyy-MM-dd");        
    }    
};
```
说明：如果是JDK8的应用，可以使用Instant代替Date，LocalDateTime代替Calendar，DateTimeFormatter代替SimpleDateFormat，官方给出的解释：simple beautiful strong immutable thread-safe。
* 【强制】高并发时，同步调用应该去考量锁的性能损耗。能用无锁数据结构，就不要用锁；能锁区块，就不要锁整个方法体；能用对象锁，就不要用类锁。说明：尽可能使加锁的代码块工作量尽可能的小，避免在锁代码块中调用RPC方法。
* 【强制】对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造成死锁。说明：线程一需要对表A、B、C依次全部加锁后才可以进行更新操作，那么线程二的加锁顺序也必须是A、B、C，否则可能出现死锁。
* 【强制】并发修改同一记录时，避免更新丢失，需要加锁。要么在应用层加锁，要么在缓存加锁，要么在数据库层使用乐观锁，使用version作为更新依据。说明：如果每次访问冲突概率小于20%，推荐使用乐观锁，否则使用悲观锁。乐观锁的重试次数不得小于3次。
* 【强制】多线程并行处理定时任务时，Timer运行多个TimeTask时，只要其中之一没有捕获抛出的异常，其它任务便会自动终止运行，使用ScheduledExecutorService则没有这个问题。
* 【推荐】使用CountDownLatch进行异步转同步操作，每个线程退出前必须调用countDown方法，线程执行代码注意catch异常，确保countDown方法被执行到，避免主线程无法执行至await方法，直到超时才返回结果。说明：注意，子线程抛出异常堆栈，不能在主线程try-catch到。
* 【推荐】避免Random实例被多线程使用，虽然共享该实例是线程安全的，但会因竞争同一seed 导致的性能下降。
说明：Random实例包括java.util.Random 的实例或者 Math.random()的方式。
正例：在JDK7之后，可以直接使用API ThreadLocalRandom，而在 JDK7之前，需要编码保证每个线程持有一个实例。
* 【推荐】在并发场景下，通过双重检查锁（double-checked locking）实现延迟初始化的优化问题隐患(可参考 The "Double-Checked Locking is Broken" Declaration)，推荐解决方案中较为简单一种（适用于JDK5及以上版本），将目标属性声明为 volatile型。
反例：
```java
class Singleton {   
    private Helper helper = null;  
    public Helper getHelper() {  
      if (helper == null) 
      synchronized(this) {      
        if (helper == null)        
        helper = new Helper();    
      }      
      return helper;  
    }  
    // other methods and fields...  
}
```
* 【参考】volatile解决多线程内存不可见问题。对于一写多读，是可以解决变量同步问题，但是如果多写，同样无法解决线程安全问题。如果是count++操作，使用如下类实现：
```java
AtomicInteger count = new AtomicInteger(); 
count.addAndGet(1); 
```
如果是JDK8，推荐使用LongAdder对象，比AtomicLong性能更好（减少乐观锁的重试次数）。 
* 【参考】 HashMap在容量不够进行resize时由于高并发可能出现死链，导致CPU飙升，在开发过程中可以使用其它数据结构或加锁来规避此风险。 
* 【参考】ThreadLocal无法解决共享对象的更新问题，ThreadLocal对象建议使用static修饰。这个变量是针对一个线程内所有操作共享的，所以设置为静态变量，所有此类实例共享此静态变量 ，也就是说在类第一次被使用时装载，只分配一块存储空间，所有此类的对象(只要是这个线程内定义的)都可以操控这个变量。
* 【强制】在高并发场景中，避免使用”等于”判断作为中断或退出的条件。
说明：如果并发控制没有处理好，容易产生等值判断被“击穿”的情况，使用大于或小于的区间判断条件来代替。
反例：判断剩余奖品数量等于0时，终止发放奖品，但因为并发处理错误导致奖品数量瞬间变成了负数，这样的话，活动无法终止。

### 控制语句

* 【参考】下列情形，需要进行参数校验：
1） 调用频次低的方法。
2） 执行时间开销很大的方法。此情形中，参数校验时间几乎可以忽略不计，但如果因为参数错误导致中间执行回退，或者错误，那得不偿失。
3） 需要极高稳定性和可用性的方法。
4） 对外提供的开放接口，不管是RPC/API/HTTP接口。
5） 敏感权限入口。
* 【参考】下列情形，不需要进行参数校验：
1） 极有可能被循环调用的方法。但在方法说明里必须注明外部参数检查要求。
2） 底层调用频度比较高的方法。毕竟是像纯净水过滤的最后一道，参数错误不太可能到底层才会暴露问题。一般DAO层与Service层都在同一个应用中，部署在同一台服务器中，所以DAO的参数校验，可以省略。
3） 被声明成private只会被自己代码所调用的方法，如果能够确定调用方法的代码传入参数已经做过检查或者肯定不会有问题，此时可以不校验参数。

### 注释

* 【强制】类、类属性、类方法的注释必须使用Javadoc规范，使用/*内容/格式，不得使用// xxx方式。说明：在IDE编辑窗口中，Javadoc方式会提示相关注释，生成Javadoc可以正确输出相应注释；在IDE中，工程调用方法时，不进入方法即可悬浮提示方法、参数、返回值的意义，提高阅读效率。
* 【强制】所有的抽象方法（包括接口中的方法）必须要用Javadoc注释、除了返回值、参数、异常说明外，还必须指出该方法做什么事情，实现什么功能。说明：对子类的实现要求，或者调用注意事项，请一并说明。
* 【强制】所有的类都必须添加创建者和创建日期。
* 【强制】所有的枚举类型字段必须要有注释，说明每个数据项的用途。
* 【参考】特殊注释标记，请注明标记人与标记时间。注意及时处理这些标记，通过标记扫描，经常清理此类标记。线上故障有时候就是来源于这些标记处的代码。
1） 待办事宜（TODO）:（ 标记人，标记时间，[预计处理时间]） 表示需要实现，但目前还未实现的功能。这实际上是一个Javadoc的标签，目前的Javadoc还没有实现，但已经被广泛使用。只能应用于类，接口和方法（因为它是一个Javadoc标签）。
2） 错误，不能工作（FIXME）:（标记人，标记时间，[预计处理时间]） 在注释中用FIXME标记某代码是错误的，而且不能工作，需要及时纠正的情况。

### 其他

* 【强制】在使用正则表达式时，利用好其预编译功能，可以有效加快正则匹配速度。说明：不要在方法体内定义：Pattern pattern = Pattern.compile(规则);
* 【强制】注意 Math.random() 这个方法返回是double类型，注意取值的范围 0≤x<1（能够取到零值，注意除零异常），如果想获取整数类型的随机数，不要将x放大10的若干倍然后取整，直接使用Random对象的nextInt或者nextLong方法。
* 使用 System.currentTimeMillis(); 获取当前毫秒数。说明：如果想获取更加精确的纳秒级时间值，使用System.nanoTime()的方式。在JDK8中，针对统计时间等场景，推荐使用Instant类。

### 安全

* 【强制】隶属于用户个人的页面或者功能必须进行权限控制校验。
* 【强制】用户敏感数据禁止直接展示，必须对展示数据进行脱敏。
* 【强制】用户输入的SQL参数严格使用参数绑定或者METADATA字段值限定，防止SQL注入，禁止字符串拼接SQL访问数据库。
* 【强制】用户请求传入的任何参数必须做有效性验证。说明：忽略参数校验可能导致：page size过大导致内存溢出；恶意order by导致数据库慢查询；任意重定向；SQL注入；反序列化注入；正则输入源串拒绝服务ReDoS。说明：Java代码用正则来验证客户端的输入，有些正则写法验证普通用户输入没有问题，但是如果攻击人员使用的是特殊构造的字符串来验证，有可能导致死循环的结果。
* 【强制】禁止向HTML页面输出未经安全过滤或未正确转义的用户数据。
* 【强制】表单、AJAX提交必须执行CSRF安全过滤。
* 【强制】在使用平台资源，譬如短信、邮件、电话、下单、支付，必须实现正确的防重放限制，如数量限制、疲劳度控制、验证码校验，避免被滥刷导致资损。
* 【推荐】发贴、评论、发送即时消息等用户生成内容的场景必须实现防刷、文本内容违禁词过滤等风控策略。