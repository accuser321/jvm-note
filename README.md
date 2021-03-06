# jvm-note
记录学习java虚拟机的笔记,持续更新
# 一.内存问题
java虚拟机在执行java程序的过程中,会把它所管理的的内存划分为若干个不用的数据区域.包括:(方法区)(虚拟机栈)(本地方法栈)(堆)(程序计数器)
# 分解概念
# 1.程序计数器:
可以看做当前线程所执行的字节码的行号指示器.字节码解释器是通过改变这个计数器的值来选取下一条需要执行的字节码指令,分支,循环,跳转,异常处理,线程恢复等基础功能都需要依赖这个计数器来完成.由于java的虚拟机是通过线程轮流切换并分配处理执行时间的方式来实现的.为了线程切换后能恢复到正确的执行位置,每条线程都需要有一个独立的程序计算器.各条线程直接的计数器互不影响,我们称这种内存区域为”线程私有”的内存.如果线程正在执行的是一个java方法,这个计数器记录的是正在执行的虚拟机字节码指令的地址.如果正在执行的是Natvie方法,这个计数器值则为空(Underfined).此内存区域是唯一一个在java虚拟机规范中没有规定任何OutOfMemoryError情况的区域.
# 2.虚拟机栈(VM stacks):
与程序计数器一样,vm stacks也是线程私有的,它的生命周期与线程一致.vm stacks描述的是java方法执行的内容模型.每个方法被执行的时候都会同事创建一个栈帧(stack Frame).用于存储局部变量表,操作栈,动态链接,方法出口等信息.每一个方法被调用直至执行完成的过程,就对应着一个栈帧在虚拟机栈中入栈到处出栈的过程.
粗略的将内存分为队内存(Heap)和栈内存(Stack),”栈”指的就是虚拟机栈,或者是虚拟机栈中的局部变量表部分.
局部变量表存放了编译器可知的各种基本数据类型(boolean,byte,char,short,int,float,long,double)+对象引用.其中,64位长度的long和double类型的数据会占用2个局部变量空间(Slot),其余的数据类型只占用1个.
局部变量表所需的内存空间在编译期间完成分配.P25.当进入一个方法时,这个方法需要在帧中分配多大的局部变量空间是完全确定的,在方法运行期间,不会改变局部变量表的大小.在java虚拟机规范中,对这个区域规定了两种异常状况:如果线程请求的栈深度大于虚拟机所允许的深度,将抛出StackOverflowError异常;如果虚拟机栈可以动态扩展,当扩展无法申请到足够的内存时,会抛出OutOfMemoryError异常.
# 3.本地方法栈(Native Method Stacks):
与虚拟机栈所发挥的作用非常相似,区别在于虚拟机栈为虚拟机执行Java方法(也就是字节码)服务;本地方法栈为虚拟机使用到的Native方法服务.(用处不多略),本地方法栈也会抛出StackOverflowError和OutOfMemoryError.
# 4.堆(Heap):
Java虚拟机所管理的内存中最大的一块,堆是被所有线程共享的一块内存区域,在虚拟机启动时创建.此区域是存放对象实例.几乎所有的对象实例都在堆中分配.
堆是垃圾收集器管理的主要区域,被称为”GC堆”.从内存分配的角度看,线程共享的Java堆中可能划分出多个线程私有的分配缓冲区(TLAB),不过,无论如何划分,都与存放内容无关,无论哪个区域,存储的都是对象实例.
java堆可以处于物理上不连续的内存中,只要逻辑上连续即可,like our’s disk.当前的虚拟机都是按照可扩展来实现的(通过-xmx,和-xms控制).如果在堆中没有内存完成实例分配,并且堆也无法扩展,会抛出OutOfMemoryError异常.
# 5.方法区(Method Area):
与堆一样,是各个线程共享的内存区域,用于存储已被虚拟机加载的类信息,常量,静态变量,即时编译器编译后的代码,(闫鑫的观点,因为方法区中保存的是静态产量,eg:static的数据,其实可以理解为永久区),当方法区无法满足内存分配需求是,会抛出OutOfMemoryError异常.
# 6.运行时常量池(Runtime Constant Pool):
是一个方法区的一部分.class文件中除了版本,字段,方法,接口等描述信息外,还有一项信息是常量池,用于存放编译器生成字面量和符号引用.这些内容在类在加载后存放到方法区的运行时常量池中.
运行时常量池相对于class文件常量池的一特征是具备动态性,java并不要求常量一定只能在编译期产品,运行期间也可能将新的常量放入池中.eg:String类型的intern().
既然运行时常量池是方法区的一部分,自然会受到方法区内存的限制.当常量池无法申请到内存时,会抛出OutOfMemoryError异常.
# 直接内存(Direct Memory):
不是虚拟机运行时数据区的一部分,也不是Java虚拟机规范中定义的内存区域.当这部分内存被频繁使用时,可能会导致OutOfMemoryError异常.
在JDK1.4中新加入NIO,引入了一种基于通道与缓冲区的IO方式,可以使用Native函数库直接分配堆外内存,然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作.这样可以提高性能,因为避免了在java堆和Native堆中来复制数据.
本机直接内存的分配不会受到Java堆大小的限制.但,既然是内存就要受到本机硬件的限制.服务器管理员配置虚拟机参数时,一般会根据实际内存设置-Xmx等信息,但会忽略直接内存,似的各个内存区域的总和大于物理内存限制,从而导致OutOfMemoryError异常.
# 对象访问:
如下代码Object obj = new Object()中,"Object obj”会反映到Java栈的本地变量表中,作为一个reference引用类型数据出现.而”new Object()”会反映到Java堆中,形成存储了Object类型的实例数据值的结构化内存;在java堆中还必须包含能查找到此对象类型数据的地址信息,这些类型数据存储在方法区中.
reference引用类型在不同虚拟机中,主流的访问方式有两种:使用句柄和直接指针.这两种各有优势.
1.句柄访问的好处是reference中存储的是稳定的句柄地址,在对象被移动时只会改变句柄中的实例数据指针,而reference本身不需要被修改.
2.直接指针的好处是速度更快,节省了一次指针定位的时间开销.
# ————————————————————————
# 方法区溢出
  方法区用于存放Class的相关信息,如类名,访问修饰符,常量池,方法描述等,当前的很多框架,spring和hibernate对类进行增强时,都会使用CGLib这类字节码技术,增强的类越多,就需要越大的方法区来保证动态生成的class可以加载到内存中.
  
  方法区溢出是常见的内存溢出异常,一个类如果要被垃圾收集器回收,判定条件是非常苛刻的.在经常动态生成大量class的应用中,需要特别注意类的回收状况.场景eg:使用CGLib字节码增强+大量JSP或动态产生JSP文件的应用(JSP第一次运行时需要编译为Java类)+基于OSGi的应用.
