# Day 10 · 数据建模与范式 / 反范式

> 对应文档:`Day-10-data-modeling.html`
> 日期:2026-06-10
> 主题:entity / relationship(实体=有主键的业务对象,关系决定外键放哪)、normalization 范式化(每份事实只存一处,防 update / insert / delete anomaly)、1NF/2NF/3NF(the key, the whole key, and nothing but the key;底层=functional dependency,2NF 禁 partial、3NF 禁 transitive)、denormalization 反范式(有意识引入冗余:冗余列 / 汇总表 / 快照,写时维护换读时速度)、snapshot 快照 vs version reference 版本引用(源头可变→复制值;源头已 immutable 版本化→存指针)、effective dating(valid_from / valid_to,更新=关旧开新,= 数仓 SCD Type 2 / dbt snapshot)、append-only(只增不改,无更新异常,天然适合审计与重放;配唯一键幂等)、summary table 汇总表(预算好的聚合=长成表的缓存,derived 可重算)

## 今日思考题

> 文档结尾「带着这两个问题回来讨论」。先写下自己的初步想法,再来 Claude Code 聊;聊完我会在每条下面补点评(像 Day 01–09 那样)。

1. Quote 表里,**哪些字段该快照、哪些该引用**?试着给一条判断规则(提示:从「这个字段的事实属于哪个时点」想起),再用它检验:保额、客户邮箱、费率表版本、核保规则结果。
   
   - 我的想法: 判断规则应该基于哪些信息需要的是报价时的样子，哪些需要的是最新的样子，也就是提示里的“这个字段的事实属于哪个时点”。首先，为了要确保报价的可追溯性和reproducibility可复现，被用于报价模型的信息全部都需要快照，比如保额，年龄，是否抽烟，职业等等；费率表因为也属于报价需要的信息，所以也要快照。核保规则结果也是需要快照的，不能因为未来核保规则变化导致当时的核保结果出现变化。但是不用于报价的信息就应该选择引用，确保拿到的是最新的信息，比如客户的名字，邮箱，住址等身份信息。

   - **Claude 点评：** 判断规则方向全对（「事实属于哪个时点」），保额 / 邮箱 / 核保结果三个判对。⚠ 要修一处：**费率表你判了「快照」，正解是「版本引用」**——文档核心句是 Quote 行里存「输入快照 + **版本引用**」，这是两种不同机制：**快照**用于「源头可变、且没有版本管理」的事实（客户申报的年龄/吸烟状态住 Customer 表、随时会被 UPDATE → 必须把当时的值**复制**进 `applicant_snapshot`）；**版本引用**用于「源头自己已经 immutable 版本化」的事实（`rate_table_version` 发布即冻结 → 只存 `version_id` 指针，引用不可变的东西等价于快照），省下复制整张费率表，还白送反向查询「v2026-06 影响了哪些报价」（费率出错时做 impact analysis 的合规刚需）。精炼成两步规则：**① 源头以后变了，这行该不该跟着变？** 该跟 → FK 引用；不该跟 → 钉死时点。**② 钉死时点的：源头自己 immutable 版本化了吗？** 是 → 版本引用（费率表、核保规则**版本**）；否 → 复制快照（申报信息；核保**结果**是本次申请第一手产出的事实、无处可引，所以也是快照）。再补一个微妙点：「快照 or 引用」**不是字段的固有属性，是每张表按自己要回答的问题定的**——邮箱在 Quote 表该引用，但「保单文件当时发去了哪个邮箱」的发送记录里就该快照当时邮箱。同一个字段，两张表两个答案。

