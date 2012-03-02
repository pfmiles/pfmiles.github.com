---
layout: post
title: "[编程之美]寻找最大的k个数"
date: 2012-03-02 13:37
comments: true
categories: [algorithm, programming]
---
《编程之美》上的一道题： [wiki链接](http://bop1.wikispaces.com/%E7%AC%AC%E4%BA%8C%E7%AB%A0+%E6%95%B0%E5%AD%97%E4%B9%8B%E9%AD%85#x-2.5 寻找最大的K个数 ★★★);
其实在[July的文章里有更加深刻的分析](http://blog.csdn.net/v_JULY_v/article/details/6370650)，我这里只是用最小堆的方式实现一遍，体会和总结一下。

### 分析
* 这个问题，只要找到“最大的K个数”，但并不要求这K个数之间排序；
* 而且如果K很小的话，完全可以采用选择排序的方式，达到O(nk)的复杂度；
* 若K不小，则可利用这“只能容纳K个数的最小堆”的方法来做；时间复杂度是O(nlogk);用“堆”这种结构来解决此问题，很好地理解和利用了堆的特性:
    1. 找最大的K个数要使用最小堆，是因为最小堆总能最方便地剔除“最小的元素”，那么最后剩在堆中的都是最大的K个元素了；反之，若要找“最小的K个元素”则应该使用最大堆
    1. 二叉堆的同层节点之间是没有顺序的，这恰好符合我们的题目“不要求K个数之间排序”的需求，少了K个数之间的排序则能够减少不必要计算

### 程序
	#include <stdio.h>
    
    // defines the heap structure
	typedef struct {
		int size, last_pos;
		int* data;
	} Heap;

	void into_min_heap(int node, Heap* heap);
	void adjust_min_heap(Heap* heap, int cur_index);

	// find the max k digits
	void max_k(int len, int array[], int k, int rst[]) {
		// build min-heap on rst while iterating array:
		// worst time complexity: n * log k
		int i;
		Heap heap = { k, -1, rst };
		for (i = 0; i < len; i++) {
			into_min_heap(array[i], &heap);
		}
	}

	void into_min_heap(int node, Heap* heap) {
		if (heap->last_pos == -1) {
			heap->data[0] = node;
			heap->last_pos = 0;
		} else {
			if (node < heap->data[0]) {
				return;
			} else {
				if (heap->last_pos + 1 < heap->size) {
					heap->data[heap->last_pos + 1] = node;
					heap->last_pos++;
				} else {
					heap->data[0] = node;
					adjust_min_heap(heap, 0); // log k
				}
			}
		}
	}

	void adjust_min_heap(Heap* heap, int cur_index) {
		int c1_index = (cur_index + 1) * 2;
		int c2_index = (cur_index + 1) * 2 + 1;
		if (c1_index < heap->size) {
			if (heap->data[cur_index] <= heap->data[c1_index]) {
				if (c2_index < heap->size) {
					if (heap->data[cur_index] <= heap->data[c2_index]) {
						return;
					} else {
						int tmp = heap->data[cur_index];
						heap->data[cur_index] = heap->data[c2_index];
						heap->data[c2_index] = tmp;
						adjust_min_heap(heap, c2_index);
					}
				}
			} else {
				int tmp = heap->data[cur_index];
				heap->data[cur_index] = heap->data[c1_index];
				heap->data[c1_index] = tmp;
				adjust_min_heap(heap, c1_index);
			}
		}
	}

	int main(int args, char** argv) {
		int k = 3;
		int rst[k];
		int arr[10] = { 1, 20, -35, 4, 7, 11, 19, -5, 0, 18 };
		max_k(10, arr, k, rst);
		printf("The max k numbers are: ");
		int i;
		for (i = 0; i < k; i++) {
			printf("%d ", rst[i]);
		}
		return 0;
	}
### 总结
还是要理解各种常用数据结构的特性及其意义，遇到特定的问题时才好能够正确地选择出最能适合需求的这种结构。
