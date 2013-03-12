---
layout: post
title: "关于Condition Variable为什么需要一个Mutex的思考"
date: 2012-02-19 23:37
comments: true
categories: [原理探求]
sharing: false
published: true
---
1. Linux下：

``` c
pthread_mutex_lock(&mutex);
pthread_cond_wait(&cond, &mutex);
doSomething();
pthread_mutex_unlock(&mutex);
```

<!-- more -->

2. java里：

``` java
synchronized(this){
    wait();
    doSomething();
}
```

3. C#里：

``` c#
monitor.Enter();
monitor.Wait();
doSomething();
monitor.Exit();
```

4. 使用Win32API

``` c
EnterCriticalSection (&cs);
SleepConditionVariableCS (&cond, &cs, INFINITE);
doSomething();
LeaveCriticalSection (&cs);
```

不难看到，不管是哪种语言，不论使用什么程序库，无论在windows下亦或是Linux下，condition variable这个东西的用法似乎都是固定的：必须和一把锁搭配使用。

可是，问题是为什么这些库、框架、系统的设计者，在设计这套机制的时候，非要让我们好死不死再和一把锁一块儿使用呢？做这种费力不讨好的事儿，到底是因为历史原因，还是另有深意呢？

为了回答这个问题，首先考虑这么一个生产者消费者的场景。生产者会生产数据将其放到dataHandler对象中，然后向消费者发送一条消息。消费者也有dataHandler示例的句柄，并且在接收到生产者的消息后，它将通过dataHandler.getData()来获取数据。假设这个世界上的condition variable全是没有mutex机制的，则消费者的实现代码可能是这样的：

``` c
if(!bDataReady){
     pthread_cond_wait(&sigDataReady); //根据假设，我们现在不需要锁了
}
data = dataHandler.getData();
dataHandler.releaseData();
```

代码的逻辑十分简单。但是，现在我们考虑一下，如果有以下执行顺序：

- 消费者：检查bDataReady不为真
- 生产者：设置bDataReady为true
- 生产者：发送消息
- 消费者：等待condition variable

这时，虽然生产者通知了资源的可用，但由于这时候消费者尚未开始等待消息，也就没能接收到这个消息，于是只能在消息消失后无尽地等待下去。另一方面，由于消费者一直的等待，导致资源不被消费，在糟糕的情况下生产者会等待消费者消费完毕。于是乎，死锁发生了。

思考一下，为什么会发生死锁呢？其本质的原因在于生产者、消费者的行为没有保证原子性(Atomic)。这时候，我们要问了，既然pthread_cond_wait是系统级别的同步互斥工具，它怎么会不能保证原子性？当然，我并非指这个意义上的原子性。我们首先要知道condition variable的设计意图，实际上condition variable描述这么一种情景：在这个情境中，有一个人在等待一个特定事件，而另一个人会在该事件发生时告知前者。但是，condition variable本身只实现这么一种机制，它并不指出到底是什么事件发生了。因此，往往我们在等待成功，知道有事件发生的话，还需要额外代码来判断是哪个事件。注意，这就是整个问题的关键点：

实际上我们在使用condition variable时，不仅要通condition variable实现通知和被通知，还要自行实现通知和被通知前后所必要的判断、处理等业务逻辑，这才是condition variable的使用模型。知道这个后，我们就能理解为什么上面的情景会出问题了。原因在于消费者的基本行为没有确保原子性，它的判断和等待被分隔了。

我们当然可以争辩道，即使不通过和mutex合作，我们一样能使用这个假想的condition variable很好的完成功能。是的，我承认作为程序员你可以足够聪明和小心，以至于你实现的版本没有我上面这段代码的问题。但是，这难道不和你去跟设计mutex的人争辩说你明明可以自己通过精心设计实现互斥，根本不用mutex横插一脚这件事一样愚蠢且毫无意义么？谁也不能阻止你选择使用自己的办法，但是问题的关键是，我们需要一种有经过理论证明研究过的可靠的、统一的方法，因为软件开发在大部分情况下是一门工程，而不是一门任你发挥的艺术。
