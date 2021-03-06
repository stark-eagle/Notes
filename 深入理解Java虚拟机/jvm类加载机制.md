# 虚拟机类加载机制
					
### 一.类加载时机:
  
1. 类从加载到虚拟机内存~卸载出内存的生命周期(这些阶段通常是交叉混合进行的):

    加载 --> [验证 -->  准备 -->  解析](这部分统称为连接) --> 初始化 --> 使用 --> 卸载
  
2. *`主动引用`五种必须立即对类进行`初始化`的情况:*

    1. 遇到new(new实例化对象),getstatic,putstatic(读取/修改类属性,final除外)或invokestatic(调用静态方法)这4条字节码指令时
    2. 使用java.lang.reflect包的方法对类进行反射调用时
    3. 初始化某个类,发现其父类未初始化时(对父类立即初始化)
    4. 虚拟机启动时用户指定的执行主类(包含main()的)
    5. JDK1.7动态语言支持的情况(略)
  

3. *除上述五种场景外所有的方式都不会触发初始化,称为`被动引用`:*

    1. 通过子类引用父类的类属性,不会触发子类初始化:
      对于静态字段,只有直接定义这个字段的类才会被初始化.通过子类引用父类中定义的类属性,只会触发父类的初始化.
    2. 通过类数组定义来引用类,不会触发此类的初始化(SuperClass[] a=new SuperClass[10]):
      类数组是由虚拟机自动生成,直接继承于Object的子类,只会触发`[SuperClass`的初始化.
    3. 使用某一类的常量不会触发该定义类的初始化:
      在编译阶段已经将该`定义类`(定义该常量的类)的常量存储到了`调用类`(调用此常量的类)的`常量池`中,以后的每一次引用实际上都转化为`调用类`对自身常量池的引用.
    4.  一个接口在初始化时,并不要求父接口也全初始化,只有真正使用父接口(如引用接口常量)才会初始化.



### 二.类加载过程:

1. *加载:*

  + 加载过程:

    1) 通过类的权限定名获取类的二进制字节流.
    2) 将这个字节流代表的`静态存储结构`转化为方法区的运行时数据结构.
    3) 在内存中生成代表这个类的Class对象,作为这个类在方法区的各种数据的访问入口.
      
  + 数组类本身不通过类加载器创建,而是由Java虚拟机直接创建的.

    一个数组类的创建遵循以下规则:
    
      1) 若数组的组件类型是引用类型,则遵循上述加载过程加载这个类型.这个数组将在加载该组件类型的加载器的类名称空间上被标识.
      
      2) 若数组的组件类型为基本类型,则虚拟机把数组标记为与启动类加载器关联.
      
      3) 数组可见性与组件类型可见性一致,基本类型默认为public.
    
  + 加载与连接交叉进行(开始时间固定先后).
  
2. *验证:*

    目的:确保Class文件的字节流中的信息符合当前虚拟机要求,不会危害虚拟机安全.
    
  + 验证的四个阶段:
    
      1) 文件格式验证:
      
        验证字节流是否符合Class文件格式的规范,并且能被当前版本的虚拟机处理:
          a.是否以魔数0xCAFEBABY开头.
          b.主次版本号是否在当前虚拟机处理范围之内.
          c.是否支持常量池中的常量类型(检查常量tag标志).
          d.指向常量的各种索引值是否存在且符合类型.
          e.CONSTANT_Utf8_info型的常量中是否符合UTF8编码.
          f.Class文件中各个部分及文件本身是否有被删除或附加的信息.
          ...
          
      2) 元数据验证:
      
        对字节码描述的信息进行语义分析,保证其描述的信息符合Java语言规范的要求:
          a.是否有父类(除Object外都须有父类).
          b.是否继承了不允许被继承的类(final).
          c.如果不是抽象类,是否实现了父类或接口中的所有方法.
      	  d.类中字段,方法是否与父类产生矛盾
      	  ...
      
      3) 字节码验证:
      
        通过数据流和控制流分析,确定程序语义是合法符合逻辑的(最复杂的部分):
          a.保证操作数栈的数据类型与指令代码序列都能配合工作.
          b.保证跳转指令不会跳到方法体之外的字节码.
          c.保证方法体中的类型转换是有效的.
          ....
      
      4) 符号引用验证:
	
        对类自身以外(常量池中的各种符号引用)的信息进行匹配性校验.发生在虚拟机将符号引用转化为直接引用的时候(在解析阶段发生):
          a.符号引用中通过字符串描述的权限定名是否能够找到对应类.
          b.指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段.
          c.符号引用中的类,字段,方法的访问性是否可被当前类访问.
          ...
    
    通过-Xverify:none参数可关闭大部分类的验证措施,缩短加载时间.
    

