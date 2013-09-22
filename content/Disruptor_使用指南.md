Date: 2013-09-22
Title: Disruptor使用指南
Category: Java
Tags: 并发,Disruptor
Slug: disruptor-use-manual

![Mou icon](http://ifeve.com/wp-content/uploads/2013/02/Disruptor-300x144.png)

<a>By Hanbin Zheng</a>                                

**声明：本文系博主原创，欢迎转载，但请注明作者和出处，谢谢！** 

* [**Intruduction**](#intruduction)
	* [Lock vs CAS](#lockvscas)
	* [避免伪共享](#avoidfalsesharing)
	* [Linked Queue vs Array Ringbuffer](#linkedvsarray)
	* [无时不刻的缓存](#catching)
* [**Component**](#component)
	* [Sequence](#sequence)
	* [RingBuffer](#ringbuffer)
	* [SequencBarrier](#sequencbarrier)
	* [BatchEvenProcessor](#batcheventprocessor)
	* [WorkProcessor](#workprocessor)
	* [WorkerPool](#workerpool)

## [Intruduction](id:intruduction)

Disruptor 是Java实现的用于线程间通信的消息组件。核心是一个Lock-free的Ringbuffer。我使用BlockingQueue进行了简单的对比测试，结果表明使用Disruptor来进行线程间通信的效率会高将近一倍。LMAX给出的数据是使用Disruptor能够在一个线程里每秒处理6百万订单。那么Disruptor为什么会如此快呢？通过参考Martin Fowler（Disruptor的开发者之一）的技术博客和Disruptor的源代码，可以总结出以下四条原因：

### [Lock vs CAS](id:lockvscas)
关于CAS(compare and swap)请参考WIKIPEDIA相关条目[Compare and swap](http://en.wikipedia.org/wiki/Compare-and-swap)。与大部分并发队列使用的Lock相比，CAS显然要快很多。CAS是CPU级别的指令，不需要像Lock一样需要OS的支持。
所以每次调用不需要kernel entry，也不需要context switch。当然，实现的复杂程度也相对提高了。

### [避免伪共享](id:avoidfalsesharing)
关于伪共享请参考WIKIPEDIA相关条目[False sharing](http://en.wikipedia.org/wiki/False_sharing)。为了避免伪共享带来的性能下降，Disruptor对一切可能存在伪共享的地方使用Padding将两个不想关的内存隔离到两个缓存行上。可能存在伪共享的地方包括两个不相关的线程共享变量之间以及不相关线程私有变量和想成共享变量。下面分别举例子说明。
在Disruptor的实现中，有一个多线程共享的计数组件Sequence，对Sequence的操作可以说是整个Disruptor的核心，关于Sequence，在下文介绍各个组件的时候还要详细说明。这里主要说明它是怎样避免伪共享的。主要代码如下：

    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;

    static
    {
        UNSAFE = Util.getUnsafe();
        final int base = UNSAFE.arrayBaseOffset(long[].class);
        final int scale = UNSAFE.arrayIndexScale(long[].class);
        VALUE_OFFSET = base + (scale * 7);
    }

    private final long[] paddedValue = new long[15];
    
    . . .   . . .
    
    public long get()
    {
        return UNSAFE.getLongVolatile(paddedValue, VALUE_OFFSET);
    }

Sequence定义了一个长度为15的long类型数组，使用数组第八个元素计数，数组其他部分连同对象的头作为padding部分，保证在以64byte作为缓存行大小的CPU中，计数用元素不会与其他变量存在于同一个缓存行中。关于Java中对象在内存中具体怎样布局，可以参考[深入理解Java虚拟机：JVM高级特性与最佳实践](http://www.amazon.cn/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E8%99%9A%E6%8B%9F%E6%9C%BA-JVM%E9%AB%98%E7%BA%A7%E7%89%B9%E6%80%A7%E4%B8%8E%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5-%E5%91%A8%E5%BF%97%E6%98%8E/dp/B00D2ID4PK/ref=sr_1_2?ie=UTF8&qid=1379761486&sr=8-2&keywords=%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%9AJVM%E9%AB%98%E7%BA%A7%E7%89%B9%E6%80%A7%E4%B8%8E%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)

另一个例子是关于线程私有变量的：

	private static class Padding
    {
        /** Set to -1 as sequence starting point */
        public long nextValue = Sequence.INITIAL_VALUE, cachedValue = Sequence.INITIAL_VALUE, p2, p3, p4, p5, p6, p7;
    }

    private final Padding pad = new Padding();
    
这段代码用在单生产者的应用场景中。在这种应用场景下，这个计数器不需要是线程安全的，使用Sequence过于heavy了，但仍然需要通过padding将其与其他线程共享的变量隔离开来。p2到p7的作用就是这个。

### [Linked Queue vs Array Ringbuffer](id:linkedvsarray)

Disruptor选择使用Array ringbuffer来构造lock-free队列，而不是选择Linked queue。

首先，数组是预分配的，这样不仅避免了Java GC带来的运行开销，而且对缓存来说更加友好，由于在ringbuffer上进行的操作是顺序执行的，保证了缓存命中率。使用Disruptor的时候，为了更好的利用ringbuffer的这个优点，需要将ringbuffer的元素设计的可重用，使生产者在生产消息或产生事件的时候尽量对ringbuffer元素中得属性进行更新，而不是新建。

其次，数组在定位元素的时候是使用索引，而链表在定位元素的时候使用对象引用（地址）。在lock-free队列中使用链表需要使用Double-CAS来克服ABA问题（关于double-CAS和ABA问题，请参coolshell上[关于无锁队列的文章](http://coolshell.cn/articles/8239.html)），而在数组中，可以通过递增的序号来标示不同时刻访问的相同元素，可以很自然得克服ABA问题，在需要访问数组元素的时候，只要用需要将序号对数组大小取余就可以得到数组索引。在Disruptor中Sequence就充当了递增的序号的角色。每次对ringbuffer的访问都会导致相应的Sequence增加。需要注意的是，由于Sequence是递增的，所以在到达最大值以后，会溢出，编程最小的负数，但这通常不是问题，因为要使long类型递增到溢出，即使每秒钟1000 000 000次递增，也需要上百年时间。

### [无时不刻的缓存](id:catching)
为了高效，Disruptor可谓无所不用其极，它绝不会放过任何利用缓存的信息的机会，看一个例子。

	public long next(int n)
    {
        if (n < 1)
        {
            throw new IllegalArgumentException("n must be > 0");
        }

        long nextValue = pad.nextValue;

        long nextSequence = nextValue + n;
        long wrapPoint = nextSequence - bufferSize;
        long cachedGatingSequence = pad.cachedValue;

        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
        {
            long minSequence;
            while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
            {
                Thread.yield();
            }

            pad.cachedValue = minSequence;
        }

        pad.nextValue = nextSequence;

        return nextSequence;
    }
    
这个函数是在单生产者的应用场景下生产者获取n个可用元素时执行的代码。在Disruptor里，需要多线程共享的序号，用Sequence表示，它是线程安全的，但同时访问Sequence的效率会降低，而在单线程内使用的序列号，则使用long类型，它是相对高效的。得益于序列号是递增的，就可以使用long类型cache住访问Seqence的结构，优先使用cache住的序号，只有当cache住的序号不满足条件时，才去访问Sequence。


## [Component](id:component)
### [Sequence](id:sequence)
Sequence是Disruptor最核心的组件。生产者对RingBuffer的互斥访问，生产者与消费者之间的协调以及消费者之间的协调，都是通过Sequence实现。那么Sequence是什么呢？首先Sequence是一个递增的序号，说白了就是计数器；其次，由于需要在线程间共享，所以Sequence是引用传递，并且是线程安全的；再次，Sequence支持CAS；操作最后，为了提高效率，Sequence通过padding来避免伪共享。

### [RingBuffer](id:ringbuffer)
RingBuffer是存储消息的地方，通过一个名为cursor的Sequence对象指示队列的头，协调多个生产者向RingBuffer中添加消息，并用于在消费者端判断RingBuffer是否为空。巧妙的是，队列尾并没有在RingBuffer中，而是由消费者维护。这样的好处是多个消费者处理消息的方式更加灵活，可以在一个RingBuffer上实现消息的单播，多播，流水线以及它们的组合。其缺点是在生产者端判断RingBuffer是否已满是需要跟踪更多的信息，为此，在RingBuffer中维护了一个名为gatingSequence的Sequence数组来跟踪相关Seqence。

### [SequencBarrier](id:sequencbarrier)


### [BatchEvenProcessor](id:batcheventprocessor)
### [WorkProcessor](id:workprocessor)
### [WorkerPool](id:workerpool)