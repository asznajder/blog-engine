---
layout: post
title: How to avoid thread pool induced deadlocks?
excerpt: "Thread pools are very powerful, but used unwisely can even cause deadlocks."
modified: 2016-02-28
tags: [scala, concurrency, java, jvm, threads, future, async, asynchronous]
comments: true
image:
  feature: sample-image-5.jpg
  credit: WeGraphics
  creditlink: http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---

<div style="text-align: justify">
&nbsp;&nbsp;&nbsp;&nbsp;Thread pool induced deadlocks tend to be really frustrating. What do I mean by this? These kind of deadlocks can appear only on selective machines (e.g. prod). What’s more, they may affect the code that doesn’t have any synchronization so at the first glance there are no suspicious points to start debugging. In this post, I’ll try to explain why they appear and how to avoid them.
</div>

## How does a thread pool work?

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;In order to talk about deadlocks and starvations that can appear in a thread pool we have to recall how thread pools actually work. As you probably know, thread pools are used when you want to limit the number of actually running concurrent operations. Without them, each time you start a new task within a new <i>Thread</i> object a JVM thread is created. Such threads are implemented with native threads so OS is engaged in their creation process. When you have too many threads running in your application context switches time appear to be significant what can result in a execution slow down. What’s more, constructing a new thread each time is also a time-consuming operation since it requires a switch from user mode to kernel mode. That’s why, thread pools allow you to create some group of threads that will act as executors for operations which you will submit. These threads’ lifetime depends on thread pool’s configuration, but most often some part of them is kept alive even if there are no tasks to execute.
</p>

<p>There are several kind of thread pools in Java.  The decision which of them you should use depends of course on tasks you want to execute. Nevertheless, we can say that there are two main types of thread pools in Java that are represented with classes <i>ForkJoinPool</i> and <i>ThreadPoolExecutor</i>. In short, the first one is specialized in tasks that can be divided into many smaller subtasks and their result might be joined. On the other hand, <i>ThreadPoolExecutor</i> is much more generic. We will talk in more detail about this second kind of thread pools.</p>

<p>Each <i>ThreadPoolExecutor</i> can be described using several attributes. First of all, thread pools have queues for tasks that cannot be immediately executed because all available threads are processing other requests. Such queue can be <i>bounded</i> or <i>unbounded</i>. Bounded queues are much safer because they won’t eat your whole memory, but if the queue is full, they can cause throwing the <i>RejectedExecutionException.</i> </p>

<p>Secondly, each <i>ThreadPoolExecutor</i> has <i>core pool size</i> and <i>maximum pool size</i>. New thread pool creation doesn’t really imply immediate creation of threads. A new thread is started each time a new task is submitted and the total number of threads in the pool is smaller than <i>core size</i>. What’s important, these threads are created even if there are some idle threads in the pool. If the queue is full and <i>maximum pool size</i> is bigger than <i>core size</i>, new threads are created up to the set limit to process waiting tasks. What’s important, when the queue gets empty and threads start to be idle, the process of their termination begins. Additional threads are stopped if they are idle for more than <i>keepAliveTime</i> which is another thread pool’s attribute. The thread pool is a stable state when it has <i>core size</i> threads running. What’s important to note, keeping <i>maximum size</i> the same as <i>core size</i> means that you are actually creating a <i>fixed-size pool</i>. </p>

<p>There are many other things that you can customize in <i>ThreadPoolExecutor</i> including <i>ThreadFactory</i> and hooks, but they are generally not so critical from the concurrency safety point of view. That’s why if you want to learn more about them, I encourage you to simply read the documentation.</p>

<p>You have to remember that running too small tasks in a thread-pool might be connected with some performance loss. It would be of course much smaller overhead than creating a new thread for each new task and deleting it after execution. Still, the work has to be moved to the pool so the task has to be queued and a context switch has to be made. If the task is too small, the overhead of this operations might be significant. That’s why, you have to keep your tasks reasonable small but not too small. Submitting trivial tasks will cause your code run slower than it should.</p>
</div>

## Simple stupid starvation

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;Each task occupies a thread in a pool as long as it’s not finished. This means that if you create tasks that have infinite loops or work as long as some condition is met (e.g. variable set to <i>true</i>), then the thread will stay assigned to this task. As a result, adding more tasks than the pool’s size might result in a starvation. A very stupid example of such situation is below:</p>

{% highlight scala %}
val n = Runtime.getRuntime.availableProcessors
val threadPool = Executors.newFixedThreadPool(n)

