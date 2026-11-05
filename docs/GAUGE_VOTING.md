# ve、Gauge 权重与原生预算分配规范

对应 [README](../README.md#weights)。本规范只处理原生 GOV 的池间分配；个人 LP 余额与外部奖励由 [奖励核算规范](REWARD_ACCOUNTING.md) 处理。

## 目录

- [1. 时间与职责](#time)
- [2. ve 与权重字段](#fields)
- [3. 提交和修改投票](#voting)
- [4. 周检查点与历史权重](#checkpoints)
- [5. 原生预算公式](#emissions)
- [6. 时间示例](#examples)
- [7. 严格验证](#verification)

<a id="time"></a>

## 1. 时间与职责

```text
W = 604800 秒
week(t) = floor(t / W) × W
effectiveWeek(t) = week(t) + W
```

周期为 [week, week+W)。统一 Unix 周起点，不按每个用户的首次质押时间划周。V1 一周一次确定预算权重，周内不按最新 ve 连续重算相对权重。

- VotingEscrow：回答锁仓斜率、到期日以及当前/历史 ve。
- GaugeController：保存个人支持比例，推导各 Gauge 的周权重。
- EmissionDistributor：读取历史权重，算过去区间的 GOV 并从预存预算支付。
- LiquidityGauge：使用同一套 LP 奖励积分分给用户。

Gauge 权重投票是个人奖励方向配置，不需要提案多数通过，也不进入 Timelock。新增 Gauge 的奖励资格和未来停奖状态属于协议治理，须经过提案与延迟执行。

<a id="fields"></a>

## 2. ve 与权重字段

### 2.1 ve 的统一精度

设 Q = 10^18 为锁仓斜率精度，M 为最长锁期秒数，L 为 GOV 最小单位：

```text
lockSlope = floor(L × Q / M)
veScaled(user, t) = lockSlope × max(lockEnd - t, 0)
ve(user, t) = floor(veScaled(user, t) / Q)
```

这是整数实现的定义；经济近似为 L × 剩余时间 / M。全局 ve 使用聚合的高精度点再除 Q，不要求逐用户取整后的和与全局取整结果逐单位相等。

锁仓结束按整周对齐，个人与全局历史点保存 bias、slope、timestamp，记录未来到期时的 slope 变化。历史快照不依赖遍历全部账户。

### 2.2 Controller 字段

| 字段 | 含义 |
|---|---|
| gaugeInfo[g].registered | 是否经过协议登记；历史登记不可删除 |
| gaugeInfo[g].eligibleFrom | 首个可分原生预算的周起点 |
| nativeEnabledAt[g][w] | 对应周是否释放原生奖励，使用可追溯的历史状态/变更点 |
| userVote[u][g].bps | 本人分给该池的比例，整数 0—10000 |
| userVote[u][g].slope | 最近一次投票记录的加权锁仓斜率 |
| userVote[u][g].end | 本次投票依据的锁仓到期时间 |
| userAllocatedBps[u] | 所有尚未撤销配置的 BPS 总量，包括需清理的过期配置 |
| lastVoteTime[u][g] | 检查该池十天改票间隔 |
| gaugePoint[g][w].bias | 在 w 周起点的池子绝对票权，高精度 ve 单位 |
| gaugePoint[g][w].slope | 这一点之后票权下降的速度 |
| gaugeSlopeDelta[g][w] | 到 w 时应改变的斜率，采用有符号值 |
| totalPoint[w] | 所有已取得资格 Gauge 的聚合 bias/slope |
| totalSlopeDelta[w] | 全局的预定斜率变化 |
| gaugeCursor[g] / totalCursor | 各池/全局已推进到的周 |
| pendingPoint / finalPoint | 未来周可变的待生效数据，与当前/过去周确定值的逻辑区别 |

权重统一保留高精度整数，用于比值时不用先转成完整 GOV 或用户界面百分数。

<a id="voting"></a>

## 3. 提交和修改投票

### 3.1 前置检查

voteForGauge(g, bps) 必须检查：

1. Gauge 已登记，且待生效周不早于 eligibleFrom。
2. bps 介于 0 与 10000。
3. 替换旧配置后，全用户分配总 BPS 不超过 10000。
4. 非首次修改距离本池上次修改至少十天，撤票也遵守。
5. bps > 0 时，存在有效锁仓且 lockEnd > effectiveWeek。
6. bps = 0 时允许清理过期投票，不要求新建锁仓。
7. 先完成涉及周的 Controller 检查点，不能在缺历史的状态直接覆盖旧记录。

允许部分不投；未使用 BPS 是未使用的影响力，不是一份替用户预留的 GOV。

### 3.2 新投票贡献

设 e 为待生效周，p 为本次 bps：

```text
newVoteSlope = floor(currentLockSlope × p / 10000)
newVoteBiasAtE = newVoteSlope × max(currentLockEnd - e, 0)
```

旧记录在 e 的剩余贡献：

```text
oldActiveSlope = oldVote.end > e ? oldVote.slope : 0
oldVoteBiasAtE = oldActiveSlope × max(oldVote.end - e, 0)
```

更新 e 的池点及总点：

```text
bias += newVoteBiasAtE - oldVoteBiasAtE
slope += newVoteSlope - oldActiveSlope
```

若旧到期时间还在 e 之后，撤销它此前安排的负斜率变化；再为新投票在 newEnd 安排 -newVoteSlope。全局到期计划同步更新，防止某份旧票到期时被扣两次。

最后更新 userAllocatedBps、个人 vote 结构、lastVoteTime，并发出包含 effectiveWeek 的事件。整笔操作原子执行。

### 3.3 锁仓变化及持续投票

- 加币/延长期限只改变 VotingEscrow，不自动更新 Controller 已存投票。
- 用户重投后，新贡献从下一周起使用；当前/历史周不动。
- 未重投时沿用原记录的 slope 和 end，按原承诺衰减。
- 无需每周重新投票，但分配比例不保证永远固定；不同到期日的票会以不同速度下降。
- 过期票的绝对权重归零。旧 BPS 配置需显式撤销或替换后释放，不继续贡献有效票。
- 投票不需要持有对应池 LP；质押 LP 也不会自动产生治理票。

<a id="checkpoints"></a>

## 4. 周检查点与历史权重

由上一周点推进下一周：

```text
nextBias = max(currentBias - currentSlope × W, 0)
nextSlope = currentSlope + slopeDelta[nextWeek]
```

到期先使 bias 衰减到该边界，再处理该边界的斜率变化。合法状态不应产生负 slope；实现必须校验有符号运算，不用随意截零掩盖错误。

各 Gauge 与全局点通过同样的记录增减维护。推进按周循环，不遍历所有投票用户。每次默认最多 32 个周边界；未完成时保存游标，由后续公开调用继续。

### 4.1 历史冻结与查询

- 未来 e 周点仍可被本周的合法改票更新。
- w 开始后，不再接受生效时间为 w 的新投票或资格修改。
- w 的最终权重可以延迟物化到存储，但必须由已有记录确定，不能依赖物化时的用户当前余额。
- relativeWeight(g, w) 的界面值可表示为 floor(gaugeBias × Q / totalBias)。
- 资金分配直接用完整 bias 比值，不先截断界面百分数。
- 若查询所需检查点尚未完成，明确返回需推进，不静默使用旧周数据。

### 4.2 分母、停奖与零状态

totalBias(w) 包含当周已取得资格 Gauge 的投票权重。停奖 Gauge 的权重保留在分母，但原生额度不释放，后续用户改投才改变其他池占比。

空池不会自动把原生预算送给别池。S = 0 的那段原生额度由预算库记为 skippedNative。

totalBias = 0 时全部原生预算为零；不是平均分。资格新增、停奖、恢复都只能被治理安排到未来周，历史可领取量不受影响。

<a id="emissions"></a>

## 5. 原生预算公式

### 5.1 储备和记录字段

GOV 固定发行后，208,000 GOV 预存在 EmissionDistributor；单周 B = 1,000 GOV，计划 208 周。

| 字段 | 含义 |
|---|---|
| emissionStart / emissionEnd | 固定周对齐计划区间 |
| weeklyBudget | 单周上限 B，以 GOV 最小单位计 |
| activated | 储备核对、绑定和初始配置完成后开启 |
| nativeCursor[g] | 本池已经处理到的时间，不是用户领取进度 |
| nativeReleased[g] / totalReleased | 本池/系统已转到 Gauge 的 GOV |
| skippedNative[g] | 本池已过去但因空池等未转出的额度 |
| storedBudget / computedBudget(g,w) | 某池某周由历史权重确定的预算 |
| initialRewardReserve | 初始奖励储备，用于预算与余额核对 |

预算库不保管用户 LP，不维护个人 rewards。它只接受已登记 Gauge 的受限拉取，用户直接调用不能指定金额。

### 5.2 单池周预算

设 W_g 为本周该池 bias，T 为本周总 bias：

```text
A(g,w) = floor(B(w) × W_g(w) / T(w))
```

若超出发行区间、资格未生效、该周停奖或 T = 0，则 A = 0。

B(w) 在计划内为固定周预算，否则为零。采用安全全精度 mulDiv，避免先截断相对权重再乘 B 的二次舍入。

各池 A 的和不超过 B。取整剩余留在储备中，不作为某个池的隐性收入。

### 5.3 时间累计与精确速率

对某一固定周 [w, w+W)：

```text
V(g,w,t) = floor(A(g,w) × clamp(t-w, 0, W) / W)
delta(g,w,a,b) = V(g,w,b) - V(g,w,a)
```

该周的速率是精确分数 A/W。累计函数最终在周末等于 A，同一周多次补账的差额可望远镜相加，不因调用次数改变总释放。

跨周时先按边界切段，再求和：

```text
deltaNative(g,a,b) = Σ delta(g,w,max(a,w),min(b,w+W))
```

资金操作不能把 a 到 b 统一套用 b 时刻的最新权重。

### 5.4 与 Gauge 积分衔接

1. Gauge 在任何 LP 余额变化前，基于旧 S 请求处理到 toTime。
2. Distributor 逐历史周核对预算、资格、启停状态与游标。
3. 非空池收到已过去时段的 GOV，进入统一积分内核。
4. 空池游标照常推进，未释放部分保留并记录。
5. Gauge 再结算用户，最后才改 LP 余额。

领取晚不等于发奖从领取那天才开始。公共检查点必须同时推进 Gauge 的积分与预算库游标，不能只把钱先转进柜台。

未完成历史时可以公开分批推进；stake/withdraw/claim 等资金入口在未完整推进到 now 时明确回退，不能漏算几周后继续改余额。

<a id="examples"></a>

## 6. 时间示例

### 6.1 改票何时生效

假设所有初始锁仓条件均有效：

| 时间 | 操作或状态 |
|---|---|
| w0 + 1 天 | 提交 A/B = 40/60，待 w1 生效 |
| w1 起点 | 开始使用该周最终 40/60 权重 |
| w1 + 5 天 | 距初投十一天，可以改为 70/30，待 w2 生效 |
| w1 剩余时间 | 仍按 40/60 |
| w2 起点 | 开始按 70/30 |

以上比例示例假设其他用户贡献与锁期变化不会改变对应周的最终占比。个人配置 40/60 不一定等于全系统最终 40/60，最终结果要汇总所有有效投票。

### 6.2 延迟补账

假设 A 在某周预算 400 GOV，下一周预算 700 GOV：

- 第一周开始至第 3.5 天：累计 200。
- 第一周第 3.5 天至周末：新增 200。
- 下一周开始至第 1.75 天：新增 175。
- 从第一周第 3.5 天一次补到下一周第 1.75 天：合计 375。

不能将这整段按新周 700 的速率计算，也不能收到 375 后又把它重新安排未来七天，否则会把已经发生的奖励推迟并重复定时。

### 6.3 没有重投，权重也可能变化

使用归一化示意单位：某周 A 的 bias=100、每周减少20；B 的 bias=100、每周减少10。初周两者各 50%；下一周为 80/90，约 47.06%/52.94%；A 对应票到期后可降为零。

因此“支持比例配置持续保留”与“绝对票权随时间下降”需要同时实现。这些示意数值是权重点，不是用户存款或奖励币数量。

<a id="verification"></a>

## 7. 严格验证

必须为下列项目提供单元、集成及独立参考模型验证：

- 多用户、多池 BPS 合计，超过 10000 必须回退。
- 个人权重来自锁仓量、期限、比例，不来自 LP 质押量。
- slope/bias 更新与取消旧到期记录正确，不重复减票。
- 过去周不因改票、加锁、新登记或停奖安排被改写。
- 当前周与未来待生效周分开，周边界前/时/后一秒有明确结果。
- 改票十天边界，首次投票、零票撤销、过期清理路径可运行。
- 加锁后重投与不重投的差异符合规则。
- 多个锁期到期后，全局点与测试逐用户枚举结果在规定整数语义下一致。
- 总票为零、全部票过期、空池、停奖和恢复不发生除零或隐性补发。
- 完整 bias 比值计算预算，各池预算和不超过 B。
- 同一周预算累计 V 在终点等于 A，任意拆段增量之和与整段一致。
- 原生释放不会二次走外部七天流；历史款项只计入一次。
- 多年不维护后，分批推进等于持续维护。
- 外部奖励不读取此处权重，协议提案不会替用户自动提交 Gauge 投票。

资金上界：totalReleased 不超过已过去时间允许的全局预算，也不超过初始奖励储备。测试应明确未分配、跳过、未到期和取整剩余的去向。

这些是待实现合约的验收要求；公式示例核对不等同于合约实现已经通过测试。

