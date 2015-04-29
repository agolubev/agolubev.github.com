---
layout: post
title: "Example of reactive solution with scalaz-stream. Part 1"
date: 2015-04-26 14:16:38 -0400
comments: true
categories: scala scalaz-stream reactive
---

In this post I'm going to solve a problem of traversing remote tree structure asynchronously using scala. Two approaches will be implemented. First solution will use ```futures```, the second one will be based on event streams with help of [scalaz-stream][scalaz-stream] library. Both approaches will be reactive by some extent.<!-- more -->
___


Idea of reactive programming was expressed for the first time in 1997 in the paper Functional Reactive Animation by Conal Elliot and Paul Hudak. It's not a long ago actually. Still, since then we've got several stable frameworks and libraries for programming in reactive way.
Few names come to mind right away: [ReactiveX][reactivex], [Akka][akka]. [Scalaz-stream][scalaz-stream] is another one that's not so feature reach but is the closest to functional programming paradigm as was branched out from scalaz project. We'll start form pure scala solution and then solve the same problem with help of scalaz-stream. In the next post I'm going to compare these two solutions in detail.

Let's start from a problem description. Need to design a tool that traverses tree structure on a remote service and gets data associated with leafs. This is formalization of the problem solved in [Playinzoo][playinzoo] plugin. Plugin starts with Play framework, reads through tree structures of remote Zookeeper and returns result as configurational attributes. Zookeeper has only commands to get data associated with a node and obtain list of children for specific node. We can expect that algorithm will be pretty much like dealing with tree structures. As we'll work with remote service and probably load configuration for several subtrees it would be good to get data in parallel. That's when the fun begins. With having multiple threads doing something asynchronously we need to avoid mutable state or synchronize access to this state. 

Let's speculate on how to solve this. As the first step we can think about simplified problem without parallel loading, that is, no asynchronous calls. Naive approach in this case will be recursive iteration via tree model. We need to have function that does remote calls with error handling. Connection management (aka closing it at the right moment, timeouts) will be simple as well. 

With asynchronous loading things become complicated. To manage set of threads we'll need a thread pool explicitly or implicitly. Another problem we need to think about is when we should stop waiting for tasks that are running. As algorithm will be recursive with unknown tree depth we should consider some monitoring functionality. The simplest way here will be: incrementing counter when task is starting and decrementing when it's returning result. When counter is zero it means that there is no tasks running and we can return a result. In addition we'll need some queue to store results of remote calls (like paths to children nodes) and collection to accumulate loaded data from leafs.

###  First solution - pure scala
Ok, let's stop talking and start some coding. First we should decide on data types we are going to use:
```scala
type NodePath = String
type NodeData = String
type Node = (List[NodePath],Option[NodeData])
```

Type ```Node``` contains information we get from remote service. This can be list of children or node data if it's a leaf.
With this defined we can implement method for loading node information from remote service asynchronously.

```scala
  def loadNodeInfo(path: NodePath, results: BlockingQueue[Node]) = future {
    val node:Node = ??? //request remote service: get children, if leaf - get data
    node
  } onComplete {
    case Success(node) => results.add(node)
    case Failure(e) => results.add((Nil,None))
  }
```

Function ```loadNodeInfo``` creates ```future``` as task to load data from service. When future is done, result will be added to the blocking queue. Now it's time to implement function that actually do the job of creating futures and processing node information.

```scala
import java.util.concurrent.{LinkedBlockingQueue, BlockingQueue, Executors}
import java.util.concurrent.atomic.AtomicInteger
import scala.concurrent._
import scala.util.{Failure, Success}

implicit val ec = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(8))

def loadFromZk(paths: List[NodePath]): List[NodeData] = {

  //creating future as task to load data from Zk. When it's done - put the result into queue
  def loadNodeInfo(path:NodePath, results: BlockingQueue[Node]) = {...}

  //collection to accumulate loaded data
  var readProperties = List.empty[NodeData]

  //point of synchronization of loading tasks and monitor
  val zkLoadedResults = new LinkedBlockingQueue[Node]()
  zkLoadedResults.add((paths,None))

  //counter for the running tasks
  val runningTasksCounter = new AtomicInteger(paths.size)

  var node:Node = null
  //this is where monitor logic is waiting for tasks result
  while ( { node = zkLoadedResults.take(); node } != null) {
    runningTasksCounter.decrementAndGet()
    for(path <- node._1){
      runningTasksCounter.incrementAndGet()
      loadNodeInfo(path, zkLoadedResults)
    }
    for(data <- node._2) readProperties = data :: readProperties
    //if counter is 0 we done
    if (runningTasksCounter.get == 0) return readProperties
  }
  readProperties //actually we'll never rich this point
}
```
Function ```loadFromZk``` which does the magic. It uses ```zkLoadedResults``` as a queue for ```Node``` information. Method pulls out the next result, creates load task for each received path and adds data to result list in case it's available. ```runningTasksCounter``` counts how many tasks are now running. Actually in real life some of them are running but others are waiting for a vacant thread in a pool.

