# 合约职责、核心函数与调用关系

本文件对应 [README](../README.md#contracts)。这里列出待实现接口，不是已发布 ABI。直接部署每个 Gauge，统一使用同一个实现及奖励库；不部署工厂、独立领奖铸币模块或协议费用模块。

## 1. 合约与依赖

| 模块 | 固定依赖 | 保存什么 |
|---|---|---|
| GOVToken | 部署时分配参数 | 固定供应量、余额、授权 |
| VotingEscrow | GOVToken、最长锁期 | 锁仓本金、个人与全局 ve 历史 |
| GaugeController | VotingEscrow、Timelock | Gauge 资格、个人投票、周权重、启停计划 |
| EmissionDistributor | GOVToken、Controller、固定预算/时间表 | 原生储备、每 Gauge 原生游标、释放/跳过量 |
| LiquidityGauge | LP、GOV、Controller、Distributor、Timelock | 一份 LP 余额、按 Token 的奖励流和个人奖励 |
| RewardAccounting 库 | 无外部托管地址 | 统一积分/余数算法，由 Gauge 调用 |
| ProtocolGovernor | VotingEscrow、Timelock | 提案快照、票数、生命周期 |
| TimelockController | 复用固定版本组件 | 排队操作、执行时间与治理角色 |

绑定的 LP、原生 Token、Controller、Distributor 在部署后不可换。V1 不设置代理升级入口。

登记时核对 Gauge 代码及上述绑定、LP 资产支持和 Token 角色冲突，并记录部署版本。一个 LP 只登记一个 Gauge；停奖不删除登记和历史。新增 LP 若已用作奖励 Token 也必须拒绝，反向登记同样检查。

未完成登记和奖励计划激活前，不开放正式质押或外部注资。激活可早于未来发行起点，期间原生增量为零，不妨碍按规则办理已开放业务。

## 2. 核心函数

### 2.1 GOVToken

使用标准 ERC-20 查询、授权和转账。构造时一次性产生固定总量，部署脚本按清单分配并将 208,000 GOV 转入预算库。无外部 mint、改供应量上限或更换发币地址入口。

### 2.2 VotingEscrow

| 入口 | 调用者与作用 |
|---|---|
| createLock(amount,end) | 本人创建一份锁仓，校验整周对齐及最长锁期 |
| increaseAmount(amount) | 本人向未过期锁仓加 GOV |
| extendLock(end) | 本人延长未过期锁仓，不允许缩短 |
| withdraw() | 本人到期后取回，更新检查点；不支付额外锁仓奖励 |
| balanceOf(user) / totalSupply() | 查询当前 ve |
| balanceOfAtTime(user,t) / totalSupplyAtTime(t) | 查询严格过去的历史 ve |
| lockSlope(user) / lockEnd(user) | 向 Controller 提供投票所需数据 |
| checkpoint(maxWeeks) | 公开推进全局历史，不修改别人的本金 |
| clock() / CLOCK_MODE() | 提供与治理一致的 timestamp 时钟 |

ve 不可转让、不可委托。所谓 withdraw 是锁仓本金取回，与 Gauge 中同名 LP 退出区分。

### 2.3 GaugeController

| 入口 | 调用者与作用 |
|---|---|
| registerGauge(gauge,eligibleFrom) | Timelock；部署初始化期间的受限 bootstrap 路径例外 |
| finalizeBootstrap() | 一次性关闭初始登记权限，不能重新开启 |
| voteForGauge(gauge,bps) | 用户配置自己的支持比例，下一周生效 |
| checkpointGauge(gauge,maxWeeks) | 公开按周推进单池权重 |
| checkpointTotal(maxWeeks) | 公开推进全局权重 |
| weightAt(gauge,week) / totalWeightAt(week) | 读取完整历史 bias |
| relativeWeightAt(gauge,week) | 返回明确精度的占比，仅作查询；资金计算用完整比值 |
| scheduleNativeStatus(gauge,enabled,week) | Timelock，只安排未来周，不改历史 |
| userVote(user,gauge) / allocatedBps(user) | 查询个人支持配置及已使用比例 |

Gauge 投票不会创建 Governor 提案，也不允许用户直接设置全池的百分比；它只增加本人的那份票权贡献。

### 2.4 EmissionDistributor

| 入口 | 调用者与作用 |
|---|---|
| activate() | 受限一次性初始化；核验足额储备、固定绑定和未来起点 |
| pullNative(toTime) | 仅已登记 Gauge；按游标和历史权重算金额，读取旧 S 后转账或跳过 |
| budgetAt(gauge,week) | 查询完整 bias 计算出的单周预算 |
| previewNative(gauge,toTime,maxWeeks) | 有界预览区间累计，返回是否完整 |
| nativeCursor(gauge) / released(gauge) | 查询本池推进和已实际转入量 |

pullNative 不接受用户数额、个人地址或任意收款地址；收款者固定为调用 Gauge。它只由 Gauge 公共/内部检查点路径调用。

新增捐款不修改 weeklyBudget 或 emissionEnd。没有向管理员返还任意奖励储备的接口。

### 2.5 LiquidityGauge 与奖励库

| 入口 | 调用者与作用 |
|---|---|
| stake(amount) | 本人质押或追加，同一份 LP 余额 |
| withdraw(amount) | 本人部分或全部退出，先结历史，不自动转奖励 |
| exit() | 取回本人全部 LP，保留已赚奖励 |
| claim(token) | 本人领取一种奖励，支付给本人 |
| claimable(user,token,maxWeeks) | 有界模拟，返回金额、处理时间、complete 标志 |
| checkpoint(maxWeeks) | 公开推进全部奖励到共同时间 |
| addReward(token,distributor) | Timelock；先补账，再登记空流 |
| setRewardDistributor(token,address) | Timelock；不改变既有用户债权 |
| fundReward(token,amount) | 对应外部出资者；旧流先结算再接收新资金 |
| pauseDeposits() | Guardian 或 Timelock，仅暂停新增质押 |
| unpauseDeposits() | Timelock 恢复 |
| balanceOf(user) / totalStaked() | 内部质押余额查询，不是可转让 ERC-20 |
| rewardState(token) / userReward(user,token) | 独立奖励账本查询 |

内部共用步骤：

- _checkpointAll(toTime)：先确定原生/外部的新增量，再逐 Token 调用 accrue。
- _settleUser(user)：对全部已登记 Token 结算旧余额。
- _sealGlobalRemainders()：余额变化前封存旧人群的全局分子余数。
- RewardAccounting.accrue / settle：纯会计内核，不另存一份 LP。
- 支付只更新请求 Token 的 paid/rewards，不清除其他 Token 或用户小数余数。

完整数学与顺序以 [奖励规范](REWARD_ACCOUNTING.md) 为准。

### 2.6 ProtocolGovernor 与 Timelock

Governor 提供 propose、castVote、queue、execute、cancel、state、proposalSnapshot 等接口。历史票权直接来自 ve，使用自定义票权查询覆盖，不额外部署可委托治理代币。

默认门槛和时间见 README。提案在 snapshot 时刻的状态最终确定后才允许读取历史票权。创建门槛用创建前一秒，提案参与权及法定票数用固定 snapshot，不能随投票期间追加锁仓而回写。

Timelock 复用固定版本组件。Governor 是正式 proposer/canceller；执行可开放给任何人，但只能执行已批准、排队且到期的内容。Guardian 无权取消或执行提案。

取消、回退和重试遵守所选组件的明确状态机，不自造“失败也算执行成功”的状态。

## 3. 调用流程

### 用户质押与退出

```text
stake / withdraw / exit
  → 检查余额、暂停和上限
  → Gauge._checkpointAll(now)
      → Distributor.pullNative(toTime)
          → Controller 历史周权重
          → GOV 转入 Gauge（非空池）或记录跳过（空池）
      → 外部各流累计释放差额
      → RewardAccounting.accrue
  → RewardAccounting.settle 全部 Token
  → 封存全局余数
  → 按入口顺序核验转账并变更余额
  → 核对 LP 余额与事件
```

追加和退出都必须先结旧余额；资产转账失败则本次全部回退。stake 在受保护的转入并核验后加余额；withdraw 先扣余额再转出。若 Controller 历史不足，先用其公开分批接口推进，再推进 Gauge，不能在回退后假定进度已保存。

### 领奖

```text
claim(token) → 补全部奖励 → 结本人全部奖励
            → 先更新请求 Token 的领取账
            → 只转该 Token 给本人
```

### Gauge 权重投票

```text
voteForGauge → 读取锁仓 slope/end → 校验 BPS/间隔
             → 撤旧未来贡献、安排新贡献
             → 下一周由历史点计算权重
             → Distributor 在结算区间读取
```

### 协议治理

```text
propose → Pending → Active → Succeeded / Defeated
        → 通过后 queue → 等 Timelock → execute
        → 允许的注册或配置调用
```

这两类投票共享 ve 来源，但不共享票数记录、通过规则或执行路径。

<a id="permissions"></a>

## 4. 权限边界

| 事项 | 权限与限制 |
|---|---|
| 用户本金存取 / 自己领奖 | 本人；不提供任意领取地址重定向 |
| 用户锁仓 / 自己配 Gauge 比例 | 本人；总比例和时间规则检查 |
| 维护历史检查点 | 公开，但不可代改余额或拿走资产 |
| Gauge 登记、外部奖品/出资人配置 | 正式治理经 Timelock 执行 |
| 安排未来原生启停 | Timelock，只影响未来周 |
| 暂停新增 LP | Guardian 或 Timelock |
| 恢复新增 LP | Timelock |
| 原生预算拉取 | 已登记 Gauge 的受限检查点路径 |
| 外部注资 | 已配置出资人 |
| 额外铸币、改历史、任意提现、升级实现 | V1 不提供 |

暂停新增 LP 不暂停已有用户奖励核算与正常退出。奖励转账失败不清掉债权，也不把该 Token 的转账强行放进 withdraw。

## 5. 部署顺序与初始化

1. 固定参数，部署 GOVToken，登记初始分配清单。
2. 部署 VotingEscrow、Timelock 及 Controller。
3. 部署 Distributor，转入完整奖励储备，检查固定起止时间。
4. 部署两个固定绑定的 LiquidityGauge，完成初始登记及外部奖励配置。
5. 部署 Governor，连接 timestamp ve 查询和 Timelock 角色。
6. 初始账户锁仓、投票，使首个发行周已有合法待生效权重。
7. 关闭 bootstrap 登记口，完成管理权交接并撤销部署者临时角色。
8. 核验 proposer/executor/管理员和正常提案路径，在未来发行起点之前激活计划。
9. 输出地址、预算、参数、网络、源码版本和角色清单，启动演示。

bootstrap 只用于部署时初始装配，不能在系统激活后重新开放。若初始化未完成，不开放用户业务。测试脚本负责验证首次投票生效周与发行起点衔接。

## 6. 工程约定

核心逻辑均需要测试：单元、跨模块集成、随机操作与会计不变量。演示 UI 只调用已确定 ABI，不能在前端自己算一个数再让合约相信它。

目标文件可为 GOVToken.sol、VotingEscrow.sol、GaugeController.sol、EmissionDistributor.sol、LiquidityGauge.sol、RewardAccounting.sol、ProtocolGovernor.sol；Timelock 复用依赖组件。当前只有设计文档，函数签名以实施时锁定的 ABI 为准。

