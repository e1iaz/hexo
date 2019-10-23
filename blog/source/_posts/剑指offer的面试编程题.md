---
title: 剑指offer的面试编程题
date: 2019-09-04 17:25:30
categories: "算法"
tags: ["算法", "Java"]
---
## 实现Singleton模式
懒汉模式
```java
public class Singleton {
    private static Singleton singleton;
    private Singleton() {
        System.out.println("only noe");
    }
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
饿汉模式
```java
public class Singleton {
    //饿汉模式    
    private static Singleton singleton = new Singleton();
    private Singleton() {
        System.out.println("饿汉模式");
    }
    public static Singleton getSingleton() {
        return singleton;
    }
}
```
## 二维数组中的查找

要从右上或左下进行查找
```java
public static boolean find() {
    int[][] a = {{1, 2, 8, 9}, {2, 4, 9, 12}, {4, 7, 10, 13}, {6, 8, 11, 15}};
    int rows = a.length;
    int columns = a[0].length;
    int find = 6;
    boolean found = false;
    int row = 0;
    int column = columns - 1;
    while (row < rows && column >= 0) {
        if (a[row][column] == find) {
            found = true;
            break;
        } else if (a[row][column] > find) {
            --column;
        } else {
            row++;
        }
    }
    System.out.println(found);
    return found;
}
```
## 从尾到头打印链表
链表
```java
public static void printList() {
    ListNode list;
    Stack<ListNode> stack = new Stack();
    while (list.next != null) {
        stack.push(list);
        list = list.next;
    }
    while (!stack.empty()) {
        System.out.println(stack.pop().value);
    }
}
```
## 重建二叉树
根据前序和中序重构二叉树，有个规律就是前序的第一个为root节点，中序root节点的左边是左子树，右边是右子树
```java
public static ListNode tree(int[] qian, int qHead, int qTail, int[] zhong, int zHead, int zTail) {
    ListNode root = new ListNode(qian[qHead]);
    //        System.out.print("前：");
    //        for (int i = qHead; i <= qTail; ++i) {
    //            System.out.print(qian[i] + " ");
    //        }
    //        System.out.println();
    //        System.out.print("后：");
    //        for (int i = zHead; i <= zTail; ++i) {
    //            System.out.print(zhong[i] + " ");
    //        }
    //        System.out.println();    
    int num = 0;
    for (int i = zHead; i <= zTail; ++i) {
        if (qian[qHead] == zhong[i]) {
            num = i - zHead;
        }
    }
    if (num == 0) {
        root.left = null;
    } else {
        root.left = tree(qian, qHead + 1, qHead + num, zhong, zHead, zHead + num - 1);
    }
    if (num == zTail - zHead) {
        root.right = null;
    } else {
        root.right = tree(qian, qHead + num + 1, qTail, zhong, zHead + num + 1, zTail);
    }
    return root;
}
```
## 二叉树的下一个节点
给定二叉树和其中一个节点，如何找出中序遍历的下一个节点。
很巧妙的从左节点开始，先判断该节点有没有右节点，没有一直向上走
```java
public static ListNode tree(ListNode node) {
    ListNode next = null;
    if (node.right != null) {
        ListNode pRight = node.right;
        while (pRight.left != null) {
            pRight = pRight.left;
        }
        next = pRight;
    } else if (node.parent != null) {
        ListNode current = node;
        ListNode parent = node.parent;
        while (parent != null && current == parent.right) {
            current = parent;
            parent = parent.parent;
        }
        next = parent;
    }
    return next;
}
```

## 斐波那契问题
### 青蛙台阶问题
一只青蛙一次可以跳上1级台阶，也可以跳2级台阶，求青蛙跳上一个n级台阶总共的跳法。
我们将n级台阶当成n的函数，当$ n > 2 $时，第一次跳有两种选择，跳1级，此时跳法数目等于后面剩下的$ n - 1$级的跳法数目，即$f(n-1)$；跳2级，此时跳法数目等于后面剩下的$n-2$级的跳法数目，即$f(n-2)$。因此n级台阶的不同跳法是$f(n) = f(n-1) + f(n-2)$。
如果条件改成，一只青蛙一次可以跳上1级台阶，也可以跳上2级……也可以跳上n级，跳法总数为$f(n) = 2^{n-1}$
### 矩形填充问题
用2x1的小矩形横着或竖着去覆盖更大的矩形，请问8个2x1的小矩形无重叠地覆盖一个2x8的大矩形，共有多少种方法？
竖着放的时候，右边还剩2x7的区域，横着放的时候，右边还剩2x6的区域，$f(8) = f(7) + f(6)$
## 旋转数组的最小数字
将要给有序的数组的前若干个元素搬到数组末尾，找出最小的元素
二分法，首先定义两个指针，第一位和最后一位，正常情况下是`num[0] >= nums[last]`，然后和中间的`nums[mid]`对比，如果第一位比mid大，说明最小的在前一半，`last = mid`，继续对比。
```java
public static int findMin(int[] nums) {
    int mmin = nums[0];
    int len = nums.length;
    int first = 0, last = len - 1, mid = len / 2;
    while (last - first != 1) {
        if (nums[first] < nums[last]) {
            mmin = nums[first];
            break;
        }
        if (nums[first] == nums[last] && nums[first] == nums[mid]) {
            mmin = nums[first];
            for (int i = first; i <= last; ++i) {
                if (mmin >= nums[i]) {
                    mmin = nums[i];
                }
            }
            break;
        }
        if (nums[first] > nums[mid]) {
            last = mid;
        } else {
            first = mid;
        }
        mid = (first + last) / 2;
        System.out.println(first + " " + last);
    }
    if (last - first == 1) {
        mmin = nums[last];
    }
    return mmin;
}
```