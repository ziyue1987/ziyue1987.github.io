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
	* [SequenceBarrier](#sequencbarrier)
	* [WaitStrategy](#waitstrategy)
	* [BatchEvenProcessor](#batcheventprocessor)
	* [WorkProcessor](#workprocessor)
	* [WorkerPool](#workerpool)
* [**Use Cases**](#usecases)
	* [消息定义](#message)
	* [Producer](#producer)
	* [EventProcessor及其依赖关系](#eventprocessor)
	* [One Publisher to one BatchEventProcessor](#onepublishertoonebatcheventprocessor)
	* [One Publisher to three BatchEventProcessors Pipeline](#onepublishertothreebatcheventprocessorspipeline)
	* [One Publisher to three BatchEventProcessors MultiCast](#onepublishertothreebatcheventprocessorsmulticast)
	* [One Publisher to two WorkProcessors](#onepublishertotwoworkprocessors)
	* [One Publisher to two WorkerPools](#onepublishertotwoworkerpools)
* [**结束语**](#end)
	
	

## <a name="intruduction" id="intruduction"></a>Intruduction

Disruptor 是Java实现的用于线程间通信的消息组件。核心是一个Lock-free的Ringbuffer。我使用BlockingQueue进行了简单的对比测试，结果表明使用Disruptor来进行线程间通信的效率会高将近一倍。LMAX给出的数据是使用Disruptor能够在一个线程里每秒处理6百万订单。那么Disruptor为什么会如此快呢？通过参考Martin Fowler（Disruptor的开发者之一）的技术博客和Disruptor的源代码，可以总结出以下四条原因：

### <a name="lockvscas" id="lockvscas"></a>Lock vs CAS
关于CAS(compare and swap)请参考WIKIPEDIA相关条目[Compare and swap](http://en.wikipedia.org/wiki/Compare-and-swap)。与大部分并发队列使用的Lock相比，CAS显然要快很多。CAS是CPU级别的指令，不需要像Lock一样需要OS的支持。
所以每次调用不需要kernel entry，也不需要context switch。当然，实现的复杂程度也相对提高了。

### <a name="avoidfalsesharing" id="avoidfalsesharing"></a>避免伪共享
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

### <a name="linkedvsarray" id="linkedvsarray"></a>Linked Queue vs Array Ringbuffer

Disruptor选择使用Array ringbuffer来构造lock-free队列，而不是选择Linked queue。

首先，数组是预分配的，这样不仅避免了Java GC带来的运行开销，而且对缓存来说更加友好，由于在ringbuffer上进行的操作是顺序执行的，保证了缓存命中率。使用Disruptor的时候，为了更好的利用ringbuffer的这个优点，需要将ringbuffer的元素设计的可重用，使生产者在生产消息或产生事件的时候尽量对ringbuffer元素中得属性进行更新，而不是新建。

其次，数组在定位元素的时候是使用索引，而链表在定位元素的时候使用对象引用（地址）。在lock-free队列中使用链表需要使用Double-CAS来克服ABA问题（关于double-CAS和ABA问题，请参coolshell上[关于无锁队列的文章](http://coolshell.cn/articles/8239.html)），而在数组中，可以通过递增的序号来标示不同时刻访问的相同元素，可以很自然得克服ABA问题，在需要访问数组元素的时候，只要用需要将序号对数组大小取余就可以得到数组索引。在Disruptor中Sequence就充当了递增的序号的角色。每次对ringbuffer的访问都会导致相应的Sequence增加。需要注意的是，由于Sequence是递增的，所以在到达最大值以后，会溢出，编程最小的负数，但这通常不是问题，因为要使long类型递增到溢出，即使每秒钟1000 000 000次递增，也需要上百年时间。

### <a name="catching" id="catching"></a>无时不刻的缓存
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


## <a name="component" id="component"></a>Component
### <a name="sequence" id="sequence"></a>Sequence
Sequence是Disruptor最核心的组件。生产者对RingBuffer的互斥访问，生产者与消费者之间的协调以及消费者之间的协调，都是通过Sequence实现。那么Sequence是什么呢？首先Sequence是一个递增的序号，说白了就是计数器；其次，由于需要在线程间共享，所以Sequence是引用传递，并且是线程安全的；再次，Sequence支持CAS；操作最后，为了提高效率，Sequence通过padding来避免伪共享。

### <a name="ringbuffer" id="ringbuffer"></a>RingBuffer
RingBuffer是存储消息的地方，通过一个名为cursor的Sequence对象指示队列的头，协调多个生产者向RingBuffer中添加消息，并用于在消费者端判断RingBuffer是否为空。巧妙的是，队列尾并没有在RingBuffer中，而是由消费者维护。这样的好处是多个消费者处理消息的方式更加灵活，可以在一个RingBuffer上实现消息的单播，多播，流水线以及它们的组合。其缺点是在生产者端判断RingBuffer是否已满是需要跟踪更多的信息，为此，在RingBuffer中维护了一个名为gatingSequence的Sequence数组来跟踪相关Seqence。

### <a name="sequencbarrier" id="sequencbarrier"></a>SequenceBarrier
SequenceBarrier用来在消费者之间以及消费者和RingBuffer之间建立依赖关系。在Disruptor中，依赖关系实际上指的是Sequence的大小关系，消费者A依赖于消费者B指的是消费者A的Sequence一定要小于等于消费者B的Sequence，因为所有消费者都依赖于RingBuffer，所以消费者的Sequence一定小于等于RingBuffer中名为cursor的Sequence。

SequenceBarrier在初始化的时候会收集需要依赖的消费者的Sequence，RingBuffer的cursor会被自动的加入其中。需要依赖其他消费者和/或RingBuffer的消费者在消费下一个消息时，会先等待在SequenceBarrier上，直到所有被依赖的消费者和RingBuffer的Sequence大于等于这个消费者的Sequence。当被依赖的消费者或RingBuffer的Sequence有变化时，会通知SequenceBarrier唤醒等待在它上面的消费者。

### <a name="waitstrategy" id="waitstrategy"></a>WaitStrategy
当消费者等待在SequenceBarrier上时，有许多可选的等待策略，不同的等待策略在效率和CPU资源的占用上有所不同，可以视应用场景选择：

* BusySpinWaitStrategy
* BlockingWaitStrategy
* SleepingWaitStrategy
* TimeoutBlockingWaitStrategy
* YieldingWaitStrategy
* PhasedBackoffWaitStrategy

### <a name="batcheventprocessor" id="batcheventprocessor"></a>BatchEvenProcessor
在Disruptor中，消费者是以EventProcessor的形式存在的。其中一类消费者是BatchEvenProcessor。每个BatchEvenProcessor有一个Sequence，来记录自己消费RingBuffer中消息的情况。所以，一个消息必然会被每一个BatchEvenProcessor消费。

### <a name="workprocessor" id="workprocessor"></a>WorkProcessor
另一类消费者是WorkProcessor。每个WorkProcessor也有一个Sequence，多个WorkProcessor还共享一个Sequence用于互斥的访问RingBuffer。一个消息被一个WorkProcessor消费，就不会被共享一个Sequence的其他WorkProcessor消费。

### <a name="workerpool" id="workerpool"></a>WorkerPool
共享同一个Sequence的WorkProcessor可由一个WorkerPool管理，这时，共享的Sequence也由WorkerPool创建。

## <a name="usecases" id="usecases"></a>Use Cases
下面以Disruptor 3.2.0版本为例介绍Disruptor的初级使用方法，大部分代码是出自Disruptor源代码中得perftest部分(Disruptor代码[这里下载](https://github.com/LMAX-Exchange/disruptor))。

### <a name="message" id="message"></a>消息定义
Disruptor中消息对象可以自由定义，但是必须定义实现EventFactory<T>接口的消息对象工厂来告诉RingBuffer如何初始化消息对象。

	public final class ValueEvent
	{
		private long value;
	
    	public long getValue()
    	{
        	return value;
    	}
	
    	public void setValue(final long value)
    	{
   	    	this.value = value;
    	}
	
    	public static final EventFactory<ValueEvent> EVENT_FACTORY = new EventFactory<ValueEvent>()
    	{
        	public ValueEvent newInstance()
        	{
            	return new ValueEvent();
        	}
    	};
	}
	

### <a name="producer" id="producer"></a>Producer

Disruptor中同样没有定义生产者，而是由RingBuffer提供添加消息的接口。针对单生产者和多生产者两种应用场景，RingBuffer提供了不同的初始化方法：
	
* 单生产者应用场景

		private final RingBuffer<ValueEvent> ringBuffer =
        	createSingleProducer(ValueEvent.EVENT_FACTORY, BUFFER_SIZE, new YieldingWaitStrategy());

* 多生产者应用场景
	
		private final RingBuffer<ValueEvent> ringBuffer =
        	createMultiProducer(ValueEvent.EVENT_FACTORY, BUFFER_SIZE, new BusySpinWaitStrategy());
        	
初始化的时候需要提供消息工厂，RingBuffer大小，以及一个选定的waitStrategy。向RingBuffer中添加消息的过程分成两阶段：1，申请可用节点，并将消息放入节点中；2，提交节点。

	// 阶段1：申请节点，并将消息放入节点中
	long next = rb.next();
    rb.get(next).setValue(0);
    
    // 阶段2：提交节点
    rb.publish(next);

### <a name="eventprocessor" id="eventprocessor"></a>EventProcessor及其依赖关系
Disruptor定义了两种EventProcessor：BatchEventProcessor和WorkProcessor。两种EventProcessor都实现了Runnable接口，在组装完成后可以直接放入线程中执行。

用户需要实现自己的EventHandler来告诉EventProcessor在收到消息的时候怎样处理。

用户还需要结合SequenceBarrier来构造各个EventProcessor之间及其和RingBuffer之间的依赖，，关于依赖的定义，已经在上文解释过了。这里需要说明的是，我们在使用Queue构造pipeline的时候，类似于接水管，每一个步骤需要哪些处理，就用Queue接过去，处理完成后再用Queue接到下一个步骤。这种方式固然实现起来简单，但是消息需要穿过各个Queue，必要的时候还需要对消息进行复制，这会产生大量对Queue的并发访问操作，效率很低。在Disruptor里，相邻的两个步骤被解释成步骤2中的EventProcessor依赖步骤1中的EventProcessor，消息在RingBuffer中依次被步骤1中的EventProcessor和步骤2中的EventProcessor处理。

不仅EventProcessor对RingBuffer有依赖关系，RingBuffer对EventProcessor也有反向依赖。RingBuffer需要保证在生产者比消费者快得情况下，新生产的消息不会覆盖未被完全消费（即被所有EventProcessor处理）的消息。为了做到这一点，RingBuffer会追踪有依赖关系的EventProcessor中最小的Sequence（如果不能根据依赖关系判断Sequence大小，则全部追踪）。需要追踪的Sequence会加入到RingBuffer的gatingSequence数组中。下面通过几个use case说明两种EventProcessor和RingBuffer如何组装。

### <a name="onepublishertoonebatcheventprocessor" id="onepublishertoonebatcheventprocessor"></a>One Producer to one BatchEventProcessor
这是最简单的场景，一个BatchEventProcessor

	// 构造RingBuffer
    private final RingBuffer<ValueEvent> ringBuffer =
        createSingleProducer(ValueEvent.EVENT_FACTORY, BUFFER_SIZE, new YieldingWaitStrategy());
        
    // 构造BatchEventProcessor 及依赖关系
    private final SequenceBarrier sequenceBarrier = ringBuffer.newBarrier();
    private final ValueAdditionEventHandler handler = new ValueAdditionEventHandler();
    private final BatchEventProcessor<ValueEvent> batchEventProcessor = new BatchEventProcessor<ValueEvent>(ringBuffer, sequenceBarrier, handler);
    
    // 构造反向依赖
    ringBuffer.addGatingSequences(batchEventProcessor.getSequence());

### <a name="onepublishertothreebatcheventprocessorspipeline" id="onepublishertothreebatcheventprocessorspipeline"></a>One Producer to three BatchEventProcessors Pipeline

三个BatchEventProcessor构成一个pipeline，对一个消息先后进行加工。

	// 构造RingBuffer
    private final RingBuffer<FunctionEvent> ringBuffer =
        createSingleProducer(FunctionEvent.EVENT_FACTORY, BUFFER_SIZE, new YieldingWaitStrategy());

	// 构造BatchEventProcessor 及依赖关系
	// stepOneBatchProcessor依赖RingBuffer
    private final SequenceBarrier stepOneSequenceBarrier = ringBuffer.newBarrier();
    private final FunctionEventHandler stepOneFunctionHandler = new FunctionEventHandler(FunctionStep.ONE);
    private final BatchEventProcessor<FunctionEvent> stepOneBatchProcessor =
        new BatchEventProcessor<FunctionEvent>(ringBuffer, stepOneSequenceBarrier, stepOneFunctionHandler);

	// stepTwoBatchProcessor依赖RingBuffer和stepOneBatchProcessor
    private final SequenceBarrier stepTwoSequenceBarrier = ringBuffer.newBarrier(stepOneBatchProcessor.getSequence());
    private final FunctionEventHandler stepTwoFunctionHandler = new FunctionEventHandler(FunctionStep.TWO);
    private final BatchEventProcessor<FunctionEvent> stepTwoBatchProcessor =
        new BatchEventProcessor<FunctionEvent>(ringBuffer, stepTwoSequenceBarrier, stepTwoFunctionHandler);

	// stepThreeBatchProcessor依赖RingBuffer和stepTwoBatchProcessor
    private final SequenceBarrier stepThreeSequenceBarrier = ringBuffer.newBarrier(stepTwoBatchProcessor.getSequence());
    private final FunctionEventHandler stepThreeFunctionHandler = new FunctionEventHandler(FunctionStep.THREE);
    private final BatchEventProcessor<FunctionEvent> stepThreeBatchProcessor =
        new BatchEventProcessor<FunctionEvent>(ringBuffer, stepThreeSequenceBarrier, stepThreeFunctionHandler);

	// 构造反向依赖，stepThreeBatchProcessor的Sequence最小
    ringBuffer.addGatingSequences(stepThreeBatchProcessor.getSequence());


### <a name="onepublishertothreebatcheventprocessorsmulticast" id="onepublishertothreebatcheventprocessorsmulticast"></a>One Producer to three BatchEventProcessors MultiCast

一个消息被三个BatchEventProcessor处理，但没有先后关系。

	// 构造RingBuffer
    private final RingBuffer<ValueEvent> ringBuffer =
        createSingleProducer(ValueEvent.EVENT_FACTORY, BUFFER_SIZE, new YieldingWaitStrategy());

	// 构造BatchEventProcessor 及依赖关系
    private final SequenceBarrier sequenceBarrier = ringBuffer.newBarrier();

    private final ValueMutationEventHandler[] handlers = new ValueMutationEventHandler[NUM_EVENT_PROCESSORS];

    handlers[0] = new ValueMutationEventHandler(Operation.ADDITION);
    handlers[1] = new ValueMutationEventHandler(Operation.SUBTRACTION);
    handlers[2] = new ValueMutationEventHandler(Operation.AND);

    private final BatchEventProcessor<?>[] batchEventProcessors = new BatchEventProcessor[3];

    batchEventProcessors[0] = new BatchEventProcessor<ValueEvent>(ringBuffer, sequenceBarrier, handlers[0]);
    batchEventProcessors[1] = new BatchEventProcessor<ValueEvent>(ringBuffer, sequenceBarrier, handlers[1]);
    batchEventProcessors[2] = new BatchEventProcessor<ValueEvent>(ringBuffer, sequenceBarrier, handlers[2]);

	// 构造反向依赖，三个EventProcessor没有依赖关系，将它们的Sequence全部加入
    ringBuffer.addGatingSequences(batchEventProcessors[0].getSequence(),
                                  batchEventProcessors[1].getSequence(),
                                  batchEventProcessors[2].getSequence());


### <a name="onepublishertotwoworkprocessors" id="onepublishertotwoworkprocessors"></a>One Producer to two WorkProcessors
一个消息只会被两个WorkProcessor中的一个处理。

	// 构造RingBuffer
    private final RingBuffer<ValueEvent> ringBuffer =
            RingBuffer.createSingleProducer(ValueEvent.EVENT_FACTORY,
                                            BUFFER_SIZE,
                                            new YieldingWaitStrategy());
    
    // 构造拥有两个WorkProcessor的WorkerPool                                        
    private final EventCountingWorkHandler[] handlers = new EventCountingWorkHandler[2];
    for (int i = 0; i < 2; i++)
    {
        handlers[i] = new EventCountingWorkHandler(counters, i);
    }

    private final WorkerPool<ValueEvent> workerPool =
            new WorkerPool<ValueEvent>(ringBuffer,
                                       ringBuffer.newBarrier(),
                                       new FatalExceptionHandler(),
                                       handlers);
    
    // 构造反向依赖
    ringBuffer.addGatingSequences(workerPool.getWorkerSequences());
    

### <a name="onepublishertotwoworkerpool" id="onepublishertotwoworkerpool"></a>One Producer to two WorkerPools
一个消息会被两个WorkerPool中的WorkProcessor处理，但在一个WorkerPool中只能被一个WorkProcessor处理。

	// 构造RingBuffer
    private final RingBuffer<ValueEvent> ringBuffer =
            RingBuffer.createSingleProducer(ValueEvent.EVENT_FACTORY,
                                            BUFFER_SIZE,
                                            new YieldingWaitStrategy());
    
    SequenceBarrier barrier = ringBuffer.newBarrier();
    
    // 构造拥有两个WorkProcessor的WorkerPool
    private final EventCountingWorkHandler[] handlers = new EventCountingWorkHandler[4];
    for (int i = 0; i < 4; i++)
    {
        handlers[i] = new EventCountingWorkHandler(counters, i);
    }
        
    private final WorkerPool<LesStringEvent> workerPool0 =
            new WorkerPool<LesStringEvent>(ringBuffer,
                                           barrier,
                                           new FatalExceptionHandler(),
                                           handlers[0], handlers[1]);
    private final WorkerPool<LesStringEvent> workerPool1 =
            new WorkerPool<LesStringEvent>(ringBuffer,
                                           barrier,
                                           new FatalExceptionHandler(),
                                           handlers[2], handlers[3]);
        
    // 构造反向依赖
    ringBuffer.addGatingSequences(workerPool0.getWorkerSequences());
    ringBuffer.addGatingSequences(workerPool1.getWorkerSequences());
## <a name="end" id="end"></a>结束语
本文主要讲述了Disruptor得基本使用方法，涉及少量对实现的解释，意在通过Disruptor的用用管窥Disruptor的设计思想。如果有时间，就再写一篇关于Disruptor实现的文章。本文没有涉及Disruptor定义的DSL（领域特定语言）接口，通过DSL可以更方便的使用Disruptor。