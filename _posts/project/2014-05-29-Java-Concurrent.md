---
layout: post
title : 	逐个击破之--Java Concurrent包
description : Java.util.concurrent包深入分析和介绍。
category : project
---

##背景

> 　　java.util.concurrent 包含许多线程安全、测试良好、高性能的并发构建块。可以说：**创建 java.util.concurrent 的目的就是要实现 Collection 框架对数据结构所执行的并发操作**。通过提供一组可靠的、高性能并发构建块，开发人员可以提高并发类的线程安全、可伸缩性、性能、可读性和可靠性。  
> 　　感谢Java并发大师：**Doug Lea**!


##1. 并发介绍
JDK 5.0 中的并发改进可以分为三组：  

* **JVM 级别更改** : 大多数现代处理器对并发对某一硬件级别提供支持，通常以 compare-and-swap （CAS）指令形式。CAS 是一种低级别的、细粒度的技术，它允许多个线程更新一个内存位置，同时能够检测其他线程的冲突并进行恢复。它是许多高性能并发算法的基础。在 JDK 5.0 之前，Java 语言中用于协调线程之间的访问的惟一原语是同步，同步是更重量级和粗粒度的。公开 CAS 可以开发高度可伸缩的并发 Java 类。**这些更改主要由 JDK 库类使用，而不是由开发人员使用**。
* **低级实用程序类 -- 锁定和原子类** : 使用 CAS 作为并发原语，ReentrantLock 类提供与 synchronized 原语相同的锁定和内存语义，然而这样可以更好地控制锁定（如计时的锁定等待、锁定轮询和可中断的锁定等待）和提供更好的可伸缩性（竞争时的高性能）。大多数开发人员将不再直接使用 ReentrantLock 类，而是使用在 ReentrantLock 类上构建的高级类。
* **高级实用程序类** : 这些类实现并发构建块，每个计算机科学文本中都会讲述这些类 -- 信号、互斥、闩锁、屏障、交换程序、线程池和线程安全集合类等。大部分开发人员都可以在应用程序中用这些类，来替换许多（如果不是全部）同步、wait() 和 notify() 的使用，从而提高性能、可读性和正确性。

　　本文将重点介绍 java.util.concurrent 包提供的高级实用程序类,其中包括：线程安全集合、线程池和同步实用程序。这些是初学者和专家都可以使用的"现成"类。**java.util.concurrent 规范进程的一个目标就是提供一组线程安全的、高性能的并发构建块，从而使开发人员能够减轻一些编写线程安全类的负担。**
在介绍concurrent包之前先看一下jdk中需要了解的包截图（推荐边看源码，边学习）：  
![JDK包截图](/images/projectImage/java-concurrent-packege.png)

###1.1 java.util集合
　　java.util包对应java所有的collection集合，JDK 1.2 中引入的 Collection 框架是一种表示对象集合的高度灵活的框架，它使用基本接口 List、Set 和 Map。通过 JDK 提供每个集合的多次实现（HashMap、Hashtable、TreeMap、WeakHashMap、HashSet、TreeSet、Vector、ArrayList、LinkedList 等等）。其中一些集合已经是线程安全的（Hashtable 和 Vector），通过同步的封装工厂（Collections.synchronizedMap()、synchronizedList() 和 synchronizedSet()），其余的集合均可表现为线程安全的。  
　　java.util 中的线程集合仍有一些缺点。其中的“貌似线程安全集合”（如Hashtable,Vector等），如果没有任何线程在读取，则支持使用多个并发进程写入。**但是：如果使用一个或多个读取器以及一个或多个编写器，则同步包装不提供线程安全的访问。java.util 包中的集合类都返回 fail-fast 迭代器，这意味着它们假设线程在集合内容中进行迭代时，集合不会更改它的内容**。如果 fail-fast 迭代器检测到在迭代过程中进行了更改操作，那么它会抛出 ConcurrentModificationException，这是不可控异常，此时这些“貌似线程安全集合”就变成了线程不安全集合。
###1.2 java.util.concurrent
　　java.util.concurrent 包添加了多个新的线程安全集合类（ConcurrentHashMap、CopyOnWriteArrayList 和 CopyOnWriteArraySet）。这些类的目的是提供高性能、高度可伸缩性、线程安全的基本集合类型版本。  
　　java.util.concurrent 集合返回的迭代器称为弱一致的（weakly consistent）迭代器。对于这些类，如果元素自从迭代开始已经删除，且尚未由 next() 方法返回，那么它将不返回到调用者。如果元素自迭代开始已经添加，那么它可能返回调用者，也可能不返回。在一次迭代中，无论如何更改底层集合，元素不会被返回两次。  
　　**在 JDK 5.0 之前，确保线程安全的主要机制是 synchronized 原语。访问共享变量（那些可以由多个线程访问的变量）的线程必须使用同步来协调对共享变量的读写访问。java.util.-concurrent 包提供了一些备用并发原语，以及一组不需要任何其他同步的线程安全实用程序类。**

