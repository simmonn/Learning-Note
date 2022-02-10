## 1、SEGMENT merge（段合并）概述



![文档入库到段抽象架构图](https://tva1.sinaimg.cn/large/008i3skNgy1gy9pnp3nzqj314y0n077r.jpg)

在lucene中单个倒排索引文件被称为Segment。Segment 具有不可变性，多个 Segments 汇总在一起，称为 Lucene 的Index，其对应的就是 ES 中的 Shard。一个大segment的merge操作是很消耗CPU、IO资源的，如果使用不当会影响到本身的serach查询性能。es默认会控制merge进程的资源占用以保证merge期间search具有足够资源。

一个Index会由一个或多个sub-index构成，sub-index被称为Segment。Lucene的Segment设计思想，与LSM类似但又有些不同，继承了LSM中数据写入的优点，但是在查询上只能提供近实时而非实时查询。

Lucene中的数据写入会先写内存的一个Buffer（**只能写，不能读**），当Buffer内数据到一定量后会被flush成一个Segment，每个Segment有自己独立的索引，**可独立被查询**，但数据永远**不能被更改**，这也就是为什么Lucene被称为提供**近实时而非实时查询**的原因。Segment中写入的文档不可被修改，但可被删除。Index的查询需要对一个或者多个Segment进行查询，并对结果进行合并，还需要处理被删除的文档，为了对查询进行优化，Lucene会采用段合并策略对多个Segment进行合并。



#### 在分段思想下，对数据的写操作如下：

- 新增。当有新的数据需要创建索引时，由于段的不变性，所以选择新建一个段来存储新增的数据。
- 删除。当需要删除数据时，由于数据所在的段只可读，不可写，所以Lucene在索引文件下新增了一个.del的文件，用来专门存储被删除的数据id。当查询时，被删除的数据还是可以被查到的，只是在进行文档链表合并时，才把已经删除的数据过滤掉。被删除的数据在进行段合并时才会真正被移除。
- 更新。更新的操作其实就是删除和新增的组合，先在.del文件中记录旧数据，再在新段中添加一条更新后的数据。

#### Merge的触发器



```java
  /**
   * segment flush.
   */
  SEGMENT_FLUSH,

  /**
   * commit, NRT reader reopen or a close call on the index writer.
   */
  FULL_FLUSH,

  /**
   * 显式触发
   */
  EXPLICIT,

  /**
   * 合并结束
   */
  MERGE_FINISHED,

  /**
   * IndexWriter关闭时.
   */
  CLOSING,

  /**
   * commit.
   */
  COMMIT,
  /**
   * 打开NRT readers，8以后增加.
   */
  GET_READER
```

## 3、merge流程

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gyawpuw08oj30u00wwgoq.jpg" alt="段合并流程图" style="zoom:80%;" />

#### 执行段合并流程图

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gy8popsb8lj31da0sywhz.jpg" alt="image-20220109184702912" style="zoom:50%;" />

## 4、merge策略

段合并可视化过程:https://blog.mikemccandless.com/2011/02/visualizing-lucenes-segment-merges.html

- LogMergePolicy（Lucene4 之前默认）总是合并相邻的段文件，合并相邻的段文件（Adjacent Segment）描述的是对于IndexWriter提供的段集，LogMergePolicy会选取连续的部分(或全部)段集区间来生成一个待合并段集
- TieredMergePolicy（Lucene4以后默认）中会先对IndexWriter提供的段集进行排序，然后在排序后的段集中选取部分（可能不连续）段来生成一个待合并段集，即非相邻的段文件（Non-adjacent Segment）。

   如果用一句话来描述合并策略TieredMergePolicy的特点的话，那就是：找出大小接近且最优的段集。

这里主要介绍TieredMergePolicy

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gyax6kxfshj30u01jlgo9.jpg" alt="合并策略" style="zoom:50%;" />

#### 开始：

当IndexWriter对索引有任意的更改都会调用合并策略。

#### 段集：

IndexWriter提供段集给合并策略

#### 预处理：

预处理的过程分为4个步骤，分别是排序、过滤正在合并的段、过滤大段、计算索引最大允许段的个数。

- **排序**算法为TimSort：排序规则为比较每个段中索引文件的大小，不包括被删除的文档的索引信息，大段排前。
- **过滤正在合并的段**：当IndexWriter获得一个oneMerge后，会使用后台线程对oneMerge中的段进行合并，那么这时候索引再次发生更改时，IndexWriter会再次调用TieredMergePolicy，可能会导致某些已经正在合并的段被处理为一个新的oneMerge，**为了防止重复合并，需要过滤那些正在合并中的段**。后台合并的线程会将正在合并的段添加到Set对象中，在IndexWriter调用合并策略时传入。
- **过滤大段(Large Segment)**：
  - 大段的定义：该段的SegmentSize ≥ (maxMergedSegmentBytes / 2) 并且满足 段集中的被删除文档的索引信息大小占总索引文件大小的比例totalDelPct ≤ deletesPctAllowed 或 该段中被删除文档的索引信息大小占段中索引文件大小的比例segDelPct ≤ deletesPctAllowed
- **计算索引最大允许段的个数**：allowedSegCount：该值描述了段集内每个段的大小SegmentSize是否比较接近(segments of approximately equal size)，根据当前索引大小来估算当前索引中"应该"有多少个段，如果实际的段个数小于估算值，那么说明索引中的段不满足差不多都相同（approximately equal size），那么就不会选出OneMerge

#### 段集中可以得到OneMerge

如果同时满足下面三个条件，那么说明段集中可以得到OneMerge：

- MergeType：合并类型，即上文中的MERGE_TYPE，必须是NATURAL类型
- SegmentNumber：段集中段的个数，如果SegmentNumber ≤ allowedSegCount
- remainingDelCount：剩余段集中被删除文档的总数，如果remainingDelCount ≤ allowedDelCount

##### 找出一个OneMerge

顺序遍历段集，先预判下添加一个新的段后，OneMerge的大小是否会超过maxMergedSegmentBytes，如果超过，那么就跳过这个段，继续添加下一个段，目的是使这个OneMerge的大小尽量接近maxMergedSegmentBytes，因为段集中的段是从大到小排列的，当前前提是OneMerge中段的个数不能超过mergeFactor。

###### 举个栗子吧

![IndexWriter提供的未处理的段集](https://tva1.sinaimg.cn/large/008i3skNgy1gyayacdug6j31ux0u0dis.jpg)

>  从段1开始，逐个添加到OneMerge中，当遍历到段5时发现，如果添加段5，那么OneMerge的大小，即19 (段1) + 18 (段2)+ 16 (段3) + 15 (段4) + 15 (段5) = 83，该值大于 maxMergedSegmentBytes (80)，那么这时候需要跳过段5，往后继续找，同理段6、段7都不行，直到遍历到段8，OneMerge的大小为19 (段1) + 18 (段2)+ 16 (段3) + 15 (段4) + 7 (段8) = 75，那么可以将段8添加到OneMerge中，尽管段9添加到OneMerge以后，OneMerge的大小为 19 (段1) + 18 (段2)+ 16 (段3) + 15 (段4) + 7 (段8) + 4 (段9) = 79，还是小于maxMergedSegmentBytes (80)，但是由于OneMerge中段的个数会超过mergeFactor (5)，所以不能添加到OneMerge中，并且停止遍历

![段集2](https://tva1.sinaimg.cn/large/008i3skNgy1gyayccnoe9j31v00u0q68.jpg)

##### 对上面找到的OneMerge打分

打分公式为：<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gyaymqoahpj30wi03e74i.jpg" alt="打分公式" style="zoom:67%;" /> ，mergeScore越小越好（smaller mergeScore is better)。

- skew：粗略的计算OneMerge的偏斜值(Roughly measure "skew" of the merge)，衡量OneMerge中段之间大小的是否都差不多相同，如果OneMerge中段的大小接近maxMergedSegmentBytes，即hitTooLarge为true，那么 ，否则 ，, 其中MaxSegment为OneMerge中最大的段的大小，SegmentSize为每一个段的大小，maxMergedSegmentBytes在上文中已经介绍。
- totAfterMergeBytes：该值是OneMerge中所有段的大小，这个参数描述了段合并比较倾向于(Gently favor )较小的OneMerge
- nonDelRatio：该值描述了OneMerge中所有段包含被删除文档比例，这里就不详细给出计算公式了，nonDelRatio越小说明OneMerge中包含更多的被删除的文档，该值相比较totAfterMergeBytes，对总体打分影响度更大，因为段合并的一个重要目的就是去除被删除的文档(Strongly favor merges that reclaim deletes)

最终打分公式：

![](https://tva1.sinaimg.cn/large/008i3skNgy1gyayo8cj85j31pe06iabk.jpg)

#### 替换次有OneMerge

当前层中只允许选出一个OneMerge，即mergeScore最低的OneMerge。

#### 没有新的OneMerge

遍历的对象是 段1~段12，并且选出了一个OneMerge，接着我们需要再次从 段2~段12 中选出一个OneMerge后，再从段3~段12中再找出一个OneMerge，如此往复直到找不到新的OneMerge

```
bestScore != null && hitTooLarge == false && SegmentNum < mergeFactor
```

- bestScore != null：bestScore如果为空，说明当前还没有产生任何的OneMerge，那么肯定会生成一个OneMerge
- hitTooLarge == false：如果bestScore不为空，hitTooLarge为true，也要生成一个OneMerge。
- 剩余段集个数：bestScore不为空，hitTooLarge为false，如果剩余段集个数SegmentNum小于mergeFactor就不允许生成一个OneMerge

#### 段集中剔除最优OneMerge包含的段

层内只能选出一个OneMerge，那么从段集中剔除最优，即打分最低的OneMerge中包含的段，新的段集作为新的一层继续处理。假如当前层内最优的OneMerge是从段3~段12中选出的，那么下一层的可处理的段集如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNgy1gyaz1u2fe4j31ag0luta6.jpg)

## 5、talk is cheap, show me the code

中间不重要代码已经删除

#### 1>将缓存写入到段

IndexWriter在添加文档的时候调用函数addDocuments(Iterable<? extends Iterable<? extends IndexableField>> docs)，包含如下步骤：

- org.apache.lucene.index.IndexWriter#addDocuments
  - org.apache.lucene.index.DocumentsWriter#updateDocuments  //返回为负数，触发maybeMerge
    - org.apache.lucene.index.DocumentsWriter#updateDocuments
      - org.apache.lucene.index.DocumentsWriterFlushControl#doAfterDocument
  - org.apache.lucene.index.IndexWriter#maybeProcessEvents
    - org.apache.lucene.index.IndexWriter#processEvents
      - org.apache.lucene.index.IndexWriter#maybeMerge //merge流程

当缓存flush到磁盘，形成了新的段后，就有可能触发一次段合并，所以调用maybeMerge()

maxBufferedDocs :缓冲区最大文档数，默认为-1(关闭)；可调用indexWriterConfig.setMaxBufferedDocs(int maxBufferedDocs)设定。

ramBufferSizeMB:内存缓冲区大小，默认16MB，默认按照缓冲区大小flush。可以调用IndexWriter.setRAMBufferSizeMB(double mb)设定。

------------------------------------------------------------------------------------------------------------------------------------------------------

```java
private final void maybeMerge(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments) throws IOException {
  ensureOpen(false);
  //updatePendingMerges 校验段集是否符合merge条件，如果符合，则返回符合条件的MergePolicy.MergeSpecification（记录待合并的段集OneMerge），将段集放入pendingMerges
 if (updatePendingMerges(mergePolicy, trigger, maxNumSegments) != null) {
    executeMerge(trigger);
 }
}

```

##### org.apache.lucene.index.IndexWriter#**updatePendingMerges**

**merge策略在此处需要找到符合策略的段集**，并且将段集加入到pendingMerges队列

```java
final MergePolicy.MergeSpecification spec;
if (maxNumSegments != UNBOUNDED_MAX_MERGE_SEGMENTS) {
  // 显式或者merge完成后再次调用走此流程
  assert trigger == MergeTrigger.EXPLICIT || trigger == MergeTrigger.MERGE_FINISHED :
  "Expected EXPLICT or MERGE_FINISHED as trigger even with maxNumSegments set but was: " + trigger.name();

  spec = mergePolicy.findForcedMerges(segmentInfos, maxNumSegments, Collections.unmodifiableMap(segmentsToMerge), this);
  if (spec != null) {
    final int numMerges = spec.merges.size();
    for(int i=0;i<numMerges;i++) {
      final MergePolicy.OneMerge merge = spec.merges.get(i);
      merge.maxNumSegments = maxNumSegments;
    }
  }
} else {
  // 获取NRT reader ，commit，segment_flush，full_flush走次逻辑
  switch (trigger) {
    case GET_READER:
    case COMMIT:
      spec = mergePolicy.findFullFlushMerges(trigger, segmentInfos, this);
      break;
    default:
      spec = mergePolicy.findMerges(trigger, segmentInfos, this);
  }
}
if (spec != null) {
  final int numMerges = spec.merges.size();
  for(int i=0;i<numMerges;i++) {
    // 注册merge，每次进入此方法会先校验registerDone，若为true，则返回true，否则执行后面逻辑，将		OneMerge加入到pendingMerges，标记registerDone为true，
    registerMerge(spec.merges.get(i));
  }
}
return spec;
```



ps: OneMerge，它描述了待合并的段的信息，包含的几个重要的信息如下所示：

- List<SegmentCommitInfo> segments：使用一个链表存放所有待合并的段信息SegmentCommitInfo，其中SegmentCommitInfo用来描述一个段的完整信息（除了删除信息），它包含的信息以及对应在索引文件的内容
- SegmentCommitInfo info：该字段在当前阶段是null，在后面的流程中会被赋值，它描述的是合并后的新段的信息
  List<SegmentReader> readers：该字段在当前阶段是null，在后面的流程中会被赋值，readers中的每一个SegmentReader描述的是某个待合并的段的信息，SegmentReader的介绍可以看SegmentReader系列文章
- List<Bits> hardLiveDocs：该字段在当前阶段是null，在后面的流程中会被赋值，hardLiveDocs中的每一个Bits描述的是某个待合并的段中被标记为删除的文档号集合

##### org.apache.lucene.index.ConcurrentMergeScheduler#merge

从MergeSource持有的indexWriter的pendingMerges队列中拉取OneMerge，包装为merge线程，执行线程，更新pendingMerges

```java

while (true) {

  if (maybeStall(mergeSource) == false) {
    break;
  }

  //从pendingMerges中取出OneMerge
  OneMerge merge = mergeSource.getNextMerge();

  // 从mergeSource中的OneMerge包装为一个Merge线程
  final MergeThread newMergeThread = getMergeThread(mergeSource, merge);
  mergeThreads.add(newMergeThread);
  //更新限流信息
  updateIOThrottle(newMergeThread.merge, newMergeThread.rateLimiter);
  // 启动merge 线程，执行真正的merge操作
  newMergeThread.start();
  updateMergeThreads();
}




// org.apache.lucene.index.IndexWriter#getNextMerge
  private synchronized MergePolicy.OneMerge getNextMerge() {
    if (pendingMerges.size() == 0) {
      return null;
    } else {
      // 从pendingMerges链表中取出第一个OneMerge，添加到runningMerges链表中
      MergePolicy.OneMerge merge = pendingMerges.removeFirst();
      runningMerges.add(merge);
      return merge;
    }
  }




// org.apache.lucene.index.ConcurrentMergeScheduler#updateMergeThreads
protected synchronized void updateMergeThreads() {

    // Only look at threads that are alive & not in the
    // process of stopping (ie have an active merge):
    final List<MergeThread> activeMerges = new ArrayList<>();

  //取出所有待合并线程，将其加入到正在merge的列表中
    int threadIdx = 0;
    while (threadIdx < mergeThreads.size()) {
      final MergeThread mergeThread = mergeThreads.get(threadIdx);
      if (!mergeThread.isAlive()) {
        // Prune any dead threads
        mergeThreads.remove(threadIdx);
        continue;
      }
      activeMerges.add(mergeThread);
      threadIdx++;
    }

    // 对merge列表进行归并排序,最大的段排在前面
  	/**
      public int compareTo(MergeThread other) {
        return Long.compare(other.merge.estimatedMergeBytes, merge.estimatedMergeBytes);
      }
  	*/
    CollectionUtil.timSort(activeMerges);

    final int activeMergeCount = activeMerges.size();

  	//大段数量处理，为了限流
    int bigMergeCount = 0;

    for (threadIdx=activeMergeCount-1;threadIdx>=0;threadIdx--) {
      MergeThread mergeThread = activeMerges.get(threadIdx);
      // 大于MIN_BIG_MERGE_MB = 50.0MB的为大段
      if (mergeThread.merge.estimatedMergeBytes > MIN_BIG_MERGE_MB*1024*1024) {
        bigMergeCount = 1+threadIdx;
        break;
      }
    }

    for (threadIdx=0;threadIdx<activeMergeCount;threadIdx++) {
      MergeThread mergeThread = activeMerges.get(threadIdx);

      OneMerge merge = mergeThread.merge;

      // 如果当前的merge线程id小于大段的数量-最大线程数，则暂停merge
      final boolean doPause = threadIdx < bigMergeCount - maxThreadCount;

      double newMBPerSec;
      //暂停
      if (doPause) {
        newMBPerSec = 0.0;
      } else if (merge.maxNumSegments != -1) {
        newMBPerSec = forceMergeMBPerSec;
      } else if (doAutoIOThrottle == false) {
        newMBPerSec = Double.POSITIVE_INFINITY;
      } else if (merge.estimatedMergeBytes < MIN_BIG_MERGE_MB*1024*1024) {
        // 小段不限流
        newMBPerSec = Double.POSITIVE_INFINITY;
      } else {
        newMBPerSec = targetMBPerSec;
      }

      MergeRateLimiter rateLimiter = mergeThread.rateLimiter;
      double curMBPerSec = rateLimiter.getMBPerSec();

      rateLimiter.setMBPerSec(newMBPerSec);
    }
  }
```

//merge

```java
final MergePolicy mergePolicy = config.getMergePolicy();
/**
	merge初始化，
	1.将删除文档写入硬盘;
	2.生成SegmentCommitInfo以及诊断信息，MergePolicy.OneMerge.setMergeInfo(SegmentCommitInfo）
*/
mergeInit(merge);
//真正的merge操作，耗时操作，但不持有IW的锁
mergeMiddle(merge, mergePolicy);
//此版本没有操作
mergeSuccess(merge);
//merge完成，唤醒其他线程，从runningMerges删除已完成的OneMerge
mergeFinish(merge);
//更新pendingMerges链表
updatePendingMerges(mergePolicy, MergeTrigger.MERGE_FINISHED, merge.maxNumSegments);

-------------------------------------------------------------------------
  
// merge middle
private int mergeMiddle(MergePolicy.OneMerge merge, MergePolicy mergePolicy) throws IOException {

  final SegmentMerger merger = new SegmentMerger(mergeReaders,
                                                 merge.info.info, infoStream, dirWrapper,
                                                 globalFieldNumberMap,
                                                 context);
  merge.info.setSoftDelCount(Math.toIntExact(softDeleteCount.get()));
  merge.checkAborted();

  merge.mergeStartNS = System.nanoTime();

  // This is where all the work happens:
  if (merger.shouldMerge()) {
    merger.merge();
  }

  MergeState mergeState = merger.mergeState;
  assert mergeState.segmentInfo == merge.info.info;
  merge.info.info.setFiles(new HashSet<>(dirWrapper.getCreatedFiles()));
  Codec codec = config.getCodec();

  // Very important to do this before opening the reader
  // because codec must know if prox was written for
  // this segment:
  boolean useCompoundFile;
  synchronized (this) { // Guard segmentInfos
    useCompoundFile = mergePolicy.useCompoundFile(segmentInfos, merge.info, this);
  }
  // 为true，则创建复合文件
  if (useCompoundFile) {
    createCompoundFile(infoStream, trackingCFSDir, merge.info.info, context, this::deleteNewFiles);
  }
}



-------------------------------------------------------------------------
  org.apache.lucene.index.SegmentMerger#merge
  段合并器执行真正的merge

  1、合并域信息：mergeFieldInfos
  2、合并域：mergeFields()
  3、合并标准化因子：mergeNorms()
  4、合并Points：mergePoints()
  5、合并词典和倒排表：mergeTerms()
  6、合并docValues：mergeDocValues（）
  7、合并词向量：mergeVectors()

MergeState merge() throws IOException {
  mergeFieldInfos();

  int numMerged = mergeFields();

  final SegmentWriteState segmentWriteState = new SegmentWriteState(mergeState.infoStream, directory, mergeState.segmentInfo, mergeState.mergeFieldInfos, null, context);
  final SegmentReadState segmentReadState = new SegmentReadState(directory, mergeState.segmentInfo, mergeState.mergeFieldInfos, IOContext.READ, segmentWriteState.segmentSuffix);

  if (mergeState.mergeFieldInfos.hasNorms()) {
    mergeNorms(segmentWriteState);
  }

  try (NormsProducer norms = mergeState.mergeFieldInfos.hasNorms()
       ? codec.normsFormat().normsProducer(segmentReadState)
       : null) {
    NormsProducer normsMergeInstance = null;
    if (norms != null) {
      // Use the merge instance in order to reuse the same IndexInput for all terms
      normsMergeInstance = norms.getMergeInstance();
    }
    mergeTerms(segmentWriteState, normsMergeInstance);
  }

  if (mergeState.mergeFieldInfos.hasDocValues()) {
    mergeDocValues(segmentWriteState);
  }

  if (mergeState.mergeFieldInfos.hasPointValues()) {
    mergePoints(segmentWriteState);
  }

  if (mergeState.mergeFieldInfos.hasVectors()) {
    numMerged = mergeVectors();
  }

  // write the merged infos
  if (mergeState.infoStream.isEnabled("SM")) {
    t0 = System.nanoTime();
  }
  codec.fieldInfosFormat().write(directory, mergeState.segmentInfo, "", mergeState.mergeFieldInfos, context);
}
```

## 5、merge优化场景

1、对于实时性要求不高的场景，可以增加elasticsearch refresh间隔，减少落段的频率，减少IO操作

2、调大indices.memory.index_buffer_size（默认10%），Lucene 缓冲区ramBufferSizeMB默认为16MB

3、根据业务需求，适当调整每层段数、允许合并最大段大小

4、当index不再有写入操作的时候，建议对其进行force merge：提升查询速度、减少内存开销，例如：使用低峰定时merge
