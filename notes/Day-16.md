# Day 16 · 事件驱动架构与 Kafka

> 对应文档：`Day-16-event-driven-kafka.html`
> 日期：2026-06-16

## 今日思考题

> 文档结尾「带着这两个问题回来讨论」。先写下自己的初步想法，再来 Claude Code 聊；聊完我会在每条下面补点评（像 Day 01–12 那样）。

1. 如果把 `policy-events` 这本事件日志当作 QuoteFlow 的**审计真相源**（event sourcing），相比「在 SQL 里维护一张保单当前状态表 + 另写一张 audit log」，各有什么取舍？监管复现一笔历史报价时，哪种更靠得住？
   
   - 我的想法：这道题我其实不是很确定答案，因为我觉得如果系统设计的足够好，在信息不丢失的情况下，这两种做法最终在效果上应该是一样的。直接用事件日志当审计真相源，好处就是一石二鸟，用它管理消息的同时它记录的信息本身就能作为审计用途，舍的方面我觉得就是那些事件驱动本身存在的问题：consumer需要幂等，追踪整个流程比较复杂，最终一致，调试难，需要处理schema演进的问题。「在 SQL 里维护一张保单当前状态表 + 另写一张 audit log」这种方法，我觉得问题在于audit log里是否像事件日志那样会去记录发生的时间，如果不是的话，那么就会丢失记录事件能带来的审计好处。再就是要单独维护一张状态表和audit log，本身维护难度相比只维护一个事件日志要高，还有可能因为要这样跨表记录信息，增加消息处理的latency。取的方面我不太清楚，可能是这样需要的存储空间更小？要复现历史报价的话，看起来应该是事件日志更靠得住，但就像我一开始说的，我不知道这个“当前状态表 + 另写一张 audit log”是怎么实现的，所以不确定它是不是就更不靠得住。

   **点评（Claude）：** 方向对，但没戳到最关键那层。这题真正的分水岭是一句话——**event sourcing 只有一个真相源；「状态表 + audit log」有两个，而两个真相源会漂移。** 你那句"设计够好两者等价"恰恰要反过来：event sourcing 的全部价值就是**不需要你设计得足够好**——dual-table 里 audit 是旁路记录、靠应用层纪律同步，只要有代码路径改了状态却漏写 audit、或 audit 那条 insert 在事务外失败，审计轨迹就残缺**且你察觉不到**；event sourcing 里日志就是系统实际依据运行的那份，审计与行为是同一个 artifact，**构造上不可能各说各话**。对受监管寿险，要的就是这个性质从架构来、不靠自觉。还有你完全没提的一层：复现报价不是"有没有记录变更"，是**能不能精确重算**——日志钉住 `rate_table_version`，重放能把保费从零算回来（reproducibility）；状态表只存结果、audit log 顶多记 what（traceability）。两个反直觉点你也猜反了：① 存储**更大**不是更小（留全部历史事件）；② 读当前态**更贵**（要重放，实践中靠一张 **derived** 的 read model/projection 加速，但它地位和 dual-table 那张"真相"状态表完全不同）。结论：复现历史报价 event sourcing 更靠得住，靠的是**单一真相源 + append-only 防篡改 + version pinning 能重算**，不是"它记了时间"。现实折中：很多受监管系统用 dual-write，但要把 audit log 和状态变更写进**同一事务**、保持 append-only，否则审计可信度大打折扣。

2. QuoteFlow 想把 partition 数从 6 加到 12 来提高并行度。对「同一保单事件有序」这条保证会发生什么？（提示：哈希到 partition 的映射变了）这暗示了什么运维难点？
   
   - 我的想法：Partition数的变化会导致保单的哈希到partition映射的变化，而只有保证同一保单进入的是同一partition，才能在partition里保持事件有序。这么做会violate这条保证。暗示的运维难点应该就是不能随便横向扩展，通过更多并行来减少处理时间。

   **点评（Claude）：** 机制答对了——`hash(policy_id) % N` 里 N 变、映射翻盘、同一保单可能进不同 partition、破坏有序。补三点精度：① 破坏的是一个**切换窗口**不是全盘——Kafka **不回头搬旧消息**，pol_8f3a 旧事件留在旧 partition、新事件路由到新 partition，历史被**劈成两半**，跨 partition 没有序 → cut-over 这一刻单实体全序断掉。② 运维难点更具体：partition 数**只能加不能减、且加的瞬间就破坏 key 有序**，本质是个**要趁早拍对的容量决策** → 通常一开始就多开（要 6 直接开 24）。③ 纠正你"不能横向扩展"：可以自由加 worker 提并行，**上限 = partition 数**，开 12 跑 6、随时加到 12 零破坏；痛点**只在要超过 partition 数时**。你待深挖里挂的 Day 13 consistent hashing 方向对，但注意：它减少的是"加节点时多少 key 搬家"，而对"单 key 严格有序"，**哪怕一个 key 改归属就够制造窗口** → 所以主流答案是"提前开够 partition"，不是"用一致性哈希就没事"。

## 提问 & 解答

> 讨论时冒出来的问题记这。每条：先记自己的问题，再记讨论后的结论（用自己能看懂的话）。

### Q1. retention 只有 7 天，replay 还成立吗？

**问：** 文档说"想重算过去 30 天就把 offset 拨回去"，但如果消息只保留 7 天呢？还是说 Kafka 日志里消息永远存在、只是系统里只保留 7/30 天？

