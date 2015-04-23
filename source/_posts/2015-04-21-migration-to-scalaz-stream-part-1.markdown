---
layout: post
title: "Migration to scalaz-stream. Part 1"
date: 2015-04-21 22:07:25 -0400
comments: true
categories: scala scalaz-stream reactive
---

Idea of reactive programming was expressed for the first time in 1997, not long ago. Since then we've got several stable frameworks and libraries for building code with this approach.
Few names come to mind right away (at least for something that is runnable on JVM): [ReactiveX][reactivex], [Akka][akka]. The one that is not so functional reach but in active development is [scalaz-stream][scalaz-stream]. 
The last one is the closest to functional programming paradigm as branced out from scalaz project. And that's tha main reason I'll use it furhter for solving a problem.

Below I'll describe the problem that was solved initially without reactive programming and then will show step by step how to deal with the problem with scalaz-stream library. In the second part Reader monad will be added to implementation and perhaps I'll make some comparison of old non reactive and reactive approaches.<!-- more -->
___

Let's start from problem description. Need to design a tool for iterate over remote tree and get values associated with leafs. This is formalization of problem solved in [playinzoo][playinzoo] plugin. Plugin starts with Play framework, reads through tree strutures of remote Zookeeper and returns result  as play configurational attributes. Zookeeper has only commands for getting data associating with a node and getting children of specific node. So algorithm will be pretty much like dealing with file system structure with folders and files. As we are working with remote service and probably will load configuration for several subtrees it would be good to load data in parallel. Another requirement is implementing reactive approach at the end with no mutable state. 

Solution would be simple without parall loading. Naive approach here is recursive iteration via tree model. Maybe will be good having remote calls in one method which provides error handling and proper timeout. Connection management (aka closing it at the right time) will be simple as well. 

With parallel loading we'll need to rely on thread pool with capacity from 1 to many. Consequently implementation has to be flexible here and be able to run with only one thread availabe. 

Another problem we need to think about is when to stop waiting for tasks that are running in pool. As algorithm will be recursive with unknown tree depth, we need to consider some monitoring functionality. The simplest way here is incrementing counter when task is starting and decrementing when it's return result. Also we'll need some collection to add loaded data there

Ok, let's stop talking and start some coding. Here is the simple solution written in scala, not reactive though.

```scala
import java.util.concurrent.atomic.AtomicInteger
import scala.concurrent._
import scala.util.{Failure, Success}
import scala.collection.mutable

implicit val ec = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(8))
type Path = String
type NodeData = String
type Node = (List[Path],Option[NodeData]) //result of requesting node info

def loadFromZk(paths: List[String]): List[String] = {
    //collection to accumulate loaded data
    var readProperties = List.empty[String]
 
    //point of syncronization of loading tasks and monitor
    val zkLoadingResults = new LinkedBlockingQueue[Node]()
    zkLoadingResults.add((paths,None))

    //counter for the running tasks
    val runningTasksCounter = new AtomicInteger(paths.size)  

    var node:Node = null
    //this is where monitor logic is waiting for tasks result  
    while ( { node = zkLoadingResults.take(); node } != null) { 
      runningTasksCounter.decrementAndGet() 
      for(path <- node._1){
          runningTasksCounter.incrementAndGet() 
          loadNodeInfo(path, zkLoadingResults)
      }
      for(data <- node._2) readProperties = data :: readProperties
      //if counter is 0 we done 
      if (runningTasksCounter.get == 0) return readProperties
    }
    readProperties //actually will never rich this point
}
//creating future as task to load data from Zk. When it's done - put the result into queue
def loadNodeInfo(path:String, results: BlockingQueue[Node]) = future {
      val children:List[String] = getChildren(path)
      if (children.isEmpty)
         (Nil,getData(path))
      else
         (children,None)
    } onComplete {
      case Success(node) => results.add(node)
      case Failure(e) => results.add((Nil,None))
    }
def getChildren(path: String): List[String] = ??? // Function that actually requests Zk
def getData(path: String): Option[String] = ??? // Getting data from remote Zk sevice
```

