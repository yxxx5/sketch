## libuv源码解析（一）libuv中的数据结构

对libuv这个库一直比较感兴趣，最近在组织公司前端公共体系的搭建，前端分享和培训是其中一部分。Node.js也是很多前端同学关注的话题，所以会有一系列的Node方向的分享讨论，libuv作为Node.js的底层依赖库还是有必要了解一下的，这组系列文章就是从源码的角度来分析libuv的具体实现。

_相关数据结构的使用示例会补在同目录下_

### libuv中的数据结构

这篇文章中将主要介绍libuv中依赖的三个数据结构，并简单分析他们的应用场景。

#### 头文件 heap-inl.h
在这里实现了一个二叉最小堆。

	二叉堆是一种特殊的堆，是完全二叉树或近似完全二叉树，二叉堆有两种：最大堆和最小堆。 
	
	最大堆：父结点的键值总是大于或等于任何一个子节点的键值。
	
	最小堆：父结点的键值总是晓宇或等于任何一个子节点的键值。
	
#### 具体实现

HEAP_EXPORT宏 用来定义全局静态方法。

	# define HEAP_EXPORT(declaration) static declaration

两个结构体：

	struct heap_node {
	  struct heap_node* left;
	  struct heap_node* right;
	  struct heap_node* parent;
	};
	
	struct heap {
  		struct heap_node* min;
  		unsigned int nelts;
	};
	
heap_node定义了子节点的结构，left right parent分别指向左、右和父节点；
heap中heap->min是二叉堆的root节点，heap->nelts是节点总数。

全局方法：

##### heap_init
heap_init 以 struct heap* heap 为参数，初始化为：
	
	heap->min = NULL;
	heap->nelts = 0;
	
##### heap_min
heap_min的实现很简单，以struct heap* heap 为参数，返回其min(root)节点：

	HEAP_EXPORT(struct heap_node* heap_min(const struct heap* heap)) {
  		return heap->min;
	}
##### heap_insert
heap_insert方法定义：

	void heap_insert(
			struct heap* heap,
          struct heap_node* newnode,
          heap_compare_fn less_than)
          
heap为insert目标heap；newnode是待insert的新节点。
less_than的类型为heap_compare_fn，我们先看heap_compare_fn的类型定义：


	typedef int (*heap_compare_fn)(
				const struct heap_node* a,
              const struct heap_node* b);

heap_compare_fn是一个 参数为两个heap_node指针，返回值类型为int 的 函数指针。这里heap_compare_fn less_than 通常返回0或1，在heap_insert内部通过less_than(newnode, newnode->parent)的值来判断是否需要和parent节点 swap。具体heap_compare_fn less_than可以通过不同场景来实现具体逻辑，这也是因为struct heap 和 struct heap_node并不会具体描述节点的值。

heap_insert方法目标是实现一个新节点的插入，这里主要有两部分：

1.将newnode插入到尽量左侧的底层空闲位置：
	
	struct heap_node** parent;
  	struct heap_node** child;
  	unsigned int path;
  	unsigned int n;
  	unsigned int k;

  	newnode->left = NULL;
  	newnode->right = NULL;
  	newnode->parent = NULL;
	path = 0;
  	for (k = 0, n = 1 + heap->nelts; n >= 2; k += 1, n /= 2)
    	path = (path << 1) | (n & 1);


  	parent = child = &heap->min;
  	while (k > 0) {
   		parent = child;
    	if (path & 1)
      		child = &(*child)->right;
    	else
      		child = &(*child)->left;
    	path >>= 1;
    	k -= 1;
  	}
  	newnode->parent = *parent;
  	*child = newnode;
  	heap->nelts += 1;
  	
第一步算法的实现还是挺有意思的，n = 1 + heap->nelts为底层待插入位置的索引;    

n >= 2 表示跳过root节点，n /= 2为向上路径上的父结点索引 n为偶数是左节点 奇数为右节点；  

k += 1 k表示path的有效位数  

path = (path << 1) | (n & 1);这里的实现是通过2进制0，1来记录从root的节点到插入节点索引的路径，0为左节点 1为右节点。  

举例：如果path的二进制表示为101，则表示 从root到带插入节点的路径为：  
root -> right -> left -> right  
待插入索引为13  
接着通过遍历path的有效位k 确定索引为1 + heap->nelts的newnode的parent


2.向上遍历heap 通过lessthan判断 是否需要向上swap：

	while (newnode->parent != NULL && less_than(newnode, newnode->parent))
    heap_node_swap(heap, newnode->parent, newnode);