Synchronization between main thread and tasks from pool is done with help of ```BlockingQueue```. Potentially we can add timeout for getting the next result ```zkLoadedResults.take()```. Please note that mutable state (mutable data structures and variables) localized in one method - ```loadFromZk``` so solution still can be considered as functional.

### Second solution - based on scalaz-stream
Now let's try to solve the same problem with help of scalaz-stream library. We'll apply reactive approach to process remote service responses. In other words we'll implement event stream based on pull model. In this case we'll have following as events: node paths, node data, running task counter. As there is no much documentation for the library, we will look into second implementation in more detail. As first step  we will create stream with queue as a source.

```scala
import scalaz.stream._
import scalaz.\/._
import scalaz.\/

val pathsQueue = async.unboundedQueue[String] //creating queue
val queueStream: Process[Task, String] = pathsQueue.dequeue //creating stream

val echoStream = queueStream.map(a => { println(a); a }) //print events 
Task.fork(echoStream.runLog).runAsync( _ => () )
```
You can try this on REPL and find out that the last command returns nothing. Actually, it runs stream in a separate thread waiting for new elements in ```pathsQueue```. So if you type ```pathsQueue.enqueueOne("/firstPath").run``` in REPL you'll see ```"/firstPath"``` as output. Obviously, what we just saw was pulling nature of streams in reacting on some event.

So far so good. Let's add several transformations to the stream to call ```loadNodeInfo``` asynchronously and pass the result as event further down the stream. We are going to use the same ```Node``` type from the first implementation here as well.