Syncroniation between main thread (that doing monitoring) and tasks from thread pool is done with help of LinkedBlockingQueue. Potentially we can add timout on waiting the next result. To make the solution even better we can include other functions into loadFromZk in this case we'll have state localized in function. Consquntly this would be non strict but functional. 

Now let's try to implement the same in reactive way. First step will be to create stream with queue as a source.
```scala
import scalaz.stream._
import scalaz.\/._
import scalaz.\/

val pathsQueue = async.unboundedQueue[String] //creating queue
val queueStream: Process[Task, String] = pathsQueue.dequeue //creating stream

val effectStream = queueStream.map(a => { println(a); a }) //show output to test
Task.fork(effectStream.runLog).runAsync( _ => () )
```
You can try this on REPL and find out that last command returns nothing. Actually it run stram in separate thread waiting for have anything in pathsQueue. So if you type ```pathsQueue.enqueueOne("/firstPath").run``` in REPL you'll see "/firstPath" as output. So far so good.
Let's add several stream stransformations to get the same result as in non reactive example.

```scala
import scalaz.concurrent._
import scalaz.stream._
import scalaz.\/._
import scalaz.\/

type Path = String
type NodeData = String
type Node = (List[Path],Option[NodeData])

def loadNodeInfo(path: String): Node = ???

val pathsQueue = async.unboundedQueue[String]

val loadingStream: Process[Task, String] = pathsQueue.dequeue

val loadedData = loadingStream.flatMap[Task,NodeInfo](x => Process.eval(Task.async {
  cb: ((Throwable \/ NodeInfo) => Unit) =>
    try {
      cb(right(loadNodeInfo(x)))
    } catch {
      case t: Throwable => println("Trouble"); cb(left(t))
    }
}))
```
Now need to accumulate some statistics along with ```(List[Path], Option[NodeData])``` to emulate counter for tasks in progress. Also we need to notify queue and stream that there is another node to iterate:
```scala
val dataAndStatistics = loadedData.scan[(List[Path], Option[NodeData], Int)]((Nil, None, initialNumberOfPaths))((b, o) => {
  val stat = b._3 - 1 + o._1.size
  println("Statistics: currently tasks in pool " + stat)
  if (stat == 0) pathsQueue.close.run
  (o._1, o._2, stat)
})
```
Next step will be to add all children paths to initial queue:
```scala
val downloadedProperties = dataAndStatistics.map(a => {
  if (a._1.nonEmpty) pathsQueue.enqueueAll(a._1).run
  (a._2)
})
```
Finally we need to filter out all data Nodes that we have for folders. Also we need to get all data from stream by calling runLog. As runLog returns task 
we need to execute this syncronously as want to wait for result.
```scala
val pureProperties = downloadedProperties.filter(_.isDefined).map(_.get)
val resultStream: IndexedSeq[String] = pureProperties.runLog.run
```
Cool! Let's now add some code to emulate remote call with some delays. Also we need to add some output to see asyncronos loading:
```scala
val counter = new AtomicInteger(0)
def loadNodeInfo(path: String): NodeInfo = {
  println("Request for "+path + " at "+ Thread.currentThread().getName())
  Thread.sleep(2000)
  println("Received response for "+path + " at "+ Thread.currentThread().getName())
  if (counter.get() < 5) {
    counter.incrementAndGet()
    (path + "/path"+counter.get() :: Nil, None)
  } else {
    (Nil, Some("Data for "+path))
  }
}

val paths="/first"::"/second"::Nil
pathsToLoadQueue.enqueueAll(paths).run
val initialNumberOfPaths=paths.size
```
The snippet adds two paths to queue /first and /second to load node info for them by iterating thourh tree structure. Tree structure in this case is simple chain with 5 nodes and data node at the end. At the same time function loadNodeInfo emulates requesting slow Zookeeper service for us to see asyncronous behaviour of reactive code. 

