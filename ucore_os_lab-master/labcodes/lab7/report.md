# Lab 7

## 知识点

这些是与本实验有关的原理课的知识点：

* 禁用中断的同步方法
* 信号量：实现上与原理课的讲解有细微偏差
* 条件变量、管程：实现上与原理课讲解有很大不同
* 哲学家就餐问题

此外，本实验还涉及如下知识点：

无

遗憾的是，如下知识点在原理课中很重要，但本次实验没有很好的对应：

* 纯软件实现的同步方法
* 读者-写者问题
* 死锁
* 进程间通信

## 练习0

首先，需要对之前的代码进行微小的修改：在每一次时钟中断来临时改为调用`run_timer_list`，而不是调用`sched_class_proc_tick`。同时，恢复`sched_class_proc_tick`为`static`。

## 练习1

在实现信号量时，使用到了等待队列`wait_queue_t`。

等待队列实际上就是一个链表，存放了若干等待对象。等待对象`wait_t`实际上就是进程指针和一些标志组成的结构体。等待队列各种操作实际上就是对链表操作的包装。

### 初始化操作

一个信号量维护了其数值以及一个等待队列。

初始化时需要设置信号量的数值，以及初始化等待队列。

### down（P）操作

首先判断数值是否大于0，大于0的话则直接减少1，然后立即返回。否则，将当前进程放入等待队列并运行调度器，调度到其他进程。当再次被调度回来的时候说明进程被唤醒，若等待对象的标志未被修改则进一步说明获得了信号量。等待对象的标志发生了变化，则说明是其他原因唤醒的。无论如何，进程会将自己从等待队列中移除（若还在队列中），本次操作完成。

注意，整个操作过程中要关中断来保护共享变量不被意外修改。

### up（V）操作

这个操作和down操作在某种意义下是对偶的。

首先判断该信号量的等待队列是否为空，为空则将数值增加1然后立即返回。否则，说明有进程在等待此信号量，此时，选取等待队列队首的元素唤醒即可。

同样，整个操作过程中要关中断来保护共享变量不被意外修改。

从上面可以看出，这样的实现保持了信号量的数值大于等于0的性质，这与原理课不同，其他的实现也与原理课的讲解有很大区别。

为了给用户态提供信号量，一种可能的方法是直接复用内核线程的down、up操作的函数，把它们包装成系统调用供用户进程使用。此外，还要加入用户向内核申请、释放一个信号量对象的系统调用。需要注意的是，内核线程的down以及up操作使用信号量结构体的指针作为参数，但系统调用不能这样做，因为出于安全性考虑，用户进程不能持有或间接访问内核的指针。与Lab 8中的打开的文件类似，可为每一个进程维护一个信号量指针数组，由一个用户进程的多个线程共享，创建、down、up或释放操作时都使用数组的下标来进行访问。此外，内核还应做必要的参数检查。

## 练习2

这个练习提供的注释已经写得非常详细，比实际要写的代码还要多，根据它写出C代码即可。与参考答案对比，我的实现和参考答案十分一致。

这里的条件变量的实现基于信号量，经过分析，实现的是Hoare管程。

### 初始化操作

条件变量维护了一个等待者计数器和用于实现条件变量的信号量。条件变量所在的管程维护了一个互斥体`mutex`用于保护管程，以及一个`next`信号量和`next`计数器组合起来用于模拟一个进入管程的等待队列。

初始化时，将它们全部初始化。

### wait操作

首先，会将等待者计数器增加1，记录这个线程正在等待的事实。然后，会利用`next`信号量和`next`计数器释放管程控制权，同时唤醒被阻塞的signal进程，如果没有被阻塞的signal进程（`next`计数器为0），则释放互斥体。

接着，真正开始利用条件变量的信号量等待signal进程发来通知。收到通知之后，将等待者计数器恢复（即减少1），结束等待操作。

### signal操作

首先判断是否有等待者，若没有，则直接返回。否则，将`next`计数器增加1，表示当前线程会被等待者阻塞，然后利用条件变量的信号量完成对等待者的通知。之后利用`next`信号量等待重新获得管程控制权。最后，将`next`计数器恢复（即减少1），完成本次操作。

由于上面已经讨论了一个给用户态提供信号量的方法，要为用户态提供条件变量，可以直接把内核态条件变量的实现复制到用户态，只是将其中信号量的操作改为调用上述相应的系统调用。

### 能否不基于信号量机制来实现条件变量？

我认为可以直接使用等待队列实现，但是仍然需要用到某种锁机制，可以是互斥体，也可以是自旋锁。为简单，这里描述Mesa管程的设计。

#### 初始化操作

初始化锁和等待队列。

#### wait操作

工作流程设计如下：

1. 保存中断标志并关中断
2. 释放锁
3. 将当前进程设置为睡眠状态并放入等待队列
4. 恢复中断
5. 运行调度器，切换到其他进程，等待切换回来
6. 保存中断标志并关中断
7. 将当前进程从等待队列中移除
8. 恢复中断
9. 重新获得锁，此时可能再次睡眠

注意，1、2顺序不能够互换，否则释放锁之后，另外一个进程可能在当前进程被放入等待队列之前执行signal操作，当前进程无法收到，从而导致其一直在等待队列中睡眠。

#### signal操作

工作流程设计如下：

1. 保存中断标志并关中断
2. 若等待队列不为空，则选择队列中第一个进程唤醒
3. 恢复中断
4. 可以运行调度器，切换到其他进程，从而尝试获得Hoare管程的效果

从上面可以看出，条件变量的wait操作与信号量的down操作十分类似，条件变量的signal操作与信号量的up操作十分类似。区别在于，条件变量在signal时，若等待队列为空会忽略本次操作，而信号量up操作时，若队列为空会把计数器加一，从而让之后的down操作直接通过。同时，条件变量还需要维护一个锁。