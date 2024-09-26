---
title: 算法复习_简单数据结构
date: 2024-09-26 21:30:17
tags: algorithms
categories: learning
description: 算法复习
keywords: 算法、数据结构
top_img: https://s2.loli.net/2024/03/14/WQxHO96fkU5RJSe.jpg
cover: 
comments: false
---
# 基本数据结构

> 本篇文章主要讲实现一些基本的数据结构，并且讲一些对其理解，以及最后怎么样用数组去实现这些数据结构。

## 链表

链表是使用指针实现的动态数据结构的最好和最简单的示例。对于链表来说，其对于数组(array)的优势在于：

1. 我们可以在一个列表的中间增加或者移除数据
2. 我们并不需要去预先定义这个列表的大小

同时，链表的缺点如下：

1. random access是不可能的，我们需要从头开始遍历链表
2. 我们需要动态分配以及使用指针，代码会变得复杂，可能会有内存泄漏以及segment fault
3. 链表的开销(overhead)要比数组大，因为动态分配的缘故，在内存的高效使用上不如数组，并且每一个在这种类型的list里面的item都需要一个额外的pointer

我们重新回顾一下关于链表的细节：链表就是一个node的set，每一个node都有一个value以及一个指向下一个node的指针。**如果某个pointer是null，那么这个node就是list的最后一个node。并且如果这个pointer是第一个node，那么这个list就是空的。**



下面是基本的链表功能实现，需要注意的如下：

1. **涉及对指针操作，比如删除头节点或者删除中间节点（特殊情况就是删除头节点）的时候，我们需要对指针操作，由于函数的性质，这里需要传入的是pointer to pointer，这样我们可以修改指针的值**；
2. **我们把这种node结构体中的指针next称为一种递归的manner，把node_t称为node type，这样子在我们定义的时候，我们可以直接使用node_t，而不用去写struct node；**
3. **实际上对于函数来说，修改的都是副本，但是为什么可以直接对next做指向修改？因为这里传入的也是指针的指针。**

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct node{
    int val;
    struct node* next;
}node_t;

void print_list(node_t *head){
    node_t * current = head;

    while(current != NULL){
        printf("%d\n", current->val);
        current = current->next;
    }
}

// 向链表的最后加入元素
void push(node_t* head, int val){
    node_t *current = head;
    while(current->next != NULL){
        current = current->next;
    }

    current->next = (node_t *)malloc(sizeof(node_t));
    current->next->val = val;
    current->next->next = NULL;

}

// 对于链表的最好的use case是队列和栈
// 向链表的最头加入元素
void push_back(node_t ** head, int val){
    // 由于是在链表的头部插入指针，意思是头部指针的指向会发生改变，要在函数中改变指针的指向，就要传入pointer to pointer
    node_t * new_node;
    new_node = (node_t *)malloc(sizeof(node_t));

    new_node->val = val;
    new_node->next = *head;
    *head = new_node;
}

// 去除第一个元素
// 想要去除第一个元素，就是上一个操作的逆向操作
int pop(node_t **head){
    int retval = -1;
    node_t * next_node = NULL;

    if(*head == NULL){
        return -1;
    }

    next_node = (*head)->next;
    retval = (*head)->val;
    free(*head);
    *head = next_node;

    return retval;
}

int remove_last(node_t *head){
    int retval = 0;
    // 如果这里只有一个元素，那么直接去除
    if(head->next == NULL){
        retval = head->val;
        free(head);
        return retval;
    }

    //一般情况下，我就要得到倒数第二个元素
    node_t *current = head;
    while(current->next->next == NULL){
        current = current->next;
    }

    retval = current->next->val;
    free(current->next);
    current->next = NULL;
    return  retval;
}

// remove a specific item

// 1.Iterate to the node before the node we wish to delete
// 2.Save the node we wish to delete in a temporary pointer
// 3.Set the previous node's next pointer to point to the node after the node we wish to delete
// 4.Delete the node using the temporary pointer
int remove_by_index(node_t** head, int n){
    int i=0;
    int retval = -1;
    node_t* current = *head;
    node_t* temp_pop = NULL;

    if(n==0){
        return pop(head); // 涉及到对于head指针的操作，在这种情况下最好是传入pointer to pointer
    }

    for(i=0; i<n-1; i++){
        if(current->next ==NULL){
            return -1;
        }
        current = current->next;
    }

    if(current->next==NULL){
        return -1;
    }

    temp_pop = current->next;
    current->next = temp_pop->next;
    retval = temp_pop->val;
    free(temp_pop);

    return retval;
}