##2. 详解Concurrent--CopyOnWriteArrayList 
> 我们都知道，可以用两种方法创建线程安全支持数据的 List：  
> 1. Vector  
> 2.Collections.synchronizedList()封装一个list。


###2.1 为何vector不够安全？
　　在java.util.concurrent 包中添加了名称繁琐的 **CopyOnWriteArrayList**。为什么我们想要新的线程安全的List类？为什么Vector还不够？  
　　最简单的答案是与迭代和并发修改之间的交互有关。使用 Vector 或使用同步的 List 封装器，返回的迭代器是 fail-fast 的，这意味着如果在迭代过程中任何其他线程修改 List，迭代可能失败。**也就是说在读的同时不能对集合本身进行操作（如添加，修改等）。**  
　　Vector 的非常普遍的应用程序是存储通过组件注册的监听器的列表。当发生适合的事件时，该组件将在监听器的列表中迭代，调用每个监听器。为了防止 ConcurrentModificationException，迭代线程必须复制列表或锁定列表，以便进行整体迭代，而这两种情况都需要大量的性能成本。那么怎么解决这种并发问题呢？这也就就引出了CopyOnWriteArrayList来解决此类问题。
###2.2 CopyOnWriteArrayList的引入
　　CopyOnWriteArrayList 类通过每次添加或删除元素时创建支持数组的新副本，避免了这个问题，但是进行中的迭代保持对创建迭代器时的本体进行操作。虽然复制也会有一些成本，但是在许多情况下，迭代要比修改多得多，在这些情况下，写入时复制要比其他备用方法具有更好的性能和并发性。如果应用程序需要 Set 语义，而不是 List，那么还有一个 Set 版本 -- CopyOnWriteArraySet。  
　　**这些集合在读的时候跟普通集合一样不加锁**，在修改集合操作（如写，删）等时，则会用lock拿到当前集合的集合锁：

	final ReentrantLock lock = this.lock;
	lock.lock();

　　拿到集合锁之后，此时互斥的只是其他的写操作，而读操作不受任何影响可以并发同时进行。下面是一个其中的方法供参考：  

	 public E set(int index, E element) {
	        final ReentrantLock lock = this.lock;
	        lock.lock();
	        try {
	            Object[] elements = getArray();
	            E oldValue = get(elements, index);
	            if (oldValue != element) {
	                int len = elements.length;
	                Object[] newElements = Arrays.copyOf(elements, len); //要先进行原数组的拷贝，这样读操作可以在原数组上进行，写操作在拷贝上进行
	                newElements[index] = element;
	                setArray(newElements);
	            } else {
	                // Not quite a no-op; ensures volatile write semantics
	                setArray(elements);
	            }
	            return oldValue;
	        } finally {
	            lock.unlock();
	        }
	    }

###2.3 Vector与CopyOnWriteArrayList的比较
　　前者因为使用Synchronized修饰了所有内部方法，所以读和写只能某一时刻只有一个线程来执行。而后者允许多个读操作同时进行，且允许同时有一个写操作进行，因为写操作是在数组的副本上进行的，只有写操作互斥，与读操作不互斥。且Vector可能会发生ConcurrentModificationException，而后者不会。

##3. 详解Concurrent--ConcurrentHashMap
　　正如已经存在线程安全的 List 的实现，您可以用多种方法创建线程安全的、基于 hash 的 Map -- Hashtable，并使用 Collections.synchronizedMap() 封装 HashMap。JDK 5.0 添加了 ConcurrentHashMap 实现，该实现提供了相同的基本线程安全的 Map 功能，但它大大提高了并发性。  
　　Hashtable 和 synchronizedMap 所采取的获得同步的简单方法（同步 Hashtable 中或者同步的 Map 封装器对象中的每个方法）有两个主要的不足:

* 首先，Hashtable 和 Collections.synchronizedMap 通过同步每个方法获得线程安全。这意味着当一个线程执行一个 Map 方法时，无论其他线程要对 Map 进行什么样操作，都不能执行，直到第一个线程结束才可以。
* 同时，这样仍不足以提供真正的线程安全性，许多公用的混合操作仍然需要额外的同步。虽然诸如 get() 和 put() 之类的简单操作可以在不需要额外同步的情况下安全地完成，但还是有一些公用的操作序列，例如迭代或者 put-if-absent（空则放入），需要外部的同步，以避免数据争用。

