# CSUR 审稿意见与改稿对照

<a id="review-1"></a>

## R1｜热点迁移

**审稿原文**

> The survey advocates runtime repartitioning and migration to relieve access skew (Section 3.1.2, Section 5.1). However, triggering state migration on an already saturated node inherently consumes the exact bottlenecked resources (memory bus, network I/O, and CPU). Fencing updates, snapshotting memory, and transferring state during peak saturation injects heavy coordination and I/O debt, worsening queuing and backpressure. Consequently, eager load rebalancing risks triggering a positive-feedback loop that escalates a localized hotspot into a cluster-wide metastable collapse.

> While Section 3.1.2 acknowledges access-cost modeling in principle, the Contract-Oriented Blueprint (Section 5.3) fails to formalize explicit negative-feedback boundaries or saturation guards. The authors must address this fundamental control dilemma: how can the proposed runtime guarantee that reactive state migration does not amplify the hotspot it seeks to resolve, especially under bursty skew?

**中文**　综述提出用运行时重新分区和迁移缓解访问倾斜（§3.1.2、§5.1）。但在已经饱和的节点上迁移状态，会占用原本就紧张的内存总线、网络 I/O 和 CPU。限制写入、制作快照和传输数据会增加协调与 I/O 开销，加重排队和反压。急于再平衡可能引发正反馈，把局部热点扩大为整个集群难以自行恢复的故障。

§3.1.2 承认需要估算访问成本，但 §5.3 的设计没有给出明确的负反馈条件或饱和保护。尤其当负载分布突然变得不均匀时，系统如何保证迁移不会进一步加重热点？

**我的意见**　这里得先把热键分布画清楚。一个账户本来就只能依次更新，把它整个搬走，事件还是排队。几个独立账户挤在同一个节点才有机会分摊负载。§3.1.2 沿着后一种情况算等待时间，§5.1 保留粒度和架构的讨论。

**准备怎么改**

### §3.1.2 连读稿｜局部性与拓扑条件下的成本建模

英文草案

In the fraud-detection example, several busy accounts share one worker, although their events can be processed independently. Moving some accounts to a second worker would let the two workers share the load. The decision turns on **how long events would wait under the current placement and under the proposed move**. The comparison must include events that queue up while account state is being copied, as well as those that may see shorter queues after the handoff.

For the accounts under consideration, estimate **the delay caused by the handoff**. While writes to those accounts are fenced, their events must wait. Taking a snapshot and serializing it also consume CPU time and memory bandwidth at the source. The copy then reads from source memory, crosses the network and writes to destination memory. **The slowest of these steps at its available rate limits the copy**, even if the receiving worker has idle CPU. In Megaphone’s all-at-once experiment, for example, workers serialized state faster than the network could send it. Moving one batch at a time limited the state waiting to be sent. This is why the estimate must account for **how much state is in flight at once**, as well as the total bytes moved.

A move pays off if **the shorter queues after handoff save more waiting time than the copy adds during the burst**. To start the first batch, **the source, destination and link must each have spare capacity**. When a required resource is saturated, the scheduler defers the move before it fences writes or takes a snapshot. After each batch it checks live queues and backpressure. **Rising congestion stops further transfers** until the service recovers.

中文对照

在欺诈检测示例中，几个繁忙账户的事件本可以独立处理，却都由同一个节点负责。把部分账户交给第二个节点，就能让两个节点分担负载。决定要不要搬，要比较**维持当前分配和迁移部分账户这两种情况下，事件各要等多久**。复制账户状态时积压的事件要算进去，交接后可能少排的队也要算进去。

先估算**交接这批账户会增加多少等待**。限制这些账户写入期间，它们收到的事件只能排队。制作快照和序列化还会占用源节点的 CPU 和内存带宽。随后，数据要从源端内存读出，经过网络，写入目标端内存。即使接收节点的 CPU 空闲，**复制速度仍受这条路径上实际可用速率最低的环节限制**。例如，Megaphone 在一次性迁移的实验中观察到，工作节点序列化状态的速度超过了网络发送速度。改为一批发完再搬下一批，才限制住了等着发送的数据量。因此，除了总共要搬多少字节，估算时还要看**同一时刻有多少状态正在搬运**。

如果**交接后少排队省下的时间超过这波突发期间搬运增加的等待**，迁移就有收益。启动第一批还有一个条件：**源端、目标端和链路都得有余量**。其中一处已经饱和，调度器就在冻结写入和制作快照前暂缓迁移。每搬完一批，调度器检查在线队列和反压。**拥塞继续加重就停止放行后续批次**，等服务恢复再搬。