// exercise: You must implement the function remove_by_value which receives a double pointer to the head and removes the first item in the list which has the value val.
int remove_by_value(node_t** head, int val){
    int retval = -1;
    node_t * current = *head;
    node_t * front_node = NULL;

    if(*head==NULL){
        return -1;
    }

    if(current->val == val){
        retval = val;
        pop(head);
        return retval;
    }

    while(current->val != val){
        if(current->next == NULL){
            return -1;
        }
        front_node = current;
        current = current->next;
    }

    front_node->next = current->next;
    retval = current->val;
    free(current);

    return retval;
}

int main(){
    node_t* head = NULL;
    head = (node_t *) malloc(sizeof(node_t));

    if(head == NULL){
        return 1;
    }

    head->val = 1;
    head->next = NULL;
    printf("hello world\n");
}
```

——><参考于https://www.learn-c.org/en/Linked_lists>

## 链表的数组实现：

> 我们在日常学习的过程中，老师教授给我们的都是以结构体的形式实现的链表，但是呢，比如我们要创建100000个结点，这样的话， 用结构体的话，时间太长，空间太大，反观数组，就显得很有优势

### 1.数组的优缺点

> **认识数组：**
>
> 数组是一种线性结构，存储的空间是内存连续的（物理连续），每当创建一个数组的时候，就必须先申请好一段指定大小的空间。（一次申请即可指定大小的空间）
>
> ***\*优点：\****
>
> 由于内存连续这一特征，数组的访问速度很快，直到索引下标之后，可以实现O(1)的时间复杂度的访问。
>
> **缺点：**
>
> 1.在任意位置删除和插入操作的时候，就会涉及到部分元素的移动，这样的话我们对于数组的任意位置的删除和插入的操作的时间复杂度为O(n)。
>
> 比如：
>
> 1>在i点后面插入数据，那么就需要i+1位置以及之后的元素，整体后移一位（for循环操作），然后再将插入的数据放在i+1的位置上
>
> 2>在i点之后删除元素，那么就需要，将i+1以及之后的元素，整体前移一位，总元素个数减一

以上是数组的优缺点，可以快速访问，达到O(1)，但是在任意删除和插入元素的时候，会耗时间，达到O(n)。

### 2.链表的优缺点

> ***\*认识链表\****
>
> 1.链表也是一种线性结构，但是他存储空间是不连续的（物理不连续，逻辑连续），链表的长度是不确定且支持动态扩展的。每次插入元素的时候，都需要进行申请新节点，然后赋值，插入链表中。
>
> 优点：
>
> 在插入数据或者删除数据的时候，只需要改变指针的指向即可，不需要类似于数组那样部分整体移动，整个过程不涉及元素的迁移，因此链表的插入和删除操作，时间复杂度为O(1)
>
> 缺点：
>
> 在查找任意位置的结点的数值域的时候，需要遍历，时间复杂度为O(n)

但是我们在任意位置插入或者删除元素的时候，需要查找这个指定的元素的结点位置，所以综合起来，链表的插入和删除仍为O(n)。

### 3.总结

无论数组还是链表，查找的时间复杂度都是O(n)，查找都要挨个遍历，直到找到满足的条件的数据为止，所以对于链表，如果没有给定，指针的地址，只是要插入删除第N位元素的时候，加上查找，综合起来时间复杂度为O(n)。

> 但是我们如果以数组的形式来实现链表，那么插入删除指定元素位置的时候，是不是就更加简便了呢，在第N位插入删除元素的时候，直接以O(1)的时间复杂度找到该位置结点，然后再由于链表的删除插入都是O(1)的，所以整个删除或插入操作，综合时间复杂度为O(1),比普通链表快很多。（注意这里指定第N个位置的元素，是真的第N个，但是如果要找匹配的元素，仍然是O(N)的复杂度）

### 4.实现

```c
#include <stdlib.h>
#include <stdio.h>

#define MAX_SIZE 10

// 数组模拟链表
int values[MAX_SIZE];       // 存储值的
int next[MAX_SIZE];         // 用于存储下一个节点的索引
int head = -1;              // 链表头部的索引，-1表示空链表
int free_index = 0;         // 下一个可用的空闲索引

void init(){
    for(int i=0;i<MAX_SIZE-1;i++){
        next[i] = i+1;      // 每个位置的“下一个”就是下一个可用的空闲空间（初始化的时候）,也有人直接写下一个空闲位置为 free_index ++
    }
    next[MAX_SIZE-1] = -1;  // 最后一个元素并没有下一个节点
    free_index = 0;         // 初始第一个空闲位置为0
}

void add(int value){
    if(free_index == -1){
        printf("链表已满，无法插入\n");
        return;
    }

    int new_node = free_index;      //获取空闲位置
    free_index = next[free_index];  //更新

    values[new_node] = value;
    next[new_node] = head;

    head = new_node;
}

