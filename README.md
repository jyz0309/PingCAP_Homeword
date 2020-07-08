# PingCAP_Homework
PingCAP小作业



## 题目
### 有序数据块排序
定义‘数据块’为有序数组, 若干个数据块储存在单个 SSD 上，总大小 1TB, 请给出对这若干个数据块的全局排序算法，要求内存使用不超过 16G, 并在多核环境下的优化排序速度。
提示: 有序数据块可以当做是 int64 数组。

## 基本思路
由于1T数据过于庞大，所以本次采用了1kw数据进行模拟，并限制了输入输出缓冲区的大小都只能容纳1024个元素。

**使用数据：cmake-build-debug/data*.txt**

采用了**败者树+线程池**的方式

1. 将每个数据块读入指定数量的元素进入指定输入缓冲区
2. 通过每个数据块中的第一个元素，构造一颗败者树
3. 在败者树中找到胜者，加入result输出缓冲区中，并将胜者所在的缓冲区的下一位元素进入败者树中
4. 对败者树进行调整，相比遍历找到最小值来说，所需要进行的对比更少
5. 若输入缓冲区全部被读取完，则会通知线程池分配线程，对输入缓冲区进行填充
6. 当输出缓冲区到达指定数量后，会对输出缓冲区中的数据进行文件写入,并重新填充输出缓冲区

![image](https://github.com/jyz0309/PingCAP_Homework/raw/master/image/process.png)
### 线程池工作原理
1. 线程池中通过`threadpool_create`函数来初始化线程池,并创建线程放入数组中,并创建一个指定大小的任务队列
2. 当有任务需要让线程处理的时候,将任务放入任务队列中,通知线程池分配线程对该任务进行执行,如果此时线程池没有可分配的线程,则会在任务队列尾部进行等待
3. 形式上相当于是生产者消费者模式,其中主线程为生产者,负责生产任务放入任务队列,线程池中的线程相当于消费者,负责对任务队列中的任务进行执行
4. 通过锁和条件变量,来保证不会出现死锁,顺序不当的问题

### 败者树工作原理
1. 将每个输入缓冲区的首节点放入败者树的叶子节点中
2. 节点两两比较,较小的即为胜者,迭代直到根节点,记录当前最小值输出到输出缓冲区
3. 将最小值对应的输入缓冲区的下一个节点升上来叶子节点,由于其他的节点已经比较过,所以本次比较只需要自身与父节点比较即可,很大程度上减少了开销

## 结果

1kw的数据下进行排序，败者树所需要时间:2.26717s **(包含输入输出文件)**

每次找最小值都进行一次遍历的方式所需要的时间:2.79231s **(不包含输入输出文件)**

sort函数排序：5.7347s(不包含输入输出文件)

8kw的数据下进行排序，败者树所需要时间：20.192s

每次找最小值都进行一次遍历的方式所需要的时间:25.79231s **(不包含输入输出文件)**

sort函数排序：35s

可以看到，在这种有着有序数据块的归并排序下，败者树多路归并表现的性能是比sort()原生函数、每次找最小值都进行一次遍历的方式的效率好很多的。

## 各文件作用
```
Buffer_Node.h: 缓冲区对象的定义
Buffer_Node.cpp:缓冲区对象方法的逻辑代码
thread_pool.h: 线程池的定义
thread_pool.cpp: 线程池的相关函数逻辑代码
K_Merge.h: 败者树多路归并的函数定义
K_Merge.cpp: 多路归并逻辑代码
main.cpp: 主函数
关键变量说明:
DATA_NUM:数据块的数量
BUFFER_NODE_SIZE:缓冲区可容纳的元素数量
```

## 后续可优化的点

1. 目前的读取、写入文件依旧是使用最基本的方式，会有磁盘拷贝到内核态的过程的开销浪费，后续可以改进为零拷贝。
2. 目前只在控制输入文件处做了线程池的处理，后续可以把输出结果文件的IO也放入线程池的任务列表中进行处理。
3. 目前缺陷：有多少数据块，败者树就有多少个叶子节点，后续可以先用归并排序将数据块合并为指定数量，然后再进行败者树处理



