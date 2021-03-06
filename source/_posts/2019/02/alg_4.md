---
layout: post
title: 数据结构与算法之美（四）
date: '2019-02-20'
categories: 数据结构与算法
mathjax: true
abbrlink: 3b09d701
---

# 排序（sort）- 上

## 几种经典排序算法及其时间复杂度级别

- 冒泡、插入、选择：O($n^2$) 基于比较
- 快排、归并：O($n\log n$) 基于比较
- 计数、基数、桶：O(n) 不基于比较

<!-- more-->

## 如何分析一个排序算法

### 学习排序算法的思路

明确原理、掌握实现以及分析性能。

### 分析排序算法性能

从执行效率、内存消耗以及稳定性 3 个方面分析排序算法的性能。

### 执行效率

从以下 3 个方面来衡量：

1. 最好情况、最坏情况、平均情况时间复杂度
2. 时间复杂度的系数、常数、低阶：排序的数据量比较小时考虑
3. 比较次数和交换（或移动）次数

### 内存消耗

通过空间复杂度来衡量。

针对排序算法的空间复杂度，引入原地排序的概念，原地排序算法就是指空间复杂度为 O(1) 的排序算法。

### 稳定性

如果待排序的序列中存在值等的元素，经过排序之后，相等元素之间原有的先后顺序不变，就说明这个排序算法时稳定的。

## 冒泡排序

### 冒泡排序算法原理

1. 冒泡排序只会操作相邻的两个数据。
2. 对相邻两个数据进行比较，看是否满足大小关系要求，若不满足让它俩互换。
3. 一次冒泡会让至少一个元素移动到它应该在的位置，重复 n 次，就完成了 n 个数据的排序工作。
4. 优化：若某次冒泡不存在数据交换，则说明已经达到完全有序，所以终止冒泡。

### 冒泡排序代码实现

```Java
/**
    * 冒泡排序
    *
    * @param a 待排序数组
    * @param n 数组长度
    */
public void bubbleSort(int[] a, int n) {
    if (n <= 1) return;

    for (int i = 0; i < n; ++i) {
        // 提前退出冒泡循环的标志位
        boolean flag = false;
        for (int j = 0; j < n - i - 1; ++j) {
            if (a[j] > a[j + 1]) { // 交换
                int tmp = a[j];
                a[j] = a[j + 1];
                a[j + 1] = tmp;
                flag = true;  // 表示有数据交换
            }
        }
        if (!flag) break;  // 没有数据交换，提前退出
    }
}
```

### 冒泡排序性能分析

#### 冒泡排序的执行效率

最小时间复杂度：数据完全有序时，只需进行一次冒泡操作即可，时间复杂度是 O(n)。
最大时间复杂度：数据倒序排序时，需要 n 次冒泡操作，时间复杂度是 O($n^2$)。
平均时间复杂度：通过有序度和逆序度来分析。

什么是有序度？

有序度是数组中具有有序关系的元素对的个数，比如`[2,4,3,1,5,6]`这组数据的有序度就是 11，分别是`[2,4][2,3][2,5][2,6][4,5][4,6][3,5][3,6][1,5][1,6][5,6]`。同理，对于一个倒序数组，比如`[6,5,4,3,2,1]`，有序度是 0；对于一个完全有序的数组，比如`[1,2,3,4,5,6]`，有序度为$n*(n-1)/2$，也就是 15，完全有序的情况称为满有序度。

什么是逆序度？

逆序度的定义正好和有序度相反。核心公式：逆序度=满有序度-有序度。

排序过程，就是有序度增加，逆序度减少的过程，最后达到满有序度，就说明排序完成了。

冒泡排序包含两个操作原子，即比较和交换，每交换一次，有序度加 1。不管算法如何改进，交换的次数总是确定的，即逆序度。
对于包含 n 个数据的数组进行冒泡排序，平均交换次数是多少呢？最坏的情况初始有序度为 0，所以要进行 n*(n-1)/2 交换。最好情况下，初始状态有序度是$n*(n-1)/2$，就不需要进行交互。我们可以取个中间值$n*(n-1)/4$，来表示初始有序度既不是很高也不是很低的平均情况。
换句话说，平均情况下，需要 $n*(n-1)/4$ 次交换操作，比较操作可定比交换操作多，而复杂度的上限是 O($n^2$)，所以平均情况时间复杂度就是 O($n^2$)。

#### 冒泡排序的空间复杂度

每次交换仅需 1 个临时变量，故空间复杂度为 O(1)，是原地排序算法。

#### 冒泡排序的算法稳定性