// 从链表里面删除一个值为 value 的第一个节点
void remove_by_value(int value){
    if(head == -1){
        printf("链表为空\n");
        return;
    }

    int current = head;
    int prev = -1;

    while(current != -1 && values[current] != value){ // 这里current 为-1表示这里走到头了，因为最后一个元素的next的值为-1
        prev = current;
        current = next[current];
    }

    if(current == -1){
        printf("链表中未找到值为 %d 的节点\n", value);
        return;       
    }

    // 如果删除的是头节点
    if(prev == -1){
        head = next[head];
    }
    else{
        next[prev] = next[current];
    }

    //删除完要更新 free_index,这里核心宗旨是不改变next[free_index]，而改变free_index之前的指向，
    //也就是把current 当成当前的空闲，并且当前空闲的下一个空闲就是 free_index，最后更新 free_index为current
    next[current] = free_index;
    free_index = current;
}

void print_list(){
    int current = head;
    printf("链表: ");
    while(current != -1){
        printf("%d ->", values[current]);
        current = next[current];
    }
    printf("NULL\n");
}

int main(){
    init();

    add(10);
    add(20);
    add(30);
    print_list();

    remove_by_value(10);
    print_list();

    remove_by_value(20);
    print_list();

    remove_by_value(30);
    print_list();

    return 0;

}

```



## 链栈

栈是一种后进先出的数据结构，我们可以用链表来实现栈，这样的话，我们可以在栈的顶部插入和删除元素，这样的话，我们可以在O(1)的时间复杂度内完成这个操作。栈的实现方式有很多，比如数组，链表，这里首先介绍链表的实现方式。

![image-20240926201907962](https://s2.loli.net/2024/09/26/1OMJE8zYuNibKhq.png)

单链表可以在表头、表尾或者其他任意合法位置插入元素，如果只能在单链表的表尾插入和删除元素，那么就可以将其视为链栈。
因此，在单链表的基础上，我们再维护一个**top**指针即可。

```c
#include <stdlib.h>
#include <stdio.h>

// 链栈节点
typedef struct stack_node{
    struct stack_node * next;
    void *data;
}stack_node;

// 链栈体, top指向的栈节点的结构体，栈结构体还包括一个长度信息
typedef struct stack{
    struct stack_node* top;
    int length;
}stack;

// 栈函数体清单
// stack_create         O(1)
// stack_release        O(N)
// stack_push           O(1)
// stack_pop            O(1)
// stack_empty          O(N)

stack* stack_create(){
    stack* stack = (struct stack*)malloc(sizeof(struct stack));
    if(stack == NULL) return NULL;

    stack->length = 0;
    stack->top = NULL;

    return stack;
}

stack* stack_push(stack* stack, void * data){
    stack_node *node = (stack_node*)malloc(sizeof(stack_node));
    if(node == NULL){
        return NULL;
    }

    node->data = data;

    // 插入
    node->next = stack->top;
    stack->top = node;

    stack->length++;
    return stack;
}

void* stack_pop(stack* stack){
    // 出栈时，首先临时保存栈顶节点指针，用于在返回栈顶节点数据域的值之前的释放操作
    stack_node *curr = stack->top;
    if(curr == NULL){
        return NULL;
    }

    void *data = curr->data;
    stack->top = stack->top->next;

    free(curr);
    stack->length--;
    return data;
}

// 清空栈中的所有元素，但是不删除栈
void stack_empty(stack* stack){
    int length = stack->length;
    stack_node *curr, *next;
    curr = stack->top;

    while(length--){
        next = curr->next;
        free(curr);
        curr = next;
    }

    stack->length = 0;
    stack->top = NULL;

    // while(stack->length--){
    //     stack_pop(stack);
    // }
    // stack->top = NULL;
    // 也可以换成注释这块，但是效率会大大降低，因为每次进函数都会压栈什么的，并且会有很多额外的开销，比如更新stack的状态信息，这两种代码编写在编译运行的时候就能看出来很大的差别
}

void stack_release(stack *stack){
    stack_empty(stack);
    free(stack);
}

int main()
{
    char a = 'a';
    char b = 'b';
    char c = 'c';

    /* 创建一个栈 */
    stack *stack = stack_create();

    printf("%p\n", stack_pop(stack)); // %p 会输出指针的值，通常是一块内存地址，如果是空那么就输出nil或者0x0，取决于编译平台

    /* 压栈 */
    stack_push(stack, &a);
    stack_push(stack, &b);
    stack_push(stack, &c);
    /* 出栈 */
    while (stack->length > 0)
    {
        printf("%c\n", *(char *)stack_pop(stack));
    }

    /* 压栈 */
    stack_push(stack, &a);
    stack_empty(stack);
    printf("%p\n", stack_pop(stack));

    /* 释放栈 */
    stack_release(stack);
    return 0;
}
```