3. *准备:*

    正式为类分配内存并设置类变量初始值(数据类型的零值)的阶段,这些变量所使用的内存都将在方法区中进行分配.<br>
    这时候进行内存分配的仅包括`类变量`(static修饰的静态变量),而不包括实例变量(实例变量将在对象实例化时随着对象一起分配在Java堆上).

    特殊情况: 如果是被final修饰,则一开始就初始化为其指定的值.
  

4. *解析:*

    虚拟机将常量池内的符号引用替换为直接引用的过程.
    
    `符号引用`: 以一组符号来描述所引用的目标,符号可以是任何形式的字面量,能无歧义地定位到目标(未必在内存中).(与虚拟机实现的内存布局无关)
    
    `直接引用`: 可以是直接指向目标的指针,相对偏移量或是一个能间接定位到目标的句柄.(和虚拟机实现的内存布局相关)
    
  + 解析动作:

      1) 类或接口解析:
      2) 字段解析
      3) 类方法解析
      4) 接口方法解析
      (下面三种与动态语言相关)
      5) 方法类型解析
      6) 方法句柄解析
      7) 调用点限定符解析
      
 
  
  
5. *初始化:*

    初始化阶段是根据程序员在程序中制定的主观计划区初始化类变量和其他资源,即执行类构造器<clinit>()方法的过程.
    
  + \<clinit\>执行过程的特点和细节:
      1) \<clinit\>()是由编译器自动收集类中所有`类变量的赋值动作`和`静态语句块(static{})中的语句`合并并而成的.
      编译器收集的顺序是由语句在源文件中出现的顺序所决定的
      注:静态语句块只能访问到定义静态语句块之前的变量;无法访问在它之后的变量,但却可以赋值.
      

      		  public class Test{
      			  static{
      				  i = 0;				//赋值语句正常编译通过
      				  System.out.println(i);			//编译器会提示非法向前引用
      			  }
      			  static int i = 1;
      		  }
      

      2) 虚拟机会保证在子类\<clinit\>()方法执行之前,父类的\<clinit\>()方法已经执行完毕.
      
      3) 由于父类的\<clinit\>()先执行,则意味着父类的静态语句块要优先于子类的变量赋值操作.
      
      4) \<clinit\>()对于类或接口来说并不是必需的,若没有静态语句块,也没有类变量赋值操作,则可以不生成
      \<clinit\>()方法.
      
      5) 接口与类不同,执行接口的\<clinit\>方法不需要先执行父接口的\<clinit\>()方法.只有使用到父接口中的
      类变量时,父接口才会初始化(接口实现类初始化也不会执行接口的\<clinit\>()方法).
      
      6) 虚拟机会保证一个类的\<clinit\>()方法在多线程的情况下能被正确的加锁,同步(如果\<clinit\>()中有
      耗时很长的操作,可能导致多个线程阻塞).
      
      
      
      
+ 类加载器:

  通过一个类的权限定名来获取描述此类的`二进制字节流`.

  1. 类与类加载器:
    
      `类与加载它的类加载器`一起确立这个类在虚拟机中的唯一性.
    (来源于同一个class文件的两个类,被同一个虚拟机加载,只要加载的类加载器不同,那这两个类必定不相等)
  
  2. 双亲委派模型:
    
      + Java虚拟机的角度存在两种类加载器:

        1) `启动类加载器`Bootstrap ClassLoader(C++实现,虚拟机的一部分).
        2) 其他类加载器(Java语言实现,继承抽象类ClassLoader)
   
      + 开发人员角度存在四种类加载器:

        1) `启动类加载器`Bootstrap ClassLoader:
        加载<JAVA_HOME>\lib目录下,或者被-Xbootclasspath参数指定的路径下的能被虚拟机识别的类库.
        2) `扩展类加载器`Extension CLassLoader:
        加载<JAVA_HOME>\lib\ext目录下,或者被java.ext.dirs变量指定的路径下的类库.
        3) `应用程序类加载器`Application ClassLoader(默认类加载器):
        一般也称为系统类加载器.加载用户类路径(ClassPath)上所指定的类库.
        4) `自定义类加载器`
      
      + 双亲委派模型工作流程:

        1) 一个类加载器收到类加载请求,先把请求`委派`给`父类加载器`,因此所有的请求都会传递到顶层的启动类加载器中.
        2) 当父类加载器反馈自己无法完成这个加载请求(它的搜索范围内找不到这个类)时,子加载器才会`尝试自己去加载`.

        - 优点: 

          Java类随着类加载器一起具备了一中带有优先级的层次关系.保证了Java类型体系中的基础行为(唯一性).
    
  
  3. 打破双亲委派模型:
  Tomcat 自定义了一套双亲委派模型,当应用使用Spring管理对象时,必然会打破双亲委派模型: 
  Spring使用线程上下文下载器委托下层加载器加载对象