如果两个值相等，就不会交换位置，故是稳定排序算法。

## 插入排序

### 插入排序算法原理

首先，我们将数组中的数据分为 2 个区间，即已排序区间和未排序区间。初始已排序区间只有一个元素，就是数组的第一个元素。插入算法的核心思想就是取未排序区间中的元素，在已排序区间中找到合适的插入位置将其插入，并保证已排序区间中的元素一直有序。重复这个过程，直到未排序中元素为空，算法结束。

### 插入排序代码实现

```Java
/**
    * 插入排序
    *
    * @param a 待排序数组
    * @param n 数组长度
    */
public void insertionSort(int[] a, int n) {
    if (n <= 1) return;

    for (int i = 1; i < n; ++i) {
        int value = a[i];
        int j = i - 1;
        // 查找插入的位置
        for (; j >= 0; --j) {
            if (a[j] > value) {
                a[j + 1] = a[j];  // 数据移动
            } else {
                break;
            }
        }
        a[j + 1] = value; // 插入数据
    }
}
```

### 插入排序性能分析

#### 插入排序的时间复杂度

如果要排序的数组已经是有序的，我们并不需要搬移任何数据。只需要遍历一遍数组即可，所以最好时间复杂度是 O(n)。如果数组是倒序的，每次插入都相当于在数组的第一个位置插入新的数据，所以需要移动大量的数据，因此最坏时间复杂度是 O($n^2$)。而在一个数组中插入一个元素的平均时间复杂都是 O(n)，插入排序需要 n 次插入，所以平均时间复杂度是 O($n^2$)。

#### 插入排序的空间复杂度

从上面的代码可以看出，插入排序算法的运行并不需要额外的存储空间，所以空间复杂度是 O(1)，是原地排序算法。

#### 插入排序的算法稳定性

在插入排序中，对于值相同的元素，我们可以选择将后面出现的元素，插入到前面出现的元素的后面，这样就保持原有的顺序不变，所以是稳定的。

## 选择排序

### 选择排序算法原理

选择排序算法也分已排序区间和未排序区间。但是选择排序每次会从未排序区间中找到最小的元素，并将其放置到已排序区间的末尾。

### 选择排序代码实现

```Java
/**
    * 选择排序
    *
    * @param a 待排序数组
    * @param n 数组长度
    */
public static void selectSort(int[] a, int n) {
    if (n <= 1) return;

    for (int i = 0; i < n; i++) {
        int min = i;
        for (int j = i; j < n; j++) {
            if (a[j] < a[min]) min = j;
        }
        if (min != i) {
            int temp = a[i];
            a[i] = a[min];
            a[min] = temp;
        }
    }
}
```

### 选择排序性能分析

#### 选择排序的时间复杂度

选择排序的最好、最坏、平均情况时间复杂度都是 O($n^2$)。为什么？因为无论是否有序，每个循环都会完整执行，没得商量。

#### 选择排序的空间复杂度

选择排序算法空间复杂度是 O(1)，是一种原地排序算法。

#### 选择排序的算法稳定性

选择排序算法不是一种稳定排序算法，比如`[5,8,5,2,9]`这个数组，使用选择排序算法第一次找到的最小元素就是 2，与第一个位置的元素 5 交换位置，那第一个 5 和中间的 5 的顺序就变量，所以就不稳定了。正因如此，相对于冒泡排序和插入排序，选择排序就稍微逊色了。

## 思考

### 1. 冒泡排序和插入排序的时间复杂度都是 O($n^2$)，都是原地排序算法，为什么插入排序要比冒泡排序更受欢迎呢？

冒泡排序移动数据有 3 条赋值语句，而选择排序的交换位置的只有 1 条赋值语句，因此在有序度相同的情况下，冒泡排序时间复杂度是选择排序的 3 倍，所以，选择排序性能更好。

### 2. 如果数据存储在链表中，这三种排序算法还能工作吗？如果能，那相应的时间、空间复杂度又是多少呢？

一般而言，考虑只能改变节点位置，冒泡排序相比于数组实现，比较次数一致，但交换时操作更复杂；插入排序，比较次数一致，不需要再有后移操作，找到位置后可以直接插入，但排序完毕后可能需要倒置链表；选择排序比较次数一致，交换操作同样比较麻烦。综上，时间复杂度和空间复杂度并无明显变化，若追求极致性能，冒泡排序的时间复杂度系数会变大，插入排序系数会减小，选择排序无明显变化。

### 三种排序算法比较

![三种排序算法比较](https://img.jiangchen.tech/20210505012405.png)