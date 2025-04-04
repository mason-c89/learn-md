# 什么是 elasticSearch

elastic Search, 也就是 es，是一个开源的搜索引擎。它介于**应用**和**数据**之间，只要将数据写入 es，应用就可以通过一些关键词搜索到数据。效果就像某度搜索一样。

## 工作原理

### 倒排索引
回到之前的例子，依次遍历文本进行匹配确实低效。那有更优解吗？有，我们可以对文本进行切分。比如“I am Mason”切分为“I”、“am”、“Mason”三部分。这个操作叫**分词**，分词后每部分，我们称之为一个**词项**，也就是**term**。记录词项和文本id的关系，于是之前的文本就变成这样子：

| term    | 文本 id |
| ------- | ------- |
| I       | 1, 2, 3 |
| am      | 1       |
| Mason   | 1, 2    |
| follow  | 2       |
| forward | 3       |
| the     | 3       |
| video   | 3       |

当我们想要搜索 `Mason` 的时候，只需要匹配到 `Mason` 词项，就可以立马得到它所在的文档 id 是 1 和 2。但这有个问题，短短三句话，就已经有这么多词项了，要是换成三篇文档，那词项就会多得离谱，怎么在这么多的词项里匹配出 Mason 呢？挨个遍历的话，时间复杂度就是 `O(N)`, 太低效了。

### 解决方案
我们可以将词项 **按字典序** 从小到大排序，通过二分查找的方法，直接将时间复杂度优化为 `O(lgN)`。就像下面这样：

| term    | 文档 id |
| ------- | ------- |
| follow  | 2       |
| forward | 3       |
| I       | 1, 2, 3 |
| am      | 1       |
| Mason   | 1, 2    |
| the     | 3       |
| video   | 3       |

我们将这堆排好序的词项，称为 **Term Dictionary**，而词项对应的文档 id 等信息的集合，就叫 **Posting List**。它们共同构成了一个用于搜索的数据结构，它就是**倒排索引(Inverted Index)**。

### 存储优化
但倒排索引还有个问题，Term Dictionary 数据量很大，放 **内存** 并不现实，因此必须放在 **磁盘** 中。但查询磁盘是个较慢的过程。有优化手段吗？有，我们聊下 **Term Index**。

### Term Index 是什么
我们可以发现，词项和词项之间，有些 **前缀** 是一致的，比如 `follow` 和 `forward` 前面的 `fo` 是一致的，如果我们将部分 term 前缀提取出来，就可以用更少的空间表达更多的 **term**。基于这个原理，我们可以将 Term Dictionary 的 **部分** 词项提取出来，用这些 词项 的前缀信息，构建出一个 **精简的目录树**。目录树的节点中存放这些词项在磁盘中的偏移量，也就是指向磁盘中的位置。这个目录树结构，体积小，适合放内存中，它就是所谓的 **Term Index**。可以用它来加速搜索。

### Stored Fields 是什么
到这里，搜索功能就有了。但有个问题，前面提到的倒排索引，搜索到的是 **文档 id**，我们还需要拿着这个 id 找到 **文档内容本身**，才能返回给用户。因此还需要有个地方，存放完整的文档内容，它就是 **Stored Fields**（行式存储）。

### Doc Values 是什么
有了 id，我们就能从 Stored Fields 中取出文档内容。

但用户经常需要根据某个字段排序文档，比如按时间排序或商品价格排序。但问题就来了，这些字段散落在文档里。也就是说，我们需要先获取 Stored Fields 里的文档，再提取出内部字段进行排序。也不是说不行。

但其实有更高效的做法。我们可以用 **空间换时间** 的思路，再构造一个 **列式存储** 结构，将散落在各个文档的某个字段， **集中** 存放，当我们想对某个字段排序的时候，就只需要将这些集中存放的字段一次性读取出来，就能做到针对性地进行排序。这个列式存储结构，就是所谓的 **Doc Values**。

### segment
在上文中，我们介绍了四种关键的结构： **倒排索引** 用于搜索， **Term Index** 用于加速搜索， **Stored Fields** 用于存放文档的原始信息，以及 **Doc Values** 用于排序和聚合。这些结构共同组成了一个 **复合** 文件，也就是所谓的 " **segment**", 它是一个具备 **完整搜索功能的最小单元**。

### lucene 是什么
我们可以用多个文档生成一份 segment， **如果** 新增文档时，还是写入到这份 segment，那就得同时更新 segment 内部的多个数据结构，这样并发读写时性能肯定会受影响。那怎么办呢？我们定个规矩。 **segment 一旦生成，则不能再被修改**。如果还有新的文档要写入，那就生成新的 segment。这样 **老的 segment 只需要负责读，写则生成新的 segment**。同时保证了读和写的性能。

但 segment 变多了，你怎么知道要搜索的数据在哪个 segment 里呢？问题不大， **并发同时读** 多个 segment 就好了。

但这也引入了新问题，随着数据量增大，segment 文件越写越多，文件句柄被耗尽那是指日可待啊。是的，但这个也好解决，我们可以不定期合并多个小 segment，变成一个大 segment，也就是 **段合并**(segment merging)。这样文件数量就可控了。

