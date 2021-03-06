Java的锁优化
高效并发是JDK1.6的一个重要主题，HotSpot虚拟机开发团队在这个版本上花费了大量的精力去实现各种锁优化技术，如适应性自旋（Adaptive Spinning）、锁消除（Lock Elimination）、锁粗化（Lock Coarsening）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）等，这些技术都是为了在线程之间更高效地共享数据，以及解决竞争问题，从面提高程序的执行效率。

自旋锁与自适自旋

我在Java中的线程安全与实现方法一文中讨论互斥同步的时候，提到了互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给系统的并发性能带来了很大的压力。
共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得。

如果物理机器有一个以上的处理器，能让两个或以上的线程同时并行执行，我们就可以让后面请求锁的那个线程‘稍等一会儿’，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁。
为了让线程等待，我们只须让线程执行一个忙循环（自旋），这项技术就是所谓的自旋锁。
自旋锁在JDK1.4.2中就已经引入，只不过默认是关闭的，可以使用-XX:+UseSpinning参数来开启，在JDK1.6中就已经改为默认开启了。

自旋等待不能代替阻塞，且先不说对处理器数量的要求，自旋等待本身虽然避免了线程切换的开销，但它是要占用处理器时间的，所以如果锁被占用的时间很短，自旋等待的效果就会非常好，反之如果锁被占用的时间很长，那么自旋线程只会白白消耗处理器资源，而不会做任何有用的工作，反而会带来性能的浪费。

因此，自旋等待的时间必须要有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程了。自旋次数的默认值是10次，用户可以使用参数-XX:PreBlockSpin来更改。



在JDK1.6中引入了自适应的自旋锁。自适应意味着自旋的时间不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在进行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将请允许自旋等待持续相对更长的时间，比如100个循环。另一方面，如果对于某个锁，自旋很少成功获得过，那在以后获取这个锁时将可能省略掉自旋过程，以避免浪费处理器资源。
有了自适应自旋，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测就会越来越准确，虚拟机就会变得越来越“聪明”了。

锁消除：
锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。
锁消除的主要判定依据来源于逃逸分析的数据支持，如果判断到一段代码中，在堆上的所有数据都不会逃逸出去被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。
变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是程序员自己应该是秀清楚的，怎么会在明知道不存在数据急用的情况下要求同步呢？
我们看一下下面源码：

public String concatString(String s1, String s2, String s3) {
  return s1 + s2 + s3;
}

这段代码不管是从字面上，还是程序语义上都没有同步。
我们也知道，由于String是一个不可变的类，对字符串的连接操作总是通过生成新的String对象来进行的，因此Java编译器会对String连接做自动优化。在JDK1.5之前，会转化为StringBuffer对象的连续append()操作，在JDK1.5及以后的版本中，会转化为StringBuilder对象的连续append()操作。即会变成如下代码：

public String concatString(String s1, String s2, String s3) {
  StringBuffer sb = new StringBuffer();
  sb.append(s1);
  sb.append(s2);
  sb.append(s3);
  return sb.toString();
}

每个StringBuffer.append()方法中都有一个同步块，锁就是sb对象。虚拟机观察变量sb，很快就会发现它的动态作用域被限制在concatString()方法的内部。也就是sb的所有引用永远不会“逃逸”到concatString()方法之外，其他线程无法访问到它，所以这里虽然有锁，但是可以被安全地消除掉，在即时编译之后，这段代码就会忽略掉所有的同步而直接执行了。

锁粗化
原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小---只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待锁的线程也能尽快地拿到锁。
大部分情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。

轻量级锁：
轻量级锁是JDK1.6中加入的新型锁机制，它名字中的“轻量级”是相对于使用操作系统互斥来实现的传统锁而言的，因此传统的锁机制就被称为“重量级”锁。首先需要强调一点的是，轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重要级锁使用操作系统互斥量产生的性能消耗。

要理解轻量级锁，必须从HotSpot虚拟机的对象（对象头部分）的内存布局开始介绍。
hotSpot虚拟机的对象头(Object Header)分为两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码(HashCode)、GC分代年龄(Generational GC Age)等，这部分数据的长度在32位和64位的虚拟机中分别为32个和64个Bits，官方称它为Mark Word，它是实现轻量级锁各偏向锁的关键。
另外一部分用于存储指向方法区对象类型数据的指针，如果是数组对象的话，还会有一个额外的部分用于存储数组长度。
对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。

以32位的HotSpot虚拟机中对象未被锁定的状态下，MarkWord的32个Bits空间中的25个用于存储对象哈希码(HashCode)，4Bits用于存储对象分代年龄，2Bits用于存储锁标块位，1Bits固定为0，在其他状态（轻量级锁定、重量级锁定、GC标记、可偏向）下对象的存储内容如下表：

标志位	存储内容	状态
01	对象哈希码、对象分代年龄	未锁定
00	指向锁记录的指针	轻量级锁定
10	指向重量级锁的指针	膨胀（重量级锁定）
11	空，不需要记录信息	GC标记
01	偏向线程ID、偏向时间戳、对象分代年龄	可偏向
在代码进入同步块的时候，如果此同步对象没有被锁定（锁标志为“01”状态），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word），这时候线程堆栈与对象头的状态如图1所示。然后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针。如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位(Mark Word的最后两个Bits)u将转变为“00”，即表示此对象处于轻量级锁定的状态，这时候线程堆栈与对象头的状态如图2所示。

图1 轻量级锁CAS操作之前堆栈与对象的状态

图2： 轻量级锁CAS操作之后堆栈与对象的状态
如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程抢占了。
如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥锁）的指针，后面等待锁的线程也要进入阻塞状态。

轻量级锁能提升程序同步性能的依据是“对于绝大部分锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用CAS操作避免了使用互斥量的开销，但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。

偏向锁
偏向锁也是JDK1.6中引入的一项锁优化，它的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，边CAS操作都不做了。
偏向锁的偏，就是偏心，偏袒。它的意思是这个锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。
假设当前虚拟机启用了偏向锁（启用参数-XX:+UseBiasedLocking，这是JDK1.6的默认值），那么，当锁对象第一次被线程获取的时候，虚拟机将会把对象逆流而上中的标志位设为“01”。即偏向模式。同时使用CAS操作把获取到这个锁的线程ID记录在对象的Mark Word之中，如果CAS操作成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作（例如Locking、Unlocking及对Mark Word的Update等）。

当有另外一个线程去尝试获取这个锁时，偏向模式就宣告结束。根据锁对象目前是否处于被锁定的状态，撤销偏向（Revoke Bias）后恢复到未锁定（标志位为“01”）或轻量级锁定（标志位为“00”）的状态。后续的同步操作就如上面介绍的轻量级锁那样执行。偏向锁、轻量级锁的状态转化及对象Mark Word的关系如下图所示。

偏向锁可以提高带有同步但无竞争的程序性能。它同样是一个带有效益权衡（Trade Off)性质的优化，也就是说它并不一定总是对程序运行有利，如果程序中大多数的锁总是被多个不同的线程访问，那偏向模式就是多余的。


偏向锁、轻量级锁的状态转化及对象Mark Word的关系


![image](https://github.com/liutengyuan1803/lean_repo/blob/master/images/1.jpg)
