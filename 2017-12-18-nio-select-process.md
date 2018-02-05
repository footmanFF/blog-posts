---
layout: post
title: NIO select process
date: 2017-12-18
tags: Nio, select
---

#### 《Java Nio》4.3.1 节 selection process 阅读的笔记：

Java NIO 比较系统的资料就只能找到英文版的这本书了，对应的中文版翻译实在是太差，只能看英文，这一节的内容比较重要，看了好几遍才看明白，特别记录下来。

selector 包含维护着三种 SelectionKey：

| key类型              | 定义                                       |
| ------------------ | ---------------------------------------- |
| registered key set | 已注册的 SelectionKey                        |
| selected key set   | 已经就绪的 SelectionKey                       |
| cancelled key set  | registered key set 中，已经调用了 cancel() 但是还没有去除注册关系的 selectionKey |

<!-- more -->

selector 的 select 方法被调用时，都会做下面这些事情：

1. > The cancelled key set is checked. If it's nonempty, each key in the cancelled set is removed from the other two sets, and the channel associated with the cancelled key is deregistered. When this step is complete, the cancelled key set is empty.

   cancelled key 如果为非空，其中的每个 key 如果存在在其他两个 key 集合中的话，那么这些 key 都会被删除，而且相关的 channel 会被注销。最终 cancelled key 会为空。

2. > The operation interest sets of each key in the registered key set are examined.Changes made to the interest sets after they've been examined in this step will not be seen during the remainder of the selection operation.

   检查每个注册的 key 的关注的操作集，在这个检查之后如果有对 key 关注的操作集进行修改，那么这些修改不会影响后续的其他选择操作（selection operation）。

   后一点好理解，如果 selector 的 select 操作在执行中，其中注册的 key 关注的操作集被随意修改，那就没办法最终确定到底哪些 key 被「选中」了。

   > Once readiness criteria have been determined, the underlying operating system is queried to determine the actual readiness state of each channel for its operations of interest. Depending on the specific select() method called, the thread may block at this point if no channels are currently ready, possibly with a timeout value.

   一旦 readiness criteria 确定下来，底层系统就去查询，以确定每个 channel 对关注操作集的实际就绪状态。根据调用的 select 方法是不是带超时时间参数，调用线程会阻塞一段时间。

   这段简单来说就是让系统去确定到底哪些 key 是可以被「选中」了。

   > Upon completion of the system calls, which may have caused the invoking thread to be put to sleep for a while, the current readiness status of each channel will have been determined. Nothing further happens to any channel not found to be currently ready. For each channel that the operating system indicates is ready for at least one of the operations in its interest set, one of the following two things happens:

   系统调用完成后，可能导致调用线程休眠一段时间，每个通道的当前就绪状态将被确定下来。线程休眠一段时间应该是调用线程在等待系统调用完成，所以才 sleep。

   这次系统调完成后，不会有更多的当前是就绪状态的通道被「感知」到。结合前一句，如果系统调用只调一次，那么新的就绪状态不被感知到也很正常，这句可能是为了突出「只调一次系统调用，不再管新就绪的 channel」。

   系统调用后，每个通道的关注操作集中至少有一个操作是就绪的，这时候会分两种情况。（这一段比较好理解）然后是 a 和 b 两种情况。

   这时候因为完成了系统调用，已经确定了哪些通道是「就绪」了。再回想上面 selector 维护的三种 key 集合，那么需要针对这些「就绪」了的通道，去维护对应的 key 集合了。这些 key 集合才是让 selector 的外部能知道到底哪些 key 是当前就绪的，这些 key 是窗口。

   如果一个 channel 就绪了，那么就要把他对应的 SelectionKey 添加到 selected key set 中去。一个 SelectionKey 会维护着关注的操作（就绪的 channel 操作），如果一个通道不同的就绪操作被系统调用返回，那么势必要维护这个「关注操作」集合，这个集合是在 SelectionKeyImpl 类中，用一个整形数字维护的，他的一个二进制位表示一种操作是否就绪，1代表就绪，0 反之。代码如下：

   ```java
   public class SelectionKeyImpl extends AbstractSelectionKey{
       private int readyOps;
   }
   ```

   维护这个「关注操作」集合的逻辑就是下面的 a b 两段，其实这个时候想想就能大概知道他怎么做的了：

   > - a. If the key for the channel is not already in the selected key set, the key's ready set is cleared, and the bits representing the operations determined to be currently ready on the channel are set.

   如果 channel 对应的 SelectionKey 不在 selected key set 中，SelectionKey 的就绪操作集（ready set）会被清空，然后当前被系统调用确定为就绪的操作被设置到就绪操作集中去。

   > - b. Otherwise, the key is already in the selected key set. The key's ready set is updated by setting bits representing the operations found to be currently ready. Any previously set bits representing operations that are no longer ready are not cleared. In fact, no bits are cleared. The ready set as determined by the operating system is bitwise-disjoined into the previous ready set. [2] Once a key has been placed in the selected key set of the selector, its ready set is cumulative. Bits are set but never cleared.
   >
   >   [2] A fancy way of saying the bits are logically ORed together.

   如果 channel 对应的 SelectionKey **已经**在 selected key set 中，那么就去用系统调用确定为就绪的操作更新就绪操作集中对应的操作。有点绕啊 … 

   但是其他被设置成「没有就绪」的标识不会被清理。由系统调用确定的就绪集按位分散更新到先前的就绪集中去。一旦一个 SelectionKey 被放入 selector 的 selected key set 中，其就绪集就是累积的（不会做清理）。

   一个奇特的方式说这些位是逻辑「或」在一起。

   最后一句是真谛，每次有新的就绪的 readyOp 进来，就拿这个 readyOp 去和 readyOps 执行按位或，最终设置一个 channel 上所有就绪的操作。