2. 夜间批量重算要给每份 in-force 保单写一条重算结果。这张结果表你会怎么建——**可变还是 append-only**?如果同一保单同一晚跑了两次(重试),建模上怎么不留下两条互相矛盾的记录?
   
   - 我的想法: 我认为应该建成可变，如果建成append-only的话，假设由于某些原因导致一些保单需要回滚重算（请求重试），那么重试前和重试后都会有记录，但对于夜间重算来说我应该只想保留一份正确的重算结果记录，所以应该建成可变才对。建模上怎么不留下两条互相矛盾的记录，具体我不太清楚怎么实现，但我认为同一保单同一晚跑了两次这种情况，假设两次都成功，那么就以更新的那一次重算结果作为最后的结果。我在想是不是可以用一个idempotency key去判断这两次重算针对的是同一保单，如果是同一保单的重算的话，重试的那一次的结果就去覆盖掉之前一次的结果（类似于update）。

   - **Claude 点评：** 结论要反过来：**append-only**（讨论中你也自己纠正了——你把题读成「重试场景怎么处理」，题问的是表本身的形态）。但答案里的 **idempotency key 直觉是满分**，它的数据库实现就是唯一约束：`UNIQUE(policy_id, run_date)` + `INSERT ... ON CONFLICT DO NOTHING`——重试的 INSERT 撞上约束直接变 **no-op，第二行根本写不进去**（不是「写两行再去重」，是把 duplication 结构性地排除）。表语义 = 每个 (保单, 重算日) 恰好一行：**跨晚 append-only**（昨晚的结果是「计算产出的事实」，要喂 reserving / 再保上报，UPDATE 它 = 销毁证据，和费率表「关旧开新」同一个道理）、**晚内幂等**（同键重试收敛成一行）。两个 nuance：① 重算应是 **deterministic** 的（同输入快照 + 同版本 = 同结果，reproducibility 公式），同一晚两次结果若不同 = 输入/版本在中途漂移——这恰是审计要抓的异常，所以 `DO NOTHING` + 不一致告警 **优于** `DO UPDATE` 盲覆盖（「后来者赢」会静默掩盖问题）；② 若审计连「跑了几次、为什么重试」都要 → **attempt-level** 设计：每次 attempt 一行 + 显式 `run_id / attempt_no / status` + 「最新成功」视图。两行 attempt **不是 duplication**——范式意义的 duplication 是「同一份事实存多处」，而「attempt 1 在 02:13 跑过」「attempt 2 在 02:47 跑过」是两个不同事件、两份事实（和「快照不违反范式」同一把尺子：长得一样 ≠ 同一份事实）。真正的反模式只有中间态：想要一行一结果、却不建约束，靠 timestamp 默默多行，把去重逻辑推给每个读方。一句话选型：**「跑了几次」本身是不是要记录的事实？** 不是 → result-level + 唯一键；是 → attempt-level + 显式维度。

## 提问 & 解答

> 讨论时冒出来的问题记这。每条:先记自己的问题,再记讨论后的结论(用自己能看懂的话)。

### Q1. 范式三级怎么理解？「the key, the whole key, and nothing but the key」是什么意思？

> 详见补充图解 `notes/Day-10-normal-forms.html`（每级一组伪数据「违例表 → 修复表」对照）。

**问：** 大学学过三级范式、现在印象模糊了。展开讲讲 1NF / 2NF / 3NF，怎么理解那句口诀？

**答：** 口诀的原型是法庭宣誓词 "the truth, the whole truth, and nothing but the truth, so help me God"，改成对**非主键列**的宣誓、结尾双关 so help me **Codd**（E. F. Codd，关系模型之父）。底层只有一个概念：**functional dependency（函数依赖）**——X → Y 即「知道 X 就唯一确定 Y」，三级范式管的全是「非键列到底函数依赖于谁」。**1NF**（前提，口诀还没出场）：每格恰好一个原子值——塞 `"TERM_LIFE;TPD;TRAUMA"` 逗号串只能 `LIKE '%TPD%'` 查（索引报废）、没法给单个险种挂保额/外键，拆成子表。**2NF**（the **whole** key）：禁 **partial dependency**——非键列只依赖复合主键的一半（`quote_coverage` 里塞 `quote_created_at`，跟「哪个险种」无关、被抄 N 遍）；判别：**遮住一半主键，这列还能确定吗？**只在复合主键表有戏——单列 surrogate key 让 2NF 自动成立。**3NF**（**nothing but** the key）：禁 **transitive dependency**——绕道依赖另一个非键列（`quote_id → customer_id → customer_email`，email 描述的是客户不是这次报价）；判别：**这列描述的到底是主键，还是表里另一个非键列？**实务上日常翻车 95% 在 3NF。三种违规病征统一：同一份事实被抄多遍 → update / insert / delete anomaly 三兄弟上门。最后一个分界：`applicant_snapshot` 长得像违反 3NF 但不是——它存「报价那一刻的申报状态」，这份事实**属于 quote_id 本身**；完整版判别问题是「**这列的事实属于谁、属于哪个时点？**」

