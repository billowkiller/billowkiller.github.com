看了 druid contributor [FANGJIN YANG](http://imply.io/post/2015/11/04/big-data-zoo.html) 的技术博客觉得颇为有趣，结合之前看的几篇文章，在这里总结记录下。

运行多个数据基础服务系统类似于在计算机中运行多个运用程序，每个程序都被设计来处理一个特定的问题，并没有什么万金油方案，流行的lambda架构也会为人诟病。

在基础数据管理技术的领域内，大致可以分成四个类别：`delivery, storage, processing, and querying`。根据用例和需求，用这四种技术一种或多种组合可以满足可扩展性的端到端数据服务。

## Big Picture

互联网中的原始数据很多都是存储在服务器日志中，日志代表了事件，可能是一个点击、一次消费等。记录下来的事件都是只读的，并且事件流中的事件数目不可知。这些事件都是极为重要的，富含商业价值，可以说是互联网数据的基石，因此正确的传输来的尤为重要。正确的传输包括两个方面，**一是传输到正确的处理程序，二是不重不丢**。

如果事件传输到处理系统中，那么接下来发生的就是典型的ETL（extracts, transforms, and loads）过程。这个过程的目的是清理和丰富事件，使得后期的分析更加容易。转化后的事件可以直接存储或者进入 query 系统。第一种方式对应存入分布式文件系统，这是为了以后的使用。第二种方式对应数据库，在这里存储和查询被模糊化，因为大多数数据库都是支持存储和查询的，但现在越来越多的系统中存储和查询层会分离，目的在于解耦后获得的高可用和高性能。

![](http://imply.io/image/blog-assets/flow.png)

上图是一个完整的流程，当然在实际应用中有些步骤会被省略。另外正如前文所说的，数据基础服务系统不是只有一个，每个步骤中的组件可能也不止一个，但它们并不是 `either-or` 的关系，需要仔细对照组件功能和用例进行选择。

## Delivery

前文有提到，传输的要义是传输到正确的处理程序和不重不丢。当然，后者的要求颇高，需要在性能和功能之间做 trade-off，通常的做法可能是 at-most-once 或 at-least-once，并在业务层做写特殊的处理。代表性的组件有： Apache Kafka、RabbitMQ 和 Apache Flume。

前两者都是使用 pub-sub 的模式，Kafka 是基于 transaction logs 的，而 RabbitMQ 是基于分布式队列的。Kafka在我的[这篇文章](http://billowkiller.com/blog/2016/04/06/kafka/)中有详细的介绍。Flume则是通过 push-event 的方式进行消息传输，可以对接到 Hdfs 或 Hbase。

还有其他许多的日志系统和消息传输系统 [Message Broker](https://en.wikipedia.org/wiki/Message_broker)。或者可以单独使用，或者可以拼接一块，但是目的都是传输日志。

## Storage

存储作为数据服务的中心其实一点都不为过。任何的数据终归都是要存储下来，供以后的处理和查询。同样的，存储的形式也是最为多变，它和其他分类的 overlap 也是最多的。例如前文提到的查询，以及上节中的传输，Kafka 本身就提供数据存储服务，只要 retention 时间够久，数据都可以重复展现。这里面用的最多的应该就是 HDFS 了。

## Processing

从计算层面上过来说， processing 和 querying 可能会有一定的 overlap，可能都会通过一些计算引擎或者说算子处理数据。但是二者的目的并不一样，processing system 更多的是在转化数据，如上文中所提到的 ETL， 另外对于数据量来说，通常输入和输出会比较类似；querying system 的输出通常远小于输入，因为数据会经过搜索、过滤和聚合。二者在数据管理服务中通常也会分开，一个要求更多的长期运行的稳定性和吞吐量，另一个要求低延迟。processing system 也会分为两种：流式和批量。

### Batch Processors

![](http://imply.io/image/blog-assets/batch-processing.png)

批量处理会操作大量的静态数据，也可以看到输入和输出都是对接存储系统的。因为对接的是存储系统，所以批量处理有一个天然的好处是数据可重放；而且批量处理带来的准确率在普遍意义上来说要高于流式处理，因为流式系统或者实时系统会遇到许多意外的情况，可以看到[facebook的实时数据处理](http://billowkiller.com/blog/2016/07/13/realtime-data-processing-at-facebook/)。

典型的批处理是Hadoop MapReduce，但由于算法的普适性，它对于一些工作流的支持并不大好，特别是迭代计算和热点计算。这点可以看 Spark 和 Tez。

### Stream Processors

![](http://imply.io/image/blog-assets/stream-processing.png)

从应用场景中可以看到流式处理输入是传输系统，例如 Flume 和 Kafka。数据是被处理 `cleaned up, enriched, or otherwise transformed` 后再进入存储系统的。典型的系统是 Samza, Storm, Spark Streaming，在[facebook的实时数据处理](http://billowkiller.com/blog/2016/07/13/realtime-data-processing-at-facebook/)中多有介绍，这里就不再赘述。

在流式处理中可以完成简单的查询、聚合、分析功能，但并适用于 querying system，更多的是配合使用。流式处理被用来做 ETL，查询系统用于细节分析。

## Querying

Querying 应该是在这四个里面比重占据最大的一块，因为之前的一切处理都是为了查询服务的。查询的主要



![](http://imply.io/image/blog-assets/radstack.png)