3. > Step 2 can potentially take a long time, especially if the invoking thread sleeps.
   >
   > Keys associated with this selector could have been cancelled in the meantime. When Step 2 completes, the actions taken in Step 1 are repeated to complete deregistration of any channels whose keys were cancelled while the selection operation was in progress.

   第二步可能很耗时，特别当调用线程执行 sleep。这个 sleep 应该是前面提到的，在执行系统调用的时候，线程会等待他的完成而 sleep，**但是不太确定**。

   当前 selector 的这个 SelectionKey 可能已经在任何时候被 cancel。当第二步完成，第一步执行的操作会被重复执行，为了在 selector 的 select 方法在执行中时，有些 channel 的 selectionKey 被取消（cancel）以后去注销这部分 channel。

4. > The value returned by the select operation is the number of keys whose operation ready sets were modified in Step 2, not the total number of channels in the selection key set. The return value is not a count of ready channels, but the number of channels that became ready since the last invocation of select(). A channel ready on a previous call and still ready on this call won't be counted, nor will a channel that was ready on a previous call but is no longer ready. These channels could still be in the selection key set but will not be counted in the return value. The return value could be 0.

   select 方法返回的 int 是第二步中收集到的被更新的就绪操作集（operation ready set），而不是 channel 的数量。返回的 int 不是当前就绪的渠道数量，而是自上次调用 select 方法以后，就绪的渠道数量。如果一个渠道在上次调用中是就绪，在这一次也是就绪的，那么这个渠道不会被最终计入到返回的 int 中去。如果一个渠道在前一次调用时就绪，在这次调用不就绪，当然也不会被计入。这些 channel 任然在当前的 selection key 中，但是不会被计入返回的 int 中。返回的 int 是可能为 0 的。


> Using the internal cancelled key set to defer deregistration is an optimization to prevent threads from blocking when they cancel a key and to prevent collisions with in-progress selection operations. Deregistering a channel is a potentially expensive operation that may require deallocation of resources (remember that keys are channel-specific and may have complex interactions with their associated channel objects). Cleaning up cancelled keys and deregistering channels immediately before or after a selection operation eliminates the potentially thorny problem of deregistering channels while they're in the middle of selection. This is another good example of compromise in favor of robustness.

使用内部的 cancelled key set 推迟 channel 的注销是一种优化措施，以防止线程在取消一个 selectionKey 时被阻塞，并防止与正在进行的 select 操作发生冲突。这里说的应该是前面步骤一和步骤三。试想如果不用 cancelled key set 去记录已经被取消的 selectionKey，那么 cancel 调用和正常 select 调用一定会发生冲突，两者都关心到底哪些 selectionKey 被注册进 selector，然后就只能加锁，加锁明显会降低性能。有了 cancelled key set 以后（零时称他为 ckset），任何时候调用 selectionKey 的 cancel，对应的 key 都会被收集到 ckset，那么在执行真正的 select process 之前和之后（分别是步骤一和步骤二）去遍历这个集合并删除或者注销对应的 channel，那么就能在不加锁的情况下满足 select 和 cancel 不冲突，可以并行跑。

#### 后续的看下 4.3.3 的一段

> The secret to using selectors properly is to understand the role of the selected key set maintained by the selector. (See Section 4.3.1, specifically Step 2 of the selection process.) The important part is what happens when a key is not already in the selected set. When at least one operation of interest becomes ready on the channel, the ready set of the key is cleared, and the currently ready operations are added to the ready set. The key is then added to the selected key set

比较重要的其实是 4.3.1 的 步骤二，当一个 selectionKey 不在 selected set 中，但是又来了一个就绪的操作，selectionKey 的 ready set 会被清空，当前新来的就绪操作会被加进这个 ready set。当前的这个 selectionKey 随后会被加进 selected key set 中去。

**那么为什么这个步骤二重要啊？？**

猜测应该是这样的，selectionKey 如果在 registered key set 中，但是没在 selected set 中，并且在前一次的 select 操作中，这个 selectionKey 是被触发过的（select 调用中有就绪的操作）。那么当前的这次 select 调用发生时，其中的步骤二开始时候，他内部的 ready operation set 可能是非空，这时候就必须要去清空他，因为这里面的就绪操作其实是前一次 select 调用的就绪的操作。

接着一段：

> The way to clear the ready set of a SelectionKey is to remove the key itself from the set of selected keys. The ready set of a selection key is modified only by the Selector object during a selection operation. The idea is that only keys in the selected set are considered to have legitimate readiness information. That information persists in the key until the key is removed from the selected key set, which indicates to the selector that you have seen and dealt with it. The next time something of interest happens on the channel, the key will be set to reflect the state of the channel at that point and once again be added to the selected key set.

SelectionKey 内部的 ready set 是不能被除了 selector 以外的对象修改的。清除这个 ready set（或者说接收 ready set 并针对做出处理，然后清理调）只能通过将 SelectionKey 从 selected key set 中删除。ready set 只作为信息发布出去，表示「喏，你看，就这些就绪的操作」下次再有新的操作集合，只会在全新创建的 ready set 上，而不会在上一次的 ready set 上做修改，简单来说当 select 调用结束以后，ready set 就是不可变的了，知道下次 select 调用执行时。这里可以和上面的猜测作印证。美滋滋 ~

