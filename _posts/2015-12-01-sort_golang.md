---
date: 2015-12-01
layout: post
title: 内部排序算法(Golang实现)
thread: 2015-12-01-sort_golang.md
categories: 算法
tags: Golang
---

<pre>
package main

import (
    "fmt"
)

func main() {
    //保存需要排序的Slice
    arr := []int{9, 3, 3, 4, 7, 2, 4, 7, 2, 1, 4, 7, 2, 11, 12, 11, 18, 19, 12,
	 3, 4, 7, 2, 1, 0, 11, 12, 11, 13, 4, 7, 2, 1, 0, 11, 12, 11, 18,0,1,0,1}

    //实际用于排序的Slice
    list := make([]int, len(arr))

    copy(list, arr)
    QuickSort(list)
    fmt.Println("快速排序：\t", list)

    copy(list, arr)
    QuickSortX(list)
    fmt.Println("快速排序X：\t", list)

    copy(list, arr)
    list = MergeSort(list)
    fmt.Println("二路归并排序：\t", list)
	
    copy(list, arr)
    BubbleSort(list)
    fmt.Println("冒泡排序：\t", list)

    copy(list, arr)
    BubbleSortX(list)
    fmt.Println("冒泡排序X：\t", list)

    copy(list, arr) //将arr的数据覆盖到list，重置list
    InsertSort(list)
    fmt.Println("直接插入排序：\t", list)

    copy(list, arr)
    ShellSort(list)
    fmt.Println("希尔排序：\t", list)

    copy(list, arr)
    SelectSort(list)
    fmt.Println("简单选择排序：\t", list)

    copy(list, arr)
    HeapSort(list)
    fmt.Println("堆排序：     \t", list)

}


//region 快速排序
/*
步骤：
1.从数列中挑出一个元素作为基准数
2.分区过程，将比基准数大的放到右边，小于或等于它的数都放到左边。(每次归位一个基准数到它最终应该在的位置)
3.再对左右区间递归执行第二步，直至各区间只有一个数
PS:
快速排序里面比较精妙，要注意基准数的选择和哨兵指针移动的先后顺序
*/
func QuickSort(list []int) {
    if len(list) <= 1 {
        return
    }
    var low int = 0
    var high int = len(list) - 1
    //以list[0]为基准数
    for low < high {
        if list[high] >= list[0] {
            high--
            continue
        }
        if list[low] <= list[0] {
            low++
            continue
        }

        list[low], list[high] = list[high], list[low]
    }
    //low == high
    if list[low] < list[0] {
        list[low], list[0] = list[0], list[low]
    }
    QuickSort(list[:low])
    QuickSort(list[low+1:])
}

func QuickSortX(list []int) {
    if len(list) <= 1 {
        return
    }
    key, i := list[0], 1
    low, high := 0, len(list)-1
    for low < high {
        if list[i] > key {
            list[i], list[high] = list[high], list[i]
            high--
        } else {
            list[i], list[low] = list[low], list[i]
            low++
            i++
        }
    }
    QuickSortX(list[:low])
    QuickSortX(list[low+1:])
}
//endregion

//region 二路归并排序
/*
步骤:
1.将待排序序列R[0...n-1]看成是n个长度为1的有序序列，将相邻的有序表成对归并，得到n/2个长度为2的有序表；
2.将这些有序序列再次归并，得到n/4个长度为4的有序序列；
3.如此反复进行下去，最后得到一个长度为n的有序序列。

归并排序其实要做两件事：
（1）“分解”——将序列每次折半划分。
（2）“合并”——将划分后的序列段两两合并后排序。
*/
func MergeSort(list []int) []int {   //着重理解该函数
	if len(list) <= 1 {
        return list
    } 
    mid := len(list)/2
    left := MergeSort(list[:mid])
    right := MergeSort(list[mid:])
    return merge(left,right)
}

