# LP 质押与多 Token 奖励核算规范

本规范对应 [README](../README.md#staking)。它定义待实现的整数算法；原生 GOV 与外部奖励使用同一份 LP 余额、同一个积分内核及同一种领取流程，仅“本次新增奖励从哪里计算”不同。

## 目录

- [1. 单位与核心字段](#fields)
- [2. 两种奖励的新增量](#sources)
- [3. 统一整数积分内核](#kernel)
- [4. 操作顺序](#operations)
- [5. 外部奖励注资与续期](#funding)
- [6. 数值演算](#example)
- [7. 精度、守恒与严格验证](#verification)

<a id="fields"></a>

## 1. 单位与核心字段

### 1.1 单位约定

所有奖励使用对应 Token 自身的最小单位，不先换成十八位或价格单位。不同 Token 的积分和负债互不相加。

- LP 使用十八位小数，奖励可有不同小数位。
- 积分精度 P = 10^27。
- S = totalStaked，b_u = balanceOf[user]，二者均为 LP 最小单位。
- V1 约束 totalStaked < P，使每次封存的全局余数小于一个奖励最小单位。
- 每 Token 累计实际注资不超过 uint128 上界；外部奖励登记数最多八种，加 GOV 共九种。
- 所有下述数学式使用无限精度整数表述；实现使用安全 mulDiv 与 mulmod，不能直接假定中间乘积不会溢出。

### 1.2 质押状态

| 字段 | 作用 |
|---|---|
| lpToken | Gauge 固定托管的 LP |
| totalStaked | 账本上全部用户质押量 S，不读取合约实际 LP 余额替代 |
| balanceOf[user] | 该用户质押量 b_u |
| checkpointTime | 全部奖励已经处理到的共同时间 |
| paused | 只限制新增质押 |
| rewardTokens[] | GOV 及已登记外部 Token；添加后不删除 |
| registeredReward[token] | 防止重复登记、检查领取及注资入口 |

### 1.3 每 Token 的 RewardState

| 字段 | 单位与含义 |
|---|---|
| rewardPerTokenStored | 当前全局积分 I；每单位 LP 对应的累计奖励乘 P |
| lastUpdateTime | 本 Token 已结算到的时间戳 |
| globalRemainder | 本轮除以 S 后的分子余数，单位为奖励最小单位 × P |
| sealedDustScaled | 质押人群变化前封存的旧全局余数累计，单位同上 |
| totalFunded | 实际转入 Gauge 并被正确入账的该奖励数量 |
| totalAllocated | 经过有效质押供给分配到积分内核的奖励数量 |
| totalPaid | 已实际支付给用户的数量 |
| idleUnallocated | 仅外部奖励：S 为零时已经过释放期但未分给用户的预算 |
| distributor | 仅外部奖励：允许注资的地址，由协议治理配置 |
| periodStart / periodFinish | 当前外部奖励流的起止时间 |
| streamBudget | 当前外部奖励流的总预算，包含上次尚未处理计划及明确重排的储备 |
| vestedAccounted | 当前流已经处理过的累计释放量，不是已经领取量 |
| rewardRate | 查询概念：streamBudget / (periodFinish-periodStart)，返回分子/分母或明确精度；不用截断后的整数速率驱动账本 |

GOV 的时间释放状态由 EmissionDistributor 保存。Gauge 对 GOV 的 totalFunded 只记录预算库实际转来的量，不把整个系统储备提前算成本池资产。

### 1.4 每 user × token 的 UserRewardState

| 字段 | 单位与含义 |
|---|---|
| userRewardPerTokenPaid | 该用户已结算到的全局积分 I_u |
| rewards | 已结算、尚未领取的整最小单位奖励 |
| userRemainder | 用户除以 P 后尚不能整付的分子余数，严格小于 P |
| paid | 该用户该 Token 已实际领取的数量，供核对与事件使用 |

三个状态不能混淆：全局积分更新、用户应得结算、实际 Token 支付是三个不同动作。

<a id="sources"></a>

## 2. 两种奖励的新增量

### 2.1 原生 GOV

EmissionDistributor 根据 [历史周预算](GAUGE_VOTING.md#emissions) 计算区间 [nativeCursor[g], toTime) 的预算增量。

- 调用者必须是已登记 Gauge，且 toTime 不得晚于当前区块时间。
- Distributor 从 Gauge 读取本次操作前的 totalStaked，不接受调用者上报的任意奖励金额。
- S > 0：推进游标并转入该区间应分的 GOV，Gauge 检查实际到账，将增量交给统一内核。
- S = 0：推进游标并记录 skippedNative，不向 Gauge 转入，也不补给后来者。
- 全系统已释放量不超过截至该时刻的计划累计上界。
- Distributor 不对用户逐人记账；用户由 Gauge 统一领取，不再调用独立 Minter。

nativeCursor 只能通过 Gauge 的检查点路径推进。禁止独立公开调用 Distributor 先拿钱、以后再给 Gauge 补积分，否则会破坏历史持仓归属。

### 2.2 外部奖励 Token

令 a = periodStart，z = periodFinish，F = streamBudget：

```text
vested(t) = floor(F × clamp(t-a, 0, z-a) / (z-a))
delta = vested(toTime) - vestedAccounted
vestedAccounted = vested(toTime)
```

使用累计释放量之差，避免“每秒速率先取整”导致计划余额漂移。

- S > 0：delta 进入统一积分内核。
- S = 0：delta 进入本 Token 的 idleUnallocated，积分不增长。
- 奖励流已结束时，累计 vested 最终精确等于该流的 streamBudget。
- 原生权重变化不会更改外部奖励流。
- 外部资金在注资时已实际到账，时间经过仅把它从未处理预算转为用户分配或空池储备。

### 2.3 共用时间推进

公开 Gauge.checkpoint(maxWeeks) 可以只推进一段历史，但必须让所有已登记奖励一起处理到共同的 toTime。

对于 stake、withdraw、exit、claim、fundReward、addReward 等入口，必须先把共同游标推进到当前交易时间。单次上限不足时，明确回退 NeedsCheckpoint；用户先用公开分批 checkpoint 持久推进，再重试资金操作。

维护调用只更新全局积分，不遍历用户。余额改变前的 S 和 b 在尚未处理区间保持不变，这是延迟补账仍能正确归属奖励的前提。

如果所需 Controller 周历史尚未完成，应返回 NeedsControllerCheckpoint。维护者先公开分批推进 Controller 的单池及全局周记录，再推进 Gauge。不能为继续办理资金业务而悄悄使用旧周权重。

<a id="kernel"></a>

## 3. 统一整数积分内核

### 3.1 全局 accrue(token, delta, S)

只有 S > 0 且 delta 是合法新增预算时调用：

```text
numerator = delta × P + globalRemainder
deltaI = numerator // S
globalRemainder = numerator % S
rewardPerTokenStored += deltaI
totalAllocated += delta
```

其中 // 为向下整数除法。余数保留，使“供给不变时多次补账”与“合并补账”的积分结果一致。

实现可按以下方式避免中间乘积溢出：

```text
q = mulDiv(delta, P, S)
r = mulmod(delta, P, S) + oldGlobalRemainder
deltaI = q + r // S
newGlobalRemainder = r % S
```

旧 globalRemainder 必须来自相同 S 及相同质押人群。不能在余额改变后直接沿用。

### 3.2 个人 settleUser(user, token)

```text
deltaIndex = rewardPerTokenStored - userRewardPerTokenPaid[user][token]
numerator = balanceOf[user] × deltaIndex + userRemainder[user][token]
newWhole = numerator // P
userRemainder[user][token] = numerator % P
rewards[user][token] += newWhole
userRewardPerTokenPaid[user][token] = rewardPerTokenStored
```

同样使用 mulDiv / mulmod 处理乘积。个人余数归原账户保留，不能在领取或全部退出时清零。

用户未操作时，不需要主动给他逐次写 rewards；其尚未结算部分可通过旧 balance × 积分差得出。

### 3.3 质押变化前封存全局余数

在增加、减少任何质押余额之前，对每种奖励执行：

```text
sealedDustScaled[token] += globalRemainder[token]
globalRemainder[token] = 0
```

原因：旧余数属于旧持仓人群，不能把它带到新分母后给新加入者参与分配。

将封存值保留为不可再分配的精度储备，不允许赞助者或管理员提走。原生、外部均使用同样策略。S < P 因而每次封存小于一个奖励最小单位；N 次余额变化后封存总量严格小于 N 个最小单位。不能把这类已明确归类的储备藏进用户负债。

个人余数不封存、不转给其他人；它已经归属于具体用户。

### 3.4 即时可领值

如果能完整模拟全局补账：

```text
pendingWhole = floor(
    (b_u × (simulatedIndex - userPaidIndex) + userRemainder) / P
)
claimable = rewardsStored + pendingWhole
```

查询不得将多个 Token 的数值相加。历史过长、超出查询上限时，应返回 complete=false 和下一游标，而不是把旧存储值冒充最新可领值。

<a id="operations"></a>

## 4. 操作顺序

以下为语义伪代码，不是可直接部署的 Solidity。全部入口使用适当重入保护；任一步失败回退本次全部状态变化。

### 4.1 stake(amount)：首次及追加质押

```text
require amount > 0 and not paused
require totalStaked + amount < P
checkpointAll(now)              // 使用旧 S，把历史奖励全部推进到现在
settleUserForAllTokens(user)    // 使用旧 b，锁定本人的历史收益
sealGlobalRemaindersForAllTokens()
safeTransferFrom(lpToken, user, gauge, amount)
require actualReceived == amount
balanceOf[user] += amount
totalStaked += amount
emit Staked(...)
```

首次进入的旧 b 为零，因此不会获得此前积分差带来的奖励；用户 paidIndex 会更新到当前积分。追加质押时，旧余额先结算，新增数量从此后参与。

### 4.2 withdraw(amount)：部分或全部取回

```text
require amount > 0 and amount <= balanceOf[user]
checkpointAll(now)
settleUserForAllTokens(user)
sealGlobalRemaindersForAllTokens()
balanceOf[user] -= amount
totalStaked -= amount
safeTransfer(lpToken, user, amount)
emit Withdrawn(...)
```

withdraw 不自动执行奖励 Token 转账。部分退出后，剩余 LP 继续参与；全部退出后，rewards、userRemainder、paid 等历史记录保留。

exit() 是取回当前全部 LP 的便利入口，沿用同一结算路径；退出后可单独 claim。零余额用户仍可领取已赚奖励。

### 4.3 claim(token)：领取某一种奖励

```text
require token is registeredReward
checkpointAll(now)
settleUserForAllTokens(user)
amount = rewards[user][token]
rewards[user][token] = 0
paid[user][token] += amount
totalPaid[token] += amount
if amount > 0:
    safeTransfer(token, user, amount)
emit RewardPaid(...)
```

claim 不改变 LP 余额，不封存全局余数。公共结算可能增加其他 Token 的已结算应得值，但不会清空或支付其他 Token。

重复 claim 且无新时间/奖励增量时，支付为零。某个外部 Token 转账失败，本次交易回退，原领奖资格不丢失；另一 Token 可另行领取。

### 4.4 checkpoint 与查询

checkpoint(maxWeeks) 只推进预算与全局积分；用户历史在自己的资金/领取操作中结算。公开维护函数不可代改别人的余额、领取地址或已领记录。

任何查询或操作都不能以“只处理请求的 Token”为由漏结用户变动前其他 Token 的旧账。

<a id="funding"></a>

## 5. 外部奖励注资与续期

新增奖励由协议治理配置 Token 与出资者。addReward 必须先补账到当前时间，新流默认预算为零，积分从零开始；不能把旧 Token 的积分复制过去。

fundReward(token, amount)：

1. 校验已登记外部 Token 及指定出资者，GOV 不走此入口。
2. checkpointAll(now)，处理旧奖励流已经过时间对应的金额。
3. 实际转入新预算，核对到账；totalFunded 只增加这次新到账数量。
4. 取旧流未处理计划余额、空池储备，与新预算合并。
5. 重置流起点为 now、结束为 now + WEEK，vestedAccounted 归零，已累计用户负债不变。

```text
oldUnprocessed = oldStreamBudget - oldVestedAccounted
nextStreamBudget = actualReceived + oldUnprocessed + idleUnallocated
idleUnallocated = 0
```

旧计划中已归用户但未领取的金额包含在 totalAllocated 内，不能再次放进 nextStreamBudget。

普通续资要求实际新到账大于零。零额重新安排只允许旧流已结束且 idleUnallocated > 0；不允许仅凭零额调用反复延长仍在运行的未来计划。

示例：原流未来剩 30，空池储备 10，新到 100，则新流 140。另欠用户未领的 20 必须继续保留，不计入 140。

无计划或已过结束时间，且无新注资时，积分不因空白时间增长。直接转入的未记账奖励资产不自动续期，无任意提取这些余额的入口。

<a id="example"></a>

## 6. 数值演算

### 6.1 一个完整质押过程

使用外部奖励 R：第 0 天存入 840 R，七天均匀释放，即每日 120 R；表内假定无额外小数误差。

| 时刻 / 区间 | 操作后的 A/B 质押 | 本区间 A/B 收益 | 累计 A/B 收益 |
|---|---|---|---|
| 第 0 天 | A 100，B 0 | 尚无时间经过 | 0 / 0 |
| 第 0—1 天 | 第 1 天 B 加入 100 前先补账 | 120 / 0 | 120 / 0 |
| 第 1—2 天 | A 100，B 100 | 60 / 60 | 180 / 60 |
| 第 2 天 | A 追加 100，变为 A 200，B 100 | 先结旧余额 | 180 / 60 |
| 第 2—3 天 | A 200，B 100 | 80 / 40 | 260 / 100 |
| 第 3 天 | A 部分退出 100 | 先结旧余额，之后 A 100，B 100 | 260 / 100 |
| 第 3—4 天 | A 100，B 100 | 60 / 60 | 320 / 160 |
| 第 4 天 | B 全部退出 | B 的 160 保留，可稍后领 | 320 / 160 |
| 第 4—5 天 | A 100，B 0 | 120 / 0 | 440 / 160 |
| 第 5 天 | A 全部退出 | A 的 440 保留 | 440 / 160 |
| 第 5—7 天 | S = 0 | 0 / 0，240 进入未分配储备 | 440 / 160 |

结果：用户累计 600，空池储备 240，总量为 840。退出不清空应得值，后来入场者也不能直接取得这 240。

### 6.2 多 Token 隔离

同时再加入 Token X，七天释放 420 X，采用相同持仓路径：

- A 应累计 220 X，B 应累计 80 X。
- 空池储备 120 X。
- R 可用十八位精度，X 可用六位精度；分别按最小单位验算。
- claim(R) 不能支付 X 或清空 X 的资格。
- 部分退出必须同时结算 R、X 和 GOV，只有一种 LP 持仓，不维护三份 stake 余额。

### 6.3 零额与同一时刻

同一时间 claim 不生成额外经过时间；零额 stake/withdraw 回退，不能用它绕过保护。公开 checkpoint 或 claim 可以补账。不得让额外检查点改变相同持仓人群下的最终分配。

<a id="verification"></a>

## 7. 精度、守恒与严格验证

### 7.1 积分内核的精确守恒式

在同一已处理时间，对某个 Token 定义：

```text
unsettledScaled =
    Σ balanceOf[u] × (globalIndex - userPaidIndex[u])

P × totalAllocated =
    P × (totalPaid + Σ rewardsStored[u])
    + Σ userRemainder[u]
    + unsettledScaled
    + globalRemainder
    + sealedDustScaled
```

该式在参考模型中应严格相等，不能靠随意误差容限通过。用户集只由测试模型遍历，生产合约不遍历所有用户。

它说明“已经分配的预算”全部落在已支付、整单位待领、个人小数、尚未个人结算、全局小数或已封存精度储备中，没有凭空消失或多出。

### 7.2 资金来源与余额

外部奖励还应满足：

```text
totalFunded =
    totalAllocated + idleUnallocated + (streamBudget - vestedAccounted)

actualTokenBalance >= totalFunded - totalPaid
```

直接捐赠可能使实际余额更大，但不增加已承诺资金。计划重排只改变未处理预算和储备的归类，不增加 totalFunded。

对原生 GOV：

- Gauge 的 totalFunded 等于已实际释放并进入该池有效分配的量。
- EmissionDistributor 预算储备 + 已转给全部 Gauge 的量至少等于初始奖励储备，额外捐赠单列。
- 累计释放不超过原生计划的累计时间上界，所有 Gauge 当周预算之和不超过全局周预算。
- LP 实际余额不低于 totalStaked；锁仓 GOV 不低于未取回锁仓本金。

### 7.3 必须实现的验证

- 按第 6 节逐步核对首次、追加、部分退出、全部退出和事后领取。
- 使用独立整数模型验证 accrue/settle/余数封存，不能调用被测合约内部公式作预期答案。
- 固定持仓时，将一个区间拆成多次 checkpoint/claim，最终结果一致。
- 持仓变化时严格验证封存策略及上方守恒式，而不是忽略余数。
- 随机操作序列下持续核验 LP 足额、每 Token 守恒、用户已领不超应得。
- 验证六位与十八位奖励、微额数量、极大合法数量及溢出边界。
- 空池期间不进入用户积分；原生跳过、外部储备的去向分别准确。
- 续资只重排未归用户部分，已领取和已累计负债均不被重用。
- 一种奖励转账失败、重入尝试或直接捐赠，不破坏其他账本。
- 长时间闲置，公开分批推进与持续推进得到相同结果。
- 所有失败操作保持原余额和债权；不允许“失败却先清领奖记录”。