##### heap node_swap

	static void heap_node_swap(struct heap* heap,
                           struct heap_node* parent,
                           struct heap_node* child)
                           
heap_node_swap实现 在heap中 交换parent父 child子 两个节点：

	static void heap_node_swap(struct heap* heap,
                           struct heap_node* parent,
                           struct heap_node* child) {
    /*兄弟节点*/
  	struct heap_node* sibling;
  	struct heap_node t;
	/*交换两个指针指向的 数据 阶段一*/
  	t = *parent;
  	*parent = *child;
  	*child = t;
  	
	/* 阶段二 */
  	parent->parent = child;
  	if (child->left == child) {
    	child->left = parent;
    	sibling = child->right;
  	} else {
    	child->right = parent;
    	sibling = child->left;
  	}
  	
  	/* 阶段三 */
  	if (sibling != NULL)
    	sibling->parent = child;

  	if (parent->left != NULL)
    	parent->left->parent = parent;
  	if (parent->right != NULL)
    	parent->right->parent = parent;

  	if (child->parent == NULL)
    	heap->min = child;
  	else if (child->parent->left == parent)
    	child->parent->left = child;
  	else
    	child->parent->right = child;
}

需要补图...

##### heap_remove


##### heap_dequeue

#### 头文件 queue.h
queue.h中定义了一个双向链表 链表的类型定义如下：
	
	/*包含两个指针元素（void*）的数组*/
	typedef void *QUEUE[2];
	
整个双向列表的方法都是由宏实现.

##### 4个 Private macros：
 	
 	QUEUE_NEXT(q) 返回q的后继
 	QUEUE_PREV(q) 返回q的前驱
 	QUEUE_PREV_NEXT(q) 返回q前驱的后继
 	QUEUE_NEXT_PREV(q) 返回q后继的前驱
 	
具体实现：

	#define QUEUE_NEXT(q)       (*(QUEUE **) &((*(q))[0]))
	/**/
	#define QUEUE_PREV(q)       (*(QUEUE **) &((*(q))[1]))
	/**/
	#define QUEUE_PREV_NEXT(q)  (QUEUE_NEXT(QUEUE_PREV(q)))
	#define QUEUE_NEXT_PREV(q)  (QUEUE_PREV(QUEUE_NEXT(q)))


参数q为指向QUEUE的指针 QUEUE* 
返回为 QUEUE*

QUEUE_PREV_NEXT和QUEUE_NEXT_PREV是基于QUEUE_NEXT、QUEUE_PREV实现，QUEUE_NEXT和QUEUE_PREV实现的这么复杂，主要有两方面考虑：

+ 因为后面有 QUEUE_NEXT(q) = (q); 这样的赋值操作，所以需要转成 左值
+ 为了保留类型信息

关于C的左值和右值的区别，本质在于是否能够获取内存地址，也就是能够做取地址操作。
所以：

(QUEUE *) （(*(q))[0]） 
类型QUEUE* 是rvalue不能做取地址操作.

##### 11个 Public macros:


