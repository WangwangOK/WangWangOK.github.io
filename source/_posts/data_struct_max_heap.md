---
title: 最大堆（创建、删除、插入和堆排序）
tags: 
- 算法
categories: 编程
---
  什么是最大堆和最小堆？最大（小）堆是指在树中，存在一个结点而且该结点有儿子结点，该结点的data域值都不小于（大于）其儿子结点的data域值，并且它是一个完全二叉树（不是满二叉树）。
<!-- more -->
  注意区分选择树，因为___选择树（selection tree）___概念和最小堆有些类似，他们都有一个特点是___“树中的根结点都表示树中的最小元素结点”___。同理最大堆的根结点是树中元素最大的。那么来看具体的看一下它长什么样？（最小堆这里省略）
![图-1](http://upload-images.jianshu.io/upload_images/619906-6866218c0158cfdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>这里需要注意的是：在多个子树中，并不是说其中一个子树的父结点一定大于另一个子树的儿子结点。最大堆是树结构，而且一定要是完全二叉树。

#### 最大堆ADT
那么我们在做最大堆的抽象数据类型（ADT）时就需要考虑三个操作：
（1）、创建一个最大堆；
（2）、最大堆的插入操作；
（3）、最大堆的删除操作；
最大堆ADT如下：

```
struct Max_Heap {
  object: 由多个元素组成的完全二叉树，其每个结点都不小于该结点的子结点关键字值
  functions:
    其中heap∈Max_Heap,n,max_size∈int,Element为堆中的元素类型，item∈ Element
    Max_Heap createHeap(max_size)       := 创建一个总容量不大于max_size的空堆
    void max_heap_insert(heap, item ,n) := 插入一个元素到heap中
    Element max_heap_delete(heap,n)     := if(heap不为空) {return 被删除的元素 }else{return NULL}
}
///其中:=符号组读作“定义为”
```

#### 最大堆内存表现形式
  我们只是简单的定义了最大堆的ADT，为了能够用代码实现它就必须要考虑最大堆的内存表现形式。从最大堆的定义中，我们知道不管是对最大堆做插入还是删除操作，__我们必须要保证插入或者删除完成之后，该二叉树仍然是一个完全二叉树__。基于此，我们就必须要去操作某一个结点的父结点。
  第一种方式，我们使用链表的方式来实现，那么我们需要添加一个额外的指针来指向该结点的父结点。此时就包括了左子结点指针、右子结点指针和父结点指针，那么空链的数目有可能是很大的，比如叶子结点的左右子结点指针和根结点的父结点指针，所以不选择这种实现方式（关于用链表实现一般二叉树时处理左右子结点指针的问题在线索二叉树中有提及）。
  第二种方式，使用数组实现，在二叉树进行遍历的方法分为：先序遍历、中序遍历、后序遍历和层序遍历。我们可以通过层序遍历的方式将二叉树结点存储在数组中，由于最大堆是完全二叉树不会存在数组的空间浪费。那么来看看层序遍历是怎么做的？对下图的最大堆进行层序遍历：
![](http://upload-images.jianshu.io/upload_images/619906-f1c0ab6c89bc68d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![层序遍历流程变化](http://upload-images.jianshu.io/upload_images/619906-4ef737815fce2175.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)从这里可以看出最后得到的顺序和上面图中所标的顺序是一样的。
  那么对于数组我们怎么操作父结点和左右子结点呢？对于完全二叉树采用顺序存储表示，那么对于任意一个下标为i(1 ≤ i ≤ n)的结点：
（1）、父结点为：___i / 2（i ≠ 1）___，若i = 1，则i是根节点。
（2）、左子结点：___2i（2i ≤ n）___， 若不满足则无左子结点。
（3）、右子结点：___2i + 1(2i + 1 ≤ n)___，若不满足则无右子结点。![](http://upload-images.jianshu.io/upload_images/619906-4fc0e1d7aa95f1de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>最终我们选择___数组___作为最大堆的内存表现形式。

基本定义:

```
#define MAX_ELEMENTS 20
#define HEAP_FULL(n) (MAX_ELEMENTS - 1 == n)
#define HEAP_EMPTY(n) (!n)
typedef struct {
    int key;
}element;
element heap[MAX_ELEMENTS];
```
下面来看看最大堆的插入、删除和创建这三个最基本的操作。
### 最大堆的插入
最大堆的插入操作可以简单看成是__“结点上浮”__。当我们在向最大堆中插入一个结点我们必须满足完全二叉树的标准，那么被插入结点的位置的是固定的。而且要满足父结点关键字值不小于子结点关键字值，那么我们就需要去移动父结点和子结点的相互位置关系。具体的位置变化，可以看看下面我画的一个简单的图。

```
void insert_max_heap(element item ,int *n){
    if(HEAP_FULL(*n)){
      return;
    }
    int i = ++(*n);
    for(;(i != 1) && (item.key > heap[i/2].key);i = i / 2){/// i ≠ 1是因为数组的第一个元素并没有保存堆结点
      heap[i] = heap[i/2];/// 这里其实和递归操作类似，就是去找父结点
    }
    heap[i] = item;
}
```
![](http://upload-images.jianshu.io/upload_images/619906-5ee33c128001bd5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由于堆是一棵完全二叉树，存在n个元素，那么他的高度为:___log2(n+1)___，这就说明代码中的for循环会执行___O(log2(n))___次。因此插入函数的时间复杂度为：___O(log2(n))___。

### 最大堆的删除

__最大堆的删除操作，总是从堆的根结点删除元素__。同样根元素被删除之后为了能够保证该树还是一个完全二叉树，我们需要来移动完全二叉树的最后一个结点，让其继续符合完全二叉树的定义，从这里可以看作是___最大堆最后一个结点的下沉___操作。例如在下面的最大堆中执行删除操作：
![](http://upload-images.jianshu.io/upload_images/619906-2b9aae16dedbf7fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
解答：
1）、对于最大堆的删除，我们不能自己进行选择删除某一个结点，我们只能删除堆的根结点。因此在图a中，我们删除根结点20；
2）、当删除根结点20之后明显不是一个完全二叉树，更确切地说被分成了两棵树。
3）、我们需要移动子树的某一个结点来充当该树的根节点，那么在(15,2,14,10,1)这些结点中移动哪一个呢？显然是移动结点1，如果移动了其他结点（比如14，10）就不再是一个完全二叉树了。
4）、此时在结点（15，2）中选择较大的一个和1做比较，即15 > 1的，所以15上浮到之前的20的结点处。
5）、同第4步类似，找出（14，10）之间较大的和1做比较，即14>1的，所以14上浮到原来15所处的结点。
6）、因为原来14的结点是叶子结点，所以将1放在原来14所处的结点处。
```
![](http://upload-images.jianshu.io/upload_images/619906-cd7ca08102c205b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这图中是用``temp``分别和图中的``max``做比较，来看``temp``是否会下沉

```
element delete_max_heap(int *n){
  int parent, child;
  element temp, item;
  temp = heap[(*n)--];
  item = heap[1];
  parent = 1,child=2;
  for(;child <= *n; child = child * 2){
   if( (child < *n) && heap[child].key < heap[child+1].key){/// 这一步是为了看当前结点是左子结点大还是右子结点大，然后选择较大的那个子结点
        child++;
      }
      if(temp.key >= heap[child].key){
        break;
      }
      heap[parent] = heap[child];///这就是上图中第二步和第三步中黄色部分操作
      parent = child;/// 这其实就是一个递归操作，让parent指向当前子树的根结点
   }
  heap[parent] = temp;
  return item;
}
```
同最大堆的插入操作类似，同样包含n个元素的最大堆，其高度为:___log2(n+1)___，其时间复杂度为：___O(log2(n))___。
> 总结：由此可以看出，在已经确定的最大堆中做删除操作，被删除的元素是固定的，需要被移动的结点也是固定的，这里我说的被移动的元素是指最初的移动，即最大堆的最后一个元素。移动方式为从最大的结点开始比较。

### 最大堆的创建
为什么要把最大堆的创建放在最后来讲？因为在堆的创建过程中，有两个方法。会分别用到最大堆的插入和最大堆的删除原理。创建最大堆有两种方法：
（1）、先创建一个空堆，然后根据元素一个一个去插入结点。由于插入操作的时间复杂度为___O(log2(n))___，那么n个元素插入进去，总的时间复杂度为___O(n * log2(n))___。
（2）、将这n个元素先顺序放入一个二叉树中形成一个完全二叉树，然后来调整各个结点的位置来满足最大堆的特性。
现在我们就来试一试第二种方法来创建一个最大堆：假如我们有12个元素分别为：

```
{79,66,43,83,30,87,38,55,91,72,49,9}
```
将上诉15个数字放入一个二叉树中，确切地说是放入一个完全二叉树中，如下：
![](http://upload-images.jianshu.io/upload_images/619906-f14895940e2c1ded.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但是这明显不符合最大堆的定义，所以我们需要让该完全二叉树转换成最大堆！怎么转换成一个最大堆呢？
  最大堆有一个特点就是__其各个子树都是一个最大堆__，那么我们就可以从把最小子树转换成一个最大堆，然后依次转换它的父节点对应的子树，直到最后的根节点所在的整个完全二叉树变成最大堆。那么从哪一个子树开始调整？
>我们从该完全二叉树中的最后一个非叶子节点为根节点的子树进行调整，然后依次去找倒数第二个倒数第三个非叶子节点...

#### 具体步骤
在做最大堆的创建具体步骤中，我们会用到最大堆删除操作中结点位置互换的原理，___即关键字值较小的结点会做下沉操作___。
- 1）、就如同上面所说找到二叉树中倒数第一个非叶子结点``87``，然后看以该非叶子结点为根结点的子树。查看该子树是否满足最大堆要求，很明显目前该子树满足最大堆，所以我们不需要移动结点。___该子树最大移动次数为1___。
![](http://upload-images.jianshu.io/upload_images/619906-dfabe32d4e8d1df1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 2）、现在来到结点``30``，明显该子树不满足最大堆。在该结点的子结点较大的为``72``，所以结点72和结点30进行位置互换。___该子树最大移动次数为1___。
![](http://upload-images.jianshu.io/upload_images/619906-15d2b9fc8dde0ddf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 3）、同样对结点``83``做类似的操作。___该子树最大移动次数为1___。
![](http://upload-images.jianshu.io/upload_images/619906-7405036f2377c8b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 4）、现在来到结点``43``，该结点的子结点有``{87,38,9}``，对该子树做同样操作。由于结点43可能是其子树结点中最小的，所以___该子树最大移动次数为2___。
![](http://upload-images.jianshu.io/upload_images/619906-ceeb433a5b997243.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 5）、结点``66``同样操作，___该子树最大移动次数为2___。
![](http://upload-images.jianshu.io/upload_images/619906-0a942422c933d117.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 6）、最后来到根结点``79``，该二叉树最高深度为4，所以___该子树最大移动次数为3___。
![](http://upload-images.jianshu.io/upload_images/619906-59de31f16ee7c2ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

自此通过上诉步骤创建的最大堆为:
![](http://upload-images.jianshu.io/upload_images/619906-c7887558467cf8c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>所以从上面可以看出，该二叉树总的需要移动结点次数最大为：__10__。

#### 代码实现
```
void create_max_heap(void){
    int total = (*heap).key;
    /// 求倒数第一个非叶子结点
    int child = 2,parent = 1;
    for (int node = total/2;node > 0; node--) {
        parent = node;
        child = 2*node;
        int max_node = 2*parent+1;
        element temp = *(heap + parent);
        for(;child <= max_node && max_node <= total; child = child * 2,max_node = 2*parent+1){
            if ((*(heap + child)).key < (*(heap + child + 1)).key) {/// 取右子结点
                child++;
            }
            if (temp.key > (*(heap + child)).key) {
                break;
            }
            *(heap + parent) = *(heap + child);
            parent = child;
        }
        *(heap + parent) = temp;
    }
}

/**
 *
 * @param heap  最大堆；
 * @param items 输入的数据源
 * @return 1成功，0失败
 */
int create_binary_tree(element *heap,int items[MAX_ELEMENTS]){
    int total;
    if (!items) {
        return 0;
    }
    element *temp = heap;
    heap++;
    for (total = 1; *items;total++,(heap)++,items = items + 1) {
        element ele = {*items};
        element temp_key = {total};
        *temp = temp_key;
        *heap = ele;
    }
    return 1;
}
///函数调用
int items[MAX_ELEMENTS] = {79,66,43,83,30,87,38,55,91,72,49,9};
element *position = heap;
create_binary_tree(position, items);
for (int i = 0; (*(heap+i)).key > 0; i++) {
  printf("binary tree element is %d\n",(*(heap + i)).key);
}
create_max_heap();
for (int i = 0; (*(heap+i)).key > 0; i++) {
  printf("heap element is %d\n",(*(heap + i)).key);
}
```
上诉代码在我机器上能够成功的构建一个最大堆。由于该完全二叉树存在n个元素，那么他的高度为:___log2(n+1)___，这就说明代码中的for循环会执行___O(log2(n))___次。因此其时间复杂度为：___O(log2(n))___。

### 堆排序
  堆排序要比空间复杂度为___O(n)___的归并排序要慢一些，但是要比空间复杂度为___O(1)___的归并排序要快！
  通过上面__最大堆创建__一节中我们能够创建一个最大堆。出于该最大堆太大，我将其进行缩小以便进行画图演示。
![](http://upload-images.jianshu.io/upload_images/619906-8b1f79e817875e10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最大堆的排序过程其实是和最大堆的删除操作类似，由于最大堆的删除只能在根结点进行，当将根结点删除完成之后，就是将剩下的结点进行整理让其符合最大堆的标准。
- 1）、把最大堆根结点``91``“删除”，第一次排序图示：
![](http://upload-images.jianshu.io/upload_images/619906-35b9c48c809f231b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
进过这一次排序之后，``91``就处在最终的正确位置上，所以我们只需要对余下的最大堆进行操作！这里需要注意一点：

>⚠️⚠️⚠️注意，关于对余下进行最大堆操作时：
并不需要像创建最大堆时，从倒数第一个非叶子结点开始。因为在我们只是对第一个和最后一个结点进行了交换，所以___只有根结点的顺序不满足最大堆的约束，我们只需要对第一个元素进行处理即可___

- 2）、继续对结点``87``进行相同的操作：
![](http://upload-images.jianshu.io/upload_images/619906-b14307ce0d5322e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同样，``87``的位置确定。

- 3）、现在我们来确定结点``83``的位置：
![](http://upload-images.jianshu.io/upload_images/619906-df936cc131e51cf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 4）、经过上诉步骤就不难理解堆排序的原理所在，最后排序结果如下：
![](http://upload-images.jianshu.io/upload_images/619906-9a2dbb64df3a5702.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

经过上诉多个步骤之后，最终的排序结果如下：

```
[38、43、72、79、83、87、91]
```
很明显这是一个正确的从小到大的顺序。

#### 编码实现
这里需要对上面的代码进行一些修改！因为在排序中，我们的第0个元素是不用去放一个哨兵的，我们的元素从原来的第一个位置改为从第0个位置开始放置元素。

```
void __swap(element *lhs,element *rhs){
    element temp = *lhs;
    *lhs = *rhs;
    *rhs = temp;
}

int create_binarytree(element *heap, int items[MAX_SIZE], int n){
    if (n <= 0) return 0;
    for (int i = 0; i < n; i++,heap++) {
        element value = {items[i]};
        *heap = value;
    }
    return 1;
}

void adapt_maxheap(element *heap ,int node ,int n){
    int parent = node - 1 < 0 ? 0 : node - 1;
    int child = 2 * parent + 1;/// 因为没有哨兵，所以在数组中的关系由原来的：parent = 2 * child => parent = 2 * child + 1
    int max_node = max_node = 2*parent+2 < n - 1 ? 2*parent+2 : n - 1;
    element temp = *(heap + parent);
    for (;child <= max_node; parent = child,child = child * 2 + 1,max_node = 2*parent+2 < n - 1 ? 2*parent+2 : n - 1) {
        if ((heap + child)->key <= (heap + child + 1)->key && child + 1 < n) {
            child++;
        }
        if ((heap + child)->key < temp.key) {
            break;
        }
        *(heap + parent) = *(heap + child);
    }
    *(heap + parent) = temp;
}

int create_maxheap(element *heap ,int n){

    for (int node = n/2; node > 0; node--) {
        adapt_maxheap(heap, node, n);
    }
    return 1;
}

void heap_sort(element *heap ,int n){
    ///创建一个最大堆
    create_maxheap(heap, n);
    ///进行排序过程
    int i = n - 1;
    while (i >= 0) {
        __swap(heap+0, heap + i);/// 将第一个和最后一个进行交换
        adapt_maxheap(heap, 0, i--);///将总的元素个数减一，适配成最大堆，这里只需要对首元素进行最大堆的操作
    }
}
```
调用：
```
/// 堆排序
int n = 7;
int items[7] = {87,79,38,83,72,43,91};
element heap[7];
create_binarytree(heap, items, n);
heap_sort(heap, n);
```
在实现堆排序时最需要注意的就是当没有哨兵之后，父结点和左右孩子结点之间的关系发生了变化：
```
parent = 2 * child + 1;///左孩子
parent = 2 * child + 2;///右孩子
```
关于对排序相关的知识点已经整理完了。其时间复杂度和归并排序的时间时间复杂度是一样的。