(1 to n + 1).foreach {  count =>
  threadPool.submit(new Runnable {
    override def run(): Unit = {
      while (true) {
        println(count)
        Thread.sleep(1.second)
      }
    }
  })
}
{% endhighlight %}

<p>&nbsp;&nbsp;&nbsp;&nbsp;We have here a thread pool of size <i>n</i> but submit to it <i>n + 1</i> tasks. As you may expect the first <i>n</i> tasks will get their own threads and the last task will be queued. Unfortunately it will stay in this queue forever, because other tasks are not going to finish and free their threads. Please notice that the starvation appeared because we used the fixed size thread pool that doesn’t create extra threads for enqueued tasks. It’s <i>max size</i> is the same as <i>core size</i> so no extra threads will be created. What’s more, the task would get starved also in thread pools that create more threads than <i>core size</i>. Why? Because in most cases one task in a queue would be too little to create more threads.</p></div>

## Just a little bit less stupid starvation

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;Tasks starvation in thread pools seems to be quite easy to debug. They appear mainly when you submit too many tasks to a pool or an infinite loop appears. Still, there are some situations where the code works fine at your desktop but might cause troubles on other machines. Why? For example your pool size depends on some machine specific attribute (like the number of processors) and you create tasks without considering this attribute. Here is an example:</p>

{% highlight scala %}
val TasksNumber = 8 // magic number
val n = Runtime.getRuntime.availableProcessors // might be 8 or 16, but also 2
val threadPool = Executors.newFixedThreadPool(n)

(1 to TasksNumber).foreach {  count =>
  threadPool.submit(new Runnable {
    override def run(): Unit = {
      while (true) {
        // Do something important that uses Count
      }
    }
  })
}
{% endhighlight %}

<p>&nbsp;&nbsp;&nbsp;&nbsp;Let’s now assume that everything works fine on your desktop with 8 cores. Unfortunately, if your friend would run this code on a machine with 4 cores, the half of your tasks would get starved. Why? Because the thread pool size would be 4 and you would submit 8 tasks to it. As a result, your application is not correct since only 50% of the requested work will be done.</p>

<p>The above two examples are of course very trivial and unnatural. Their only purpose was to illustrate what happens when there are some tasks waiting for a free thread. They give us some intuition that will help to better understand deadlocks.</p></div>

# Thread pool induced deadlocks

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;But why have we talked so much about the starvation so far? Unfortunately, it can sometimes cause deadlocks. Thread pool induced deadlocks may differ on the difficulty of finding their cause, but they most often follow the same schema.  The simplest reason of deadlock is depending in one task (<span style="color: red"><i>T1</i></span>) on the results of another task (<span style="color: blue"><i>T2</i></span>). If a thread pool is full and the second task (<span style="color: blue"><i>T2</i></span>) gets queued, the first task (<span style="color: red"><i>T1</i></span>) might block and wait infinitely. Classic deadlock appears then: thread in a pool is occupied by the first task (<span style="color: red"><i>T1</i></span>) which waits for the second task (<span style="color: blue"><i>T2</i></span>) that cannot be executed because all threads in a pool are busy. Here is a very simple example of such situation:
</p>

{% highlight scala %}
class Consumer(queue: BlockingQueue[Long]) extends Runnable {
  val running = new AtomicBoolean(false)

  override def run(): Unit = {
    running.set(true)
    while (running.get()) {
      try {
        val element = queue.take()
        consume(element)
      } catch {
        case e: InterruptedException =>
        running.set(false)
      }
    }
  }
}

class Producer(queue: BlockingQueue[Long]) extends Runnable {
  override def run(): Unit = {
    val element = produce()
    queue.put(element)
  }
}

val n = Runtime.getRuntime.availableProcessors
val threadPool = Executors.newFixedThreadPool(n)

val queue = new LinkedBlockingQueue[Long]()
val consumer = new Consumer(queue)
val producer = new Producer(queue)
{% endhighlight %}

<p>&nbsp;&nbsp;&nbsp;&nbsp;In this example we have two types of tasks: <i>Producers</i> and <i>Consumers</i>. The first group is creating some elements and pushes them to the <i>BlockingQueue</i>. On the other hand, <i>Consumers</i> read data from the queue and processes them. The <i>Consumer</i> thread gets blocked when there are no elements in a queue. What’s important, the blocked task still holds the thread.</p>

