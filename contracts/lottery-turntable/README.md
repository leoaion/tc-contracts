## 转盘模型抽奖合约

### 功能

- 合约owner 新增、删除管理员
- 管理员角色 新建抽奖，设置抽奖活动规则
- 管理员角色 新增奖品，设置中奖概率
- 管理员角色 开启、关闭抽奖
- 用户抽奖
- 根据抽奖id查询抽奖信息
- 根据抽奖id和奖品id查询奖品及中奖信息

### 合约结构

- LotteryData合约：抽奖数据合约，记录新建的抽奖、奖品、概率、中奖信息。所有数据的修改只能通过外部逻辑合约执行。
- LotteryCore合约：可操作抽奖数据合约。可新增admin角色，只有admin角色可执行逻辑合约的抽奖方法。owner角色管理admin角色。
- LotteryCoreWithRule：带有各种规则限制的抽奖逻辑合约，继承自LotteryCore，在此基础上新增了各种规则限制，开发者可以自定义修改。具体规则可以参考合约里的数据结构注释。

数据逻辑结构分离，基本数据结构固定，外层逻辑更新不影响数据合约。

### 合约部署流程

1. 使用 owner 地址作初始化参数，部署 LotteryData 合约
2. 使用 owner 地址和 LotteryData 地址作参数，部署 LotteryCoreWithRule 合约（构造函数将设置msg.sender为admin）
3. 调用 LotteryData.setLotteryCore 方法，参数为 LotteryCoreWithRule 地址，设置可执行数据合约的逻辑合约地址

### LotteryCoreWithRule 逻辑合约功能操作步骤

1. 部署 LotteryCoreWithRule 合约时，已设置 owner 同时为 admin 角色
2. 使用 admin 角色的账户调用 LotteryCoreWithRule.createLottery 方法新建抽奖，参数为 _lotteryName及其他规则参数，返回 lotteryId
3. 使用 admin 角色的账户调用 LotteryCoreWithRule.addLotteryPrize 向 lotteryId 的抽奖新增抽奖奖品，参数为 奖品名称，奖品数量，*中奖概率倒数*
4. 重复步骤3，直到奖品设置完毕
5. 使用 admin 角色的账户调用 LotteryCoreWithRule.startLottery 开启 id 为 lotteryId 的抽奖活动
6. 用户参与抽奖调用 LotteryCoreWithRule.userDraw ，直到抽奖活动出发结束条件（奖品发完 或 admin 角色账户关闭抽奖活动）
7. 查询 LotteryData.getLotteryPrizeInfo 奖品及用户中奖信息

### LotteryData 数据合约功能操作步骤

1. 根据抽奖id查询抽奖活动信息 LotteryData.getLotteryInfo(_lotteryId)
2. 根据抽奖id查询奖品和中奖用户信息 LotteryData.getLotteryPrizeInfo(_lotteryId, prizeIndex)
3. 根据抽奖id查询抽奖活动状态 LotteryData.getLotteryStatus(_lotteryId)
4. 获取lotteries数组长度 LotteryData.getLotteriesLength()
5. 获取抽奖的奖品数组长度 LotteryData.getLotteryPrizesLength()

### LotteryCoreWithRule 合约注意事项

- 新建合约时的输入参数为 owner地址、lotteryData地址
- 新建抽奖活动时，活动的起始时间和结束时间都是时间戳，参数*每天抽奖起始时间*和*每天抽奖结束时间*的输入都是整点数的整数值（如8：00则输入8）
- 每天抽奖起始和结束时间的限制是按照UTC(+8)时区来限制的，其他时区请自行修改参数

### LotteryCoreWithRule 自定义规则

```
struct LotteryRule {     
    uint startTime;         // 抽奖活动起始时间戳
    uint endTime;           // 抽奖活动结束时间戳
    uint daysStartTime;     // 每天抽奖起始时间，0 为不限制
    uint daysEndTime;       // 每天抽奖结束时间，0 为不限制
    uint participateCnt;    // 抽奖活动总次数限制， 0 为不限制
    uint perAddressPartCnt; // 每个地址能参与的抽奖次数，0为不限制
}
```
新建抽奖活动时的参数根据需求自定义抽奖规则

### 关键实现

1. 计算中奖数组

在管理员启动抽奖时，合约方法计算此次抽奖的所有奖品的中奖概率倒数的最小公倍数，然后使用最小公倍数除以每个奖品的中奖概率，得出每个奖品的中奖数组。  
例：  
抽奖 TestLottery 的奖品信息及中奖数组  

| 奖品名称 | 中奖概率 | 概率倒数 | 中奖数组        |
| -------- | -------- | -------- | --------------- |
| 奖品A    | 1%       | 100      | [0]             |
| 奖品B    | 2%       | 50       | [1, 2]          |
| 奖品C    | 5%       | 20       | [3, 4, 5, 6, 7] |

如上：三种奖品的中奖概率倒数的最小公倍数为100，以100除以每个奖品的概率倒数，得到的结果做中奖数组的长度，所有结果的值依次递增。

2. 用户抽奖随机逻辑

用户抽奖时，以抽奖时的`uint(keccak256(拼接字符串 (上一块的blockhash + msg.sender + 全局自增index)))`的uint值做随机数，对上面计算的最小公倍数取模，得到随机值。如果随机值结果在以上奖品的中奖数组里，即表示抽得对应的奖品。  

### 注意事项

hash的产生依赖的是产生随机数的交易时的上一块的hash，迅雷链对外的交易确认是秒级确认，但并不会立即同步对外的交易记录，以此保证上一块交易hash不会被用来作恶。