BriskStream 估算同一台共享内存机器上算子相对位置对执行的影响，可用于讨论放置后的访问代价。跨节点复制的代价用 Megaphone 的迁移实验说明。Megaphone 的图 20 及随后讨论记录了序列化快于网络发送时的暂存开销，支持第二段关于迁移粒度的判断。第三段与 §5.3 的饱和保护是本文拟议的设计条件。[BriskStream 原文](https://arxiv.org/abs/1904.03604) · [Megaphone 原文](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf)

### §3.1.1，补清示例 A（main.tex 第 415 行）

改前 · 英文

> The model logic has not changed, but **a small set of hot keys now serializes queue and lock paths**.

改前 · 中文

> 模型逻辑没有变化，但**少数热键使事件在队列和锁上排队，只能依次通过**。

问题　“热键导致排队”有两种读法：一个键的更新本来就得串行，或者几个能同时处理的键恰好挤在一个节点。前一种情况下搬家也要排队。建议把示例明确设成后一种，读者才知道为什么考虑迁移。

改后 · 英文草案

> Suppose **several independently updated account keys are assigned to the same worker**. After the campaign starts, events for these accounts arrive faster than this worker can process them. **A second worker has spare capacity and could take over some of the accounts.**

改后 · 中文

> 假设**几个可以独立更新的账户键都被分到了同一个工作节点**。营销活动开始后，这些账户的事件到达速度超过了该节点的处理速度。**另一个工作节点有余量，可以接手其中一部分账户。**

### §3.1.2，第 1 段（main.tex 第 421 行）

改前 · 英文

> That is a **cost-modeling problem** rather than a detection problem alone. Once access patterns become visible, the harder task is to **price them correctly** under skew and topology.

改前 · 中文

> 这涉及**成本建模**，只检测到热点还不够。在能观察到访问模式后，更难的是**准确估算这些访问的代价**，把负载倾斜和硬件拓扑都考虑进去。

问题　这两句只说“要算成本”。建议删掉这层重复，用前台事件的等待时间作比较对象，并交代比较多长时间。

改后中英文见本条开头的 §3.1.2 连读稿第 1 段。

### §3.1.2，第 2 段（main.tex 第 423 行）

改前 · 英文

> Across these works, it is more accurate to think in terms of **an access-cost surface shaped by contention and hardware placement** than in terms of a static data structure.

改前 · 中文

> 综合这些工作，把访问成本理解为**由资源争用和硬件放置共同决定的成本曲面**，比只从静态数据结构出发更准确。

问题　“成本曲面”没有说明搬运代价从哪来。沿着源端读取、网络发送、目标端写入这条路径，可以看出哪一步让在线事件多等，也能解释 Megaphone 为什么分批搬运。

改后中英文见本条开头的 §3.1.2 连读稿第 2 段。

### §3.1.2，第 3 段（main.tex 第 425 行）

改前 · 英文

> **Partitioning and synchronization choices also shape later recovery behavior** by determining checkpoint grain, replay skew, and migration debt under disturbance.

改前 · 中文

> **分区和同步方式也会影响后续恢复**，因为它们决定了检查点的粒度、重放负载的不均衡程度，以及扰动下迁移留下的额外工作。

问题　这一段从迁移开销转向故障恢复，留下了前面那个决定：这波热点值不值得搬？先用交接代价和预计节省的排队时间作比较，再看源端此刻能否承受第一批。

改后中英文见本条开头的 §3.1.2 连读稿第 3 段。

### §3.1.3，补查的衔接句，R1 未点名（main.tex 第 429 行）

改前 · 英文

> Example A again: **once the fraud detector's access-cost model decides that migration is warranted**, the runtime must execute the move without losing in-flight transactions or stalling downstream operators.

改前 · 中文

> 再看示例 A，**一旦欺诈检测系统的访问成本模型判定值得迁移**，运行时就必须在不丢失在途事务、不使下游算子停顿的情况下完成迁移。

问题　现在一句话跳过了上一节最需要解释的决定。把“值得搬”设为明确前提，这一节只讲决定之后怎样接手，避免旧、新节点对同一条事件处理两次。

改后 · 英文草案

> **If moving a set of keys is expected to reduce waiting and the source can sustain the copy**, the next question is where processing resumes. **The old and new owners must agree on the last event processed for each key before the new owner accepts updates.**

改后 · 中文

> **如果预计迁移这组键能减少等待，而且源节点能承受复制开销**，下一个问题就是处理该从哪里继续。**新所有者开始接受更新前，新旧所有者必须就每个键最后已处理的是哪条事件达成一致。**

### §5.3，过载时能否启动或继续（main.tex 第 713 行）

改前 · 英文

> The decision kernel is where the state catalog and observation streams become **boundary-aware choices**.

改前 · 中文

> 状态目录和观测数据在决策内核中转化为**考虑了各项边界的选择**。

问题　只在队列越过上限时减速，可能在第一批快照时就已挤占前台。先把保护放到冻结写入之前，再规定执行中的停发条件、安全完成已提交交接的方式和重启条件。

改后 · 英文草案

> **Before it fences writes, the scheduler reserves CPU time and bandwidth for live events and for transfers already under way.** It admits one small batch if the source, destination and link can carry the batch’s peak load with a safety margin, and the write pause stays within the service limit. Stale readings or a saturated resource postpone the copy. When arrivals already exceed processing capacity, the service must throttle incoming work or add capacity. **If the live queue keeps growing or backpressure rises, the scheduler stops issuing batches**, slows interruptible copies and lets committed handoffs finish using their reservations. It resumes at a low rate once capacity returns and congestion stays below the lower restart threshold for a recovery window. **Enforced reservations bound the load from new batches. Events arriving during a batch already in flight can still incur a delay.**

改后 · 中文

> **冻结写入前，调度器先为在线事件和已经开始的迁移留出 CPU 和带宽。** 源端、目标端和链路的剩余容量都能承受一小批迁移的峰值负载，还能留出余量，且停写时间在服务允许范围内，才启动这一批。读数过期或有一处资源饱和，就推迟复制。若事件到达得已经比处理得快，服务要先限流或扩容。**在线队列持续增长或反压加重时，调度器停止发起新批次**，降低可中断复制的速率，让已提交的交接用预留资源完成。资源恢复、拥塞持续低于较低的重启阈值一段时间后，再从低速恢复。**按预算执行能限制新批次增加的负载；已在途的一批仍可能让刚到的事件多等一会儿。**



CPU、内存带宽和网络带宽分别按下面的式子预算。对每种资源 r，在同一控制窗口内要求

$$
L_r^{\max}+M_{r,\mathrm{committed}}^{\max}+M_{r,\mathrm{new}}^{\max}+H_r\le C_r^{\min}.
$$

以源端网络为例，左侧依次是前台流量、已批准的迁移、新批次的峰值流量和预留带宽，右侧是该窗口内可用带宽的下界。多个控制器共用这本账，才能给前台留足带宽。用历史均值代替峰值时，突发流量可能冲破预留。停写时长另设上限。评估时继续观察突发过后的队列：若输入已回落，重试或反复迁移仍使有用吞吐长期偏低，就要追查是哪项额外工作使过载持续。[亚稳态故障的原始定义](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf)

### §5.2 表 7｜冲突时谁先执行，队列何时恢复

*作者补查：R1 问迁移能否避免放大热点，核对表 7 对稳定性的承诺。*

[main.tex 第 696 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L696)

改前 · 英文

> overlapping controllers **stop oscillating under burst or skew**

改前 · 中文

> 相互作用的控制器**在突发或倾斜下停止振荡**

问题　规定迁移与准入同时争资源时谁优先，只能解决当下这一次冲突。下一轮观测有延迟，控制器仍可能反复启动、暂停。此处必须与 §5.3“有条件地限制新增迁移工作”的承诺一致。

改后 · 英文草案

> **Conflicting actions have a defined order.** To assess oscillation, record migration starts, pauses and restarts alongside **queue growth and recovery after a burst**.

改后 · 中文

> **冲突动作有明确的执行次序。** 要判断是否振荡，应记录迁移启动、暂停和重启的时间，并对照**突发期间队列怎样增长、突发结束后怎样恢复**。

### 附录 B.5｜策略骨架也要遵守正文的饱和保护

*作者补查：R1 问饱和保护，核对补充材料中的执行规则。*

[extended_design_notes.tex 第 63 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/supplement_sections/extended_design_notes.tex#L63)

改前 · 英文

> with explicit **rollback or retry semantics when telemetry is delayed or conflicting**

改前 · 中文

> 在遥测延迟或彼此冲突时，明确规定**回滚或重试的语义**

问题　旧信号不足以支持新动作。若遥测一过期就反复重试，恰好可能走回 R1 的正反馈。把“尚未开始一批”和“所有权已交接”分成两种情形。

改后 · 英文草案

> **Stale or conflicting observations close admission to new batches.** Keep enough capacity to finish a committed handoff, record its outcome, and **recheck headroom before admitting another batch**.

改后 · 中文

> **遥测过期或冲突时，停止放行新批次。** 保留完成已提交交接所需的资源，记录交接结果，并在**放行下一批之前重新检查余量**。

### 附录 D 的三例示意图｜图里还在写“见到热点就重分区”

*作者补查：R1 问何时允许迁移，核对案例图是否误导。*

[running_examples_triptych.tex 第 26 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/figures/running_examples_triptych.tex#L26)

改前 · 英文

> **repartition hotspots**\\adjust snapshots

改前 · 中文

> **重新分区热点**／调整快照

问题　只改正文，读图的人仍会把重分区理解为默认动作。把是否放行迁移画出来，让图和 §3.1.2 的两个判断一致。

改后 · 英文草案

> **Compare waiting times** → check spare capacity → **defer or migrate one batch**

改后 · 中文

> **比较两种方案下的等待时间** → 检查资源余量 → **暂缓，或放行一批迁移**

**作者补查 · 引文备查。** R1 用到的 Megaphone 在本地 `refs.bib` 第 836 行起只列出前三位作者；原论文列出六位，后三位为 Vasiliki Kalavri、John Liagouris、Timothy Roscoe。[原论文首页](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf)

<a id="review-2"></a>

## R2｜引言提出的问题，§3 和 §4 怎样接上

**审稿原文**

> My main concern is that the line of argument is not easy to follow. I had to read the paper several times to understand how the central argument develops, and it was still not clear to me how the discussion of prior work leads to and supports that argument.

> Please clarify what Section 3 establishes, what Section 4 adds, and how the comparisons support the central argument. (See comments in "Should anything be deleted from or condensed in the paper?")

> Sections 3 and 4: Both discuss substantial prior work, but I found it difficult to see how the argument progresses from one section to the other. The authors should clarify what Section 3 establishes and what Section 4 adds, while reducing repeated descriptions of existing systems.

**中文**　我最担心的是论证难以跟上。我读了几遍才明白中心论点如何展开，但仍不清楚已有工作怎样导出并支持这一论点。请说明第 3 节确立了什么，第 4 节新增了什么，以及这些比较如何支持中心论点。两节都介绍了大量工作，应减少重复。

**我的意见**　引言第 141 行已经点明问题。难跟的是后面的推理：分开研究访问、执行和更新，会漏看哪部分服务代价？引言用一个短例说明困难。§3 沿一项决策追到后果，§4 再比较两类系统在哪一步可以借用同一认识、在哪一步需要各自的规则。

**准备怎么改**

### 引言｜保留第 141 行的问题，说明第 143 行的困难

*作者补查，归入 R2 的总体阅读意见。R2 没有点名引言。* [引言首段](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L141) · [第二段](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L143)

**第 1 段的末句保留。**

> Across these settings, the central question is whether a runtime can keep **shared, long-lived, performance-critical state governable without sacrificing service stability**.
>
> 这些场景的共同问题是，运行时能否在**维持服务稳定的同时，管好共享、长期存在且影响性能的状态**。

第 2 段已经交代，文献往往分别研究算子放置、内存布局和模型适应。困难出在段尾：“局部收益”和“后来的约束”究竟落在哪两个操作上？这里给一个短例，第 147 行接着引出访问、执行、演化三个维度。后面的分类、贡献和章节顺序沿用原稿。

改前 · 英文

> What matters, then, is not optimization within one phase, but control over a lifecycle in which **a local gain often comes back as a later constraint**.

改前 · 中文

> 因此，重要的是控制整个生命周期，而不只是优化单个阶段，因为**一个环节的收益往往会变成后续环节的约束**。

改后 · 英文草案

> A server can reclaim a completed request’s prefix to make room for a waiting request. **The waiting request can start sooner. If the first user returns, the server may have to compute the same prefix again.** The allocator sees the memory saving immediately. The cost appears later, in another request’s latency.

改后 · 中文

> 服务节点可以回收一个已结束请求的前缀缓存，给排队的请求腾位置。**排队的请求可以早些开始；原来的用户再来时，节点却可能得重新计算同一段前缀。** 分配器立刻就能看到省下的内存，而代价到后来才会出现，体现在另一次请求的延迟上。

### §1 首句｜历史背景（原文对照）

*作者补查 · 原文对照*　[main.tex 第 141 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L141)

> Systems have **moved away from largely stateless throughput engines** toward long-running services shaped by evolving state.

> 系统**已经从基本无状态的吞吐引擎转型**为受状态演化影响的长期运行服务。

原句是历史背景。R2 的总体意见更直接地落在本段末句和下一段，见前面的引言对照。

### §3 开头（main.tex 第 409 行）

*作者补查 · 原文对照*

> **A local gain in one dimension often returns as a downstream constraint on the other two.**

> **一个维度上的局部收益，常常会变成另外两个维度随后要面对的约束。**

本句所处的段落已经交代三个维度和三个贯穿案例。保留这个开头，把衔接的修改落在 §3.4 末段和 §4 首段。

### §3.4 末段｜把案例的结果交给 §4

*R2 点名 §3、§4；第 525 行是作者补查的衔接段。* [§3.4 末段](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L525)

改前 · 英文

> The example isolates **four serving-lifecycle boundaries**. Admission-versus-restoration tradeoffs unfold over time, not at one instant.

改前 · 中文

> 这个例子区分出**四个服务生命周期边界**。准入与恢复之间的取舍随时间展开，而非发生在某一个时刻。

问题　用这两个请求收住案例，再用同一件事解释第 3 节的三个维度如何连在一起。读者此时需要的是本章结论。

改后 · 英文草案

> Reclaiming the cached prefix lets the waiting request start sooner. If the original user returns, the server must fetch that prefix from another tier or compute it again. **The same decision changes who runs now, how much work the server does later, and how long the returning user waits.** This is the connection among access, execution, and reuse that Section 3 traces. To judge the benefit of reclamation in this example, the measurement has to include both requests.

改后 · 中文

> 回收前缀缓存，让排队的请求得以早些开始。如果原来的用户回来，服务节点就得从其他存储层取回这段前缀，或者重新计算。**同一个回收决定，既改变了眼下哪个请求能先运行，也改变了节点以后要做多少工作，以及回访用户要等多久。** 这就是第 3 节沿着具体机制梳理出的访问、执行与复用之间的联系。要判断这次回收是否划算，两次请求都得算进去。

### §4 开头（main.tex 第 530 行）

改前 · 英文

> The preceding sections established a **mechanism inventory**, but coverage alone does not show **which lessons transfer**.

改前 · 中文

> 前几节整理出了**一份机制清单**，但仅凭覆盖了哪些机制，看不出**哪些经验能用于其他领域**。

问题　第 3 节已经追到了每项动作在本系统里的后果。到这里再换领域，同时固定“丢弃状态以释放内存”这个动作，就能看出决定能否沿用的条件：被删的东西以后还能不能找回来？

改后 · 英文草案

> Section 3 followed an evicted prefix to the next request that needed it. **If the input and the same model parameters are still available, the server can rebuild the prefix, and the returning request waits for that work.** A continual learner can also free memory by deleting an old training example. **If that was its only copy, the learner can no longer replay the example. Whether the deletion hurts accuracy depends on what the remaining examples teach the model.** Both systems free space at once. To judge the choice, we need to know what the next request or training update can still do with the state that remains.

改后 · 中文

> 第 3 节追到了缓存回收后的下一次请求。**只要输入和对应的模型还在，服务就能重算前缀，回来复用它的请求要为这次计算多等一段时间。** 持续学习系统也可以删掉旧训练样本来腾空间。**如果删掉的是唯一副本，以后训练就用不到这个样本了。准确率会不会下降，取决于模型还能从留下的样本中学到什么。** 两种做法都能马上腾出内存。要判断删得值不值，还得看下一个请求或下一轮训练拿剩下的状态能做什么。

*R2 点名 §3、§4 的论证和 §4.2.3 的两篇缓存文献。下面五组是我们按 §4.2 原有小节补查的对照。改后段落替换所摘原句及其重复解释。§4.2 开头保留图表导读，第 538 行概括四类问题的句子可删。*

### §4.2.1｜流处理要从哪个时刻接着算？

送审原句节选，见 [main.tex 第 569 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L569)。

改前 · 英文

> Progress semantics answer **when a transition is safe**, while ownership-transfer mechanisms answer **where it may occur**; strong systems need both.

改前 · 中文

> 进度语义回答**何时可以安全地转换**，所有权转移机制回答**转换可以在哪里发生**。健全的系统两者都需要。

**我的意见**　“何时”和“哪里”听起来像同一项操作的两半。Flink 论文这里解决的是故障后从哪条记录重放，Megaphone 解决的是正常运行时谁来处理某个键的下一条记录。

改后 · 英文草案

> Flink and Megaphone both have to tell which records have already contributed to an operator’s state. In Flink’s checkpoint protocol, an operator waits for a barrier on each input and **saves its state at that boundary. After a failure, it restores the checkpoint and replays the later input.** Megaphone faces a different event. The stream is still running, but a group of keys is about to move. Once the old worker has processed the records before the scheduled handoff time, it **sends the state for those keys to the new worker, which can then process their later records.** Other keys can keep moving through the operator during the handoff.

改后 · 中文

> Flink 和 Megaphone 都得分清哪些记录已经算进了算子状态。Flink 做检查点时，算子等每路输入的屏障都到齐，**保存这个分界点上的状态。发生故障后，系统恢复检查点，再重放后面的输入。** Megaphone 面对的是另一种情况。流还在运行，但一组键要换节点。旧节点处理完交接时刻以前属于这组键的记录，便**把这组键的状态交给新节点，由新节点处理后续记录。** 交接期间，算子仍可处理其他键。

[Flink，§3.3](https://kth.diva-portal.org/smash/get/diva2%3A1059537/FULLTEXT01.pdf) · [Megaphone，§§3–4](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf)

### §4.2.2｜一个查询来不及算完，能少算一些数据吗？

送审原句节选，见 [main.tex 第 575 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L575)。

改前 · 英文

> Read together, these papers expose three recurring families: **state-path shaping, budgeted approximation, and online boundary steering**. Quality budgets become scheduling inputs, not post-hoc validation checks.

改前 · 中文

> 这些工作合起来呈现出三类反复出现的方法：**调整状态的处理路径、在预算内做近似、在线调节边界**。质量预算成为调度的输入，而不再只是事后的验证项。

**我的意见**　按这三个名字分组，读者仍不知道“质量预算”在哪篇论文里真的改变了执行方式。两篇都想缩短等待，但只有 BlinkDB 的用户允许查询返回估计值。

改后 · 英文草案

> FineStream runs window queries on a chip where the CPU and GPU share memory. To finish a window sooner, it **divides the work between the two processors while keeping the same records in the computation.** BlinkDB serves analytical queries whose users can accept an estimate. Given a time limit or an error bound, it **chooses how large a sample to query and reports the estimate with its uncertainty.** Moving work to the GPU changes when a window result is ready. Reading a smaller sample can also change the value returned to the user.

改后 · 中文

> FineStream 在 CPU 和 GPU 共享内存的芯片上执行窗口查询。为让一个窗口早点算完，它**把工作分给两个处理器，参与计算的仍是原来那些记录。** BlinkDB 面对的用户可以接受估计值。用户给出时间限制或误差要求后，系统**据此选择要查多大的样本，并同时报告估计值及其不确定性。** 把工作分给 GPU，改变的是窗口结果什么时候出来。少读一些数据，连用户拿到的数值也可能改变。

[FineStream，USENIX ATC 2020](https://www.usenix.org/conference/atc20/presentation/zhang-feng) · [BlinkDB，EuroSys 2013](https://dspace.mit.edu/entities/publication/37668545-3cfd-4ba8-87b6-6a617d888e7d)

<a id="sec423-kv"></a>

### §4.2.3｜前缀命中后，请求发到哪里，缺的块要不要搬？

送审原句节选，见 [main.tex 第 591 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L591)。

改前 · 英文

> Practical runtimes still lack **portable semantics** for when KV fragments may be shared, transferred, restored, or invalidated across continuous batching, adapter multiplexing, speculative branches, and disaggregated decode paths.

改前 · 中文

> 实际运行时仍缺少**可移植的语义规则**，来规定在连续批处理、适配器复用、推测分支和预填充与解码分离等场景中，何时可以共享、传输、恢复 KV 片段，或将其作废。

**我的意见**　原句断言缺少共享和传输规则。R2 给出的两篇论文已经把其中一部分做出来了。要看清余下的问题，可以沿一个请求依次看派给谁、缺的块怎么取、取到后能否用。

改后 · 英文草案

> Suppose the worker with the longest matching prefix also has the longest queue. Mooncake **can send the request to another worker if it expects that worker to produce the first token sooner.** Its choice accounts for the queue, the prefix available there, and the cost of fetching blocks. MemServe also considers queues and cache hits when routing. Once it has picked a worker, it makes a further choice. If another worker holds more of the prefix, MemServe **fetches the extra blocks when their estimated transfer time is no greater than the prefill time they would save. Otherwise it recomputes that part locally.**
>
> LMCache gives a scheduler operations to **look up cached blocks, pin them against eviction, and move them between locations.** The distinction from a stream handoff matters here. A second worker can read a copy of a completed prefix. The original worker can keep its copy. If the service switches adapters, it must check whether **the cached blocks match the prefix state required by the current model and adapter.**

改后 · 中文

> 假设前缀匹配最长的节点恰好也排着最长的队。Mooncake **如果预计另一个节点能更早给出首个 token，就可以把请求派过去。** 它把排队、本地已有多少前缀、取回缓存块要花多久一起算进去。MemServe 派发请求时同样考虑队列和缓存命中。选定节点以后，它还要做一个决定。如果别处多缓存了一截前缀，MemServe 会估算两个时间。**搬来这一截所需的时间如果小于或等于省下的预填充时间，就搬；搬运更慢，就在本地重算。**
>
> LMCache 提供的操作让调度器可以**查找缓存块、固定它们以免被回收，再把块搬到需要的位置。** 这里和流分片交接有一个实际区别。另一个节点可以读取已经算好的前缀，原节点也可以继续保留自己的副本。服务如果切换了适配器，复用前就要核对，**这些缓存块里存的是不是当前模型和适配器所需要的前缀状态。**

[Mooncake，期刊版调度算法](https://doi.org/10.1145/3773772) · [MemServe，§5.3.1 式 (1)–(2)](https://arxiv.org/html/2406.17565v3) · [LMCache，控制接口](https://arxiv.org/html/2510.09665v2)

### §4.2.4｜新向量来了，查询要等磁盘索引重建吗？

送审原句节选，见 [main.tex 第 597 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L597)。

改前 · 英文

> These triggers often live in different modules, but the user-facing answer quality depends on **their ordering**. A retriever can appear accurate in a static benchmark while its **publication lag, deletion debt, or rebuild backlog** makes the live memory layer unsafe to expose.

改前 · 中文

> 这些触发条件往往分属不同模块，但用户拿到的答案质量取决于**它们的执行顺序**。一个检索器在静态测试中可能很准，但**发布滞后、积压的删除或重建任务**可能使在线记忆层无法安全地对外开放。

**我的意见**　原句把上线安全、查询质量和维护积压揉在一起。先用同一件事来比：新增向量写到哪里，正在运行的查询怎么找到它，后台还要做多少工作。

改后 · 英文草案

> FreshDiskANN keeps its main search graph on SSD. It **puts new vectors in an in-memory index and searches both indexes**, filtering deleted entries from the results. Queries can find an insertion before the next merge. When the temporary indexes reach a memory threshold, a background merge incorporates the updates into the disk graph. SPFresh partitions the vectors by proximity. It **inserts a new vector into the nearest partition. If the partition grows too large, it splits it and reassigns affected vectors near the new boundary.** Thus a burst of insertions leaves different work behind: a merge with the disk graph in FreshDiskANN, or repairs to the affected partitions in SPFresh. Query latency and recall during that work tell us what users experience.

改后 · 中文

> FreshDiskANN 的主搜索图放在 SSD 上。它**先把新向量写进内存索引，查询时同时搜索磁盘和内存索引**，再从结果中滤掉已删除的条目。因此不用等下次合并，查询就能找到新向量。临时索引占用的内存达到阈值后，后台才把更新并入磁盘图。SPFresh 按向量之间的距离分区。它**把新向量直接插入最近的分区。分区过大就拆分，再重新分配新边界附近受影响的向量。** 同样来一批插入，FreshDiskANN 随后要把更新并入磁盘图，SPFresh 则可能要修整受影响的分区。在这些工作进行时测查询延迟和召回率，才能看出用户等了多久、找回了多少近邻。

[FreshDiskANN，§5](https://suhasjs.github.io/files/freshdiskann-arxiv.pdf) · [SPFresh，LIRE 局部重分区](https://www.microsoft.com/en-us/research/publication/spfresh-incremental-in-place-update-for-billion-scale-vector-search/)

### §4.2.5｜留下的旧样本，什么时候用来训练？

送审原句节选，见 [main.tex 第 603 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L603)。

改前 · 英文

> The reusable comparison is therefore between **lifecycle controls over one bounded store** rather than between named replay heuristics alone.

改前 · 中文

> 因而，可复用的比较在于考察**同一个容量有限的存储上的生命周期控制**，而不只是比较各个有名字的重放方法。

**我的意见**　两篇论文都设了样本容量上限，但模型需要更新的时间不同。MIR 在新数据到达时就要更新模型，GDumb 可以把训练推迟到评价时。

改后 · 英文草案

> MIR updates the model as new batches arrive. It **estimates how a step on the new batch would change the loss on stored examples. It replays the examples with the largest predicted increases alongside the new batch.** GDumb can defer training until evaluation. As examples arrive, it **keeps a buffer whose class counts it tries to balance. At evaluation it trains a fresh model using only that buffer.** For MIR, the immediate question is which old example to replay in this update. For GDumb, dropping the last example of a class leaves the next training run with no example of that class unless another arrives.

改后 · 中文

> MIR 每收到一批新数据就要更新模型。它**先试算这批数据会怎样改变模型，再看一批候选旧样本中哪些损失会上升最多，最后把受影响最大的旧样本与新数据一起用于真正的更新。** GDumb 可以等到评价时再训练。数据陆续到来时，它**先存样本，尽量让各类样本数均衡。到了评价时，只用留下的样本从头训练一个模型。** MIR 当下要选的是这一步该重放哪个旧样本。对 GDumb 来说，如果删掉了某类最后一个样本，以后也没有同类样本再来，下次训练就没有这一类的例子可用了。

[MIR，§3.1](https://proceedings.neurips.cc/paper/2019/file/15825aee15eb335cc13f9b559f166ee8-Paper.pdf) · [GDumb，算法 1](https://www.ecva.net/papers/eccv_2020/papers_ECCV/papers/123470511.pdf)

### §3.3.4｜延迟压实为什么会变成热点？

*作者补查：R2 问 §3 如何支撑 §4，检查案例结论能否从条件推出。*

[main.tex 第 510 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L510)

改前 · 英文

> In Example C, **deferred index compaction** (an evolution deferral) **concentrates future query traffic onto a shrinking set of fresh shards** (an access hotspot).

改前 · 中文

> 在示例 C 中，**推迟索引压实**（延后演化操作）**使未来查询集中到越来越少的新鲜分片**（访问热点）。

问题　少做一次压实，查询可能仍走原来的副本，只是后台积累了更多维护工作。要写出热点转移，例子里须交代查询路由只选择已发布所需版本的副本。

改后 · 英文草案

> Suppose **queries may use only replicas that have published the required index version**. If maintenance delays publication on one replica, **the other eligible replicas receive its share of the traffic**. With no eligible alternative, the query must wait or be served under a relaxed freshness requirement.

改后 · 中文

> 假设**查询只能使用已发布所需索引版本的副本**。如果维护工作延误了一个副本的发布，**其余符合条件的副本就会分担它原先的流量**。如果没有其他合格副本，查询只能等待，或者在允许放宽新鲜度要求时使用旧版本。

### §5.1.2｜控制器放在哪里，并不直接决定一致性

*作者补查 · 原文对照*　[main.tex 第 672 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L672)

> Embedded control is fast but fragmented, sidecar-style control is modular but delayed, and **shared services are consistent** but can bottleneck if their APIs are too fine-grained

> 嵌入式控制快但分散，边车式控制模块化但有延迟，而**共享服务具有一致性**，不过 API 粒度过细时可能成为瓶颈

这是控制器部署位置的架构比较。本轮保留原句供查阅，迁移的饱和保护集中改 §5.3。

### §7｜让结论回答引言的问题

*作者补查 · 原文对照*　[main.tex 第 779 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L779)

> Progress will therefore depend on runtimes that keep **state identity, observability, and policy precedence stable under disturbance**.

> 因此，进展将依赖于能在扰动下保持**状态身份、可观测性和策略优先级稳定**的运行时。

结论首句已经扣回访问、执行和演化。先把 §3、§4 的推理接上，结论不另起一套写法。

## R2｜第 2 节的定义和表 3

**审稿原文**

> Section 2 would be easier to follow with less background and fewer repeated definitions (especially, formal definition seems unnecessary), and a clearer explanation of how the three analytical dimensions relate to the five comparison questions.

> Section 2: The section introduces the scope, background, definitions, and analytical framework, but their roles in the argument are not sufficiently clear. I suggest condensing the background and overlapping definitions, and explaining how the three analytical dimensions and the five questions in Table 3 will be used to examine the literature.

**中文**　第 2 节应减少背景和重复定义，形式化定义似乎没有必要。范围、背景、定义和分析框架各自在论证中起什么作用不够清楚。请说明三个分析维度与表 3 的五个问题如何配合使用。

**我的意见**　把表 3 放到一个实际例子旁边。先让读者看一页缓存如何影响下一次请求，再读“五个字段”，这些词才有着落。

**准备怎么改**

§2.4 保留状态定义和表 1。把 §2.5.1 的窗口和 KV 页例子并到定义后面。§2.6.1 保留表 3 和下面这段缓存例子，删去重复的字母元组及“只允许一个字段不同”的规则。§2.3 和 §3 重复介绍的系统删去一遍，历史沿革放在附录 A。

### §2.4，对照 §2.5.1 再次定义 state 的首句（main.tex 第 233 行）

改前 · 英文

> State is **any runtime object read, written, moved, retained, or reused** across these execution boundaries.

改前 · 中文

> 状态是跨越这些执行边界**被读取、写入、移动、保留或复用的任何运行时对象**。

问题　列出几乎所有操作，仍然难以判断一次函数调用里的临时量算不算。把 §2.5.1 里更有区分力的“影响未来执行”提到这里，后面直接举例。

改后 · 英文草案

> We call an object state when **it persists beyond one operation and its contents or location affect later work**. A KV page qualifies because later decoding steps can read the keys and values it holds.

改后 · 中文

> 当一个对象**在一次操作结束后仍然存在，且它的内容或位置会影响后续工作**时，我们就称它为状态。KV 页符合这个定义，因为后续解码步骤可以读取其中存放的键和值。

### §2.6.1，表 3 前（main.tex 第 368 行）

改前 · 英文

> Two papers are directly comparable when their tuples **differ in at most one field**; they address fundamentally different control problems when they differ in **σ or β**.

改前 · 中文

> 当两篇论文的比较元组**至多有一个字段不同**时，它们可以直接比较。如果不同的字段是 **σ 或 β**，它们解决的就是本质不同的控制问题。

问题　如果唯一不同的字段正是 σ 或 β，这两条规则给读者两个方向。删去按字段数量判定可比性的句子，示范怎样实际用表 3。

改后 · 英文草案

> **Access, execution and reuse organize the survey. Table 3 follows a decision within any one of those topics.** In a serving system, for example, the allocator may **reclaim a completed request’s cached prefix**. This frees pages for the current batch. A later request with the same prefix may need to recompute it, increasing its time to first token. Measuring through that return visit captures both effects.

改后 · 中文

> **访问、执行和复用是综述的三个主题。表 3 用来追踪每个主题里的一项具体决定。** 例如，分配器可以**回收已结束请求留下的前缀缓存**，让当前批次用上腾出的页。后来的请求如果又用到同一前缀，就可能要重算，等得更久才收到首个 token。把实验延长到这次回访，就能同时看到两次请求受到的影响。

### §2.1｜纳入标准怎样用在边界论文上

*作者补查 · 原文对照*　[main.tex 第 188 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L188)

> Recent papers enter the analytical core only when they expose **a new state object, runtime control seam, or evaluation boundary**.

> 近期论文只有展示**新的状态对象、运行时控制环节或评价边界**才进入核心分析。

这是选文方法。R2 要求解释三个维度和表 3，调整定义与表 3 的引入即可，保留原稿的选文说明。

## R2｜表 5、6、8 的依据

**审稿原文**

> Tables 5, 6, and 8 should be more clear, and shows how the cited works reflect these differences.

> Section 4 also includes comparison tables (Table 5), but the text needs to explain what can be concluded from them and why. Making these steps explicit would help readers understand how the surveyed evidence supports the paper’s central argument and the design recommendations that follow.

> The survey should distinguish what existing systems achieve, under which conditions, and what remains unresolved. The gaps listed in Tables 5 and 6 need clearer support from the cited findings.

**中文**　表 5、6、8 应更清楚地展示所引工作如何体现表中的差异。对于第 4 节的表 5，正文需要解释能从表中得出什么，以及为什么。请区分已有系统做到了什么、在什么条件下做到、什么仍待解决。表 5 和表 6 的缺口需要所引文献提供更清楚的依据。

**我的意见**　现在几篇论文共用一行，末尾放一个“缺口”，我看不出它指哪篇。建议直接写论文里的操作和实验，把几个系统放在一起后，再写它们各自管到了哪一步。

**准备怎么改**

### 表 5，服务分页与阶段控制行，末列（main.tex 第 630 行）

改前 · 英文

> **lifecycle closure over admit, expose/transfer, evict/reclaim, and restoration debt**

改前 · 中文

> **让准入、开放或传输、驱逐或回收以及后续恢复成本构成完整的生命周期闭环**

问题　“生命周期闭环”盖住了真正要追的那笔时间：旧请求回来时，要等多久才能重新用上前缀？把这次回访写进表格。

改后 · 英文草案

> **Time from a return request’s arrival to its first token, including any prefix reload or recomputation after eviction.**

改后 · 中文

> **从回访请求到达到收到首个 token 的时间，包括驱逐后重新加载或重算前缀所花的时间。**

### 表 6，保留治理行，末列（main.tex 第 650 行）

改前 · 英文

> **retention-store contract with precedence**

改前 · 中文

> **规定操作优先级的保留存储契约**

问题　“优先级”没有说是哪两个决定打架。一个旧样本可能最值得拿来重放，预算一缩却成了要腾空间的对象。

改后 · 英文草案

> **Replay selects an example for the current update. Eviction determines whether that example will still be available for the next update.**

改后 · 中文

> **重放策略选出本轮更新要用的样本；淘汰策略决定下轮更新时，这个样本还在不在。**

### 表 8，服务生命周期控制行（main.tex 第 751 行）

改前 · 英文

> **allocator-only or phase-only evaluation**

改前 · 中文

> **只测分配器，或只测某一个阶段**

问题　原表把“只测分配器”列为常见盲点，却没有逐篇指出测量时段。这里写成一项回访测试，把请求顺序和要测的两段耗时给出来。

改后 · 英文草案

> **Proposed test:** reclaim pages during a burst, then replay requests that reuse their prefixes. **Measure both the memory freed during the burst and the time those later requests spend restoring or recomputing state.**

改后 · 中文

> **建议的测试：**在突发流量期间回收缓存页，然后重放会复用这些前缀的请求。**同时测量突发期间释放了多少内存，以及后续请求花了多久恢复或重算状态。**

**表 5、6 合并。** 同一行把论文的做法与它对后续请求、更新或故障恢复的影响放在一起。

| 系统问题 | 论文中的做法与结果 | 运行时的后续影响 |
|---|---|---|
| 流处理进度 | MillWheel 用逻辑时间组织流聚合，保存处理进度和持久状态。论文用连续异常检测说明故障后如何接着处理事件。[MillWheel](https://research.google/pubs/millwheel-fault-tolerant-stream-processing-at-internet-scale/) | 分片换节点时，接收方还需要取得写权限，并从已提交的位置继续处理。 |
| 分片迁移 | Megaphone 的一次性迁移实验中，源端打包状态的速度超过了网络发送速度，待发数据堆在源端。分批传输限制了这部分积压。[Megaphone，图 19–20](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf) | 即使只搬一小批，源端仍要花 CPU 时间和内存带宽打包。节点已经饱和时，迁移准入应当暂缓。 |
| 硬件放置 | FineStream 在集成 CPU–GPU 上细分窗口计算任务，比较了不同执行安排的吞吐和延迟。[FineStream](https://www.usenix.org/conference/atc20/presentation/zhang-feng) | 换用独立设备后，中间结果可能需要跨互连传输，数据搬运时间进入每个窗口的完成时间。 |
| 近似查询 | BlinkDB 按响应时间或精度要求选择样本，在 TPC-H 和实际分析负载上测量查询耗时与误差。[BlinkDB](https://dspace.mit.edu/entities/publication/37668545-3cfd-4ba8-87b6-6a617d888e7d) | 下游若根据估计值触发告警，估计误差在告警阈值附近可能改变决策。 |
| KV 分页与阶段调度 | vLLM 按需分配 KV 块，在延迟相近时提高了相对基线的吞吐。Sarathi-Serve 把预填充拆块，使运行中的解码请求能继续推进。[PagedAttention](https://arxiv.org/abs/2309.06180) · [Sarathi-Serve](https://www.usenix.org/conference/osdi24/presentation/agrawal) | 显存紧张时，分页影响能同时容纳多少请求；长提示挤占计算时，分块影响活跃请求等待下一个 token 的时间。 |
| 前缀复用与传输 | Mooncake 在派发请求时估计各节点的排队和处理时间。MemServe 在节点已选定时还比较获取额外前缀和重算的时间。LMCache 提供查找、固定和搬运缓存块的操作。[Mooncake](https://doi.org/10.1145/3773772) · [MemServe](https://arxiv.org/html/2406.17565v3) · [LMCache](https://arxiv.org/html/2510.09665v2) | 回访时先判断缓存块能否用于当前模型和适配器，再比较等块传来和就地重算哪种能更早给出首个 token。 |
| 适配器复用 | S-LoRA 将请求所需的适配器从主存调入 GPU，在一个内存池中管理适配器权重和 KV 张量，并测量多适配器服务的吞吐。[S-LoRA](https://arxiv.org/abs/2311.03285) | 适配器切换后，复用旧 KV 前须核对生成这些块时使用的模型参数是否与新请求兼容。 |
| 索引维护 | FreshDiskANN 先把插入写到内存索引，随后并入磁盘图。SPFresh 把新向量写入最近的分区，分区过大时拆分并调整附近向量。[FreshDiskANN](https://arxiv.org/abs/2105.09613) · [SPFresh](https://doi.org/10.1145/3600006.3613166) | 在相同的插入和查询序列上，可观察后台合并或局部修整期间的查询延迟与召回率。 |
| 按需检索 | Self-RAG 在生成过程中决定何时检索文档，并在问答、事实核查和长文本生成任务上评价结果。[Self-RAG](https://arxiv.org/abs/2310.11511) | 语料持续更新时，同一次检索能找到哪些文档，还取决于索引已发布到哪个版本。 |
| 有限记忆与重放 | MIR 在每次更新时从缓冲区挑出容易受干扰的旧样本重放。GDumb 按类别留样，评价时只用缓冲区从头训练。[MIR](https://proceedings.neurips.cc/paper/2019/file/15825aee15eb335cc13f9b559f166ee8-Paper.pdf) · [GDumb](https://www.ecva.net/papers/eccv_2020/papers_ECCV/papers/123470511.pdf) | 同样大小的缓冲区，在 MIR 中影响下一步能挑到哪些旧样本，在 GDumb 中决定评价时全部可用的训练数据。 |


**表 8 与 R1 对应的测试。** 同一条多热键输入轨迹比较三种策略：维持放置、检测到热点即迁移、按资源余量分批迁移。固定初始状态、目标端容量、候选键集和前台限流规则，并将负载从突发恢复到原来的水平。记录源端与目标端队列、前台达标吞吐、尾延迟、迁移字节数和启停时刻。突发结束后继续记录至队列恢复，或达到预先给定的观测上限。策略若限制准入，同时报告接受、延期和拒绝的事件数，从而区分队列缩短是因为少接了事件，还是因为事件处理得更快。

表 8 的标题改为“建议的扰动测试”。每行给出一次负载变化，以及从变化开始到恢复期间要记录的指标。

| 场景 | 负载与对照组 | 记录指标 |
|---|---|---|
| 热键迁移 | 同一输入和初始队列，比较不迁移、立即迁移、按余量分批迁移 | 队列长度、尾延迟、达标吞吐、拒绝量、恢复时间 |
| KV 回收 | 同一请求序列与显存预算，第二波分别复用或不复用被回收的前缀 | 第一波腾出的显存、第二波的重算或加载时间与首 token 延迟 |
| 索引维护 | 固定查询与更新序列，改变何时修复、何时开放新版本 | 查询延迟与召回，以及删除到生效的时间和积压更新数 |
| 样本保留 | 同一学习任务流，在相同更新步缩减内存预算 | 每轮实际存储量、压缩或删除的样本、后续任务上的遗忘与恢复 |
| 共享中间件 | 对相同服务和更新负载，比较各自单独运行与共用链路并发运行 | 前台 SLO 违例、后台进度、哪项工作被推迟，以及结束时还欠多少工作 |

### §4.2.6｜表 3 的第五问与表 5 的五列对不上

*作者补查：R2 点名比较表，核对表头与正文是否一致。*

[main.tex 第 611 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L611)

改前 · 英文

> it shows how **the same five fields** expose recurring seams across different state lifetimes and evaluation boundaries

改前 · 中文

> 它展示了**相同的五个字段**如何揭示不同状态寿命和评价边界下反复出现的控制问题

问题　§2 问过一次操作会让谁多等。原表把各领域分行列出，读者还得自己猜。同样写着“分批”，Megaphone 和 Sarathi-Serve 分的是不同的工作，解开的也是不同的等待。把这两个表项连起来读。

改后 · 英文草案

> Table 5 gives two reasons to split work into smaller units. **Megaphone moves groups of keys in succession so fewer keys wait for a handoff at once. Sarathi-Serve splits a long prompt’s prefill so that requests already decoding can generate another token between chunks.** In the first case, look at record latency while ownership changes. In the second, look at the time between tokens for requests already in service.

改后 · 中文

> 表 5 中有两种分批。**Megaphone 让各组键依次搬迁，使同一时间等待交接的键更少。Sarathi-Serve 把长提示的预填充拆成小块，让已经开始解码的请求在块与块之间有机会再生成一个 token。** 前者要看交接期间流记录等了多久，后者要看在途请求的两个 token 之间隔了多久。

[Megaphone 的分批迁移](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf) · [Sarathi-Serve 的分块预填充与调度](https://www.usenix.org/conference/osdi24/presentation/agrawal)

### §4.3｜“分析成熟度”该用证据说明

*作者补查，归入 R2 对表 5、6 的意见。*

[main.tex 第 657 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L657)

改前 · 英文

> Across access, execution, and evolution, local gains **often disappear** once effects propagate through the state lifecycle.

改前 · 中文

> 在访问、执行和演化中，局部收益在影响沿状态生命周期传播后**往往会消失**。

问题　原句说收益“经常消失”，表内却没有统计过频率。改用 Megaphone 的一项观察：待发数据在哪儿积起来，分批搬以后哪里发生了变化。

改后 · 英文草案

> In Megaphone’s all-at-once migration experiment, **the source serialized state faster than the network could send it**. Starting the next batch only after the previous one completed limited the state waiting to be sent. **On a busy source, even the first batch needs CPU time and memory bandwidth that live events may already be using.**

改后 · 中文

> 在 Megaphone 的一次性迁移实验中，**源节点序列化数据的速度超过了网络发送速度**，打包好的数据于是积在发送端。改成上一批结束后再搬下一批，待发数据量就受到了限制。**如果源节点很忙，第一批所需的 CPU 时间和内存带宽也可能已经被在线事件占满。**

迁移粒度和重配置期间延迟的实验见 [Megaphone 图 1、图 19–20](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf)。

### §6.1｜用评价回答具体论断

*作者补查 · 原文对照*　[main.tex 第 737 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L737)

> Systems that score well on only one dimension deliver **local improvements rather than full state-management solutions**.

> 只在一个维度上表现好的系统提供的是**局部改善，而非完整的状态管理方案**。

这句给出评价原则。R2 所问的实验依据直接落在表 5、6、8，保留本句供对照。

## R2｜§3.4 的定量例子

**审稿原文**

> Section 3.4 promises a quantitative example but provides a qualitative discussion. Please supply the numbers, sources, and assumptions, or revise its title and introduction.

**中文**　§3.4 承诺提供定量示例，给出的却是定性讨论。请补上数字、来源和假设，或者修改标题与开头。

**我的意见**　五步因果链可以保留。就目前材料，我倾向于把它写成定性案例。

**准备怎么改**

### §3.4，标题与首段（main.tex 第 512 行）

改前 · 英文

> **Quantitative Propagation Trace: A Worked Example**

改前 · 中文

> **定量传播轨迹：一个演算示例**

问题　标题承诺带读者算一遍。首段甚至说用了两篇论文的代表性数字，但后面没有任何容量、时延或算式。标题和首段一起改。

改后 · 英文草案

> **How KV Eviction Affects the Next Request**

改后 · 中文

> **驱逐 KV 缓存怎样影响下一个请求**

### §3.4，替换首句（main.tex 第 515 行）

改前 · 英文

> with **representative numbers drawn from the vLLM and Sarathi-Serve literature**

改前 · 中文

> 采用**取自 vLLM 和 Sarathi-Serve 文献的代表性数字**

问题　这处是首句中的原文节选。直接写示例的假设和读者要观察的后果。

改后 · 英文草案

> Suppose **one serving worker uses a fixed pool for both active KV pages and reusable prefixes from completed requests**. A long prompt is admitted and the allocator reclaims a completed request’s unpinned prefix. If a later request needs those tokens, **it reloads the old pages if a valid copy remains elsewhere, or repeats prefill if the last copy was discarded**. Either action can delay its first token.

改后 · 中文

> 假设**同一服务节点用一个容量固定的池，同时存放活跃请求的 KV 页和已结束请求留下的可复用前缀**。接纳长提示后，分配器回收了一个已结束请求的、未被固定的前缀。后来的请求若需要这些 token，**别处还有有效副本就重新加载，最后一份副本也被丢弃就重新运行预填充**。两种做法都可能推迟首个 token 返回。

### 案例 B｜一篇文章里，“驱逐 KV”指了三种动作

*作者补查：R2 质疑 §3.4 的案例，向前核对同一个案例中的动作。*

[§3.3.4 的总结句](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L510)；对照 §3.2.2 第 458 行、§3.2.4 第 474 行和 §3.4 第 517–519 行。

改前 · 英文

> In Example B, **token-level KV eviction** (an execution decision) **reduces future prefix-reuse value** (an evolution loss).

改前 · 中文

> 在案例 B 中，**按 token 驱逐 KV**（执行决策）会**降低未来前缀复用的价值**（演化阶段的损失）。

问题　§3.2.2 说把页搬到 CPU，数据仍在，以后可以取回。§3.2.4 说的是一种近似执行方式：解码时只让注意力看到选中的 token，可能改变答案。到了 §3.4，真正要讲的是另一个请求回来后找不到可复用的完整前缀。三者都能省显存，但哪份数据还在、影响哪次请求都不同。建议让 §3.4、§5.4 和附录 C.1 始终追踪同一个可复用前缀，前两种操作只在各自涉及的地方对比。

改后 · 英文草案

> **Offloading a KV page preserves a copy that can be fetched later. Dropping selected tokens from the active attention context can change the current output.** Example B follows a third decision: **reclaiming a cached prefix after its request finishes**. If no other copy survives, the next request with that prefix must recompute it.

改后 · 中文

> **卸载 KV 页会保留一份以后可取回的副本。让当前注意力计算跳过一部分 token，则可能改变当下的输出。** 案例 B 跟踪第三种决定：**请求结束后，回收原本可供复用的前缀缓存**。如果没有其他副本，下一个使用相同前缀的请求就必须重算。

## R2｜§6.3 的研究方向

**审稿原文**

> Section 6.3 should explain the concrete challenges behind its research directions and how they remain unresolved, rather than repeat earlier recommendations.

**中文**　§6.3 应说明每个研究方向背后有什么具体困难，以及为什么仍未解决，避免重复前面的建议。

**我的意见**　这一节最值得留下的是那些在系统运行时就得决定、后果却要晚些才知道的事。缓存该不该删就是一个好例子。

**准备怎么改**

### §6.3 末句（main.tex 第 774 行）

改前 · 英文

> The common research target is **a shared control layer that makes state identity, movement cost, exposure legality, and lifecycle debt composable under one service boundary**.

改前 · 中文

> 共同的研究目标是**建立一个共享控制层，使状态身份、移动成本、开放使用的合法性和生命周期中延后的代价，能在同一服务边界下组合起来**。

问题　这里一次点了四个名词，却没说现有办法卡在哪里。以缓存回收为例，把已知量、未知量和能检验的结果各写一句。其余方向若找不到同样具体的问题，就压缩。

改后 · 英文草案

> The allocator knows **how many pages it can free by evicting a prefix**. The saving is immediate. The cost comes later, **if a user returns and has to wait for that prefix again**. A useful eviction policy would use the observed return intervals and reload times to choose which prefixes to keep within the memory budget.

改后 · 中文

> 分配器知道**删掉一段前缀能腾出多少页**。空间当场就省下来了。代价发生在以后，**用户再次回来时，可能得等系统取回这段前缀**。驱逐策略若能根据已观察到的回访间隔和加载耗时挑选前缀，就有机会在相同预算下减少这段等待。

§6.3 的另外三个研究方向可各用一个具体困难展开。

> **Observation.** A queue can fill between the time a worker reports its length and the time a migration starts. An admission rule needs the age of the reading and an estimate of how much load can arrive during that interval.
>
> **Placement.** A GPU can finish an operator sooner yet deliver its result later if moving the input saturates a shared link. Predicting completion time requires the link bandwidth left by other traffic as well as the device profile.
>
> **Evaluation.** Two runs may free the same number of pages during a burst. Their session latencies can diverge on the next visit if one run retained a reusable prefix and the other must recompute it.

中文：**观测。** 节点上报队长后，到迁移实际启动前，队列可能已经满了。调度器需要知道读数有多旧，以及这段时间还可能到达多少工作。**放置。** GPU 可以更快算完一个算子，但如果输入要经过拥挤的链路，结果反而可能更晚返回。预测完成时间时，要同时估算设备耗时和其他流量给链路留下的带宽。**评价。** 两次运行在突发期间腾出的页数可能一样。用户下次回来时，一次运行还保留着可复用前缀，另一次需要重算，两者的会话延迟这时才拉开差距。

## R2｜§4.2.3 的 MemServe 和 LMCache

**审稿原文**

> Please consider MemServe and LMCache when discussing cache sharing, state transfer, and control interfaces, as they may affect the assessment of what remains unresolved.

> I recommend to cite two more papers in \S 4.2.3. Both works manage KVCache state similar to Mooncake:

> [1] Hu, Cunchen, Heyang Huang, Junhao Hu, Jiang Xu, Xusheng Chen, Tao Xie, Chenxi Wang et al. "Memserve: Context caching for disaggregated llm serving with elastic memory pool." arXiv preprint arXiv:2406.17565 (2024).

> [2] Liu, Yuhan, Yihua Cheng, Jiayi Yao, Yuwei An, Xiaokun Chen, Shaoting Feng, Yuyang Huang et al. "LMCache: An efficient KV cache layer for enterprise-scale LLM inference." arXiv preprint arXiv:2510.09665 (2025).

**中文**　讨论缓存共享、状态传输和控制接口时，请考虑 MemServe 和 LMCache。建议在 §4.2.3 引用这两篇论文。它们同 Mooncake 一样管理 KV 缓存，可能改变对剩余问题的判断。

**我的意见**　MemServe 已经会在传输和重算间作选择，LMCache 也已经给出移动和固定缓存的接口。读者需要先看到这些差别，才能理解 §4.2.3 为什么还要单讲复用规则。

**准备怎么改**

[§4.2.3 的改前、改后与中英文对照](#sec423-kv)列在五组比较的第三组。

**参考文献怎么录。** R2 推荐的两篇都还没有收入送审版 `refs.bib`。下面两条按本段实际引用的版本和论文首页署名填写。MemServe v3 首页题名与 arXiv 摘要页不同。LMCache v2 首页前三位依次为 Yuhan Liu、Jiayi Yao、Yihua Cheng，摘要页的第二、三位顺序相反。审稿原文保留来信的写法。[MemServe v3 首页](https://arxiv.org/pdf/2406.17565v3) · [MemServe 摘要页](https://arxiv.org/abs/2406.17565v3) · [LMCache v2 首页](https://arxiv.org/pdf/2510.09665v2) · [LMCache 摘要页](https://arxiv.org/abs/2510.09665v2)

```bibtex
@misc{hu2024memserve,
  author = {Cunchen Hu and Heyang Huang and Junhao Hu and Jiang Xu and
            Xusheng Chen and Tao Xie and Chenxi Wang and Sa Wang and
            Yungang Bao and Ninghui Sun and Yizhou Shan},
  title = {{MemServe}: Flexible Mem Pool for Building Disaggregated {LLM} Serving with Caching},
  year = {2024},
  howpublished = {arXiv:2406.17565v3},
  url = {https://arxiv.org/abs/2406.17565v3}
}

@misc{liu2025lmcache,
  author = {Yuhan Liu and Jiayi Yao and Yihua Cheng and Yuwei An and
            Xiaokun Chen and Shaoting Feng and Yuyang Huang and Samuel Shen and
            Rui Zhang and Kuntai Du and Junchen Jiang},
  title = {{LMCache}: An Efficient {KV} Cache Layer for Enterprise-Scale {LLM} Inference},
  year = {2025},
  howpublished = {arXiv:2510.09665v2},
  url = {https://arxiv.org/abs/2510.09665v2}
}
```

§4.2.3 中，派发请求的句子引 `qin2025mooncake`，比较传输与重算的句子引 `hu2024memserve`，查找、固定和移动缓存块的句子引 `liu2025lmcache`。现有 Mooncake 条目指向期刊版，可保留引用键，将 “Just Accepted, 2025” 更新为 *ACM Transactions on Storage* **22(4), 2026, pp. 1–38**。表内和正文统一引这个版本。[ACM 期刊记录](https://doi.org/10.1145/3773772)

同次核对还有两处更正。§4.2.1 使用的 `nasir2019megaphone` 少了后三位作者，完整署名为 **Moritz Hoffmann, Andrea Lattuada, Frank McSherry, Vasiliki Kalavri, John Liagouris, Timothy Roscoe**。§4.2.3 第 587 行的 **LLMnix** 应为 **Llumnix**，其引用键 `sun2024llumnix` 和文献表题名拼写正确。[Megaphone 首页](https://www.vldb.org/pvldb/vol12/p1002-hoffmann.pdf) · [Llumnix，OSDI 2024](https://www.usenix.org/conference/osdi24/presentation/sun-biao)

书目试排还涉及三个字段。Flink 补上 `pages = {28--38}`，题名中的系统名写作 `{Apache Flink}`。FreshDiskANN 引用 2021 年预印本，条目用 `@misc` 和 `howpublished = {arXiv:2105.09613}`。MIR 的 `address` 填出版地 `Red Hook, NY, USA`。[Flink 的出版记录](https://repository.tudelft.nl/record/uuid%3Adf177547-a436-4bb0-a7e2-470b83025bb0) · [FreshDiskANN 的 arXiv 记录](https://arxiv.org/abs/2105.09613) · [MIR 出版方地址](https://www.proceedings.com/info.html)

## R2｜参考文献编号和编辑措辞

**审稿原文**

> The main text and appendix use separate reference numbering, so the same number can identify different papers. A unified or clearly distinguished numbering scheme would make citations easier to follow.

> Please remove editorial phrases such as “The main paper now carries…” and “The remaining serving papers that have not yet been mentioned…”. The symbolic representation in Section 2 is also unnecessary and I feel it strange to human readers.

**中文**　正文和附录各自编号，同一个编号可能指不同论文。建议统一编号，或让两套编号容易区分。请删除“The main paper now carries…”和“The remaining serving papers that have not yet been mentioned…”这类编辑措辞。第 2 节的符号表示也显得多余、不自然。

**我的意见**　这里有两个能直接定位的问题。引用编号会撞，附录里还留了写稿时的过渡句。

**准备怎么改**

### 补充材料首页（supplement.tex 第 41 行）

改前 · 英文

> Its bibliography includes the works cited directly in the appendix.

改前 · 中文

> 附录的参考文献表收录附录中直接引用的工作。

问题　这句只解释收录哪些论文，不能让人分清同一个 [3]。建议补充材料单独用 S 前缀，同时改文中和文末编号。例如正文 [3] 仍指 Keyformer，补充材料的 BlinkDB 改为 [S3]。下句在编号实际改好后使用。

改后 · 英文草案

> **References in this supplement are numbered [S1], [S2], and so on.**

改后 · 中文

> **补充材料中的参考文献依次编号为 [S1]、[S2] 等。**

**编号方案已在隔离副本试编。** 在补充材料本地副本的 `\begin{document}` 后重定义 natbib 的两个显示命令：

```tex
\renewcommand{\citenumfont}[1]{S#1}
\renewcommand{\bibnumfmt}[1]{[S#1]}
```

用本仓库的 acmart 和 Tectonic 编译完整补充材料副本，核对了文末 [S1]–[S183] 的连续编号，并核对 PDF 中实际出现的引文跳转目标。BlinkDB 在文中显示为 S3，在文末显示为 [S3]，链接指向对应条目。正式返修采用该写法时，用最终文献集合重新编译并更新页码。

### 附录 D.2，首段（extended_domain_notes.tex 第 30 行）

改前 · 英文

> **The main paper now carries** the core predictive, structural-reuse, and temporal-lifecycle comparison.

改前 · 中文

> **主文现在涵盖了**有关预测、结构化复用和生命周期随时间变化的核心比较。

问题　“现在放在正文”是编稿过程。开头直接说这一节所比较的问题。

改后 · 英文草案

> **When a request arrives, a serving system must decide whether an existing KV prefix can be reused and where to retrieve it.**

改后 · 中文

> **一个请求到达时，服务系统必须判断已有的 KV 前缀能否复用，以及该从哪里取回它。**

### 附录 D.2，列举其余工作的一段（extended_domain_notes.tex 第 42 行）

改前 · 英文

> The remaining serving papers **that have not yet been mentioned directly** still fit the same lifecycle view.

改前 · 中文

> 其余**尚未被直接提及**的服务系统论文也符合同一种生命周期视角。

问题　“还有些论文没提到”没有信息量。删去这句，保留后文确实有区别的机制。

改后 · 英文草案

> **Keeping an adapter resident saves a later load. Reusing a prompt prefix saves computation only when the stored state is valid for the new request.**

改后 · 中文

> **让适配器保持驻留可以省去以后的加载。复用提示前缀只有在已存状态对新请求仍然有效时，才能省去计算。**

<a id="review-3"></a>

## R3｜篇幅、重复与长句

**审稿原文**

> 1. The paper is too long.  Certain sections in the main text (such as Section 3.2 and 3.3) could  overwhelm the reader with citation lists rather than comparative insights. Streamlining narrative transitions around these lists will improve readability.

> 1. Some materials in section 4 and Appendix C are similar.  It can be represented using Table.

> 2. Some of the sentenses, such as those in propagation view and life cycle, are too long.  It would be better to replace them by shorter ones.

**中文**　文章过长，§3.2、§3.3 等处的引文列表可能淹没比较见解，应精简衔接。§4 与附录 C 有些内容相似，可以用表格组织。传播视角和生命周期部分有长句，建议拆短。

**我的意见**　§3.2、§3.3 的篇幅主要花在按名字列系统。正文各留一组能看清差异的工作。附录 C 留下请求回访、交接失败这些需要按时间顺序讲的细节。

**准备怎么改**

**可以直接删掉的句子。** 作者补查，归在 R3 的篇幅和可读性意见下。

| 出处 | 送审稿原句 · 中文 | 拟处理 |
|---|---|---|
| §2.1，第 192 行 | “the paper is **not meant to function as an encyclopedic bibliography**”（原句节选：本文**无意充当百科式参考书目**。） | **删去。** 紧接文中的存储引擎例子：同一种压实操作，一篇只介绍存储布局，另一篇让调度器决定何时执行。读者由此能看出本文选文的角度。 |
| §2.2，第 202 行 | “This manuscript **does not replace specialized surveys**.”（本文**并不取代专门领域的综述**。） | **删去。** 本段直接比较已有综述按流处理、硬件或持续学习组织文献，而本文在不同领域间比较状态管理决策。 |
| §2.6.1，第 370 行 | “The tuple is meant to be **reusable rather than merely descriptive**.”（这个元组旨在**供人复用，而不只是描述**。） | **删去首句。** 上面的缓存前缀例子已经实际用过表 3，顺势进入文献比较。 |
| §3.3.2，第 496 行 | “Retention is **not a background garbage-collection task**; it is a forward-looking service-quality control.”（保留**不是后台垃圾回收任务**，而是面向未来的服务质量控制。） | 改成 **“If the index drops an older embedding today, a query tomorrow may lose the document it needs.”**／**“索引今天删掉一个旧嵌入，明天的查询就可能找不到所需的文档。”** 与前一句提到旧嵌入的例子合并。 |
| §4.1，第 534 行 | “Using that tuple, we group work by recurring control problem **rather than by venue order or application label**.”（我们按反复出现的控制问题分组，**而非按发表场所次序或应用标签**。） | **删去。** §2 已经交代分类方式。§4 开头用迁移分片和传输 KV 的比较承接 §3。 |
| §5.3，第 715 行；附录 B.5，第 47 行 | “The blueprint is **not intended to prescribe a single control-plane architecture**.”／“The blueprint is a conceptual reference model, **not a monolithic runtime**.”（蓝图**无意规定单一的控制平面架构**；它是概念参考模型，**并非单体运行时**。） | **两处都删。** 正文保留同一条迁移规则在内嵌调度器和边车控制器上怎样执行的区别；附录直接给出准入、暂停与恢复步骤。 |
| §6.2，第 768 行 | “This negative boundary is **not a weakness in the contribution**; it is what makes the contribution reviewable.”（这个不适用条件**并非贡献的弱点**，它使贡献可供评审。） | **删去。** 保留具体例子：流分片交接时要防止重复应用事件；KV 前缀复用要核对模型、适配器和前缀是否匹配。 |

### §3.2.3 的引文串（main.tex 第 468 行）

*作者补查 · 原文对照*

> Recent systems refine this lifecycle by **changing representation, tiering, or sharing semantics**.

> 近期系统通过**改变表示、分层方式或共享语义**来细化这一生命周期。

这个开场句与它后面的长名单一起压缩，见本项的两段合并稿。

### §3.3.3 的检索段（main.tex 第 504 行）

*作者补查 · 原文对照*

> Modern retrieval pipelines expose **concrete control surfaces**.

> 现代检索流水线提供了**具体的控制手段**。

这处开场句也随第 504–506 行合并，合并稿在本项后面。

### 附录 C.1 与 §4.2.3 重复的地方（extended_case_studies.tex 第 14 行）

改前 · 英文

> A locally beneficial admission may force **evict/reclaim decisions that destroy future reuse**, causing later re-materialization traffic and queue amplification.

改前 · 中文

> 一次局部有利的准入，可能迫使系统作出**破坏未来复用的驱逐或回收决定**，从而在以后产生重新生成状态的流量，使排队加剧。

问题　正文已经说过回收会增加恢复成本。附录若保留这个案例，就展示两次请求的时间顺序，以后可以把这两步放在表中。

改后 · 英文草案

> **At admission**, request B needs pages still occupied by A’s prefix. **On A’s next turn**, a miss sends that prefix back through prefill and delays the first output token.

改后 · 中文

> **接纳请求时**，请求 B 需要的缓存页仍被 A 的前缀占用。**等到 A 的下一轮请求**，一次缓存未命中让这段前缀重新经过预填充，延后了首个输出 token 的生成。

### 生命周期长句（本次选例，R3 未指定原句）（main.tex 第 676 行）

改前 · 英文

> That lesson is visible across domains: **migration and replay expose admit, place, and expose/transfer semantics in streaming**; KV management exposes admit, expose/transfer, and evict/reclaim semantics in serving; and dynamic retrieval and retention expose mutate, compact, and evict/reclaim semantics once freshness or budget pressure is no longer negligible.

改前 · 中文

> 这一经验在多个领域都可看到：**在流处理中，迁移与重放体现了准入、放置以及开放或传输的语义**。在服务系统中，KV 管理体现了准入、开放或传输、驱逐或回收的语义。在动态检索与保留中，当新鲜度或预算压力大到无法忽略时，还会涉及修改、压实、驱逐或回收。

问题　一句话依次切换流处理、服务和检索，每次还要解读几组动作名。拆成三个有主语的短句。

改后 · 英文草案

> **A stream processor transfers a shard and resumes processing at its new owner. A serving engine admits a request and later reclaims its KV pages. A retrieval service updates or compacts an index as its corpus changes.**

改后 · 中文

> **流处理系统传输分片，并由新所有者接着处理。推理引擎接纳一个请求，稍后回收其 KV 页。检索服务随着语料变化更新或压实索引。**

### §3.2.3｜把两篇论文的贡献写成同一件事

*作者补查 · 原文对照*　[main.tex 第 466 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/main.tex#L466)

> vLLM and Sarathi-Serve then make memory layout and phase asymmetry explicit by **treating KV memory as a paged state object whose footprint evolves differently in prefill and decode**.

> vLLM 和 Sarathi-Serve 通过**把 KV 内存视为在预填充和解码阶段占用量变化不同的分页状态对象**，使内存布局与阶段差异显式化。

这一句把分页与分块调度写到了一起。具体归因在下方的整段替换稿中修正。

两篇原文分别把主要机制写为 KV 块管理和分块预填充调度。[PagedAttention](https://arxiv.org/abs/2309.06180) · [Sarathi-Serve](https://www.usenix.org/conference/osdi24/presentation/agrawal)

§3.2.3 第 466–470 行用 PagedAttention、Sarathi-Serve 和 DistServe 比较内存占用、解码等待和跨节点传输。缓存的复用规则留在 §4.2.3。§3.3.3 第 504–506 行并成一段，比较请求到达时的检索决定与索引更新对后续查询的影响。剩余文献中，有独立机制的放进附录 A，重复介绍的删掉。

**两段合并后的写法。** §3.2.3 合并第 466、468、470 行，§3.3.3 合并第 504、506 行。两处英文合计由 457 词缩至 150 词，不计引文。

> PagedAttention allocates KV memory in blocks as requests grow. Sarathi-Serve splits long prefills into smaller chunks so that ongoing decode requests can continue to advance. DistServe places prefill and decode on separate workers, introducing a transfer between stages. **The limiting resource changes across these designs: space for active sequences, time in the next batch, or bandwidth for the state that must move.** Section 4.2.3 compares when a cached prefix at one worker can actually serve a request at another.

中文：PagedAttention 随请求增长按块分配 KV 内存。Sarathi-Serve 将长预填充拆块，使进行中的解码可以继续。DistServe 将预填充和解码放在不同节点，从而引入阶段间传输。**几种设计面对的限制依次是活跃序列所占空间、下一批次的计算时间，以及必须搬运状态的带宽。** §4.2.3 再比较一个节点上的缓存前缀何时真能供别处的请求使用。

> A retrieval system makes decisions at two different times. Self-RAG decides during generation whether to consult external documents. FreshDiskANN updates the search index as vectors arrive or are deleted. **The first decision changes which evidence an answer uses. The second changes which evidence a future query can find.** A static retrieval test can measure the first effect while missing the delay between an update and its visibility to queries.

中文：检索系统在两个不同的时刻作决定。Self-RAG 在生成时决定是否查阅外部文档。FreshDiskANN 在向量增加或删除时更新搜索索引。**前一个决定改变当前回答采用哪些证据，后一个决定改变未来查询能找到哪些证据。** 静态检索测试可以测到前者，却可能看不到从更新发生到查询可见的时延。

这两段的机制依据见 [PagedAttention](https://arxiv.org/abs/2309.06180)、[Sarathi-Serve](https://www.usenix.org/conference/osdi24/presentation/agrawal)、[DistServe](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin)、[Self-RAG](https://arxiv.org/abs/2310.11511) 和 [FreshDiskANN](https://arxiv.org/abs/2105.09613)。

### 附录 A.4｜用相邻工作划清边界

*作者补查 · 原文对照*　[extended_landscape.tex 第 35 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/supplement_sections/extended_landscape.tex#L35)

> Together, these adjacent lines help motivate **contract-oriented blueprint and middleware pressure-point arguments** rather than one more domain-specific taxonomy.

> 总体而言，这些相邻方向为**面向契约的蓝图和中间件压力点论证**提供动机，而不是再给出一套特定领域的分类。

这句交代历史背景的用途。R3 关心的重复集中处理在 §4 和附录 C；附录 A 不另写一段。

### 附录 C.2｜给出正文没有展开的失败分支

*作者补查：R3 明确提到附录 C，核对案例能否补足正文。*

[extended_case_studies.tex 第 37 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/supplement_sections/extended_case_studies.tex#L37)

改前 · 英文

> The missing piece is a **disturbance-invariant ownership protocol** that unifies visibility, transfer legality, and replay semantics across scale-out and recovery rather than re-deriving them per subsystem.

改前 · 中文

> 缺少的是一个**在扰动下保持不变量的所有权协议**，能统一扩容和恢复时的可见性、转移合法性与重放语义，避免各子系统分别推导。

问题　案例已经到最后一步，应该告诉读者交接中途出故障怎么办。用两个具体故障时点体现补充材料的价值。

改后 · 英文草案

> Consider a failure **after the copy but before the recorded ownership change**. Recovery must identify the last committed owner and input position. **After the change commits**, retries must honor the new owner and must not apply an event twice.

改后 · 中文

> 设想故障发生在**复制结束后、所有权变更记入记录之前**。恢复时必须找到最近已提交的所有者和输入进度。**所有权变更一旦提交**，重试就必须服从新的所有权，并避免重复应用同一事件。

## R3｜传播公式与跨领域比较

**审稿原文**

> 2. The propagation equations in Section 2.5.2  are conceptually compelling. Providing a brief worked mathematical interpretation or expanding slightly on how partial derivatives manifest in concrete metrics (e.g., TTFT vs. queue length) would strengthen the mathematical rigor.

> 3. The bridge between classical distributed systems and modern AI serving/retrieval is good. However, adding a dedicated paragraph explicitly contrasting deterministic state consistency against stochastic state utility would sharpen the cross-domain synthesis in Section 4.

**中文**　§2.5.2 的传播公式很有吸引力。希望用简短的数学例子解释偏导数怎样对应具体指标，例如 TTFT 与队列长度。经典系统与现代 AI 系统的联系讲得好。建议在 §4 单列一段，对比确定性的状态一致性与带有不确定性的状态效用。

**我的意见**　传播图可以保留。把三个算子之间的偏导换成一项看得懂的量：队列里多一个请求，或前缀多一次重算，首 token 会晚多久。第 4 节用版本发布规则和检索质量分别说明一致性、效用。

**准备怎么改**

### §2.5.2，偏导数后面的解释（main.tex 第 326 行）

改前 · 英文

> In words, **a local perturbation in one stage propagates and reappears as a constraint in the other two**.

改前 · 中文

> 用文字来说，**一个阶段中的局部扰动会传播，并重新表现为另外两个阶段的约束**。

问题　这句话没有给出被改变的量和被观测的量。若要保留数学，用一个假设明确的小例子替代直接对阶段算子求导。

改后 · 英文草案

> Suppose a worker processes requests in arrival order, one at a time. A new request arrives behind q requests, each with s milliseconds of service remaining. Let t be the time from starting the new request to producing its first token. Then **time to first token is qs + t**. With two requests ahead at 5 ms each, the queue adds 10 ms. At the same queue length, **losing a cached prefix adds a further 4 ms if the request spends 4 ms recomputing it before decoding**. Continuous batching changes the calculation because several requests can share a decoding step.

改后 · 中文

> 假设一个节点按到达顺序逐个处理请求。新请求到达时，前面有 q 个请求，每个还需 s 毫秒服务。设新请求从开始处理到产生首个 token 用时 t，则**首 token 延迟为 qs + t**。前面有两个请求、每个各需 5 毫秒时，光排队就要 10 毫秒。保持队长不变，**如果丢失缓存前缀使请求在解码前多花 4 毫秒重算，首 token 就再晚 4 毫秒**。这些数字用于演算。连续批处理允许多个请求共用一个解码步，等待时间需要另算。

### §4.2.4 末句（main.tex 第 599 行）

改前 · 英文

> In both cases, the systems question is when the runtime can still treat memory as **legally useful**.

改前 · 中文

> 两种情况下，系统都要判断，运行时何时仍能把记忆视为**符合使用规则且有用**。

问题　一个版本按协议可以发布，与它对用户有没有帮助，是两件事。“legally useful”把两者揉在一起。建议在 §4 明确各用什么去判断。

改后 · 英文草案

> A stream processor checks **which worker owns a shard** before accepting a write. A retrieval service checks an index version before publishing it. Both checks have a definite outcome for a given system state. Once the index is live, **the value of its contents depends on the query**. Recall measured over queries sampled from the target workload estimates how often useful documents are retrieved.

改后 · 中文

> 流处理器在接受写入前检查**哪个节点拥有这个分片**。检索服务在发布索引前检查版本。对给定的系统状态，这两项检查都有明确结果。索引上线以后，**里面的内容有没有用，要看用户问什么**。从实际负载中抽取一组查询测召回率，可以估计相关文档被找回的比例。

**§4 的放置与 §2 的取舍。** 将上面的中英文对照作为 §4.1 末尾独立一段，接在比较原则后面；§4.2.4 结尾删去 “legally useful” 那句，只回指读者现在已经知道的区别。§2.5.2 保留阶段关系图，删除三条未定义的算子偏导，用排队段落说明固定其他条件时的有限变化；实际批处理含阶段重叠，不能套用该式推算 p99。

### 附录 D.3｜记忆的价值未必随时间缓慢下降

*作者补查 · 原文对照*　[extended_domain_notes.tex 第 65 行](https://github.com/DataSysResearch/parallel-distributed-state-management-survey/blob/cca9f5a4c6f023335156723a8bc33a38aba1a080/supplement_sections/extended_domain_notes.tex#L65)

> whose **value decays gradually** and whose maintenance debt is easy to hide if the runtime only reports steady-state quality

> 它的**价值逐渐衰减**，如果运行时只报告稳态质量，累积的维护代价就容易被掩盖

保留为效用讨论的对照。正式修稿若保留此句，只将 **value decays gradually** 收窄为 **value can change**（价值可能变化）。一致性和预期效用的实质区别在 §4 集中讨论。
