---
title: 《深入理解Java虚拟机》读书笔记
date: 2019-11-25 17:25:30
categories: ["Java", "读书笔记"]
tags: ["Java", "Java虚拟机", "读书笔记"]
---

# 《深入理解Java虚拟机》

## 第一部分 走进Java

### 第一章 走进Java

和其他Java书一样，先概述Java和历史，和别的书不同的是有介绍Java虚拟机的历史。  
Sun Classic VM，第一款商用Java虚拟机，只能使用纯解释器方式来执行Java代码，要使用JIT编译器就必须外挂，但外挂了JIT编译器，JIT就完全接管了虚拟机的执行系统，解释器不再工作。  

> JIT，Just-in-time，即时翻译。当Java执行runtime环境时，每遇到新的类型，JIT在此时会针对这个类进行编译，编译后的程式被优化成相对简短的原生指令码，使得这个程式的执行速度较原先解释性（将Java指令转译为对等的微处理器指令）要快一些，且会将翻译过的代码缓存起来以降低性能损耗。但是这样做有个缺点，虽然执行速度快了，但是编译时间变长。  
在Sun Classic中一旦外挂了JIT，代码运行一次也要经过编译，无论它们执行的频率是否有价值去编译。这就导致了不敢使用编译时耗较高的优化技术。

Sun Hospot VM是现在使用最广的虚拟机，提供热点代码探测技术，可以通过执行计数器找到最具有编译价值的代码，然后通知JIT编译器以方法为单位进行编译。通过编译器与解释器协同工作，在相应时间与执行性能中取得平衡，可以执行更多的代码优化技术。