**答：** 不是"永久存、只给你看 7 天"——Kafka 没有隐藏的永久副本。retention 删的是**日志里那段真实字节**，过期 segment 文件被物理删除。所以 **replay 窗口 = retention 窗口**：retention 设 7 天，第 8 天起就重放不了，offset 拨回去落到已删区间报 `offset out of range`。两个概念别混：**offset** 是每个 consumer 自己记的进度，**retention** 是日志本身的属性、跟谁读到哪无关。要拿"日志即审计"，得显式配成留够久：① 无限 retention；② log compaction（只保每 key 最新态——给不了完整历史）；③ tiered storage（老 segment 卸到 S3 仍可查）；④ 最常见：Kafka 当热缓冲（短 retention）+ 事件落一份到持久冷存满足 7 年留存。这也是 event sourcing 的隐藏成本：要当真相源，得为"留 7 年"的存储买单。

### Q2. command / 待办任务能出现在 Kafka 里吗？

**问：** 虽说待办任务用队列更顺手，但 Kafka 这种 event-driven 日志型系统里能不能出现 command？

**答：** 能，很常见。关键是拆开两个**正交维度**：**语义**（command 祈使 / event 陈述）vs **传输**（queue 消费即删 / log append-only+offset+replay）。文档把 command→queue、event→log 配在一起是说"最自然搭配"，不是硬规则。command-on-Kafka 就是**拿 Kafka 当 work queue**（consumer group"每 partition 给一个 worker"正好是点对点分发）。换来：命令也能 replay/审计、按 key 有序、少一套基础设施。牺牲：没有 per-message ack / 重投 / DLQ / 延迟投递（毒丸要自己写跳过+转发）、**head-of-line blocking**（一条卡住后面全堵），且是广播媒介干点对点的活。一句话钉死：**transport ≠ intent**，命令骑在 Kafka 上语义仍是命令、拿不到 event 的解耦红利。最干净的搭配是两者共存、各走各的 topic：**command 进、event 出**（收 `RepricePolicy` → 干活 → 发 `PolicyRepriced`），即 CQRS 雏形。

### Q3. 物理上 Kafka 到底是个啥？

**问：** Kafka 物理意义上是个什么东西？

**答：** 一群跑在多台机器上的 **broker** 进程（JVM）。**partition 不是抽象概念**——它就是某 broker 磁盘上的一个目录，里面是一串 **segment 文件**（`.log`）；写一条消息 = 往当前 segment 文件末尾追加字节，offset = 字节在文件里的位置。"日志"不是比喻，是真在往文件尾巴写。**为什么快**：只做顺序写 + 靠 OS page cache + 读用 zero-copy（sendfile）。**容错**：每 partition 一个 leader broker + 若干 follower 副本复制日志。**协调**：老版本用 ZooKeeper，新版本用内置的 **KRaft = Kafka Raft**——这正是 Day 19 的共识 / leader election / quorum，是 Kafka 控制面的底座。

### Q4. 为什么叫 Kafka？

**问：** 这名字哪来的？

**答：** 来自作家 **Franz Kafka**。Kafka 是 LinkedIn 做的，主创 **Jay Kreps** 起名：这是个为"写"优化到极致的系统（write-heavy log），而 Franz Kafka 是个 **writer**，加上他大学读过很多卡夫卡、喜欢，就用了。**不是**因为 "Kafkaesque"（荒诞官僚噩梦）——那是大家事后玩的梗。

## 我的理解（用自己的话复述今天的核心）

> 不看文档，凭记忆写 2–4 句。能写清楚 = 真的懂了。

- 先分清命令command（需要做事，期待结果）和事件event（已经发生的事，只有最后的状态）。对于事件驱动的系统，发送方只负责报告事件，而不需要去管有谁接收到了这个信息，这是一种pub/sub的形式，方便添加新的接收方去处理信息（发送方不需要做任何改动）。Kafka这种事件日志系统，由于记录了完整的事件流程，可以直接当作audit trail来用。但是事件驱动不是免费的，要解决没有显式的完整流程逻辑，最终一致，调试困难，schema演进等等问题。

## 待深挖 / 存疑

> 还没想透、或想之后回头补的点。

- [ ] 思考题 2 接 Day 13：partition 数从 6 改成 12，`hash(policy_id) % N` 的映射整体翻盘，同一保单的新老事件可能落到不同 partition → 有序保证被打破。这正是 Day 13 consistent hashing 想缓解的「rehash 大搬迁」问题——回看一致性哈希怎么把「加节点」的影响局部化。
- [ ] Day 17（批处理 vs 流处理）：拿到这条事件流之后，到底「攒一批一起算」还是「来一条算一条」？同一条 `policy-events` 既能喂夜间重定价的 batch，也能喂实时看板的 stream。
- [ ] Day 18（幂等、重试、DLQ、exactly-once）：事件驱动里 at-least-once 依旧成立、consumer 仍要幂等（`event_id` 去重闸门），这天是主菜；schema 演进 + 重放老消息崩掉的问题也归这条线。
- [ ] Day 24（可观测性）：一笔业务横跨 N 个互不知情的订阅者，靠 `correlation_id` 把碎片缝回一条链 = 分布式链路追踪，这天专讲。

## 一句话串联

> 今天的内容和我的 pricing engine / 之前学过的哪天，能怎么联系起来？

- 把 `policy-events` 当 event log + 钉住 `rate_table_version`，正是 pricing / 夜间重定价最需要的 reproducibility：多年后监管问"这张保单当时为什么是这个价"，答案是"把那条事件配上当时钉住的费率表版本**重放、确定性地算回来**"，而不是"翻日志大概是这个原因"。这条线串起 Day 14（幂等 consumer 把 offset + 副作用写进**同一事务**）和 Day 13（partition key = shard key 的同一套思想）。
