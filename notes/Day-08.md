# Day 08 · 关系数据库与 ACID

> 对应文档：`Day-08-relational-db-acid.html`
> 日期：2026-06-08
> 主题：事务（transaction，BEGIN…COMMIT/ROLLBACK 的全成或全退单元）、ACID 四字母（Atomicity 原子性靠回滚不留半成品 / Consistency 一致性守约束 / Isolation 隔离 / Durability 持久性 COMMIT 即落非易失存储）、四档隔离级别（Read Uncommitted / Committed / Repeatable Read / Serializable）、三种并发异常（dirty read 脏读 / non-repeatable read 不可重复读 / phantom read 幻读）、MVCC（多版本并发控制，读快照、读不加锁）、WAL（write-ahead log 预写日志，撑原子性与崩溃恢复）、数据库为何配当 source of truth（写路径别图快、durability 是 write-back 风险的反面）

## 今日思考题

> 文档结尾「带着这两个问题回来讨论」。先写下自己的初步想法，再来 Claude Code 聊；聊完我会在每条下面补点评（像 Day 01–07 那样）。

1. QuoteFlow 夜间给**百万**保单批量重定价，事务边界该画在「每张保单一个小事务」还是「整批一个大事务」？各有什么取舍（想想锁持有时间、失败回滚范围、日志大小）？
   
   - 我的想法：对于百万保单这种请求量很大的情况，事务边界应该画在每张保单（或是一小部分保单一起）而不是整批一个大事务。小事务可以让每个锁的持有时间相对较短，如果是整个大事务因为保单数量多导致完成一项任务的时间长，所以容易长时间持有锁，影响其他任务的处理。失败回滚范围同理，小事务方便进行小范围回滚，而整个大事务出问题要回滚整个百万保单，重启的成本过高。日志大小对小事务来说可以保持的较小，大事务则需要将很多内容放进日志里，甚至有可能撑爆日志。不过我在想大事务有没有什么好处，可能在不会fail的情况下，大事务整体处理请求，可能能省掉一些分散为小事务所要花费的成本，事务形态也相对更简单一些（i.e. 不需要分发小事务到不同机器处理之类的）？

   - **Claude 点评：** 主线全对，而且你结尾那句疑问正是题眼。补三点：① 大事务在 Postgres 里还有个隐形代价——它一直不 commit，就把 MVCC 的「最老活跃事务」horizon 钉住，autovacuum 没法回收死元组 → **表膨胀（table bloat）**，不只是「日志大」。② 回滚范围还有一层：百万保单有**夜间窗口**，大事务崩在第 99 万张 → 前面正确的全 ROLLBACK、从 0 重跑 → 很可能**撑爆窗口、违约 SLA**；小事务能**断点续跑**（前提：`last_processed` 标记 / 幂等键，Day 18）。**这里 atomicity 的「全有全无」反而是负担**——每张保单的重定价彼此独立，你根本不想要「一张失败把另外 99 万正确的也撤掉」。③ 你那句「大事务有没有好处」是满分直觉，但好处不是你猜的「省得分发」——是 **commit 开销**：每次 COMMIT = 一次 WAL 落盘 fsync。一百万个「每张一事务」= 一百万次 fsync = 慢到爆。这股力把你推回大事务。两股力一拉扯，**真正的答案就是你括号里写的 chunked batch**（每 1000 张一个事务）——摊薄 commit 开销 + 控住锁/回滚/日志/bloat。你不是在两极选，正解是中间那档，而你已经写出来了。