<p>The above code should work fine on most machines. Still, there are at least two tricky corner cases in which such approach would stop working. The first one is running this code on a single CPU computer. In such case, the thread pool would have only one thread so the <i>Consumer</i> task would take it and <i>Producer</i> would get queued and starved. As a result, there is no way then that the <i>Consumer</i> would process anything so we have deadlock although the producer has been created. Another corner case is creating too many <i>Consumers</i> so <i>Producers</i> would also be queued. The simplest solution to this problem is of course creating separate pools for both types of tasks.</p>

<p>The problematic code above could be very easily modified to the following situations which also result in a deadlock: 

<ul>
    <li><i>T2</i> is a task that releases some lock on which <i>T1</i> is blocked;</li>
    <li><i>T2</i> is created by T1 to compute something and <i>T1</i> at some point of time blocks to get the result (Java Futures get);</li>
    <li><i>T2</i> modifies the shared state on which execution of <i>T1</i> depends;</li>
    <li>….</li>
</ul>
</p>

<p>All the described examples are quite easy to understand but as the dependences in your business logic get more complicated the harder it gets to find the reason of a deadlock. What’s funny, it’s almost always the same: some threads simply got starved.</p></div>

## Scala’s Futures vs deadlocks

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;Let’s dive into another example. Imagine the following situation. You would really like to visit Tokio but airplane tickets to Japan are very expensive. You know that from time to time it’s possible to buy tickets with a huge discount and you don’t want to miss such opportunity. That’s why you decided to write an application that will periodically check tickets price and automatically buy them if it’s below some set threshold. Both operations (price check and booking) require to make calls to some external services. We could do something like this:</p>

{% highlight scala %}
implicit val ecn = ExecutionContext.fromExecutorService(
    Executors.newFixedThreadPool(n))

Future { // first future
  val flightInfo = Await.result(Future { // second future, first future waits for a result
          priceChecker.getFlightsLowestPrice(DreamedDestination)
  }, 30.seconds)

  println(s"Managed to get price: ${flightInfo.price}")
  if (flightInfo.price < FlightPriceThreshold) {
    Future {
      flightsBooker.bookFlight(flightInfo.id)
    } onSuccess { case _ =>
      sendSms(s"You're flying to $DreamedDestination!")
    }
  }
}
{% endhighlight %}

<p>&nbsp;&nbsp;&nbsp;&nbsp;Will this code work? Most often yes, but in some circumstances a deadlock might appear. Everything depends on the number of free threads in a pool. But why actually something can go wrong? It’s all about creating <i>Future</i> that’s responsible for fetching the lowest flight price and blocking on it’s result. As an effect of this call, current <i>Future</i> (so also thread) gets blocked and another task (second <i>Future</i>) is created that needs a thread to execute. If there are no free threads in a pool, the task will get starved what will result in a timeout after 30 seconds and an exception.</p>

<p>There is an easy way to make above example working. The recommended approach is to simply get rid of blocking and use only fully asynchronous code. Here is an example of sample non-blocking implementation:</p>

{% highlight scala %}
val fetchedPrices = Future {
  PriceChecker.getFlightsLowestPrices(dreamedDestination)
}

val booking = fetchedPrices.map { flight =>
  println(s"Managed to get price: ${flight.price}")
  if (flight.price < FlightPriceThreshold) {
    FlightsBooker.bookFlight(flight.id)
  }
}

booking onSuccess { case _ =>
  Future {
    sendSms(s"I bought you a ticket. You're flying to $dreamedDestination!")
  }
}
{% endhighlight %}

</div>

## Summary

<div style="text-align: justify">
<p>&nbsp;&nbsp;&nbsp;&nbsp;Going back to the question for the post title, most of the thread pool induced deadlocks are the result of tasks starvation. That’s why, in order to avoid deadlocks you have to avoid situations where your tasks might never get a thread. First of all, you have to ensure that you’re using the right types of thread pools and you correctly assigned tasks to them. What’s more, configuring thread pools using machine specific attributes should be done carefully. On a high level, it’s always good to consider asynchronous approach.</p> 
</div>

## Sources

<ul>
    <li><a href="https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html">https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html</a></li>

    <li><a href="http://blog.jessitron.com/2014/01/fun-with-pool-induced-deadlock.html">http://blog.jessitron.com/2014/01/fun-with-pool-induced-deadlock.html</a></li>

    <li><i>Programming Concurrency on the JVM</i>, Venkat Subramanaim, The Pragmatic Programmers</li>

    <li><a href="http://www.drdobbs.com/parallel/use-thread-pools-correctly-keep-tasks-sh/216500409?pgno=1">http://www.drdobbs.com/parallel/use-thread-pools-correctly-keep-tasks-sh/216500409?pgno=1</a></li>
</ul>