　　对比来说，当多个线程需要访问同一Map 时，ConcurrentHashMap可以获得更高的并发性。  
　　**ConcurrentHashMap允许多个读操作并发进行，读操作并不需要加锁:**如果使用传统的技术，如HashMap中的实现，如果允许可以在hash链的中间添加或删除元素，读操作不加锁将得到不一致的数据。ConcurrentHashMap实现技术是保证HashEntry几乎是不可变的。HashEntry代表每个hash链中的一个节点，其结构如下所示：  

	static final class HashEntry<K,V> {  
	    final K key;  
	    final int hash;  
	    volatile V value;  
	    final HashEntry<K,V> next;  
	}  
　　可以看到除了value不是final的，其它值都是final的，这意味着不能从hash链的中间或尾部添加或删除节点，因为这需要修改next引用值，所有的节点的修改只能从头部开始。对于put操作，可以一律添加到Hash链的头部。但是对于remove操作，可能需要从中间删除一个节点，这就需要将要删除节点的前面所有节点整个复制一遍，最后一个节点指向要删除结点的下一个结点。这在讲解删除操作时还会详述。为了确保读操作能够看到最新的值，将value设置成volatile，这避免了加锁。  
　　**ConcurrentHashMap允许多个修改操作并发进行，其关键在于使用了锁分离技术:**它使用了多个锁来控制对hash表的不同部分进行的修改。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的hash table，它们有自己的锁。**只要多个修改操作发生在不同的段上，它们就可以并发进行。**  
　　在大多数情况下，ConcurrentHashMap 是 Hashtable或 Collections.synchronizedMap-(new HashMap()) 的简单替换。然而，其中有一个显著不同，即 ConcurrentHashMap 实例中的同步不锁定映射进行独占使用。实际上，没有办法锁定 ConcurrentHashMap 进行独占使用，它被设计用于进行并发访问。为了使集合不被锁定进行独占使用，还提供了公用的混合操作的其他（原子）方法，如 put-if-absent。ConcurrentHashMap 返回的迭代器是弱一致的，意味着它们将不抛出ConcurrentModificationException。可以通过下图说明：  
![JDK包截图](/images/projectImage/Java-hashMap-concurrentHashMap.jpg)
> ConcurrentHashMap的源码可以自行查看JDK源码，有点麻烦，这里就不举例了～。～关于其实现细节可以参考：
> [ConcurrentHashMap之实现细节](http://www.iteye.com/topic/344876 "ConcurrentHashMap之实现细节")

##4.详解Concurrent--线程池ThreadPoolExecutor
关于线程池的讲解已经在前面的文章中给出了：[伟哥教你破解--Java线程谜团](http://itweige.com/Java-Thread/ "伟哥教你破解--Java线程谜团")
##5. 详解Concurrent.locks--ReentrantLock
　　ReentrantLock 是具有与隐式监视器锁定（使用 synchronized 方法和语句访问）相同的基本行为和语义的 Lock 的实现，但它具有扩展的能力。  
　　作为额外收获，在竞争条件下，ReentrantLock 的实现要比现在的 synchronized 实现更具有可伸缩性。（有可能在 JVM 的将来版本中改进 synchronized 的竞争性能。）  
　　**这意味着当许多线程都竞争相同锁定时，使用 ReentrantLock 的吞吐量通常要比 synchronized好**。换句话说，当许多线程试图访问 ReentrantLock 保护的共享资源时，JVM 将花费较少的时间来调度线程，而用更多个时间执行线程。  
　　虽然 ReentrantLock 类有许多优点，但是与同步相比，它有一个主要缺点 -- 它可能忘记释放锁定。建议当获得和释放 ReentrantLock 时使用下列结构：

	Lock lock = new ReentrantLock();
	...
	lock.lock();
	try {
	  // perform operations protected by lock
	}
	catch(Exception ex) {
	 // restore invariants
	}
	finally {
	  lock.unlock();
	}


　　因为锁定失误（忘记释放锁定）的风险，所以对于基本锁定，强烈建议您继续使用 synchronized，除非真的需要 ReentrantLock 额外的灵活性和可伸缩性。  
　　ReentrantLock 是用于高级应用程序的高级工具 -- 有时需要，但有时用原来的方法就很好。几乎 java.util.concurrent 中的所有类都是在 ReentrantLock 之上构建的，ReentrantLock 则是在原子变量类的基础上构建的。所以，虽然仅少数并发专家使用原子变量类，但 java.util.concurrent 类的很多可伸缩性改进都是由它们提供的。