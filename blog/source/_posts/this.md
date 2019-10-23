---
title: this
date: 2018-11-17 17:25:30
categories: "Java"
tags: ["Java"]
---
刚开始没有想写this的，因为感觉这个很简单，但是后来看了后面的解释后，发现有一些细节需要注意的
```java
public class Main {
    int petalCount = 0;
    String s = "initial value";
    Main(int petals) {
        petalCount = petals;
        System.out.println("Constructor w/ int arg only, petalCount=" + petalCount);
    }
    Main(String ss) {
        System.out.println("Constructor w/ String arg only, s=" + ss);
        s = ss;
    }
    Main(String s, int petals) {
        this(petals);
        //this(s); call to 'this()' must be first statement in constructor body this.s = s;
        System.out.println("String & int args");
    }
    Main() {
        this("hi", 47);
        System.out.println("default constructor (no args)");
    }
    void printOetalCount() {        
        //this(11);call to 'this()' must be first statement in constructor body
        System.out.println("petalCount = " + petalCount + " s = " + s);
    }
    public static void main(String[]args) {
        Main x = new Main();
        x.printOetalCount();
    }
}
```
结果是

```cmd
Constructor w/ int arg only, petalCount=47
String & int args
default constructor (no args)
petalCount = 47 s = initial value
```
构造器Main(String s,int petals)表明，尽管可以用this调用一个构造器，但去不能调用两个，并且要将调用的构造器放到开始，否则会编译出错
printPetalCount()表示，除了构造器外，编译器禁止在其他任何方法中调用构造器