```scala
import scalaz.concurrent._
import scalaz.stream._
import scalaz.{\/-, \/}
//the same data model
type NodePath = String
type NodeData = String
type Node = (List[NodePath], Option[NodeData])

val pathsQueue = async.unboundedQueue[NodePath]
val loadingStream: Process[Task, NodePath] = pathsQueue.dequeue

def loadNodeInfo(path: NodePath): Node = ???

val loadedData = loadingStream.flatMap[Task, Node](x =>
  Process.eval(Task.async {
    cb: ((Throwable \/ Node) => Unit) => cb(\/-(loadNodeInfo(x)))
  }))
```
Starting point for our stream as well as source for events is ```pathsQueue```. It contains node paths as instructions to load children or data (if it's a leaf) from remote service. Queue can be updated asynchronously and for simplicity has no bounds. The line ```Process.eval(Task.async{...})``` wraps task into a stream, then it runs async task that will do callback ```cb``` as soon as result is ready.

As next step we need to accumulate some statistics along with ```(List[NodePath], Option[NodeData])``` to emulate ```runningTasksCounter``` for tasks running in parallel. 

```scala
val dataAndStatistics = 
    loadedData.scan[(List[NodePath], Option[NodeData], Int)]((Nil, None, initialNumberOfPaths))((a, c) => {
  val stat = a._3 + c._1.size - 1
  println("* Currently tasks in pool " + stat)
  if (stat == 0) pathsQueue.close.run 
  (c._1, c._2, stat)
})
```
Important line here is one that sends ```close``` event to queue ```pathsQueue.close.run``` to let all stream activities shut down normally with special ```Complete``` event.
For scan function ```(a, r)``` attributes are like for ```foldLeft```. ```a``` is accumulated value, ```c``` is current pulled event.

At first I thought about splitting stream into two - one will be data oriented, another will gather statistics and shut down system. Unfortunately for now there is no such build-in function in scalaz-stream we can use. (In ReactiveX, for example, there is ```groupBy``` function that does what we want.)

Next we need to notify stream that there are other nodes to iterate. To do this we add new paths to the queue and return only ```Option[NodeData]``` as event for the next stream as we don't need anything else:
```scala
val nodeDataStream = dataAndStatistics.map(a => {
  if (a._1.nonEmpty) pathsQueue.enqueueAll(a._1).run
  a._2
})
```
Finally we filter out all events with no data and transform ```Stream[Option[NodeData]]``` to ```Stream[NodeData]```. After this we get all data from stream by calling ```runLog```. (Alternatively there are possibility to run ```runLast``` which returns only last event or ```run``` that returns ```Unit```.) ```runLog``` function returns ```Task``` object. As soon as we are going to run the task synchronously and wait for the result we should call ```runLog.run```.
```scala
val pureNodeDataStream = nodeDataStream.filter(_.isDefined).map(_.get)
val result: IndexedSeq[NodeData] = pureNodeDataStream.runLog.run
```

Cool! Let's now add some code to emulate remote call with few seconds delay and add some output to verify asynchronous loading:

```scala
def loadNodeInfo(path: NodePath): Node = {
  println("> Request info for node " + path + " at " + Thread.currentThread().getName)
  Thread.sleep(2000)
  println("< Received response for node " + path + " at " + Thread.currentThread().getName)
  val depth = path.count(_ == '/')
  if (depth < 3)
    (path + "/node" + depth :: Nil, None)
  else
    (Nil, Some("Data for " + path))
}

val paths = "/first" :: "/second" :: Nil
pathsQueue.enqueueAll(paths).run //enqueue paths for root node. enqueueAll returns task so need to call run
val initialNumberOfPaths = paths.size
```

Function ```loadNodeInfo``` emulates request/response to quite slow Zookeeper service. It emulates tree structure as simple chain with 2 folder nodes and data node at the end. The code snippet also adds two paths to queue ```/first``` and ```/second``` to iterate though these two subtrees.  

When we run [full code example][first-example] in REPL we'll see that ```loadNodeInfo``` calls are still not running in parallel:

```sh
> Request info for node /first at pool-1-thread-4
< Received response for node /first at pool-1-thread-4
* Currently tasks in pool 2
> Request info for node /second at pool-1-thread-5
< Received response for node /second at pool-1-thread-5
* Currently tasks in pool 2
> Request info for node /first/node1 at pool-1-thread-1
< Received response for node /first/node1 at pool-1-thread-1
* Currently tasks in pool 2
> Request info for node /second/node1 at pool-1-thread-7
< Received response for node /second/node1 at pool-1-thread-7
* Currently tasks in pool 2
> Request info for node /first/node1/node2 at pool-1-thread-1
< Received response for node /first/node1/node2 at pool-1-thread-1
* Currently tasks in pool 1
> Request info for node /second/node1/node2 at pool-1-thread-3
< Received response for node /second/node1/node2 at pool-1-thread-3
* Currently tasks in pool 0
resultStream: IndexedSeq[String] = Vector(Data for /first/node1/node2, Data for /second/node1/node2)
```
We can fix this by create Stream of Streams during ```loadingStream``` transformation and using ```nondeterminism.njoin``` to merge and send results as events to one stream.

```scala
val loadedData = nondeterminism.njoin(10, 1)(loadingStream.map[Process[Task, Node]](x =>
  Process.eval(Task.async {
    cb: ((Throwable \/ Node) => Unit) => cb(\/-(loadNodeInfo(x)))
  })))
```

Few words about magic numbers in njoin call: 10 is pool capacity and 1 is queue length for all tasks running in the pool. You can find final code [here][example-second]. When we run it we get the correct result:

```sh
> Request info for node /first at pool-1-thread-2
> Request info for node /second at pool-1-thread-5
< Received response for node /first at pool-1-thread-2
< Received response for node /second at pool-1-thread-5
* Currently tasks in pool 2
* Currently tasks in pool 2
> Request info for node /first/node1 at pool-1-thread-1
> Request info for node /second/node1 at pool-1-thread-6
< Received response for node /first/node1 at pool-1-thread-1
* Currently tasks in pool 2
> Request info for node /first/node1/node2 at pool-1-thread-4
< Received response for node /second/node1 at pool-1-thread-6
* Currently tasks in pool 2
> Request info for node /second/node1/node2 at pool-1-thread-5
< Received response for node /first/node1/node2 at pool-1-thread-4
* Currently tasks in pool 1
< Received response for node /second/node1/node2 at pool-1-thread-5
* Currently tasks in pool 0
resultStream: IndexedSeq[String] = Vector(Data for /first/node1/node2, Data for /second/node1/node2)
```
Finally we can see that requests are running in parallel, independently on each other. Consequently, as a result we got functional solution for our tree traversal problem that based on streams and is reactive by it's nature.

As summary I'll provide the list of links I found for getting into reactive programming and scalaz-stream:

* A list of good articles is available right on streaz-scala project on [Additional resources][stream-resources] page;
* When you're dealing with scalaz streams it's essential to understand scalaz type - Task. So you should look for short [tutorial][task-tutorial] about Task as well;
* Also there is a presentation with overview of scalaz-stream functions [available here][stream-presentation];
* And of course Coursera lectures [Principles of Reactive Programming][coursera] that give outstanding vision on this topic.

[reactivex]: http://reactivex.io/
[akka]: http://akka.io/
[playinzoo]:  https://github.com/agolubev/playinzoo
[scalaz-stream]:  https://github.com/scalaz/scalaz-stream
[stream-resources]: https://github.com/scalaz/scalaz-stream/wiki/Additional-Resources
[stream-tutorial]: https://www.chrisstucchio.com/blog/2014/scalaz_streaming_tutorial.html
[task-tutorial]: http://timperrett.com/2014/07/20/scalaz-task-the-missing-documentation/
[stream-presentation]: http://pchiusano.github.io/talks/scalaz-stream-nescala-2014/scalaz-stream-nescala-2014.html
[reverse-presentation]: https://dl.dropboxusercontent.com/u/1679797/NYT/Reactive%20in%20Reverse.pdf
[first-example]: https://gist.github.com/agolubev/13bc4c1bc865de97f64c
[example-second]: https://gist.github.com/agolubev/0c2430c750eaa4096436
[coursera]: https://class.coursera.org/reactive-002 