我们通过写例子来了解这些Public macros的用途：

	#include <stdio.h>
	#include "queue.h"



	struct handle_t {
    	int val;
    	QUEUE node;
	};

	void handle_init(struct handle_t* handle_p, int val);

	static QUEUE* q;

	static QUEUE queue;

	static QUEUE queue1;

	static QUEUE queue2;


	int main() {
    	struct handle_t* handle_p;
		/*QUEUE_INIT(q) 初始化链表*/
    	QUEUE_INIT(&queue);
    	QUEUE_INIT(&queue1);
    	QUEUE_INIT(&queue2);

    	struct handle_t handle1;
    	handle1.val = 1;
    	handle_init(&handle1, 1);


    	struct handle_t handle2;
    	handle_init(&handle2, 2);

    	struct handle_t handle3;
    	handle_init(&handle3, 3);

    	struct handle_t handle4;
    	handle_init(&handle4, 4);

    	struct handle_t handle5;
    	handle_init(&handle5, 5);

    	struct handle_t handle6;
    	handle_init(&handle6, 6);

		//从尾部插入节点
    	QUEUE_INSERT_TAIL(&queue, &handle1.node);
    	QUEUE_INSERT_TAIL(&queue, &handle2.node);
    	QUEUE_INSERT_TAIL(&queue, &handle3.node);
		//从头部插入节点
    	QUEUE_INSERT_HEAD(&queue1, &handle4.node);
    	QUEUE_INSERT_HEAD(&queue1, &handle5.node);
    	QUEUE_INSERT_HEAD(&queue1, &handle6.node);


    	/*
     	*  QUEUE_ADD(h, n) 链接 h和n 两个链表
     	* */
    	QUEUE_ADD(&queue, &queue1);

    	/*   QUEUE_FOREACH(q, h)用于遍历队列 打印结果为：
     	*   这里q初始值为QUEUE_HEAD(h) 通过QUEUE_NEXT(q) 遍历链表直到（q）== (h)停止
     	*
     	*   通过QUEUE_DATA取到我们真正需要的数据结构，取到val值 具体实现会在后面分析，打印结果为
     	*
     	*   1
     	*   2
     	*   3
     	*   6
     	*   5
     	*   4
     	*
     	* */
    	QUEUE_FOREACH(q, &queue) {
        	handle_p = QUEUE_DATA(q, struct handle_t, node);
        	printf("%d\n", handle_p->val);
    	};
    	printf("################\n");



    	/* QUEUE_HEAD(h)的内部实现就是调用QUEUE_NEXT(h)
     	*
     	* */
    	q = QUEUE_HEAD(&queue);
    	/* REMOVE q 的操作 其实就是链接 q的prev和q的next:
     	*
     	*   QUEUE_PREV_NEXT(q) = QUEUE_NEXT(q);                                       \
         QUEUE_NEXT_PREV(q) = QUEUE_PREV(q);
     	*
     	* */
    	QUEUE_REMOVE(q);
    	/*
     	*  打印：
     	*  2
     	*  3
     	*  6
     	*  5
     	*  4
     	*
     	* */
    	QUEUE_FOREACH(q, &queue) {
       	handle_p = QUEUE_DATA(q, struct handle_t, node);
        	printf("%d\n", handle_p->val);
    	};
    	printf("################\n");


    	/*  QUEUE_MOVE(h, n) 内部通过QUEUE_SPLIT实现，将链表h 移动到 n
     	*
     	*  QUEUE_MOVE(&queue, &queue2); 等于下面QUEUE_SPLIT的写法
     	*  QUEUE_SPLIT(&queue, QUEUE_HEAD(&queue), &queue2);
     	*
     	* */
    	//QUEUE_MOVE(&queue, &queue2);

    	/*   QUEUE_SPLIT(h, q, n) 以q将链表h分割为 h 和 n两部分
     	*
     	* */
    	QUEUE_SPLIT(&queue, &handle5.node, &queue2);


    	/*
     	*  5
     	*  4
     	* */
    	QUEUE_FOREACH(q, &queue2) {
        	handle_p = QUEUE_DATA(q, struct handle_t, node);
        	printf("%d\n", handle_p->val);
    	};
    	printf("################\n");

    	/*
     	*  2
     	*  3
     	*  6
     	* */
    	QUEUE_FOREACH(q, &queue) {
        	handle_p = QUEUE_DATA(q, struct handle_t, node);
        	printf("%d\n", handle_p->val);
    	};
    	printf("################\n");


    	return 0;
	}

	void handle_init(struct handle_t* handle_p, int val) {
    	handle_p->val = val;
    	QUEUE_INIT(&handle_p->node);
	}
	
通过这个例子对queue.h的使用和源码实现有了基本了解，最后看一下 QUEUE_DATA(ptr, type, field)的实现：
	
	#define QUEUE_DATA(ptr, type, field)                                          \
  	((type *) ((char *) (ptr) - offsetof(type, field)))
  	
  	
熟悉linux内核编程的同学会经常见到宏函数container_of(ptr, type, member),QUEUE_DATA和container_of的实现和作用基本一致：

	通过结构体中一个成员的地址找到这个结构体的首地址，得到这个结构体的指针。
	
这里用到了offset宏 定义在stddef.h头文件中，用于得到结构体中一个成员相对于结构体起始字节的偏移量；通过 （char*）(ptr) 得到了ptr的首地址，ptr首地址 减去 偏移量 就能得到结构体的首地址。
	
以下面的例子分析:

	#include "queue.h"

	struct handle_t {
    	int val;
    	QUEUE node;
	};


	int main (){
    	struct handle_t handle;

    	handle.val = 1;
    	QUEUE_INIT(&handle.node);
		/*
			QUEUE_DATA找到了 通过成员&handle.node 找到了结构体数据handle的地址
			并将其转为 handle_t*
		*/
    	struct handle_t* handle_p = QUEUE_DATA(&handle.node, struct handle_t, node);
    	
		//1
    	printf("%d\n", handle_p->val);
	}
	

#### 头文件 tree.h
这里定义了伸展树和红黑树，在libuv里只使用了 红黑树。

	