### Q2. SCD（slowly changing dimension）是什么？和今天的 effective dating 什么关系？

**问：** 学 dbt 的时候听过 SCD，记忆模糊了，展开讲讲。

**答：** SCD 是 Kimball 数据仓库方法论的词。仓库分 **fact table**（业务事件度量，一行一事件、只进不改）和 **dimension table**（描述性属性：客户/产品/渠道，fact JOIN dimension 来切片）；"slowly changing" = 维度属性会变、但相对事实流入的速度变得**慢**。源系统改了属性，仓库怎么办按 Type 编号：**Type 0** 永不更新；**Type 1** 原地覆盖（无历史，全世界只剩「现在」）；**Type 2** 关旧行插新行（`valid_from / valid_to / is_current`，完整历史）★；Type 3 加 `previous_xxx` 列（只回退一步，少用）。**Type 2 = 今天的 effective dating，一字不差**——每个版本还领自己的 surrogate key。dbt 的 **snapshot** 就是机器替你做的 Type 2：每次 `dbt snapshot` 对比源表当前态 vs 快照表 current 行，发现变化就关旧插新（`dbt_valid_from / dbt_valid_to` 自动列）。**关键差别**：OLTP 侧的 effective dating 是应用在**事务里**写版本、每次变更当场捕获 = **权威记录**（监管 as-at 复现靠它）；仓库 snapshot 是**定期跑批重建**的历史——粒度只到快照频率（一天跑一次、一天内改两次就丢中间态）、开始 snapshot 之前的历史永远补不回，只是分析便利品。fact 表用 Type 2 维度有两个姿势，今天全见过：fact 行直接存版本的 surrogate key（load 时钉死 = **版本引用**）；或 point-in-time join（`ON t >= valid_from AND (valid_to IS NULL OR t < valid_to)` = 文档的**时间旅行查询**，SQL 同款）。Type 1/2 还能按列混用——rating factor 列走 Type 2、无关紧要列走 Type 1（dbt `strategy='check'` + `check_cols`）。

### Q3. 汇总表是什么、长什么样？

**问：** 反范式三形态里的「汇总表」具体指什么？

**答：** **Summary / aggregate table** = 把高频聚合查询的结果**预先算好、存成一张真的表**——冗余列冗余的是「别人家的字段」，汇总表冗余的是「计算结果」。例：运营 dashboard 要「每日新业务概况」，直接对亿级 quote 表跑 `GROUP BY` 是低选择性大扫描（Day 9）、每次打开都重算；建 `daily_new_business_summary(summary_date, product, quotes_count, apps_count, policies_issued, annualised_premium)`，**一行 = 一个 (日期, 产品) 粒度的聚合**，夜批填好，dashboard 查一个月 = 读 60 行、毫秒级、与明细表多大无关。每日 in-force 汇总（按产品聚合保单数/总保额/年化保费）同款——正是 reserving / 再保上报的口径。两个设计决策：**① 粒度（grain）**——对齐 dashboard 实际要切的维度；粗了答不了细问题，细了行数逼近明细。**② 谁维护**（反范式的标准追问）——夜批刷新（简单好对账、白天数据滞后）/ 写时增量 `UPDATE +1`（实时，但写放大 + 当天所有写抢**同一行**的锁 = Day 8 lock contention）/ **materialized view**（数据库代管，`REFRESH` 一句话，本质还是批刷）。**与快照的关键区别**（问一句「丢了怎么办」）：汇总表是 **derived data**——真相永远在明细表，随时可完整重算，本质是「**长成表的缓存**」（Day 7 的 precompute，失效策略 = 刷新策略），允许先粗糙后修正；快照**不可重建**，丢了就是审计事故，必须事务内一次写对。dbt 的 mart / aggregate model 就是汇总表在分析侧的化身。

