---
layout: post
title: "QuickSort and Derivatives"
date: 2014-08-09 03:18
comments: true
categories: algorithm
tags: [algorithm, quicksort]
---

记录下快排相关的一些东西。包括普通快排，迭代快排和单链表快排。

----------

<!--more-->

## 普通快排 ##

``` c++
/* A typical recursive implementation of quick sort */

/* This function takes last element as pivot, places the pivot element at its correct position in sorted array, and places all smaller (smaller than pivot) to left of pivot and all greater elements to right of pivot */
int partition (int arr[], int l, int h) {
    int x = arr[h];
    int i = (l - 1);
 
    for (int j = l; j <= h- 1; j++) {
        if (arr[j] <= x) {
            i++;
            swap (arr[i], arr[j]);
        }
    }
    swap (arr[i + 1], arr[h]);
    return (i + 1);
}
 
/* A[] --> Array to be sorted, l  --> Starting index, h  --> Ending index */
void quickSort(int A[], int l, int h) {
    if (l < h){        
        int p = partition(A, l, h);
        quickSort(A, l, p - 1);  
        quickSort(A, p + 1, h);
    }
}
```

这种实现方式有很多可以值得改进的地方：

1. 上面的实现使用最后一个元素作为pivot，这对于已经排好序的数组来说是个灾难。可以随机选取一个元素或者直接用中位数来替代。
2. 为了减少递归层次，可以先递归数组中个数少的一半。
3. 对于小数组来说，插入排序可能更快。可以综合下插入和快排。
4. 使用了递归和函数调用栈来存储中间值，并且还需要存储其他的信息。此外还包括存储函数调用的活动记录，恢复上层函数执行的费用。

可以使用迭代来替代递归。

### 迭代快排 ###


``` c++
/* This function is same in both iterative and recursive*/
int partition (int arr[], int l, int h) {
    int x = arr[h]， i = (l - 1);
 
    for (int j = l; j <= h- 1; j++){
        if (arr[j] <= x){
            i++;
            swap (arr[i], arr[j]);
        }
    }
    swap (arr[i + 1], arr[h]);
    return (i + 1);
}

void quickSortIterative (int arr[], int l, int h) {
	// initialize top of stack
    int stack[ h - l + 1 ]; int top = -1;
 
    // push initial values of l and h to stack
    stack[ ++top ] = l;  stack[ ++top ] = h;
 
    // Keep popping from stack while is not empty
    while ( top >= 0 ){
        // Pop h and l
        h = stack[ top-- ];  l = stack[ top-- ];
 
        int p = partition( arr, l, h );

        if ( p-1 > l ) {
            stack[ ++top ] = l; stack[ ++top ] = p - 1;
        }
 
        if ( p+1 < h ) {
            stack[ ++top ] = p + 1; stack[ ++top ] = h;
        }
    }
}
```

## 文艺快排 ##

思路和数据的快速排序一样，都需要找到一个pivot元素、或者节点。然后将数组或者单向链表划分为两个部分，然后递归分别快排。

针对数组进行快排的时候，交换交换不同位置的数值，在分而治之完成之后，数据就是排序好的。那么单向链表是什么样的情况呢？除了交换节点值之外，是否有其他更好的方法呢？可以修改指针，不进行数值交换。这可以获取更高的效率。

在修改指针的过程中，会产生新的头指针以及尾指针，要记录下来。在partition之后，要将小于pivot的的部分、pivot、以及大于pivot的部分重新串起来成为一个singly linked list。

在partition时，我们用最后的节点作为pivot。当我们扫描链表时，如果节点值大于pivot，将节点移到尾部之后；如果节点小于，保持不变。

在递归排序时，我们先调用partition将pivot放到正确的为止并返回pivot，然后，递归左边，递归右边，最后在合成一个单链表。

``` c++
struct node *partition(struct node *head, struct node *end,  
                      struct node **newHead, struct node **newEnd)
{
   struct node *pivot = end;
   struct node *prev = NULL, *cur = head, *tail = pivot;

   while(cur != pivot)
   {
       if(cur->data < pivot->data)
       {
          if((*newHead) == NULL)
               (*newHead) = cur;
           prev = cur;  
           cur = cur->next;
       }
       else
       {
           if(prev)
               prev->next = cur->next;
           struct node *tmp = cur->next;
           cur->next = NULL;
           tail->next = cur;
           tail = cur;
           cur = tmp;
       }
   }
   if((*newHead) == NULL) (*newHead) = pivot;
   (*newEnd) = tail;
   return pivot;
}
struct node *quickSortRecur(struct node *head, struct node *end)
{
   if(!head || head == end)
       return head;
   struct node *newHead = NULL, *newEnd = NULL;
   struct node *pivot = partition(head, end, &newHead, &newEnd);
   if(newHead != pivot)
   {
      struct node *tmp = newHead;
       while(tmp->next != pivot)
           tmp = tmp->next;
       tmp->next = NULL;
       newHead = quickSortRecur(newHead, tmp);
       tmp = getTail(newHead);
       tmp->next =  pivot;
   }
   pivot->next = quickSortRecur(pivot->next, newEnd);
   returnn ewHead;
}
void quickSort(struct node **headRef)
{
   (*headRef) = quickSortRecur(*headRef, getTail(*headRef));
   return;
}
```