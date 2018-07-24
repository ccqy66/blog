---
title: Java并发深入之-CAS
date: 2018-02-12 15:53:44
tags:
- Java
- 并发
- CAS
categories:
- Java
- 并发
---
#### 什么是原子操作
> 原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何上下文的切换。
注：原子操作可以是一个步骤，也可以是多个操作步骤，但是其顺序不可以被打乱，也不可以被切割只执行其中的一部分。
<!--more-->
#### 源码剖析

以Java的`AtomicInteger`为例进行分析：

- 首先分析该类的成员变量：
```Java
      // setup to use Unsafe.compareAndSwapInt for updates
      private static final Unsafe unsafe = Unsafe.getUnsafe();
      private static final long valueOffset;
      private volatile int value;
```
  该类共有三个成员属性。
  - unsafe：该类是JDK提供的可以对内存直接操作的工具类。
  - valueOffset：该值保存着AtomicInteger基础数据的内存地址，方便unsafe直接对内存的操作。
  - value：保存着AtomicInteger基础数据，使用volatile修饰，可以保证该值对内存可见，也是原子类实现的理论保障。
- 再谈静态代码块（初始化）
```Java
      static {
          try {
              valueOffset = unsafe.objectFieldOffset
                  (AtomicInteger.class.getDeclaredField("value"));
          } catch (Exception ex) { throw new Error(ex); }
      }
```
该过程实际上就是计算成员变量value的内存偏移地址，计算后，可以更直接的对内存进行操作。
- 核心方法`compareAndSet(int expect,int update)`
```Java
      public final boolean compareAndSet(int expect, int update) {
          return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
      }
```
  在该方法中调用了unsafe提供的方法：
  ```Java  
  public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

  ```
  由于该方法是native方法，为了了解其实现，我们翻阅代码，看看hotspot是如何实现的：
  ```Java
      jboolean sun::misc::Unsafe::compareAndSwapInt (jobject obj, jlong offset,jint expect, jint update)  {
        jint *addr = (jint *)((char *)obj + offset); //1
        return compareAndSwap (addr, expect, update);
      }

      static inline bool compareAndSwap (volatile jlong *addr, jlong old, jlong new_val)    {   
        jboolean result = false;   
        spinlock lock;    //2
        if ((result = (*addr == old)))    //3
          *addr = new_val;    //4
        return result;  //5
      }
      ```
  - 1：通过对象地址和value的偏移量地址，来计算value的内存地址。
  - 2：使用自旋锁来处理并发问题。
  - 3：比较内存中的值与调用方法时调用方所期待的值。
  - 4：如果3中的比较符合预期，则重置内存中的值。
  - 5：如果成功置换则返回true，否则返回false；
  综上所述：compareAndSet的实现依赖于两个条件：
    - volatile原语：保证在操作内存的值时，该值的状态为最新的。（被volatile所修饰的变量在读取值时都会从变量的地址中读取，而不是从寄存器中读取，保证数据对所有线程都是可见的）
    - Unsafe类：通过该类提供的功能，可以直接对内存进行操作。
- 核心方法`getAndIncrement()`
```Java
      public final int getAndIncrement() {
          return unsafe.getAndAddInt(this, valueOffset, 1);
      }
      ```
  同样使用unsafe提供的方法：
  ```Java
      public final int getAndAddInt(Object var1, long var2, int var4) {
          int var5;
          do {
              var5 = this.getIntVolatile(var1, var2);//1
          } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));//2
          return var5;
      }

      //getIntVolatile方法native实现
      jint sun::misc::Unsafe::getIntVolatile (jobject obj, jlong offset)   
      {   
        volatile jint *addr = (jint *) ((char *) obj + offset);    //3
        jint result = *addr;    //4
        read_barrier ();    //5
        return result;    //6
      }
      inline static void read_barrier(){
        __asm__ __volatile__("" : : : "memory");
      }
      ```
  - 1: 通过volatile方法获取当前内存中该对象的value值。
    - 3：计算value的内存地址。
    - 4：将值赋值给中间变量result。
    - 5：插入读屏障，保证该屏障之前的读操作后后续的操作可见。
    - 6：返回当前内存值
  - 2：通过`compareAndSwapInt`操作对value进行+1操作，如果再执行该操作过程中，内存数据发生变更，则执行失败，但循环操作直至成功。

#### CAS面临的问题
- 问题一：ABA问题
 - 问题阐述：
 从上面的分析我们知道：在操作某一个变量的值时首先会检查该值是否被修改（通过与预期值进行比较）。只有再没有修改时，才会操作该值。但是这样会存在一个问题：假设该变量的初始值是A,同时有三个线程操作该变量，线程1操作将变量由A置为B,线程2操作将变量由B置为A,线程3操作时，比较当前的变量与预期值相同，执行操作。但实际上该变量在线程3操作之前是被修改了的。这就是ABA问题。
 - 问题解决：
  - 可以在变量前面增加一个版本号，每次操作都会将版本号加一，即使该变量的值被多次修改，最终和预期值相同，也可以区分出来。操作流程就如：1A-2B-3A
  - 使用jdk提供的类`AtomicStampedReference`：http://blog.hesey.net/2011/09/resolve-aba-by-atomicstampedreference.html

#### 参考文档
- http://zl198751.iteye.com/blog/1848575
- http://blog.hesey.net/2011/09/resolve-aba-by-atomicstampedreference.html