在Java的展望里，提到了模块化，这个在Java 9中已经实现了；混合语言，一个项目中不同的应用层使用不同的编程语言，不过这样真的好么，维护性很低啊；多核并行，提高单机的性能；64位虚拟机，书上说64位虚拟机要付出更大的代价，比如更高的内存消耗，但是在[stackoverflow](https://stackoverflow.com/questions/4931304/why-is-the-64bit-jvm-faster-than-the-32bit-one?tdsourcetag=s_pctim_aiomsg) 上提出他的64位要比32位性能更号，还没有实践过。

自己编译JDK，这个就比较坑了，最开始想和书上装7，后来发现要6的环境，索性就编译9，然后apt-get安装8。Linux下在configure检查下环境没有问题，但是make之后就报错，查了好久，发现是版本问题，glibc >= 2.24的情况下，readdir_r方法被deprecated了，不支持了，openjdk在11修复了问题，之前的不管了。

>8可以修改./hotspot/make/linux/makefiles/gcc.make的WARNINGS_ARE_ERRORS = -Werro，注释掉或者改为-Wno_all。
9的话是没有的，可以bash configure --disable-warnings-as-errors --enable-debug解决

## 第二部分 自动内存管理机制

### 第二章 Java内存区域与内存溢出异常

#### 运行时数据区域

Java虚拟机在执行Java程序的时会把它管理的内存划分不同的区域，每个区域有不同的用途，包括以下几个运行时数据区域，如下图  
![member.png](https://i.loli.net/2019/11/30/kSrPAahcdTBQRqM.png)  

##### 程序计数器

PC寄存器，在会汇编里学的，是存放下一条指令的位置，也就是运行前指针+1，而在JVM里是当前指令的位置，也就是运行完指针+1。  
JVM多线程是通过线程轮流切换并分配处理器执行时间的方式，这个和操作系统相近，单核CPU一次只能运行一个进程，而在运行的Java进程中会再分线程，一次只能执行一个线程。为了在线程切换后能够恢复正确的执行位置，每个线程都要有自己的程序计数器，且相互不会有影响，也就是“线程私有”的内存。  
执行Java方法存的是正在执行的指令地址，执行Native方式则为空。  

##### Java虚拟机栈

Java虚拟机栈也是线程私有，生命周期与线程相同，其描述的是Java方法执行的内存模型，即每个方法在执行时会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每个方法从调用到完成，对应了一个栈帧在虚拟机栈中的入栈出栈。最好的理解就是方法包含方法，一个方法调用的方法没有完成，该方法的栈帧上面始终有调用方法的栈帧。  
局部变量表存放编译器可知的各种基本数据类型、对象引用和returnAddress类型。64位长度的long和double会占用两个局部变量空间，其他类型占1个。局部变量表所需要的内存空间在编译期间完成分配，在运行期间不会改变局部变量表大小。  
虚拟机栈规定了两种异常情况：如果线程请求的栈深度大于虚拟机所允许的深度，抛出StackOverflowError；如果可以动态扩展，且扩展时无法申请到足够的内存，就会抛出OOM  

##### 本地方法栈

和虚拟机栈相似，不同在于一个是执行Java的，一个是执行Native方法的。HotSpot把这两个栈合二为一。  

##### Java堆

Java堆是所有线程共享的一块内存区域，在虚拟机启动时创建，其唯一的目的就是存放对象实例，几乎所有的对象实例都在这里分配的。  
Java堆是垃圾回收器管理的主要区域，从回收的角度来看，基于现在收集器采用的分代收集，可以分为新生代和老年代，再细分可以再分Eden区、From Survivor区、To Survivor区；从内存分配来看，可能划分出多个线程私有的分配缓冲区。  
Java堆可以在物理空间上不连续，只要逻辑上连续即可。既可以实现固定大小，也可以扩展（-Xmx和-Xms），我记得阿里规范上要求固定大小。如果堆中没有内存，抛出OOM。  

##### 方法区

与Java堆一样，各线程共享的内存区域，存储已被虚拟机加载的类信息、常量 、静态变量、即时编译后的字节码等数据。虽然它在Java堆中，但是它和Java堆又不一样，因为hotspot的团队把GC粉黛收集扩展至方法区，使用永久代来实现方法区，这样hotspot就可以像Java堆那样管理内存，但也仅在hotspot中存在永久代。在Java8的时候hotspot删除了永久代，将永久生成的数据移入到Java堆，其余部分移入到native memory，元数据。  
jvm堆方法区的规范很宽松，除了Java堆的规范以外，还可以选择不是先垃圾回收。这个区域很少会涉及到垃圾回收，主要是针对常量池的收回和对类型的卸载，一般效果不是很好。如果方法区无法满足内存分配时，抛出oom。  

> Native memory：jvm中除去Java heap，系统为jvm保留的内存  
> 类型卸载：用户自定义的类加载器的类可以被卸载，而系统自带的类加载器所加载的类，在虚拟机的生命周期中永远不会被卸载。

##### 运行时常量池

运行时常量池是方法区的一部分，用于存放类加载后由编译器生成的变量和符号。Java并不要求常量要在编译器产生，运行期也可以把新的常量放入池中，比如String的intern()。

> 我查了下发现运行时常量池分为class文件常量池、运行时常量池和字符串常量池。其中字符串常量池在6之前是方法区的一部分，7在堆内存，8之后移除了永久代，存在本地内存当中。

##### 直接内存

直接内存(Direct Memory)，不会受Java堆的大小，受本机总内存大小和处理器寻址空间的限制。类似于native memory，硬件中的缓冲区可以共享，以减少相同字节在内存中被复制的次数  
[off-heap, native heap, direct memory and native memory](https://stackoverflow.com/questions/30622818/off-heap-native-heap-direct-memory-and-native-memory)

#### HotSpot虚拟机对象探秘

##### 对象的创建

当虚拟机遇到一条new的指令时，会检查指令的参数是否在常量池中有这个类的符号引用，且检查这个符号引用的类是否被加载、解析、初始化，如果没有则执行加载、解析、初始化。  
类加载检查完后就需要分配内存，大小已经确定。从内存区划分一块有两个实现方法，第一种是占用和空闲各占一边，中间通过指针分割，只需要将指针移动需要的大小就完成内存分配；第二种是占用和空闲交错存在的，这个时候就需要分配一个空闲列表来记录空闲内存，然后选择合适的位置分配。具体使用哪种方式取决于垃圾回收器。  
在分配内存的时候还有个问题就是并发问题，两个线程同时分配内存，两个线程同时读到一个起始位置，就会出现内存被重新分配。解决方法有两个，一个是CAS，一个是为每个线程预先分配一块内存，称为本地线程分配缓冲(TLAB)，如果TLAB用完需要重新申请一个，这个时候就需要同步锁定。这个肯定是要每个线程新分配内存就同步要好很多的。  
内存分配结束后就会将内存空间全部初始化为零值（除了对象头），如果使用TLAB可以在初始化零值，这样就会保证类中定义的值不需要赋值也能直接用，而在函数内变量不初始化是无法使用的。  
接下来就是对象头的内容的设置，例如属于哪个类、对象的哈希值、gc分代年龄。  
对于虚拟机来讲，对象已经创建完成，但对于代码才刚刚开始，因为要执行init方法依照代码去完成初始化。

> 这个常量池应该是class文件常量池，包含字面量、符号引用等。  
> 字面量是比较接近Java语言层面的常量概念 ，如文本字符串、申明为final的常量值等
符号引用属于编译原理方面的概念，包括类和接口的全限定名、字段的名称和描述符、方法的名称和描述符
> 为什么创建对象的时候就能确定大小呢，我猜测以下， 它确定的大小是基本类型的字节加上对象的引用大小，至于数组的话在对象头中有个数组长度的标记。  
> 对象大小 = 对象头大小 + 父级继承链非静态成员域大小 + 当前加载类非静态成员域大小 + 内存对齐导致的Padding。其中对象头大小包括Mark Word、类型指针、数组长度。Mark Work用于存储对象自身的运行时数据，比如对象的hashCode、GC分代年龄、锁状态信息、线程持有锁、偏向线程ID、偏向时间戳

```c++
u2 index = Bytes::get_Java_u2(pc + 1);
ConstantPool *constants = istate->method()->constants();
if (!constants->tag_at(index).is_unresolved_klass())
{
    // Make sure klass is initialized and doesn't have a finalizer
    //index是pc指针，之前指针+1后查看类信息
    Klass *entry = constants->slot_at(index).get_klass();
    InstanceKlass *ik = InstanceKlass::cast(entry);
    //判断类是否经过初始化阶段
    if (ik->is_initialized() && ik->can_be_fastpath_allocated())
    {
        //该类所需要的空间
        size_t obj_size = ik->size_helper();
        oop result = NULL;
        // If the TLAB isn't pre-zeroed then we'll have to do it
        // 判断TLAB是否初始化零值
        bool need_zero = !ZeroTLAB;
        //如果使用了TLAB就用TLAB分配空间，如果没有的话，通过CAS在堆中分配内存
        if (UseTLAB)
        {
            result = (oop)THREAD->tlab().allocate(obj_size);
        }
        // Disable non-TLAB-based fast-path, because profiling requires that all
        // allocations go through InterpreterRuntime::_new() if THREAD->tlab().allocate
        // returns NULL.
#ifndef CC_INTERP_PROFILE
        if (result == NULL)
        {
            need_zero = true;
            // Try allocate in shared eden
        retry:
            HeapWord *compare_to = *Universe::heap()->top_addr();
            HeapWord *new_top = compare_to + obj_size;
            if (new_top <= *Universe::heap()->end_addr())
            {
                if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to)
                {
                    goto retry;
                }
                result = (oop)compare_to;
            }
        }
#endif
        if (result != NULL)
        {
            // Initialize object (if nonzero size and need) and then the header
            //判断是否初始化零值，根据是否使用TLAB
            if (need_zero)
            {
                HeapWord *to_zero = (HeapWord *)result + sizeof(oopDesc) / oopSize;
                obj_size -= sizeof(oopDesc) / oopSize;
                if (obj_size > 0)
                {
                    memset(to_zero, 0, obj_size * HeapWordSize);
                }
            }
            //偏向锁
            if (UseBiasedLocking)
            {
                result->set_mark(ik->prototype_header());
            }
            else
            {
                result->set_mark(markOopDesc::prototype());
            }
            result->set_klass_gap(0);
            result->set_klass(ik);
            // Must prevent reordering of stores for object initialization
            // with stores that publish the new object.
            // 不一定准确，storestore屏障，在store2及以后写入操作执行前，保证store1的写入堆其他处理器可见
            OrderAccess::storestore();
            SET_STACK_OBJECT(result, 0);
            UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
        }
    }
}
```

##### 对象的内存分布

对象在内存中的布局可以分为三部分，对象头、实例数据、对齐填充。对象头分两部分，对象自身的运行时数据(Mark Word)和类型指针。Mark Word是对象自身定义的数据无关的额外存储成本，考虑到空间的问题，被设计成一个非固定的数据结构，32位如下
hash:25 --------->| age:4    biased_lock:1 lock:2 (normal object)  
JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)  
size:32 -------------------------------------------->| (CMS free block)  
PromotedObject*:29 ------>| promo_bits:3 ----->| (CMS promoted object)  
画成像http报文的表格就是  
![image.png](https://i.loli.net/2019/12/07/me4FWg7P9w6MvkH.png)  
64位与32位类似  
类型指针是对象指向它的类元数据的指针，虚拟机可以根据这个指针来确定这个对象是哪个类的实例。如果对象是Java数组，那么对象头还会有一块记录数组长度的数据。  
接下来存储的就是有效信息了，也是程序中定义的字段内容，会继承父类字段，存储顺序会受虚拟机的分配策略参数和字段在Java源码中定义的顺序。hotspot默认策略是long/double、int、short/char、byte/boolean、oop，即相同字节长度的字段被分配在一起，在满足这个条件下，父类字段会在子类之前。  
第三部分对齐是为了保证内存对齐，hotspot要求对象的起始位置8字节的整数倍，如果一个对象大小不满足8N，就需要对齐填充补全。  
> 这里的对象最好理解成只有基础类型的会好理解很多

##### 对象的访问定位

创建完对象后，Java可以根据栈上的reference数据来操作堆上的具体对象`SET_STACK_OBJECT(result, 0);`因为只是一个地址，所以具体怎么读，读到什么有虚拟机来确定，主流是句柄和直接指针，hotspot使用直接指针。句柄是需要维护一个句柄池，池中有对象实例数据的指针和类型的指针，reference指向的是句柄地址；直接指针是存储Java堆的地址，然后在Java堆中存放对象类型数据。句柄的好处是对象移动时只会改变句柄中的实例指针，而reference不需要改变；直接指针的好处是速度快，节省了一次指针定位的时间。

#### 实战OutOfMemoryError

在Java命令加入一些配置  
|命令|含义|  
|--|--|
|-verbose:gc|输出虚拟机GC的详细情况|
|-Xms20M|堆最小内存20M|
|-Xmx20M|堆最大内存20M|
|-Xmn10M|年轻代10M|
|-XX:+PrintGCDetails|打印GC信息|
|-XX:SurvivorRatio=8|Eden占新生代的8/10|

>-XX:SurvivorRatio=n表示Eden : Survivor = n:1，Eden占 n/(n+2)

##### Java堆溢出

Java堆是存放对象实例的，只要不断创建对象实例，并保证GC Roots到对象之间有可达路径避免被回收，就可以实现Java堆溢出。  
使用-XX:+HeapDumpOnOutOfMemoryError可以让虚拟机出现内存溢出是Dump出当前的内存转储快照以便之后分析。当出现问题后就可以根据快照进行分析，其主要是确定内存中的对象是否是必须的，也就是要区分是内存泄漏(Memeory Leak)还是内存溢出(Memory Overflow)。
分析快照的软件书中推荐的是MAT，但是要在eclipse，所以我使用JProfiler。  

```java
/**
 * VM Args:- Xms20m -Xmx20m -XX:HeapDumpOnOutOfMemoryError
 */
static class OOMObject {
}
public static void main(String[] args) {
    List<OOMObject> list = new ArrayList<>();
    while(true) {
        list.add(new OOMObject());
    }
}
```

![image.png](https://i.loli.net/2019/12/14/iAZzYBf6NpWRgoC.png)  
可以很清晰，这个Object数组占用了极高的内存，然后查看这个对象到GC Roots的引用链，查找对象是如何与GC Roots相关联而不被回收掉的  
![image.png](https://i.loli.net/2019/12/14/FHGqKudacBpswv9.png)
如果不是内存泄漏，意味着所有对象都必须存活，这个时候查看是否需要调整-Xmx和-Xms，物理内存是否可以再调大一些；检查对象的生命周期是否过长。

##### 虚拟机栈和本地方法栈溢出

在hotspot中并不区分虚拟机栈和本地方法栈，虽然后-Xoss设置本地方法栈的参数，但是是无效的，只能通过-Xss来定义。  
Java虚拟机规范中描述了两种异常：栈深度大于虚拟机允许的最大深度，抛出StackOverflowError；虚拟机扩展本地方法栈的时候无法申请到足够的空间，抛出OutOfMemory异常。下面的方法抛出SOF  

```c++
/**
 * VM args: -Xss180k
  * 书中是128k，现在最小是180k了
 */
private int stackLength = 1;
public void stackLeak() {
    stackLength++;
    stackLeak();
}
public static void main(String[] args) {
    JavaVMStackSOF oom = new JavaVMStackSOF();
    try {
        oom.stackLeak();
    } catch (Throwable e) {
        System.out.println("stack length:" + oom.stackLength);
        throw e;
    }
}
```

书中说在单线程下无法抛出OOM，在多线程下可以抛出OOM。32位Windows是下每个进程用户可以内存为2G，本地方法栈大小为2G - Xmx - MaxPermSize。这就意味着，每个线程分配的栈容量大，则创建的线程数就变少，如果无法修改栈容量，可以减少堆容量来为多线程分配更多内存空间。  
多线程溢出确实会造成Windows假死，而Linux下不会，书上说12章回讲。  

>OutOfMemeory书中写的不是很好，在Oracle的虚拟机规范中是这么写的
>
>- If the computation in a thread requires a larger Java Virtual Machine stack than is permitted, the Java Virtual Machine throws a   StackOverflowError.  
>- If Java Virtual Machine stacks can be dynamically expanded, and expansion is attempted but insufficient memory can be made available to effect the expansion, or if insufficient memory can be made available to create the initial Java Virtual Machine stack for a new thread, the Java Virtual 
Machine throws an OutOfMemoryError.  
>
>翻译过来是  
>
>- 如果线程中的计算所需的Java虚拟机堆栈超出允许的范围，则Java虚拟机将抛出StackOverflowError。  
>- 如果可以动态扩展Java虚拟机堆栈，并且尝试进行扩展，但是没有足够的内存来实现扩展，或者如果没有足够的内存来为新线程创建初始Java虚拟机堆栈，则Java虚拟机机器抛出一个OutOfMemoryError。  
>
>[Java虚拟机规范(Java Virtual Machine Specification)](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.2)。7和9使用的相同的描述  

##### 方法区和运行时常量池溢出

方法区和运行时常量池在1.7开始溢出了持久代，因此在1.6可以使用-XX:MaxPermSize来这是持久代最大内存。String.intern()是一个Native方法，如果常量池中存在当前字符串, 就会直接返回当前字符串. 如果常量池中没有此字符串, 会将此字符串放入常量池中后, 再返回。  
openjdk已经不提供1.6的二进制文件了（我没找到），我就在GitHub上找了一个。  

```java
List<String> list = new ArrayList<String>();
int i = 0;
while (true) {
    list.add(String.valueOf(i++).intern());
}
```

但是，书上的代码不溢出，它会一直创建，使用JProfilter发现它会一直在堆上创建，我添加了-Xmx=10M抛出了OOM，也就是说我添加的-XX:PermSize=10m -XX:MaxPermSize=10m没有生效，但是减少MaxPermSize的值比如2m确实能抛出OutOfMemoryError:PermGen space，就很奇怪了，总不会是这个是魔改版的吧。 ~~应该是魔改版，后面的String.intern()返回引用的测试也是1.7的输出。~~ 也不对，换了zulu的openjdk也是没有溢出，返回引用也是1.7的输出。  
理论上讲，String.intern()的对象会存储在永久代，然后返回永久代的引用，而StringBuilder创建的对象在Java堆上，所以str1.intern() == str1是错误的。而在1.7之后intern()只是在常量池中记录首次出现的实例。  
至于7及以后的方法区溢出时运行时产生大量类填满方法区，比如动态生成大量class的应用。  

>String.intern()可以看这篇博客 [深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

##### 本机直接内存溢出

DirectMemory类似于native memory，是共享的硬件底层缓冲区，通过-XX:MaxDirectMemorySize指定，不指定默认与-Xmx一样。代码中越过了DirectByteBuff类，直接通过反射获取Unsafe实例及逆行内存分配，因为DIrectByteBuffer分配内存抛出异常时，并没有真正像操作系统申请内存，而是通过计算然后抛出，真正申请的时unsafe.allocateMemory()  

```java
private static final int _1MB = 1024 * 1024;
public static void main(String[] args) throws Exception {
    Field unsafeField = Unsafe.class.getDeclaredFields()[0];
    unsafeField.setAccessible(true);
    Unsafe unsafe = (Unsafe) unsafeField.get(null);
    while(true) {
        unsafe.allocateMemory(_1MB);
    }
}
```

书中说DirectMemory导致内存溢出， 一个特征时Heap Dump文件中不会看到明显异常，而我加了-XX:+HeapDumpOnOutOfMemoryError后并没有快照生成。

### 垃圾收集器与内存分配策略

#### 对象已死么

##### 引用计数算法

给对象中添加一个引用计数器，当有地方引用它时，计数器+1，当引用失效时，计数器-1，当计数器为0的时候，表示该对象没有被引用，所以可以被回收。  

```java
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    private byte[] bigSize = new byte[2 * _1MB];
    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objA.instance = objA;
        objA = null;
        objB = null;
        System.gc();
    }
}
```

这两个对象相互引用，但是引用计数器不为0，于是无法回收它们。难道不能设置null为0么？？  
[GC [PSYoungGen: 7997K->648K(75776K)] 7997K->648K(248320K), 0.0016632 secs]  
实际上JVM还是会回收了它们，因为JVM才用的是可达性分析算法

##### 可达性分析算法

这个算法思路是通过一系列为“GC Roots”的对象作为起点，从这些节点开始向下搜索，所走过的路叫做引用链，当一个对象到GC Roots没有任何引用链的时候，就可以证明对象不可用，可以被回收掉。  

```java
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024 * 1024;
    private byte[] bigSize = new byte[2 * _1MB];
    public static void testGC() throws InterruptedException {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objA.instance = objA;
        objA = null;
        objB = null;
        ReferenceCountingGC objC = new ReferenceCountingGC();
        ReferenceCountingGC objD = new ReferenceCountingGC();
    }
}
```

这段代码可以通过JProfiler分析  
![image.png](https://i.loli.net/2019/12/14/Ylsiehnyfz3pBLT.png)
可以看到这个类是有5个实例引用的，而查看堆的快照的时候只有3个实例(AB为null了)，点击上面的Run GC，实例引用变成了3个了  
![image.png](https://i.loli.net/2019/12/14/qBXYLdMA4JoGkHe.png)
可作为GC Roots的在源码中有10种，为枚举类型，且还分MarkFromRootsTask和ScavengeRootsTask，即标记和清楚，但是枚举是一样的，只是顺序不同

```c++
enum RootType {
    universe              = 1,      //JVM内部使用的引用
    jni_handles           = 2,      //JNI方法
    threads               = 3,      //所有Java线程当前栈帧的引用和虚拟机内部线程
    object_synchronizer   = 4,      //所有synchronized锁住的对象引用
    flat_profiler         = 5,      //性能分析器
    management            = 6,      //内存池、线程对象和线程快照对象
    jvmti                 = 7,      //JVMTI导出
    system_dictionary     = 8,      //JVM内部使用的引用
    class_loader_data     = 9,      //所有已加载的类
    code_cache            = 10      //本地代码，比如JIT
};
```

这些过于官方了，还是书中的比较好理解，关注点相对少一点。

- 虚拟机栈中以用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI

##### 再谈引用

在JDK1.2后，提出了强引用、软引用、弱引用、虚引用。  
强引用：只要强引用还在，就不会被回收，比如new的对象。  
软引用：还有用，但不是必须的对象，之后再内存不够的时候才会被回收，可以作为缓存  
弱引用：比软引用更弱一些，生命周期直到下次垃圾回收的时候  
虚引用：并不会决定对象的生命周期，也无法通过虚引用来取得一个对象的实例，其主要是用来追踪对象被垃圾回收器回收的活动，必须和引用队列联合使用。  

>引用队列 ReferenceQueue，当引用对象所指向的内存空间被GC回收后，该引用对象被追加到引用队列的末尾  
>我们通过get方法得到其指定对象，它的唯一作用是当其他指向对象被回收之后，自己被加入引用队列，用作记录该引用指向的对象已被销毁，虚引用能在其指向的对象从内存中移除掉之后才会加入到引用队列，程序可以通过判断引用队列中是否已经加入虚引用，来了解被引用的对象是否被垃圾回收

##### 生存还是死亡

一个对象被回收至少要经历两次标记过程：可达性分析后发现与GC Roots没有引用链，那么他会被第一次标记并且根据是否有必要执行finalize()进行一次筛选。当对象没有重写finalize()方法，或者该方法已经被虚拟机调用过，就会被视为“没有必要执行”。而一个对象被判断为必须执行finalize()方法，就会讲这个对象放入到F-Queue队列中，并稍后由一个虚拟机自动创建的、低优先级的Finalized线程去执行finalize()方法。稍后GC将对F-Queue进行第二次标记，如果对象在finalize()中重新与GC Roots建立引用链，例如把自己(this)复制给某个类或者类的成员变量，就不会被回收。  

```java
@Override
protected void finalize() throws Throwable {
    super.finalize();
    FinalizeEscapeGC.SAVE_HOOK = this;
}
```

不过finalize()方法并不好，在jdk9中已经提示过时，在《Effect Java》中也不建议使用[effect java](http://sjsdfg.gitee.io/effective-java-3rd-chinese/#/notes/08.%20%E9%81%BF%E5%85%8D%E4%BD%BF%E7%94%A8Finalizer%E5%92%8CCleaner%E6%9C%BA%E5%88%B6)

###### 疑问

为什么F-Queue队列中执行缓慢或者死循环会使GC崩溃？

##### 回收方法区

方法区里面有常量和类信息，方法区在Java堆，也可以按照Java堆回收一样，只是效率比较低。常量可以像对象一样回收，比如String中没有引用，在回收阶段就会被回收。而类卸载就比较麻烦，要同时满足3个条件：  

- 该类所有实例都被回收
- 加载该类的ClassLoader已经被回收
- 该类对应的java.lang.Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法

#### 垃圾收集算法

##### 标记 - 清除算法

最基础的收集算法，分为标记和清除两个阶段。标记是与GC Roots没有引用链的，清除就是回收标记的内存。但是这个算法有两个缺点：效率问题，标记和清除两个过程效率不高；空间问题，直接清除掉会造成内存空间碎片，如果一个大的对象要分配内存，如果无法找到足够大的内存旧的出发另一次垃圾收集动作。  

##### 复制算法

将内存平均分成两半，每次只使用一半。回收的时候，将一半的内存中活的对象按顺序复制到另一半内存中，这样就不会存在空间碎片了。这种算法的代价就是内存减少了一半。现在的商业虚拟机都采用这种方法来收集新生代。新生代分为一个较大的Eden区和两个Survivor区，每次使用Eden区和一个Survivor区。回收的时候，将Eden区和Survivor存活的对象一次性复制到另一个Survivor区种，然后清除掉Eden区和Survivor。HotSpot默认Eden：Survivor = 8：1。如果新生代放不下，对象可以直接通过分配担保机制进入老年代。  

##### 标记 - 整理算法

如果对象存活率过高就要进行较多的复制，效率低。如果不想浪费50%空间，就需要额外的空间担保。通常老年代一般选择复制算法，而是使用标记 - 整理算法。标记过程与标记 - 清除一样，之后不采用清除，而是让所有存活的对象向一端移动，然后清除掉边界以外的内存。  

##### 分代收集算法

根据对象存活周期的不同将内存划分为几块，一般分为新生代和老年代。新生代有大量对象被清除，少量对象存活，使用复制算法；老年代对象存活率高，使用标记 - 整理算法。

#### HotSpot算法实现

##### 枚举根节点

在标记过程要查找与GC Roots的引用链，可做为GC Roots节点的主要是全局性的引用(常量、静态变量等)与执行上下文(例如栈帧中的本地变量表)，如果是遍历这里面的引用，必然回消耗很多时间。同时可达性分析对执行时间的敏感还体现在GC停顿上，因为在GC过程中要保证内存不变，如果在GC过程中被标记的对象被再次引用就会出现问题，所以必须停止所有Java执行线程。  
为了解决这个问题，主流JVM采用准确式GC，即当执行系统停顿下来后，虚拟机知道哪些地方存放着对象引用，在HotSpot中使用OopMap的数据结构，记录了对象偏移量和类型的映射关系，  

>书中讲的实在是简单，发现了两篇不错的文章  
>[JVM 之 OopMap 和 ememberedSet](https://www.javatt.com/p/47827) 和 [找出栈上的指针/引用](https://www.iteye.com/blog/rednaxelafx-1044951)  
>一个线程意味着一个栈，一个栈由多个栈帧组成，一个栈帧对应着一个方法，一个方法里面可能有多个安全点。 gc 发生时，程序首先运行到最近的一个安全点停下来，然后更新自己的 OopMap ，记下栈上哪些位置代表着引用。枚举根节点时，递归遍历每个栈帧的 OopMap ，通过栈中记录的被引用对象的内存地址，即可找到这些对象(GC Roots) 。

##### 安全点

安全点是一个特定位置，到了这个位置，才会线程挂起开始GC。安全点选的太少会导致GC时间过长，选的太多会增大运行的压力，要选择一个平衡点，通常会在方法调用、循环跳转、异常跳转等功能性指令会产生SafePoint。为了使所有线程能够在安全点后挂起，有两种方法：抢先式中断和主动式中断：抢先式中断是所有线程全部挂起，未运行到安全点的恢复运行到安全点；主动式中断是在安全点设置标记位，然后轮询，挂起。  

##### 安全区域  

安全区域是安全点的扩展，比如当一个线程sleep，gc显然是不会等到线程到安全点的，所以采用安全区域，即在一段代码片段中，引用关系不会发生变化。当线程执行到安全区域中代码时，标识自己已经进入safe region，如果这段时间发生GC，就不会管标记为safe region的线程了。当线程离开安全区域时，会检查系统是否完完成根节点枚举，如果完成线程继续执行，否则就会等待系统发出的可以安全离开的信号。

#### 垃圾回收器

![image.png](https://i.loli.net/2019/12/21/Vz6YwiuFxEnGLd1.png)  
HotSpot虚拟机垃圾收回器，连线表示可以组合使用

##### Serial收集器

最基本最悠久的回收器，是一个单线程的收集器，在垃圾回收的时候必须暂停其他所有工作线程，直至收集结束。Serial收集器是虚拟机运行在Client模式下的默认新生代收集器，它和其他收集器比的优点是简单而高效，对于单核环境，没有线程交互，可以获得最高的单线程收集效率。  

>client模式，客户端环境，针对GUI优化，启动速度快，运行速度不如server  
>server模式，服务器环境，启动慢，运行快  
>可以通过java -version查看  

##### ParNew收集器

Serial收集器的多线程版本，是运行在server模式下的首选新生代收集器，除了性能原因，还有个原因是除了Serial收集器意外唯一一个能与CMS收集器配合工作的。从图中就可以看出来。在单核绝对不会有Serial收集器更好，因为有线程交互的开销  

>并行：多个相同类型的线程一起工作，其他类型线程处于等待状态  
>并发：所有线程一起执行

##### Parallel Scavenge收集器

并行多线程收集器，其特点是别的收集器关注点在于尽可能减少由于垃圾回收造成的用户线程停顿，而Parallel Scavenge关注点在于达到一个可控的吞吐量，即 CPU运行用户线程时间与CPU总消耗时间的比值。  
Parallel Scavenge提供两个参数用于控制吞吐量，最大垃圾回收时间-XX:MaxGCPauseMillis和吞吐量大小-XX:GCTimeRatio。MaxGCPauseMillis允许值是大于0的毫秒数，收集器会尽量把时间控制在这个值内，但不要把这个参数设置的太小，因为GC停顿时间是牺牲吞吐量和新生代空间换取的，回收小内存比大内存快，但是更频繁，所以吞吐量也会降下来；GCTimeRadio是大于0小于100的整数，相当于吞吐量的倒数，默认值为99，即允许最大1%，1 / (1 +99)。  
也可以使用-XX:+UseAdaptiveSizePolicy来自动调节新生代大小、Eden与Survivor比例，晋升老年代对象大小，使用这个参数只需要把基本内存设置好，然后使用MaxGCPauseMillis或GCTimeRatio给虚拟机一个优化目标。