## 我的理解(用自己的话复述今天的核心)

> 不看文档,凭记忆写 2–4 句。能写清楚 = 真的懂了。

- 对数据进行建模时，要先根据业务思考应该有哪些Entity（有身份，i.e.主键的数据对象）和它们之间的Relationship。一般情况下我们要做Normalisation范式化来解决3个anomaly，到3NF就是一般业务需要的最高level了，但是根据具体的业务考量我们有时候也要进行Denormalisation反范式，比如通过牺牲写方面来减少join次数，从而增加读的速度，减少latency（但不要靠猜，要靠EXPLAIN ANALYZE去看是真的需要加冗余列，汇总表，还是能通过增加索引来解决），或是依据审计合规要求take snapshot存进数据表里，满足reproducibility。再就是audit_log要只增不改，保证audit的完整记录。

- **Claude 点评：** 覆盖很全（entity/relationship 先行、3NF 够用、**EXPLAIN ANALYZE 先于反范式**「不要靠猜」——把 Day 9 的工具接过来了，这条最值钱、快照、append-only）。补两条进复习卡：① **快照不违反范式**——Customer 存「现在的状态」、Quote 存「报价那刻我看到的状态」，时点不同 = 两份不同的事实、各自仍只有一个家；范式禁的是「同一份事实存多处」，不是「长得像的数据出现两次」。② 文档那句判断力值得原样背下：把系统里「**不可变的事实**」（报价、审计、版本、重算结果）和「**可变的状态**」（客户资料、保单状态）**分开建模**——前者 append-only 随便反范式，后者老老实实 3NF。

## 待深挖 / 存疑

> 还没想透、或想之后回头补的点。

- [ ] Day 11(NoSQL 家族):今天默认「数据放关系库」——什么时候这个默认该被挑战?QuoteFlow 里哪些数据其实更适合 KV / 文档 / 宽列?
- [ ] Day 13(分区与分片):append-only 的重算结果表 / audit_log 长到几亿行后,按时间分区怎么接住它(Day 09 点评里也埋了这条)?
- [ ] Day 18(幂等、重试、死信队列):思考题②的「同一保单同一晚跑两次」其实就是幂等问题——`UNIQUE + ON CONFLICT` 是数据库侧最便宜的实现,到时对照消息队列侧的 idempotency key + dedupe。
- [ ] Day 17(批处理与数据管道):汇总表/materialized view 的刷新调度、dbt 跑批,和 OLTP 版本表怎么喂给仓库(ETL/ELT)。

## 一句话串联

> 今天的内容和我的 pricing engine / 之前学过的哪天,能怎么联系起来?

- 今天内容我觉得主要是从数据库数据建模方面介绍了能满足Day 1一些non-functional requirements的设计。比如说以latency作为考量，思考何时应该normalise，何时应该denormalise（Day 9也提到的join多会增加latency的情况）；以reproducibility作为考量，思考什么数据需要用快照和只增不改的储存方式。

- **Claude 点评：** 接住 Day 1（NFR 取舍）+ Day 9（EXPLAIN 先行、JOIN 成本）两条线，准。再挂两个钩子：① 汇总表本质是「长成表的缓存」→ 直通 **Day 7**（precompute、失效 = 刷新策略）；② 你在分析侧早就天天用今天的内容——**dbt snapshot / SCD Type 2 就是 effective dating 的另一副面孔**，估值用「周年日当天有效的那套假设」就是 point-in-time join。OLTP、仓库、精算三个地方，同一个思想：给事实加生效期，更新永远关旧开新。