2. 同一个 QuoteFlow，为什么**报价（读）路径**和**签发/扣款（写）路径**可以选不同强度的隔离级别？你会怎么给这两条路径各定一档，理由是什么？
   
   - 我的想法：读的话只是从数据库（或者缓存）里提取数据，不涉及到对具体数据的修改，但是写的话涉及到对数据库数据的修改，一旦出问题就会导致source of truth被污染，因此写比读需要更高强度的隔离级别（Day 7讲的缓存应该同理）。对于读路径的话，我隔离级别会定在read committed这个level （Level 2），因为只要能避免dirty read的问题，i.e. 不要读到未更新的数据，就一般不会出太大问题。但是对于写路径的话，non-repeated read和phantom read就会是一个问题，比如两次读到不同值导致写入的结果不一致，所以要放到至少level 3以上甚至到level 4，避免因为这些问题导致的审计合规冲突。

   - **Claude 点评：** 方向对（读能弱、写要强），但藏着一个**精度修正**：隔离级别**不是按「读 vs 写」分类定的，是按「这个事务的逻辑会被哪种 anomaly 咬」定的**。反例戳破「写一定比读高」：财务月报**只读不写**，但要一致快照 → 得上 RR/Serializable；而 `UPDATE balance=balance-10` 这种写，配个行锁在 Read Committed 就够。两个具体修正：① 你给读路径定 Read Committed，但**报价路径恰恰可能要往上调**——报价要算一张费率表的很多行，读到一半版本变了会**混用新旧费率**（跨多行的 non-repeatable read）→ 不可复现。解法二选一：升 **Repeatable Read**（拿快照）或 Day 7 的**显式钉版本号**——这把今天和昨天缝上了。② 写路径「直接拉满 Serializable」常是错的：它触发 **serialization failure → 重试 → 吞吐塌**（billing 高吞吐扛不住）。老练做法是**点名要守的不变量、用最便宜工具守**：怕开两张同号保单 → `UNIQUE` 约束；怕重复扣款 → 幂等键 + `SELECT FOR UPDATE` 行锁；只有真有 write-skew 才上 Serializable。③ 小脚注：**Postgres 没有真正的 dirty read**（Read Uncommitted 等同 Read Committed），所以读路径「避开 dirty read」在 PG 上免费送，你真正权衡的是要不要快照。

## 提问 & 解答

> 讨论时冒出来的问题记这。每条：先记自己的问题，再记讨论后的结论（用自己能看懂的话）。

### Q1. Serializable 具体怎么实现？它怎么解决 phantom read？

**问：** Serializable 这一档是怎么做到的？尤其 phantom read（同一查询两次行集变化）怎么被挡住？

**答：** 先分清：**Serializable 是一个「保证」，不是一种机制**——它承诺并发事务跑完，结果**等价于「它们一个接一个串行跑」的某个顺序**。怎么兑现，有两套**完全不同**的实现，而 phantom 正是区分它们的试金石（难点：**你没法锁一行还不存在的数据**）。

- **锁派（2PL + predicate / next-key lock，悲观）**：传统两阶段锁实现（MySQL InnoDB 的 Serializable、SQL Server）。普通行锁锁不住「不存在的行」，于是升级成**锁一段范围 / 锁谓词**——你那条 `WHERE ...` 的 SELECT 不只锁命中行，**还锁住谓词覆盖的索引区间（gap）**，别人想往这段 INSERT 就**被阻塞**，幽灵无从出现。代价：写者干等、并发降、可能 deadlock。
- **SSI（Serializable Snapshot Isolation，乐观）**：PostgreSQL 9.1+，建在 MVCC 快照上、**全程不阻塞**。大家各自在快照上跑，背后用 `SIReadLock`（**追踪标记、不是会挡人的锁**）记下你做过的**谓词读**；别人往这个范围写 → 形成 **rw-dependency**，提交时若依赖**绕成危险的环** → **abort 一个**，报 `could not serialize access`，应用**重试**。代价：重试白干一遍。

一句话：**phantom 的破法——锁派去锁「行将要落进的那段范围」，SSI 派不锁、改成「事后发现你读过的范围被人塞了东西、就判你重来」。** 落到 QuoteFlow「一人最多一张某类保单」：**最便宜解其实不是 Serializable，是一条 `UNIQUE(applicant_id, policy_type)`**——第二条 INSERT 直接撞约束失败。Serializable 留给**没有单一约束能表达**的 write-skew（如「某再保合约下所有保单的风险敞口加起来不能超上限」：读聚合、写一行、约束跨多行）。