func merge(left, right []int) (result []int) {
    i,j := 0,0
    for i < len(left) && j < len(right){
        if left[i] < right[j]{
            result = append(result,left[i])
			i++
        }else{
            result = append(result,right[j])
			j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return
}
//endregion

//region 冒泡排序
/*
步骤：
1.比较相邻的元素。如果第一个比第二个大，就交换他们两个
2.对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。在这一点，最后的元素应该会是最大的数
3.针对所有的元素重复以上的步骤，除了最后一个
4.持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较
*/
func BubbleSort(list []int) {
    for i := 0; i < len(list)-1; i++ {
        for j := 0; j < len(list)-i-1; j++ {
            if list[j] > list[j+1] {
                list[j], list[j+1] = list[j+1], list[j]
            }
        }
    }
}

//优化：如果没有交换发生，代表已经有序，即可结束
func BubbleSortX(list []int) {
    var exchange bool = false
    for i := 0; i < len(list)-1; i++ {
        for j := 0; j < len(list)-i-1; j++ {
            if list[j] > list[j+1] {
                list[j], list[j+1] = list[j+1], list[j]
                exchange = true
            }
        }
        if !exchange {
            break
        }
        exchange = false
    }

}
//endregion

//region 插入排序
/*
步骤：
1.从第一个元素开始，该元素可以认为已经被排序
2.取出下一个元素，在已经排序的元素序列中从后向前扫描
3.如果被扫描的元素（已排序）大于新元素，将该元素后移一位
4.重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
5.将新元素插入到该位置后
6.重复步骤2~5
*/
func InsertSort(list []int) {
    var temp, i, j int
    for i = 1; i < len(list); i++ {
        temp = list[i]
        for j = i - 1; j >= 0 && temp < list[j]; j-- {
            list[j+1] = list[j]
        }
        list[j+1] = temp
    }
}

//region 希尔排序
/*
基本思想：
把记录按步长 gap 分组，对每组记录采用直接插入排序方法进行排序。
随着步长逐渐减小，所分成的组包含的记录越来越多，当步长的值减小到 1 时，整个数据合成为一组，构成一组有序记录，则完成排序。
*/
func ShellSort(list []int) {
    for gap := (len(list) + 1) / 2; gap >= 1; gap = gap / 2 {
        for i := 0; i+gap < len(list); i++ {
            InsertSort(list[i : i+gap+1])
        }
    }
}
//endregion

//region 简单选择排序
/*
步骤：
1.在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。
2.再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
3.以此类推，直到所有元素均排序完毕。
*/
func SelectSort(list []int) {
    var index int
    for i := 0; i < len(list)-1; i++ {
        index = i
        for j := i + 1; j < len(list); j++ {
            if list[index] > list[j] {
                index = j
            }
        }
        list[index], list[i] = list[i], list[index]
    }
}
//endregion


//region 堆排序
/*
步骤：
1.构造最大堆（Build_Max_Heap）：
	若数组下标范围为0~n，考虑到单独一个元素是大根堆，则从下标n/2开始的元素均为大根堆。
于是只要从n/2-1开始，向前依次构造大根堆，这样就能保证，构造到某个节点时，它的左右子树都已经是大根堆。

2.堆排序（HeapSort）：
	由于堆是用数组模拟的。得到一个大根堆后，数组内部并不是有序的。因此需要将堆化数组有序化。
思想是移除根节点，并做最大堆调整的递归运算。第一次将heap[0]与heap[n-1]交换，再对heap[0...n-2]做最大堆调整。
第二次将heap[0]与heap[n-2]交换，再对heap[0...n-3]做最大堆调整。重复该操作直至heap[0]和heap[1]交换。
由于每次都是将最大的数并入到后面的有序区间，故操作完后整个数组就是有序的了。

3.最大堆调整（Max_Heapify）：
	该方法是提供给上述两个过程调用的。目的是将堆的末端子节点作调整，使得子节点永远小于父节点 。
*/
func heapAdjust(list []int, parent int, length int) {
	temp := list[parent]  // temp保存当前父节点
	child := 2*parent + 1 // 先获得左孩子

	for child < length {
		// 如果有右孩子结点，并且右孩子结点的值大于左孩子结点，则选取右孩子结点
		if child+1 < length && list[child] < list[child+1] {
			child++
		}

		// 如果父结点的值已经大于孩子结点的值，则直接结束
		if temp >= list[child] {
			break
		}

		// 把孩子结点的值赋给父结点
		list[parent] = list[child]

		// 选取孩子结点的左孩子结点,继续向下筛选
		parent = child
		child = 2*child + 1
	}

	list[parent] = temp
}

func HeapSort(list []int) {
	// 循环建立初始堆
	for i := len(list) / 2; i >= 0; i-- {
		heapAdjust(list, i, len(list)-1)
	}

	// 进行n-1次循环，完成排序
	for i := len(list) - 1; i > 0; i-- {
		// 最后一个元素和第一元素进行交换
		list[0], list[i] = list[i], list[0]

		// 筛选 R[0] 结点，得到i-1个结点的堆
		heapAdjust(list, 0, i)
	}
}
//endregion
　　
</pre>