到这里，上面提到的多个 segment，就共同构成了一个 **单机文本检索库**，它其实就是非常有名的开源基础搜索库 **lucene**。不少知名搜索引擎都是基于它构建的，比如我们今天介绍的 ES。但这个 lucene 属实过于简陋，像什么高性能，高扩展性，高可用，它是一个都不沾。

### 高性能
lucene 作为一个搜索库，可以写入大量数据，并对外提供搜索能力。多个调用方 **同时读写** 同一个 lucene 必然导致争抢计算资源。抢不到资源的一方就得等待，这不纯纯浪费时间吗！有解决方案吗？有！首先是对写入 lucene 的数据进行分类，比如体育新闻和八卦新闻数据算两类，每一类是一个 **Index Name**，然后根据 Index Name 新增 lucene 的数量，将不同类数据写入到不同的 lucene 中，读取数据时，根据需要搜索不同的 Index Name 。这就大大降低了单个 lucene 的压力。

但单个 Index Name 内数据依然可能过多，于是可以将单个 Index Name 的同类数据，拆成好几份，每份是一个 **shard 分片**， **每个 shard 分片本质上就是一个独立的 lucene 库**。这样我们就可以将读写操作分摊到多个 分片 中去，大大降低了争抢，提升了系统性能。

### 高扩展性
随着 分片 变多，如果 分片 都在同一台机器上的话，就会导致单机 cpu 和内存过高，影响整体系统性能。

于是我们可以申请更多的机器，将 分片 **分散** 部署在多台机器上，这每一台机器，就是一个 **Node**。我们可以通过增加 Node 缓解机器 cpu 过高带来的性能问题。

### 高可用
到这里，问题又又来了，如果其中一个 Node 挂了，那 Node 里所有分片 都无法对外提供服务了。我们需要保证服务的高可用。有解决方案吗？有，我们可以给 分片 **多加几个副本**。将 分片 分为 **Primary shard** 和 **Replica shard**，也就是主分片和副本分片 。主分片会将数据同步给副本分片，副本分片 **既可以** 同时提供读操作， **还能** 在主分片挂了的时候，升级成新的主分片让系统保持正常运行， **提高性能** 的同时，还保证了系统的 **高可用**。

### Node 角色分化
搜索架构需要支持的功能很多，既要负责 **管理集群**，又要 **存储管理数据**，还要 **处理客户端的搜索请求**。如果每个 Node **都** 支持这几类功能，那当集群有数据压力，需要扩容 Node 时，就会 **顺带** 把其他能力也一起扩容，但其实其他能力完全够用，不需要跟着扩容，这就有些 **浪费** 了。因此我们可以将这几类功能拆开，给集群里的 Node 赋予 **角色身份**，不同的角色负责不同的功能。比如负责管理集群的，叫 **主节点(Master Node)， 负责存储管理数据的，叫数据节点(Data Node)， 负责接受客户端搜索查询请求的叫协调节点(Coordinate Node)。集群规模小的时候，一个 Node 可以同时** 充当多个角色，随着集群规模变大，可以让一个 Node 一个角色。**随着集群规模的增大，可以让每个Node只承担一个角色**，以优化资源分配和提高效率。

### 去中心化
在Elasticsearch中，为了避免单点故障和提高系统的可扩展性，采用了去中心化的架构。这意味着没有单一的控制节点来管理整个集群的状态。相反，集群中的所有节点都参与到集群状态的管理和决策过程中。这通常通过使用像Raft这样的一致性算法来实现，确保所有节点对集群状态有一致的视图。

### Elasticsearch的写入流程
1. **客户端写入请求**：当客户端应用发起数据写入请求时，请求首先到达集群中的协调节点。
2. **路由和写入**：协调节点根据数据的路由信息，确定数据应该写入到哪个数据节点的哪个分片（Shard）。数据首先写入主分片，该分片将数据存储在底层的Lucene索引中，形成倒排索引、Stored Fields和Doc Values。
3. **同步副本分片**：主分片写入成功后，数据会被同步到所有的副本分片。
4. **确认和响应**：当所有的副本分片都成功写入数据后，主分片会向协调节点发送一个确认（ACK），表示写入操作完成。
5. **写入完成**：最后，协调节点向客户端应用返回写入完成的响应。

### Elasticsearch的搜索流程
Elasticsearch的搜索流程分为两个阶段：查询阶段和获取阶段。
- **查询阶段**：
  - 客户端应用发起搜索请求，请求到达协调节点。
  - 协调节点根据Index Name信息，将请求转发到相关的数据节点和分片。
  - 每个分片使用其底层Lucene索引的倒排索引进行并发搜索，找到匹配的文档ID，并使用Doc Values进行排序。
  - 分片将搜索结果和排序信息返回给协调节点，协调节点对结果进行聚合和排序，然后舍弃不需要的数据。

- **获取阶段**：
  - 协调节点使用文档ID从数据节点的分片中检索完整的文档内容。
  - 分片从其Stored Fields中检索文档内容，并将其返回给协调节点。
  - 协调节点将最终的搜索结果返回给客户端应用，完成搜索过程。