### Q2. MVCC 属于隔离级别的哪一档？

**问：** 文档像把 MVCC 当第 2 档（Read Committed），可它又有 snapshot 机制，像第 3 档（Repeatable Read）。到底哪一档？

**答：** **哪一档都不是——MVCC 不是「档位」，是底下那台「引擎」。** 它每次写都生成一个**新版本**的行（PG 里带 `xmin/xmax` 的新元组），读的时候按**你自己的快照**挑「对我可见的那个版本」，读不加锁、读写不互阻。**四档都在用这台引擎**；档位之间的差别**不在用不用快照，而在快照的「保质期」**：

- **Read Committed**（PG 默认）= **每条语句开始时重拍一张新快照** → 两次读之间别人 commit 了就照进来，这正是 **non-repeatable read** 为何在第 2 档发生（不是没用快照，是**每句换新**）。
- **Repeatable Read**（= **Snapshot Isolation**）= **整个事务开始拍一张、全程复用同一张** → 重复读永远一致。
- **Serializable** = 那张冻结快照 **+ 提交时查 rw 冲突环**（Q1 的 SSI）。

所以「**有快照**」不是第 3 档专属（第 2 档也用），**让它成第 3 档的是「快照被冻住、整个事务看同一张」**。比喻：MVCC 是台能随时拍照的相机；第 2 档每做一件事前重拍、第 3 档进门拍一张只看这张、第 4 档同一张加个裁判。**同一台相机三种用法 = 三档，相机本身不是「第几档」。** 脚注：PG 的 RR 比 SQL 标准更强、其实是 Snapshot Isolation，**连 phantom 都顺手挡**（标准 RR 不挡），但仍挡不住 write skew，那才轮到第 4 档。

### Q3. 两个事务互锁（deadlock）的具体例子？

**问：** 两个 transaction 互相锁住（deadlock）能给个具体例子吗？

**答：** 经典版本：两个夜间重定价 worker 都要更新 `billing` 里的 policy 100 和 200，但**遍历顺序相反**。T1 先 `UPDATE WHERE id=100`（锁住 100）、再要 200；T2 先 `UPDATE WHERE id=200`（锁住 200）、再要 100 → **T1 等 T2 放 200、T2 等 T1 放 100**，两条等待边首尾相接成**环**，谁都不可能先松手 = deadlock（不是慢，是逻辑上永远解不开）。数据库维护一张 **wait-for graph**（谁在等谁），发现环就**挑一个 victim 直接 ROLLBACK**，另一个继续；Postgres 是被阻塞事务先干等 `deadlock_timeout`（默认 **1 秒**）才去查图，报 `deadlock detected`，被砍的由**应用负责重试**。头号解法：**规定全局加锁顺序**（如永远按 `policy_id` 升序），环压根连不成（只排队、不死锁）；其余：事务尽量短（接思考题①小事务）、`SELECT ... FOR UPDATE ORDER BY id` 一把锁全、队列场景用 `SKIP LOCKED`。

**顺手分清两种「abort + retry」**：deadlock = **锁等待**成环（运行中、悲观锁派）；serialization failure = **rw 依赖**成环（提交时、乐观 SSI）——报错不同、机制不同，但**最后都靠你写的重试逻辑兜底**。

### Q4. SQL 的 COMMIT 和 git 的 commit 相通吗？

**问：** 事务里的 COMMIT 和 git 的 commit，有什么相通之处？还是只是英文 "commit" 这个词对两种情况都合适？

**答：** **不是巧合——两边故意借用同一个英文隐喻**（把一批东西**正式敲定成一个不可撤回的承诺**），且底下真有一副**共享骨架**：① 都有一个「暂定、还没生效」的**暂存区**（SQL 的 `BEGIN` 后未提交改动只对自己可见 / git 的 staging area）；② commit 是把**一批改动捆成一个不可分单元**一次落定（git 一个 commit 也是 all-or-nothing 的快照，你不会 checkout 到「半个 commit」= atomicity）；③ 落定后**持久、有了身份**。