But when I run it int REPL (functioning code is available [here][first-example]) I can see that call of loadNodeInfo is still syncronized
```
Request for /first at pool-1-thread-2
Received response for /first at pool-1-thread-2
Statistics: currently tasks in pool 2
Request for /second at pool-1-thread-2
Received response for /second at pool-1-thread-2
Statistics: currently tasks in pool 2
Request for /first/path1 at pool-1-thread-1
Received response for /first/path1 at pool-1-thread-1
Statistics: currently tasks in pool 2
Request for /second/path2 at pool-1-thread-7
Received response for /second/path2 at pool-1-thread-7
Statistics: currently tasks in pool 2
Request for /first/path1/path3 at pool-1-thread-8
Received response for /first/path1/path3 at pool-1-thread-8
Statistics: currently tasks in pool 2
Request for /second/path2/path4 at pool-1-thread-5
Received response for /second/path2/path4 at pool-1-thread-5
Statistics: currently tasks in pool 1
Request for /first/path1/path3/path5 at pool-1-thread-2
Received response for /first/path1/path3/path5 at pool-1-thread-2
Statistics: currently tasks in pool 0
resultStream: IndexedSeq[String] = Vector(Data for /second/path2/path4, Data for /first/path1/path3/path5)
```
We can fix this by using nondeterminism.njoin. So after adding this and running [code][example-second] we'll get:
```
Request for /first at pool-1-thread-6
Request for /second at pool-1-thread-8
Received response for /second at pool-1-thread-8
Received response for /first at pool-1-thread-6
Statistics: currently tasks in pool 2
Statistics: currently tasks in pool 2
Request for /second/path1 at pool-1-thread-5
Request for /first/path2 at pool-1-thread-2
Received response for /second/path1 at pool-1-thread-5
Statistics: currently tasks in pool 2
Request for /second/path1/path3 at pool-1-thread-8
Received response for /first/path2 at pool-1-thread-2
Statistics: currently tasks in pool 2
Request for /first/path2/path4 at pool-1-thread-1
Received response for /second/path1/path3 at pool-1-thread-8
Statistics: currently tasks in pool 2
Request for /second/path1/path3/path5 at pool-1-thread-3
Received response for /first/path2/path4 at pool-1-thread-1
Statistics: currently tasks in pool 1
Received response for /second/path1/path3/path5 at pool-1-thread-3
Statistics: currently tasks in pool 0
resultStream: IndexedSeq[String] = Vector(Data for /first/path2/path4, Data for /second/path1/path3/path5)
```
Finally we are seeing that requests are going in parallel and in async mode wihtout syncronization with each other.

As summary I'll provide the list of links I used for looking into reactive programming and scalaz-stream:

* A list of good articles is available right on stremz-scala project on [Additional resources][stream-resources] page
* When you're dealing with scalaz streams it's essential to understand type Task that scalaz provides - [link to tutorial][task-tutorial]
* Aslo there is presentation with overview of scalaz-strem functions [available here][stream-presentation]
* And of cause Coursera course [Principles of Reactive Programming][coursera] that gives outstanding vision on this topic

[reactivex]: http://reactivex.io/
[akka]:	http://akka.io/
[playinzoo]:	 https://github.com/agolubev/playinzoo
[scalaz-stream]:  https://github.com/scalaz/scalaz-stream
[stream-resources]: https://github.com/scalaz/scalaz-stream/wiki/Additional-Resources
[stream-tutorial]: https://www.chrisstucchio.com/blog/2014/scalaz_streaming_tutorial.html
[task-tutorial]: http://timperrett.com/2014/07/20/scalaz-task-the-missing-documentation/
[stream-presentation]: http://pchiusano.github.io/talks/scalaz-stream-nescala-2014/scalaz-stream-nescala-2014.html
[reverse-presentation]: https://dl.dropboxusercontent.com/u/1679797/NYT/Reactive%20in%20Reverse.pdf
[first-example]: https://gist.github.com/agolubev/13bc4c1bc865de97f64c
[example-second]: https://gist.github.com/agolubev/0c2430c750eaa4096436
[coursera]: https://class.coursera.org/reactive-002 