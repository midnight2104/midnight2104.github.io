---
layout: post
title: 关于Java8中HashMap扩容的一点疑问
tags: Java
---

每次存储键值对的时候，如果每个桶的位置只存储了一个值（由`hash`函数计算出来的索引比较特殊），那么这种情况下，扩容操作就会不断发生，这样是不是有点浪费空间呢？


比如下面这种情况：
```java

import java.util.HashMap;

public class Test {
    
    /**
     * java version "1.8.0_211"
     * Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
     * Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
     */
    public static void main(String[] args) {
        //指定初始容量为4
        HashMap<Integer, String> hashtable = new HashMap<>(4);

        hashtable.put(0, "000");
        hashtable.put(1, "111");
        hashtable.put(2, "222");
        //存储3的时候就会扩容了
        hashtable.put(3, "333");

        hashtable.put(4, "444");
        hashtable.put(5, "555");
        hashtable.put(6, "666");
        //存储7的时候会再一次扩容
        hashtable.put(7, "777");

        System.out.println(hashtable.size());


    }
}

```

上面的过程，用一张图来表示一下，看着更加直观一些：

![image](http://upyun.midnight2104.com/blog/1.png)


假如一直按照上述情况发生，那么每个桶就只存储了一个值，链表上没有再存储其他值了，也就没有构成一个拉链。在这情况下，散列表的查询操作会很快，直接用下标获取值，但是，空间会不会有一点点浪费？

好像，这个也么有其他解决方法了吧。

