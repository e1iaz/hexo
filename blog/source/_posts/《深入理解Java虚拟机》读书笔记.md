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

###### 对象的访问定位

创建完对象后，Java可以根据栈上的reference数据来操作堆上的具体对象`SET_STACK_OBJECT(result, 0);`因为只是一个地址，所以具体怎么读，读到什么有虚拟机来确定，主流是句柄和直接指针，hotspot使用直接指针。句柄是需要维护一个句柄池，池中有对象实例数据的指针和类型的指针，reference指向的是句柄地址；直接指针是存储Java堆的地址，然后在Java堆中存放对象类型数据。句柄的好处是对象移动时只会改变句柄中的实例指针，而reference不需要改变；直接指针的好处是速度快，节省了一次指针定位的时间。