但它们**在「各自要啃的硬骨头」上分岔**：**SQL-commit 本质是并发控制原语**（我的改动**何时**对别人可见 / 算数，难题在 isolation、锁、冲突）→ 所以**一锤定音**（没法 uncommit，要撤只能再发一个**补偿事务**）+ **立即全局可见**；**git-commit 本质是版本 / 历史原语**（沿时间留一串快照血缘，基本单写者）→ 所以**可改写**（amend / rebase / reset；commit **对象**用 hash 寻址不可变，但**历史指针可变**）+ **仅本地**（要让别人看见得单独 `push`）。历史彩蛋：早年 CVS/SVN 的 `commit` 本就是「提交到中央服务器、所有人立刻看到」——**几乎等于 SQL commit**；是 **git 故意把 commit 降权成本地动作**、又另发明了 `push`。所以 `svn commit ≈ git commit + git push`。

## 我的理解（用自己的话复述今天的核心）

> 不看文档，凭记忆写 2–4 句。能写清楚 = 真的懂了。

- 数据库依靠transaction这个功能把相关联的业务语句合在一起进行处理，并通过atomicity来确保一个transaction里的语句要么全部成功，要么任何失败导致回滚重试，这是数据库可以作为source of truth的基础。除了atomicity之外，还有consistency（数据库更改前后规则保持一致），Isolation（并发任务处理不互相影响），和Durability（commit之后立刻数据写进非易失存储，比如硬盘/SSD来保证数据不丢失）。Isolation的实现需要面对并发处理数据可能出现的三个问题：dirty read, non-repeatable read, 和phantom read，它们可以通过依次增强的隔离等级：read committed, repeatable read, 和serialization来规避。

- **Claude 点评：** 覆盖很全。一个精度修正：「atomicity 确保要么全部成功，要么**回滚重试**」——**retry 不属于 atomicity**。原子性只保证 all-or-nothing（失败就 ROLLBACK 干净），**要不要重试是应用层自己的决定**，数据库不替你重试。这个区分在思考题①里就很要紧（断点续跑 = 应用拿幂等键自己重试，不是 DB 给的）。另：「serialization」正名叫 **Serializable**。其余（C 守约束 / I 并发不互扰 / D commit 即落盘）都准。 

## 待深挖 / 存疑

> 还没想透、或想之后回头补的点。

- [ ] WAL 说「改动先写日志再改数据」——日志本身怎么保证落盘、崩溃恢复时怎么 redo/undo？跟数据库**索引/存储引擎**的关系 —— Day 9
- [ ] 隔离级别挡并发异常靠 MVCC + 锁；更深的**事务与隔离级别、分布式事务（saga）**怎么跨服务做 —— Day 14
- [ ] 单机 ACID 的 durability 靠 WAL；多副本下「COMMIT 算不算数」要看**复制与一致性（leader-follower、quorum、CAP/PACELC）** —— Day 12
- [ ] 思考题①的大事务 vs 小事务，本质是**批处理**的事务边界设计 —— Day 17
- [ ] Consistency 的「C」其实和 CAP 里的 C 不是一回事，待并排厘清 —— Day 12

## 一句话串联

> 今天的内容和我的 pricing engine / 之前学过的哪天，能怎么联系起来？

- 今天的内容和第一周的内容都有一定联系，主要讲解了数据库通过什么机制来成为系统中储存数据的source of truth，以及如何通过ACID设定边界来满足non-functional requirements里提到的auditability，reproducibility的需求。

- **Claude 点评：** 接住了 NFR 那条线。再点明今天和昨天**最硬的一根**：**Durability 就是 Day 7「write-back 缓存丢数据」的正反面**——write-back 先写易失内存、缓存一挂没刷的就凭空消失；durability 保证 COMMIT 一返回数据已落非易失存储。所以「写路径别图快、缓存只在读侧帮忙」不是口号，**它的技术理由就叫 durability**。
