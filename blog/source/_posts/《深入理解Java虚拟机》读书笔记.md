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

### 第三章垃圾收集器与内存分配策略

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

##### Serial Old收集器

Serial Old是Serial收集器的老年代版本，单线程，使用标记-整理算法，主要用于client模式下的虚拟机使用。Server模式下一种用途是1.5之前与Parallel Scavenge搭配使用，另一种用途是，作为CMS的后备预案

##### Parallel Old 收集器

Parallel Old是Parallel Scavenge收集器的老年版本，多线程，使用标记-整理算法。1.6之后才开始提供，在此之前Parallel Scavenge只能与Serial Old一同使用，但是老年代单线程无法发挥多个CPU的处理能力，吞吐量实际上比不高，知道Parallel Old出现与其配合。

##### CMS收集器

Concurrent Mark Sweep，以获取最短回收停顿时间为目的的收集器，基于标记-清除算法，整体过程分为

- 初始标记
- 并发标记
- 重新标记
- 并发清除

初始标记和重新标记仍然需要停止所有线程，初始标记以下GC Roots能否直接关联到对象，速度很快，并发标记是GC追踪标记过程，而重新标记是修正并发标记期间其他线程运行导致标记不准确的修复过程。其特点就是用户线程和并发标记并发清理一起执行。  
![image.png](https://i.loli.net/2019/12/28/2PfCcJ6FbsvgqQW.png)  
但还是三个明显缺点

- 对CPU资源敏感，虽然两个线程一起执行，但是回收依旧会占用CPU使用，所以会导致总吞吐量降低。CMS默认是(CPU数量 + 3) / 4，当cpu在4个以上时就会占用25%以上的CPU资源。如果是单核，采用用户线程与回收线程交替运行，这样减少了停顿时间，但是收集过程会变长。
- 如果在并发清理过程中用户线程产生的垃圾是不会被标记的，所以这些垃圾就是浮动垃圾，要在下次GC的时候才会被回收。同时使并行的，其他线程也需要空间，所以要预留空间，提前GC。1.5默认设置使老年代使用68%就会被激活，可以通过-XX:CMSInitiatingOccupancyFraction来设置阈值，如果设置很高，运行期间内存无法曼居程序需要，就会临时启用Serial Old收集器，由于单线程， 停顿时间会变长，性能下降。
- 由于采用的使标记-清除算法，所以会产生空间碎片，当碎片过多时，就没有连续的空间存放大对象，需要触发一次Full GC。

##### G1收集器

收集器技术发展的最前沿成果，jdk9中被提议为默认垃圾回收器，与其他收集器相比，举有以下特点：

- 并行与并发，利用多核优势，缩短STW时间
- 分代收集，与之前收集器一样分为新生代和老年代，但是不需要配合其他收集器管理GC堆
- 空间整合：整体上基于标记-整理算法，这样不会产生内存碎片，分配大对象不会提前触发GC
- 停顿时间可控：可是指定在M毫秒的时间片段内，消耗在垃圾回收的时间不得超过N毫秒

G1提出了Region区域，将堆划分为多个大小相等的独立区域，每个区域在逻辑上被映射为新生代老年代，但是新生代和老年代不再是物理隔离。  
![image.png](https://i.loli.net/2019/12/28/Hy6r94oSbshqD7M.png)  

>Humongous区表示巨大对象区，即对象大小超过Region一半的对象  

G1收集器不是全区域收集，而是跟踪各个Region里面的价值，后台维护一个优先列表，每次根据回收时间优先回收价值最大的Region。  
虽然Region按块回收效率会高，但是存在一个问题，就是并不是所有的引用对象都在一个Region里。为了避免全堆扫描，G1采用RemeberSet，每个Region对应一个RememberSet，当发现有Reference类型进行写操作会判读那引用的对象是否处在同一Region下，如果不在就通过CardTable把相关引用信息记录到被引用对象所属的Region的RememberSet中。  

>CardTable，一种数据结构，标记老年代的某块内存区域中的对象是否持有新生代对象的引用  

不计算维护RSet的操作，G1收集器大致分为四步：

- 初始标记，STW，扫描GC Roots，标记直接可达对象并压入扫描栈
- 并发标记，并发递归扫描marking stack，并标记存活对象，扫描SATB pre-write barrier
- 最终标记，STW，需要刷新SATB pre-write barri的budder
- 筛选回收，清点和重置标记状态，重置RSet，盘点活对象，如果没有活对象，即直接回收Region到free list
STAB是堆的一个快照，标记数据结构包括两个位图：previous位图(PTAMS)和next位图(NTAMS).  
![image.png](https://i.loli.net/2019/12/28/nriouQvet85CAdg.png)  
NTAMs会先扫描到Top的空间，在扫描期间，分配的对象会都在[NTAMS, Top]；扫描结束后，会将NTAMS的值赋给PTAMS，STAB给[Bottom, PTAMS]创建一个快照Bitmap，这样确保对象不会被漏标记  

#### 内存分配和回收策略

对象主要分配在新生代的Eden上，如果启动本地线程分配缓冲，将按线程优先在TLAB上分配，少数情况下可能会直接分配在老年代

##### 对象优先在Eden分配

对象在新生代Eden区分配，当Eden没有足够空间进行分配是，虚拟机就会发起一次Minor GC

```java
/**
 * -verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
private static final int _1MB = 1024 * 1024 / 2;
public static void main(String[] args) {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[3 * _1MB];
    allocation2 = new byte[4 * _1MB];
    allocation3 = new byte[4 * _1MB];
    allocation4 = new byte[8 * _1MB];
}
```

```text
Heap
 PSYoungGen      total 9216K, used 7290K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 88% used [0x00000000ff600000,0x00000000ffd1eb20,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 4096K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 40% used [0x00000000fec00000,0x00000000ff000010,0x00000000ff600000)
 PSPermGen       total 21504K, used 3137K [0x00000000f9a00000, 0x00000000faf00000, 0x00000000fec00000)
  object space 21504K, 14% used [0x00000000f9a00000,0x00000000f9d10490,0x00000000faf00000)
```

这段代码和GC日志中显示，allocation123都是存活的，虚拟机没有找到可回收的对象，当给4分配内存是，发现Eden区已经占用了6MB，不足在分配4MB，因此发生Minor GC，而123都是2MB，无法放入到Survivor区，因此只能通过分配担保机制提前转移到老年代。  
不过和书中不同，书中是将3个2MB的对象移入到老年代，而我的代码，是将4移入到老年代  

> MinorGC：新生代GC，发生在新生代的垃圾收集动作，GC频繁，回收速度快  
> MajorGC/Full GC：老年代GC，发生在老年代，出现MajorGC至少发生一次MinorGC，通常比MinorGC慢。  
> 后面再尝试多new几个大对象时，出现了GC时间的时间戳，由此看出，在我的jdk7下，大对象时直接进入老年代，而不是发生GC  
> 也就是说，对象分配现在eden中，然后再MinorGC时将对象转到Survivor里，然后再是老年代。  

##### 大对象直接进入老年代

上面的代码在我的jdk下就是大对象直接进入老年代。可以通过-XX:PretenureSizeThreshold设置大于这个值的对象直接在老年代分配

##### 长期存活的对象将进入老年代

虚拟机给每个对象定义一个对象年龄计数器，如果Eden出生并经过一次Minor GC后被放入到Survivor区时，会将年龄设为1，每次Minor GC都会加一，在到达阈值（默认15）就会晋升到老年代，可以通过-XX:MaxTenuringThreshold设置。
可以很好理解，因为要复制，复制就有消耗，对于长时间存在的对象就没有必要每次都复制，所以放到老年代。

##### 动态对象年龄判断

为了更好的使用内存状况，虚拟机提供动态对象年龄判断。即如果在Survivor空间中相同年龄所有对象大小总和大于Survivor空间的一半，年龄大于或等于该对象就可以直接进入老年代，无需等待阈值。

##### 空间分配担保

现在的jdk只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC，否则就会进行Full GC。
Jdk6 update24之前会进行一个空间担保的参数设定，如果为true则允许冒险。这个冒险是的风险是在Manor GC之前不知道有多少对象会存活，也不知道有多少对象会晋升到老年代，所以设定一个值与老年代的剩余空间进行比较；如果为false，即不开启担保，在老年代最大可用的连续的空间小于新生代所有对象总空间就会出发Full GC。

> 这个担保就是将Full GC的条件分为三段，大于新生代所有对象总空间；小于总空间但大于一个值有风险可以尝试；小于这个值Full GC

### 第四章 虚拟机性能监控与故障处理工具

#### JDK命令行工具

在jdk的bin目录下有一些由Sun公司封装了tools.jar的监控工具。

##### jps：虚拟机进程状态工具

类似于Linux下的ps命令，可以显示所有运行的虚拟机进程，虚拟机执行主类名称以及进程的本地虚拟机ID(LVMID，Local Virtual Marchine Identical)，且LVMID与操作系统的的进程ID一致。  
jps命令格式  
jps [ options ] [ hostid ]  
常用参数  
|参数|含义|
|--|--|
|-q|只输出LVMID，省略主类名称。|
|-m|输出虚拟机进程启动时传递给主类main()函数的参数|
|-l|输出主类的全名，如果进程执行的是jar包，输出jar路径|
|-v|输出虚拟机进程启动时JVM参数|

##### jstat：虚拟机统计信息监视工具

用于监视虚拟机中各种运行状态信息的命令工具，有些像Linux的top命令  
jstat命令格式  
jstat [ option vmid [ interval ] [ s | ms ] [ count ] ]  
其中如果链接到本地虚拟机，vmid就是 LVMID，如果是远程，VMID格式为  
[ protocol: ][ // ] lvmid [ @hostname [ :port ]/servername ]  
参数interval和count表示查询间隔和次数，如果不设置默认值查询1次  
常用参数
|参数|含义|
|--|--|
|-class|监视类装载、卸载数量、总空间以及类装载所耗费的时间|
|-gc|监视Java堆状况，包括Eden区、两个Survivor区、老年代、元空间（jdk6为永久代）等用量、已用空间、GC时间合计等信息|
|-gccapacity|监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间|
|-gcutil|监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比|
|-gccause|与-gcutil功能一样，但是会额外输出导致上次GC产生的原因|
|-gcnew|监视新生代GC状况|
|-gcnewcapacity|监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间|
|-gcold|监视老年代状况|
|-gclodcapacity|监视内容与-gcold基本相同，出书主要关注使用到的最大、最小空间|
|-compiler|输出JIT编译器编译过的方法、耗时等信息|
|-printcompilation|输出已经被JIT编译的方法|
|-gcmetacapacity|输出元数据空间状况
输出参数的含义
|参数|含义|
|--|--|
|S0|第一个Survivor区|
|S0C|第一个Survivor区总容量|
|S0U|第一个Survivor区使用容量|
|S1|第二个Survivor区|
|E|Eden区|
|O|Old区|
|M|元数据|
|CCS|压缩类空间|
|YGC|年轻代回收次数|
|YGCT|年轻代回收消耗时间|
|FGC|老年代回收次数|
|GCT|垃圾回收总时间|
|Loaded|加载class的数量|
|Bytes|所占用空间大小|
|Unloaded|未加载数量|
|Time|类装载所消耗时间|
|NGCMN|新生代最小容量|
|NGCMX|新生代最大容量|
|NGC|当前新生代容量|
|TT|对象在新生代存活的次数|
|MTT|对象在新生代存活的最大次数|
|DSS|期望Survivor区大小|
|Compiled|最近编译方法的数量|
|Failed|失败数量|
|Invalid|不可用数量|
|Time|时间|
|FailedType|失败类型|
|FailedMethod|失败的方法|
|Size|最近编译方法的字节码数量|
|Type|最近编译方法的编译类型|
|Method|方法名称标识|

##### jinfo：Java配置信息工具

实时查看和调整虚拟机各项参数，可以使用 -flag [ +|- ] name 或者 -flag name=value修改一部分运行起家可写的虚拟机参数。  
jinfo 命令格式  
jinfo [ option ] pid  
常用参数  
|参数|含义|
|--|--|
|\<no option>|输出虚拟机和系统所有信息|
|-flags|输出虚拟机所有信息|
|-sysprops|输出系统所有信息|
|-flag \<name>|输出虚拟机指定信息|
|-flag [ +\|- ]\<name>|新增和减少指定参数|
|-flag \<name> = \<value>|修改指定参数|

##### jmap：Java内存映像工具

用于生成堆转储快照，在Windows下只能使用-dump和-histon选项，其他选在要在Linux/Solaris下使用  
jmap命令格式  
jmap [option] vmid  
option选项  
|参数|含义|
|--|--|
|-dump|生成Java堆转储快照。格式：-dump:[live, ]format = b, file=\<filename>，其中live子参数说明是否只dump出存活的对象|
|-finalizerinfo|显示在F-Queue中等待Finalizer线程执行finalize方法的对象|
|-heap|显示Java堆详细信息|
|-histo|显示堆中对象统计信息，包括类、实例数量、合计容量|
|-F|强制生成dump快照|

##### jhat：虚拟机堆转储快照分析工具

搭配jmap使用，分许jmap生成的dump文件，内置了微信的HTTP服务器，生成的dump分析结果后可以在浏览器上查看。

##### jstack：Java堆栈追踪工具

生成虚拟机当前时刻的线程快照，是当前虚拟机内每一条线程正在执行的方法堆栈的集合，目的是定位线程长时间停顿的原因。  
jstack命令格式  
jstack [ option ] vmid  
参数
|参数|含义|
|--|--|
|-F|强制输出线程堆栈|
|--l|除堆栈外，显示关于锁的附加信息|
|-m|如果调用到本地方法栈，可以显示C/C++的堆栈|

##### HSDIS：JIT生成代码反汇编

sun公司推出的反汇编插件，源码在HotSpot中，但没有编译后的程序，需要自己下载或编译。在使用java是添加参数，-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly即可输出，使用-XX:CompileCommand可以让编译器不要内联函数

> 内联函数是指编译器将指定的函数体插入并取代每一处调用该函数的地方，从而节省了每次调用函数带来的额外时间开销。

#### JDK可视化工具

##### JConsole：Java监视与管理控制台

基于JMX的可视化监控，在bin目录下的jconsole.exe。

###### 内存

```java
/**
 * -Xms100m -Xmx100m -XX:+UseSerialGC
 */
static class OOMObject {
    public byte[] placeholder = new byte[64 * 1024];
}
public static void fillHeap(int num) throws InterruptedException {
    List<OOMObject> list = new ArrayList<>();
    for (int i = 0; i < num; ++i){
        Thread.sleep(50);
        list.add(new OOMObject());
    }
    System.gc();
}
public static void main(String[] args) throws InterruptedException {
    fillHeap(1000);
}
```

在这段代码运行后，Java堆是100M，一个OOMObject对象为64KB，1000次大约64MB，加上空线程跑堆内存使用20+MB，差不多也有100MB了，结束后如图所示  
![image.png](https://i.loli.net/2020/01/11/5ChEZ4n8BaQqrKp.png)  
堆的第一个柱状图是Eden区，最大值27328KB，根据默认Eden : Survivor = 8 : 1，则年轻代总大小为 27328 * 10 / 8=34160KB。倒是书中提到的，Eden区和Survivor基本被清空了，应该是没有是线程sleep，没有执行gc导致的，加入一行Thread.Sleep(1000)让函数内gc完成，就会出现书中描述得了。在YGC的过程中，Survivor柱状图没动过，表明Survivor的对象没有别移入到老年代，因为年龄还不够，至于有没有激活动态对象年龄判断还看不清楚。因为gc实在函数内完成的，因为所有对象都与list有关联，所以不会被回收，只需要把gc请求放到main函数中，这样所有的对象都没有关联，就会被GC掉。  

###### 线程

```java
static class SynAddRunnable implements Runnable {
    int a, b;
    public SynAddRunnable(int a, int b) {
        this.a = a;
        this.b = b;
    }
    @Override
    public void run() {
        synchronized (Integer.valueOf(a)) {
            synchronized (Integer.valueOf(b)) {
                System.out.println(a + b);
            }
        }
    }
}
public static void main(String[] args) throws Exception {
    for (int i = 0; i < 100; i++) {
        new Thread(new SynAddRunnable(1, 2)).start();
        new Thread(new SynAddRunnable(2, 1)).start();
    }
}
```

为了避免占用更多的内存， JVM为Integer对象提供一个常量池，范围是[-128, 127]，而synchronized是互斥锁，在多线程的情况下，可能会出现一个线程拿到了Integer.valueOf(1)这个对象的锁，一个对象拿到了Integer.valueOf(2)这个对象的锁，两个线程都在等待对方释放锁，如图所示。  
![image.png](https://i.loli.net/2020/01/11/4MCzQVRtba9N3TU.png)

##### VisulalVM：多合一故障处理工具

基于NetBeans平台开发， 具备了插件扩展功能，通过扩展可以做到  

- 显示虚拟机进程以及进程的配置、环境信息
- 监视应用的CPU、GC、堆等信息
- dump以及分析堆转储快照
- 方法级的程序运行性能分析
- 离线程序快照

对于可视化工具，主要还是熟悉操作。在介绍中，有个比较强的，BTrace，动态日志追踪，可以在程序运行中，加入原本不存在的调试代码。  
创建一个BTraceTest类，类中有个add(int a, int  b)方法，在BTrace标签中使用代码  

```java
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
@BTrace
public class TracingScript {
    @OnMethod(
    clazz="jvm.BTraceTest",  
    method="add",
    location=@Location(Kind.RETURN))
    public static void func(@Self jvm.BTraceTest instance, int a, int b, @Return int result) {
        println("调用堆栈: ");
        jstack();
        println(strcat("方法参数A: ", str(a)));
        println(strcat("方法参数B: ", str(b)));
        println(strcat("方法结果: ", str(result)));
    }
}
```

clazz表示要调试的类名，method表示要调试的方法名，然后再func中写明调试代码，就可以输出想要的信息。在执行后，只要在程序现中执行到add()方法就会打印出a, b和返回值的信息，对于调试还是很有帮助的。

### 第五章 调优案例分析与实战

#### 案例分析

##### 高性能硬件上的程序部署策略

一个15万的PV/天的在线文档类型网站，硬件4核CPU，16GB物理内存。选择64位jdk5，堆大小位12G，收集器使用Parallel Scavenge和Parallel Old，基于吞吐量优先收集器。结果是网站长时间失去响应。  
12G的堆空间，GC时间肯定要比2G空间要长的，而且是基于吞吐量收集器，年轻代是Parallel Scavenge，如果参数设置过小，要求每次Minor GC时间尽可能小，那么新生代空间也会变小；又因为是在线文档类型的网站，因此内存中会有很多序列化对象，如果在Minor GC过程中没有清理到，就会进入到老年代，新生代小则老年代大，Full GC就要使用过多的时间。  
对于高性能硬件部署程序，主要是有两种，一种是通过64位JDK，一种是使用若干个32位构建逻辑集群。第一种情况，需要程序能控制好Full GC次数，不然就会出现案例中现象，频繁Full GC导致网站失去响应。如果Full FC一天一次可以跑定时任务，在没人的时候触发Full GC。不过这种情况还有个问题，如果出现堆溢出，dump文件可能无法分析。  
使用若干个32位集群，需要前端搭建一个负载均衡器，这样会减少GC时间。但同样也有缺点：  

- 节点争夺全局资源，像案例中的磁盘竞争，可能会出现IO问题
- 资源池，可能会出现一个节点线程池满了，而其他节点还很空闲
- 内存上限，我觉得可以使用64位集群，只要分配堆空间小一点。
- 大量冗余的本地缓存，比如String

在案例中的解决方法是5个32位JDK逻辑集群，每个节点2GB；一个负载均衡代理；由于是文档系统，堆CPU要求不高，主要压力在于硬盘IO和内存，所以使用CMS收集器。  
这个案例也体现了微服务的优点，可能横向扩展会是现在的主要方案吧。

##### 集群间同步导致的内存溢出

一个基于B/S的MIS系统，硬件位两台2个CPU、8GB内存，每台启动3个实例，构成6节点的亲和式集群。开始数据存在数据库，由于IO对性能影响大，后来使用JBossCache构建一个全局缓存。不定期出现多次内存溢出。
JBossCache是基于JGroups进行集群间的数据通信，由于信息在传输中有个失败的可能，在确定所有注册在GMS的节点都收到正确的消息之前，消息要保存在内存中。由于是亲和式集群，就要设置一个负责安全校验的全局Filter，每当收到请求后会更新最后的操作时间，并同步到所有节点中去，防止一个用户在一段时间内登录在多台节点上。当多个请求发出，节点键的网络交互会非常频繁，在网络无法完全支持的情况下，就会出现大量消息保存在内存中，产生内存溢出。

##### 堆外内存导致的溢出错误

基于B/S的电子考试系统，4GB，运行32位Windows系统，客户端可以实时从服务器端接收考试数据。测试期间发现服务端不定时抛出内存异常，在尝试把堆大小开到最大，即1.6GB后，内存异常更频繁了。也没有dump文件产生，挂jstat发现GC不频繁，各区也很正常。
在内存溢出后从日志的异常堆栈中发现两行，关于java.nio的异常。这可能是直接内存Direct Memory的溢出，因为Direct Memory不计算在堆内，而是在剩下的0.4G中分配。而且在Direct Memory不够时，不会通知虚拟机进行垃圾回收，而是等待老年代的Full GC，然后顺便清理掉这些。因此会出现内存异常，堆开大反而会增加异常出现。除了Direct Memory，还有一些情况也会出现内存溢出而各区域正常：线程堆栈、Socket缓存、区、JNI代码、虚拟机和GC

##### 外部命令导致系统缓慢

数字校园应用系统，在压测的时候发现CPU使用过高，而CPU使用过高的并不是应用。根据Solaris的Dtrace脚本查看了当前花费CPU最高的时Linux下的fork。通过源码发现用户请求的处理时通过Runtime.getRuntime().exec()调用了外部shell脚本，这个执行消耗很大，改成API就会解决。

##### 服务器JVM进程崩溃

基于B/S的MIS系统，分布式的所有节点虚拟机都出现过进程崩溃自动关闭，每个进程崩溃前出现大量相同的异常java.net.SocketException: Connection reset。询问后发现是MIS系统工作流的待办事项变化时通过web通知OA，且是异步调用，由于两边速度完全不对等，时间一长积累的web服务没有调用完成，导致等待的线程和Socket连接越来越多，最终超过虚拟机承受能力导致进程崩溃。解决方法是改成消息队列。

##### 不恰当数据结构导致内存占用过大

一个64位虚拟机，内存配置为-Xms4g -Xmx8g，ParNew + CMS收集器。平常Minor GC约30毫秒以内，但在每10分钟加载一个约80MB的数据文件到内存形成100万个HashMap<Long, Long>Entry，在这段里Minor GC超过500毫秒。平时GC短时因为新生代大部分对象可以被清除，而分析数据时，大部分数据都存活，而ParNew是基于复制算法，大量对象被复制，所以会很慢。解决方式是加入参数-XX:SurvivorRatio=65536 -XX:MaxTenuringThreshold=0使得去除survivor区，一次Minor GC后直接升到老年代。但这也是治标的方法，治本的方法是需要改代码，因为HashMap<Long, Long>结构存数据文件空间效率太低。

##### 由Windows虚拟内存导致的长时间停顿

GUI桌面程序，偶尔出现1分钟左右的GC，通过-XX:+PrintRefeneceGC查看日志发现真正执行GC时间不长，但准备开始GC时间比较长。同时观察到程序在最小化是，资源管理器显示的占用内存大幅度减少，而虚拟内存没有变化，可能是最小化时工作内存被自动交换到磁盘中，而在这段时间发生GC需要从磁盘中恢复页面，从而导致不正常的停顿。

#### 实战：Idea运行速度调优

书中是eclipse，我尝试一下跟着书调试Idea。其参数如下

``` txt
-Xms128m
-Xmx2026m
-XX:ReservedCodeCacheSize=240m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-XX:CICompilerCount=2
-Dsun.io.useCanonPrefixCache=false
-Djava.net.preferIPv4Stack=true
-Djdk.http.auth.tunneling.disabledSchemes=""
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Dkotlinx.coroutines.debug=off
-Djdk.module.illegalAccess.silent=true
-Dide.no.platform.update=true
-Djdk.attach.allowAttachSelf=true
```

idea启动有两个jvm启动，一个是platform，一个是Launcher。当多个idea启动后会有多个Launcher出现。之前第一次启动idea很慢猜测是启动platform时间过长，而且为了以后创建多个Launcher做准备。  
platform：  

- Full GC被触发0次，0s；Minor GC触发56次，0.477秒
- 加载类44366个，耗时41.539s
- JIT编译时间为31.896s
- 虚拟机被分配665.5MB新生代（266.25的Eden和两个33.25的Surviver区间）以及1.654GB老年代

在观察VisualVM中，在加载类等占用比较高，占一半以上；而JIT编译等动作后台线程完成，是一段一段完成的时间在不断变化的，这导致刚开始使用Idea比较慢。  

##### 升级jdk1.6的性能变化及兼容问题

idea使用的是自带的jdk11版本，idea在选择jdk版本的时候都会经过很长时间的测试，因为升级jdk版本没什么太多的必要。书中提到Heap size上升，而Used heap下降，这个情况我也是遇到了，是由于没有指定Xms和Xmx导致堆内存是浮动的，在一次GC时导致堆空间不够，进行了扩容，因此Heap size上升；GC结束后，Used heap下降。  

##### 编译时间和类加载时间的优化

由于idea使用人多，可以认为编译代码是可靠的，不需要在加载的时候进行字节码验证，可以通过-Xverify:none禁用字节码验证过程。这个优化确实明显，类加载时间为30.953秒，提高了25%左右；编译时间为30.526秒，提供了25%左右  

##### 调整内存设置控制垃圾收集频率

减少GC次数可以减少启动时间，这里可以通过提高新生代和老年代的空间来减少GC次数。只是在我电脑上GC时间影响不大。  

##### 选择收集器降低延迟

idea已经选择了很好用的CMS收集器，并行收集，可以尽可能的降低停顿时间。  

## 第三部分 虚拟机执行子系统

> 一个class文件都对应唯一的类或接口的定义信息，但类或接口并不一定在文件中，比如通过类加载器直接生成  

### 类文件结构

#### 无关性基石

Java主要宣传：一次编写，到处运行(Write one, Run Anywhere)。因此不同平台的虚拟机都使用统一字节码，构成了平台无关性的基石。而虚拟机允许其他语言在其上运行，实现语言无关性。在Java规范分为Java语言规范及Java虚拟机规范。  
实现语言无关性的基础是虚拟机和字节码存储格式。虚拟机不与任何语言绑定，只与class文件绑定。class文件包含虚拟机指令集和符号表和其他辅助信息，且具有强制性的语法和结构化约束。任何语言丢可以编译成class文件。  
Java语言中的各种变量、关键字和运算符号最终由多条字节码组合而成。字节码命令所能提供的命令描述能力比Java语言本身更强大，因此一些Java语言无法有效支持的语言特性，在其他语言中实现。

#### class类文件的结构

class文件是一组8位字节为基础单位的二进制流，每个数据按照顺序排列在class文件中，中间没有任何分隔符。当需要占用8位字节以上空间，使用大端序的方式分割若干个8位字节进行存储。  
class文件格式采用类似于结构体的结构存储，其中包含无符号数和表。无符号数属于基本的数据结构，u1、u2、u4、u8表示1个字节、2个字节、4个字节、8个字节的无符号数，描述数字、索引引用、数量之或者UTD-8的字符串；标是多个无符号数或者其他表作为数据结构构成的复合数据结构，且可以描述有层次关系的复合结构的数据。  

##### 魔法数与class文件的版本

class前4个字节位魔法数，表示文件的类型，为0xCOFEBABE，接下来4个字节为版本号，4、5位为次版本号，6、7位为主版本号。  
![image.png](https://i.loli.net/2020/01/25/dRKvLaJxqjBGzTO.png)  
主版本号为0x0037，十进制为55，表示最高可以支持jdk11  

##### 常量池

![image.png](https://i.loli.net/2020/02/01/vMeqnRl1uLwXIZW.png)  
主版本号后面是常量池入口，显示说明常量池中的数量35-1个，减1是因为容量计数是从1开始的，而不是0，class文件格式规范制定时设计者把第0项空出来表示某些常量池的索引值的数据在特定情况下需要表达不引用任何一个常量池的含义，这个时候需要索引值设置为0表示。只有常量池容量计数时从1开始的，对于其他集合类型是从0开始的。  
常量池主要存放两类常量：字面量，类似于Java语言的常量；符号引用偏向于编译原理，包括类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。而class文件中没有保存内存布局信息，而是通过虚拟机在加载class文件的时候动态解析的。  
常量池中每项都是一张表，有很多种结构不同的表结构，他们共同的特点是表开始第一位是一个u1类型的标志位，表示哪种常量类型。  
|类型|项目|类型|描述|含义|
| -- | -- | -- | -- | -- |
|CONSTANT_Utf8_info|tag|u1|值为1|UTF-8编码的字符串|
||length|u2|UTF-8编码的字符串占用字节数||
||bytes|u1|长度为length的UTF-8编码的字符串||
|CONSTANT_Integer_info|tag|u1|值为3|整型字符串|
||bytes|u4|按照高位在前存储int值||
|CONSTANT_Float_info|tag|u1|值为4|浮点型型字面量|
||bytes|u4|按照高位在前存储float值||
|CONSTANT_Long_info|tag|u1|值为5|长整型字面量|
||bytes|u8|按照高位在前存储的Long值||
|CONSTANT_Double_info|tag|u1|值为6|双精度浮点型字面量|
||bytes|u8|按照高位在前存储double值||
|CONSTANT_Class_info|tag|u1|值为7|类或接口的符号引用|
||index|u2|指向全限定名常量项的索引||
|CONSTANT_String_info|tag|u1|值为8|字符串类型字面量|
||index|u2|指向字符串字面量的索引||
|CONSTANT_Fieldref_info|tag|u1|值为9|字符的符号引用|
||index|u2|指向声明字段的类或者接口描述符CONSTANT_Class_info的索引项||
||index|u2|指向字段描述符CONSTANT_NameAndType的索引项||
|CONSTANT_Methodref_info|tag|u1|值为10|类中方法的符号引用|
||index|u2|指向声明方法的类描述符CONSTANT_Class_info的索引项||
||index|u2|指向名称及类型索引描述符CONSTANT_NameAndType的索引项||
|CONSTANT_InterfaceMethodref_info|tag|u1|值为11|接口中方法的符号引用|
||index|u2|指向声明方法的接口描述符CONSTANT_Class_info的索引项||
||index|u2|指向名称及类型描述符CONSTANT_NameAndType的索引项||
|CONSTANT_NameAndType_info|tag|u1|值为12|字段或方法的部分符号引用|
||index|u2|指向该字段或方法名称常量项的索引||
||index|u2|指向该字段或方法描述符常量项的索引||
|CONSTANT_MethodHandle_info|tag|u1|值为15|表示方法句柄|
||reference_kind|u1|值必须在[1, 9]，决定了方法句柄的类型。方法句柄类型的值表示方法句柄的字节码行为||
||reference_index|u2|值必须是对常量池的有效索引||
|CONSTANT_MethodType_info|tag|u1|值为16|标识方法类型|
||descriptor_index|u2|值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符||
|CONSTANT_InvokeDynamic_info|tag|u1|值为18|标识一个动态方法调用点|
||bootstrap_method_attr_index|u2|值必须是对当前class文件中引导方法表的bootstrap_methods[]数组的有效索引||
||name_and_type_index|u2|值必须是对当前常量池的有效索引，常量池在该索引处的项必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符|  

![image.png](https://i.loli.net/2020/02/01/DjmbchOH5aVAt9I.png)  
tag位为7，标识CONSTANT_Class_info，是一个类或接口的符号引用；name_index是一个索引值，指向常量池一个CONSTANT_Utf8_info常量，表示这个类或接口的全限定名。根据值定位到第27个常量  
![image.png](https://i.loli.net/2020/02/01/QLMhqlRiy3D7Fx6.png)  
010 Editor的插件还是从0还是的，所以查看的26。tag为1表示为UTF-8编码的字符串，length表示字符串长度，bytes[]表示字符数组，这里表示Son这个类名。  
由于二进制文件不方便阅读，bin下的javap工具是专门用于分析Class字节码的工具，使用-verbose参数输出文件字节码内容  
![image.png](https://i.loli.net/2020/02/01/YsH8VTb5BpCZovE.png)  

##### 访问标志

在常量池结束后的两个字节代表访问标志，用于显示这个class文件时接口还是类，还有修饰符。
|标志名称|标志值|含义|
|--|--|--|
|ACC_PUBLIC|0x0001|是否为public类型|
|ACC_FINAL|0x0010|是否被声明为final，只有类可设置|
|ACC_SUPER|0x0020|是否允许使用invokespecial字节码指令的新语意，invokespecial指令的语意在jdk1.0.2发生过变化。为了区别这条指令使用哪种语意，jdk1.0.2之后编译出来的类的这个标志都必须为真|
|ACC_INTERFACE|0x0200|标识这是一个接口|
|ACC_ABSTRACT|0x0400|是否为abstract类型，对于接口或者抽象类来说，此标志值为真，其他类值为假|
|ACC_SYNTHETIC|0x1000|标识这个类并非由用户代码产生的|
|ACC_ANNOTATION|0x2000|标识这是一个注解|
|ACC_ENUM|0x4000|标识这是一个枚举|  
access_flags中一共有16个标志位可以使用，当前只定义了其中8个，没有使用到的标志位为0.  
![image.png](https://i.loli.net/2020/02/01/4DIm1cAbyaSrOd2.png)  
根据标志值相加，发现是ACC_PUBLIC和ACC_SUPER，0x0001 + 0x0020 = 0x0021  

##### 类索引、父类索引与接口索引集合

类索引和父类索引都是u2类型的数据，而接口索引集合是一组u2类型的数据的集合。类索引是这个类的全限定名，父类索引是父类的全限定名，接口索引集合描述这个类实现了哪些接口。类索引和父类索引指向一个类型为CONSTANT_Class_info的类描述符常量，然后根据这个常量找到CONSTANT_Utf8_info类型的常量中的全限定名字符串  
![image.png](https://i.loli.net/2020/02/01/edDYtuql8vJVah5.png)  
this_class和super_class指向常量池第19、20个，然后根据这个在索引的字符串是ForwardingSet和java/lang/Object，即为该类和父类的全限定名。interface_count表示实现接口的数量n，接下来的n个u2类型的数据指向CONSTANT_Class_info的类描述常量然后找到CONSTANT_Utf8_info类型的长岭中的全限定名字符串，这里就指向了java/util/Set  

##### 字段表集合

用于描述类或接口中声明的变量，包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。每个变量都使用标志位来确定他的修饰符。  
|类型|名称|数量|
|--|--|--|
|u2|access_flags|1|
|u2|name_index|1|
|u2|descriptor_index|1|
|u2|attributes_count|1|
|attribute_info|attributes|attributes_count|  

其中access_flags表示修饰符  

|标志名称|标志值|含义|
|--|--|--|
|ACC_PUBLIC|0x0001|字段是否为public|
|ACC_PRIVATE|0x0002|字段是否为private|
|ACC_PROTECTED|0x0004|字段是否为protected|
|ACC_STATIC|0x0008|字段是否为static|
|ACC_FINAL|0x0010|字段是否为final|
|ACC_VOLATILE|0x0040|字段是否为volatile|
|ACC_TRANSIENT|0x0080|字段是否为transient|
|ACC_SYNTHETIC|0x1000|字段是否由编译器自动产生的|
|ACC_ENUM|0x4000|字段是否为enum|  

![image.png](https://i.loli.net/2020/02/01/laFmHc9kCuA8jIZ.png)  
这个字段就是0x0012，也就是0x0010 + 0x0002，即private final。  
接下来是name_index和descriptor_index，其值对应了常量池的索引，类型为CONSTANT_Utf8_info，分别表示字段的简单名称以及字段和方法的描述符，在这里name_index是s，即变量名为s，描述符为Ljava/util/Set;描述符的作用使用来你描述字段的数据类型、方法的参数列表(包括数量、类型、顺序)和返回值，对于描述数据类型的标识符含义如下  
|标识字符|含义|
|--|--|
|B|byte|
|C|Char|
|D|double|
|F|float|
|I|int|
|J|long|
|S|short|
|Z|boolean|
|V|void|
|L|对象类型|
|[|数组|  

如果定义一个`java.lang.String[][]`类型的二维数组，记录为`[[Ljava/lang/String`。  
根据图中查找name_index是s，descriptor_index是Ljava/util/Set，这个字段就是private final Set s;  
之后会跟随一个属性表集合存储一些额外的信息，这里属性计数器为1，之后就会跟随一个Signature，用于记录泛型签名信息。签名有助于实现反射、调试以及编译，现在Java的反射API能够获取到泛型类型，最终的数据来源就是这个属性。  
字段表不会列出从父类继承的字段，但是可能列出原本Java代码中不存在的字段；Java语言中字段无法重载，但是字节码中，两个字段的描述符不一致，而字段重名就是合法的。  

##### 方法表集合

与字段表类似，结构一样，仅在访问标志和属性表集合的可选项中有所区别，因为volatile和transient不能修饰方法，因此没有对象的标志，而多了synchronized、native、strictfp和abstract对应的标志。  

|标志名称|标志值|含义|
|--|--|--|
|ACC_PUBLIC|0x0001|方法是否为public|
|ACC_PRIVATE|0x0002|方法是否为private|
|ACC_PROTECTED|0x0004|方法是否为protected|
|ACC_STATIC|0x0008|方法是否为static|
|ACC_FINAL|0x0010|方法是否为final|
|ACC_SYNCHRONIZED|0x0020|方法是否为synchronized|
|ACC_BRIDGE|0x0040|方法是否由编译器产生的桥接方法|
|ACC_VARARGS|0x0080|方法是否接收不定参数|
|ACC_NATICE|0x0100|方法是否为native|
|ACC_ABSTRACT|0x0400|方法是否为abstract|
|ACC_STRICTFP|0x0800|方法是否为strictfp|
|ACC_SYNTHETIC|0x1000|方法是否是有编译器自动产生的|

![image.png](https://i.loli.net/2020/02/01/eVd6ItXmBc3uKJx.png)  
从图中可以发现这与两个函数，根据name_index和descriptor_index对应的字符串可以知道方法名为`<init>`，描述符为`()V`为构造函数，有一个属性为Code，里面存放方法中的代码。
如果子类没有Override父类方法，在方法集合中就不会出现父类的方法信息；可能会出现编译器自动添加的方法。Java中Overload除了要与原方法有相同的简单名称之外还要求拥有过一个与原方法不同的特征签名，因此不能仅根据返回值的不同来对方法进行重载。而class文件格式中特征签名范围更大，因此只要描述符不是完全一致的两个方法也是可以共存的。

##### 属性表集合

属性表集合的限制比较宽松，任何人都可以向属性表中写入定义的属性信息。而JVM运行时会试别预定于的属性，忽略掉不认识的属性。  
|属性名称|使用位置|含义|
|--|--|--|
|Code|方法表|Java代码编译成的字节码指令|
|ConstantValue|字段表|final关键字定义的常量值|
|Deprecated|类、方法表、字段表|final关键字定义的常量值|
|Exceptions|方法表|方法抛出的异常|
|EnclosingMethod|类文件|仅当一个类为局部类或者匿名类时才能拥有这个属性，这个属性用于标识这个类所在的外围方法。|
|InnerClasses|类文件|内部类列表|
|LineNumberTable|code属性|Java源码的行号与字节码指令的对应关系|
|StackMapTable|code属性|新的类型检查器检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配|
|Signature|类、方法表、字段表|支持泛型情况下的方法签名，再Java语言中，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量或参数化类型，则Signature属性会为它记录泛型签名信息。由于Java的泛型采用擦除发实现，在稳了避免类型信息呗擦除后导致参数混乱，需要找个属性记录泛型中的相关信息。|
|SourceFile|类文件|记录源文件名称|
|SourceDebugExtension|类文件|用于存储额外的调试信息|
|Synthetic|类、方法表、字段表|标识方法或字段为编译器自动生成|
|LocalVariableTypeTable|类|特征签名代替描述符，为了引用泛型语法之后能描述泛型参数化类型而添加的|
|RuntimeVisibleAnnotations|类、方法表、字段名|为动态注解提供支持，用于指明哪些注解是运行时可见|
|RuntimeInvisibleAnnotations|类、方法表、字段表|与RuntimeVisibleAnnotations相反，用于指明哪些注解是运行时不可见的|
|RuntimeVisibleParameter|方法表|与RuntimeVisibleAnnotations类似，只是不能作用对象为方法参数|
|RuntimeInvisibleParameterAnnotations|方法表|与RuntimeInvisibleAnnotations类似，只是不能作用对象为方法参数|
|AnnotationDefault|方法表|用于记录注解类元素的默认值|
|BootstrapMethods|类文件|保存iinvokednamic指令引用的引导方法限定符|

每个属性，名称是从常量池引用CONSTANT_Utf8_info的常量，属性结构自定义，通过attribut_length说明属性占用几位  

###### Code属性

存储Java方法体中的代码，接口或抽象类中的方法不存在Code属性。Code结构如下  
|类型|名称|数量|含义|
|--|--|--|--|
|u2|attribute_name_index|1|常量固定为Code，标识该属性的属性名|
|u4|attribute_length|1|属性的长度。为整个属性长度减6个字节|
|u2|max_stack|1|操作数栈深度的最大值|
|u2|max_locals|1|局部变量表所需的存储空间，单位为Slot。方法参数（包括this）、显示异常处理器的参数、方法体中定义的局部变量都需要使用局部变量表存放。Slot可以重用|
|u4|code_length|1|字节码长度，理论上最大值可以达到232 - 1，但虚拟机规范中明确限制一个方法不允许超过65535条字节码指令|
|u1|code|code_length|存储字节码指令的一系列字节流|
|u2|exception_table_length|1|异常处理表数量|
|exception_info|exception_table|exception_table_length|异常处理表，可以不存在|
|u2|attributes_count|1||
|attribute_info|attributs|attributes_count|

![image.png](https://i.loli.net/2020/02/08/Tnsba9YdExg1m7z.png)  

max_stack和max_locals很简单，没什么说的，接下来是2A B7 00 01 B1  

- 2A：aload_0，将第0个Slot中为reference类型的本地变量推送到操作栈顶
- B7：invokespecial，以栈顶的reference类型的数据所指向的对象作为方法接收者，调用此对象的实例构造器方法、private方法或者它的父类方法，后面跟着CONSTANT_Methodref_info类型常量。
- 00 0A：invokespecial的参数
- B1：return，方法结束

依旧可以使用javap -verbose来查看计算后的字节码指令  
![image.png](https://i.loli.net/2020/02/08/XRNmespQh2ijOk3.png)  
其中locals和args_size都为1，而实际上源码中参数并没有，是因为自带的this关键字访问此方法所属对象，inc(TestClass this)与inc()的class文件是相同的。  
异常处理表的格式4个类型为u2的属性，start_pc、end_pc、handler_pc、catch_type，表示当字节码在第start_pc到end_pc行之间出现类型为handler_pc或者其子类的异常，则转达第handler_pc行继续处理，当catch_type的值为0时，表示任意异常情况都需要转向到handler_pc处进行处理。  

###### Exceptions属性

与Code属性评级，是列举方法中可能抛出的受检异常，也就是throws关键字后面列举的异常  
|类型|名称|数量|含义|
|--|--|--|--|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|number_of_exceptions|1|可能抛出异常的数量|
|u2|exception_index_table|number_of_exceptions|指向CONSTANT_Class_info型常量的索引，代表受检异常的类型|

###### LineNumberTable属性

Java源码行号与字节码偏移量之间对应的关系，可以再javac使用-g:none或-g:lines来设置是否生成，不生成的主要影响是异常堆栈不会显示出错行号  
|类型|名称|数量|含义|
|--|--|--|--|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|line_number_table_length|1|line_number_info的数量|
|line_number_info|line_number_table|line_number_table_length|包含start_pc表示字节码行号，line_number表示Java源码行号|

###### LocalVariableTable属性

描述栈帧中局部变量表中的变量与Java源码中定义的变量之间的关系，可以通过javac的-g:none或-g:vars来取消生成，取消的影响是参数名称丢失，IDE会使用arg0之类的占位符代替。

|类型|名称|数量|
|--|--|--|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|local_variable_table_length|1|
|local_variable_info|local_variable_table|local_variable_table_length|

local_variable_info类型标识栈帧与源码中局部变量的关联  

|类型|名称|数量|含义|
|--|--|--|--|
|u2|start_pc|1|局部变量声明周期开始的字节码偏移量|
|u2|length|1|作用范围覆盖的长度|
|u2|name_index|1|局部变量名称|
|u2|descriptor_index|1|局部变量描述符|
|u2|index|1|局部变量再栈帧局部变量表中Slot的位置，如果变量类型是64位时，占用Slot位index和index + 1两个|

JDK1.5引入泛型后新增LocalVariableTypeTable，与LocalVariableTable类似，还只是把descriptor_index替换成Signature  

###### SourceFile属性

记录生成这个Class文件的源码文件名称，可以通过javac的-g:none或-g:source选项来取消生成，如果取消，在抛出异常时，堆栈不会显示代码所属文件名，对于大多数类来说没有问题，因为类名和文件名时一致的，但内部类等特殊类型不行。

###### ConstantValue属性

通知虚拟机自动为静态变量赋值，只有对被static关键字修饰的变量使用。对于非static类型的变量赋值是在实例构造器`<init>`方法中进行的，对于类变量，则有两种方式可以选择：在类构造器`<clinit>`方法中或者使用ConstantValue属性。在虚拟机规范中有ConstantValue属性的字段必须设置ACC_STATIC标志。从数据结构中，除了attribute_name_index和attribute_length外，还有u2类型的constantvalue_index，指向常量池一个字面量常量的引用，根据字段不同，字面值可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info、STONSTANT_String_info常量中的一个
InnerClasses属性
内部类与宿主类之间的关系，结构中还有1个u2类型的number_of_classes，number_of_class个inner Class_info类型的inner_class，其结构如下  

|类型|名称|数量|含义|
|--|--|--|--|
|u2|inner_class_info_index|1|内部类符号引用|
|u2|outer_class_info_index|1|宿主类符号引用|
|u2|inner_name_index|1|指向常量池CONSTANT_Utf8_info类型的索引，如果匿名内部类则为0|
|u2|inner_class_access_flags|1|内部类访问标志，类似于类的access_flags，范围如下结构。|  

|标志名称|标志值|含义|
|--|--|--|
|ACC_PUBLIC|0x0001|是否为public|
|ACC_PRIVATE|0x0002|是否为private|
|ACC_PROTECTED|0x0004|是否为protected|
|ACC_STATIC|0x0008|是否为static|
|ACC_FINAL|0x0010|是否为final|
|ACC_INTERFACE|0x0020|是否为synchronized|
|ACC_ABSTRACT|0x0400|是否为abstract|
|ACC_SYNTHETIC|0x1000|是否并非由用户代码产生的|
|ACC_ANNOTATION|0x2000|是否是一个注解|
|ACC_ENUM|0x4000|是否是一个枚举|  

###### Deprecated及Synthetic属性

标志类型的布尔属性，只存在有和没有，没有值的概念。Deprecated用于表示某个类、字段或方法已经被作者定为不再推荐使用，可以使用@deprecated注解；Synthetic属性代表此字段或者方法并不是Java源码直接生产，而是由编译器自行添加。

###### StackMapTable属性

一个复杂的变长属性，会在虚拟机类加载的字节码验证阶段被新类型检查器使用，在于代替之前比较消耗性能的基于数据流分析的类型推导验证器。包含零个或多个栈映射帧，显示或隐式代表一个字节码偏移量，用于表示执行该字节码时局部变量表和操作数栈的验证类型。类检查器会通过验证目标方法的局部变量和操作数栈需要的类型来确定一段字节码指令是否符合逻辑约束。

###### Signature属性

记录泛型签名信息，结构中有一个u2类型的signature_index，指向一个常量池的有效索引，为CONSTANT_Utf8_info结构，表示类签名、方法类型签名或字段类型名称

###### BootstrapMethods属性

保存invokedynamic指令引用的引导方法限定符。如果常量池出现过CONSTANT_InvokeDynamic_info类型常量，则必须存在明确的BootstrapMethods属性，且最多只有一个该属性。结构中还有一个u2的num_bootstrap_methods和num_bootstrap_methods个bootstrap_method类型的bootstrap_methods，其结构为  

|类型|名称|数量|含义|
|--|--|--|--|
|u2|bootstrap_method_ref|1|指向常量池的有效索引，且为CONSTANT_MehtodHandle_info结构|
|u2|num_bootstrap_argumengts|1|bootstrap_arguments[]数组成员的数量|
|u2|bootstrap_arguments|num_bootstrap_arguments|对应常量池的索引，结构必须为COUNTSTANT_String_info、Class、Integer、Long、Float、Double、MethodHandle、MethodType之一|
