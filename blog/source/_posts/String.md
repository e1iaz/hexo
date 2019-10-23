---
title: String
date: 2018-05-08 17:25:30
categories: "Java"
tags: ["Java", "String"]
---
```java
String s1="test";
String s2="test";
System.out.println(s1==s2);
```
输出值是true，因为String s1是引用，java会在字符串缓冲区（String constant or literal pool）检查是由有“test”的存在，如果不存在将创建一个新的String对象“test”，并将地址给s1，如果已经存在，则不会创建新的对象，直接将地址s2，s1s2；两个引用指向同一个地址，所以输出true。
再在后面修改s2的值

```java
s2="t";
System.out.println(s1);
s2="test";
System.out.println(s1==s2);
```
既然是指向同一个地址，是否修改s2的值会对s1也造成修改呢，输出的s1还是“test”，也就是在缓冲区中新创建了一个String对象“t”，并将地址给了s2。再将s2的值改为“test”，输出s1==s2为true，也就是又指向同一个地址了。
```java
String s3=new String("test");
String s4=new String("test");
System.out.println(s3==s4);
```
输出值是false，因为这种方法是在堆内存中有两个不同的对象。
二更
```java
class Rock {
    String s1 = "hello";
    String s2 = null;
    String s3 = s2;
    Rock(String s) {
        System.out.println(s2);
        s2 = s;
    }
}
public class Main {
    public static void main(String[] args) {
        Rock t = new Rock("world");
        System.out.println(t.s1);
        System.out.println(t.s2);
        System.out.println(t.s3);
    }
}
```
结果：
```shell
nullhelloworldnull
```
可以看到s2是先为null，后调用构造函数，将s2赋值为world。
接下来我又在main函数内添加了两行
```java
String s="world";
System.out.println(s == t.s2);
```
输出为true，也就是和之前一样，依旧是在缓冲区内寻找是否有world。