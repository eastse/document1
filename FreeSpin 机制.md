# FreeSpin 机制详解

## 目录
1. [FreeSpin 的触发机制](#1-freespin-的触发机制)
2. [FreeSpin 与调控系统的关系](#2-freespin-与调控系统的关系)
3. [FreeSpin 的概率计算](#3-freespin-的概率计算)

---

## 1. FreeSpin 的触发机制

### 1.1 FreeSpin 的数据结构

FreeSpin 在系统中有多个层级的定义:

#### Proto 定义 (serverProto.proto)
```protobuf
// FreeSpin 请求标识
message FreeSpin {
  bool free = 1;  // 标识是否为免费旋转
}

// FreeSpin 数据列表
message freeSpinList {
  repeated freeData free = 1;
}

// FreeSpin 详细数据
message freeData {
  string id = 1;           // FreeSpin ID
  int32 remain = 2;        // 剩余次数
  int32 type = 3;          // FreeSpin类型
  double bet = 4;          // 下注金额
  double maxWin = 5;       // 最大赢额
  double nowWin = 6;       // 当前赢额
  freeBonus bonusData = 7; // 奖励数据
  google.protobuf.Timestamp expired = 8;  // 过期时间
  bool isAllowStop = 9;    // 是否允许停止
}

// FreeSpin 类型枚举
enum freeType {
  NormalFree = 0;      // 普通免费旋转
  GiftCodeFree = 1;    // 礼品码免费旋转
  TadaFree = 2;        // Tada免费旋转
  FreeBonue = 3;       // 免费奖励
}
```

#### 游戏特定结构 (以G77为例)
```protobuf
// SingleFree - 单次免费游戏
message SingleFree {
  repeated SinglePlate FreePlateQueue = 1;  // 免费游戏盘面队列
  double FreeWin = 2;                       // 免费游戏赢额
  int32 FreePlus = 3;                       // 额外增加的免费次数
  int32 AwardType = 4;                      // 奖励类型
  double NowMoney = 5;                      // 当前金额
  int32 FreeNowRound = 6;                   // 当前免费轮次
  int32 FreeTotalRound = 7;                 // 总免费轮次
  bool MaxFlag = 8;                         // 最大倍数标识
}

// FreeData - 免费游戏数据
message FreeData {
  repeated SingleFree FreeQueue = 1;  // 免费游戏队列
  double FreeTotalWin = 2;            // 免费游戏总赢额
}
```

### 1.2 触发条件

FreeSpin 在**普通模式**中的触发机制:

1. **Scatter 符号触发** (以G77, G103, G134为例)
   - G77: 3个以上Scatter → 8次免费旋转, 4个Scatter → 12次, 5个以上 → 20次
   - G103: 4个Scatter → 8次免费旋转, 每多1个Scatter额外+2次
   - G134: 3个Scatter → 10次免费旋转, 每多1个Scatter额外+2次

2. **免费游戏中可再次触发**
   - 在免费游戏期间,收集到特定数量的Scatter可以增加额外的免费次数
   - 记录在 `FreePlus` 字段中

3. **判断逻辑位置**
   ```
   普通模式 (Normal Mode)
     ↓
   SpinReq.IsFree = false
     ↓
   游戏逻辑判断 (各游戏内部实现)
     ↓ (满足Scatter条件)
   触发 FreeSpin
     ↓
   FreeRemainRound > 0
   ```

### 1.3 状态管理

FreeSpin 的状态通过以下字段管理:

- **FreeRemainRound**: 剩余免费轮次
- **FreeNowRound**: 当前是第几轮免费游戏
- **FreeTotalRound**: 总共有多少轮免费游戏
- **FreePlus**: 在免费游戏中额外获得的次数

这些状态在 `transformer` 中用于生成游戏历史报表:
```go
// transformer_roundqueue.go
func calculateRoundEarnedFreeSpins(isMainGame bool, roundIndex int, 
    freeTotalRounds int, thisRoundMap map[string]interface{}) int {
    
    if isMainGame {
        // 主游戏中触发
        if freeRemainRounds, ok := thisRoundMap["FreeRemainRound"].(float64); ok {
            return int(freeRemainRounds)
        }
    } else {
        // 免费游戏中额外获得
        thisFreeTotalRounds := thisRoundMap["FreeTotalRound"].(float64)
        earnedFreeSpins = int(thisFreeTotalRounds) - int(lastFreeTotalRounds)
    }
    return earnedFreeSpins
}
```

---

## 2. FreeSpin 与调控系统的关系

### 2.1 调控系统架构

FreeSpin 在调控系统中的位置:

```
SpinReq
  ↓
[1] CalculateGameModeCategoryAndGameBetMode()
  ↓
游戏模式: Normal/Card/BuyFeature/Extra
  ↓
[2] CalculateAdjustRequest()
  ├── BaseRequest (Cost, BaseBet, GameCode, GameMode)
  ├── UserAdjust (UserRtpStatus, UserRtp, UserAdjustControl)
  ├── MerchantAdjust (AdjustID, WeightID, MerchantRtp)
  ├── MerchantPool (水池数据)
  └── AdjustConfig (RTP配置, 权重配置)
  ↓
[3] StartAdjust()
  ├── CalculateRtpType() → rtpType (HIGH/MEDIUM/LOW)
  └── CalculateWeightResult()
       ↓
[4] weightrtp.CalculateWeightResult()
  ├── WeightNone (无权重)
  ├── WeightSd1/Sd2/Sd3 (标准差权重)
  ├── WeightStreamer (直播模式)
  └── WeightFeature (特色模式) ← FreeSpin 相关
       ↓
[5] CalculateWeightFeatureResult()
  └── calculateTargetFeatureIndex() ← 按序表演FreeSpin
```

### 2.2 普通模式中的调控流程

在**普通模式 (Normal Mode)** 下:

```go
// gamemodecalculator.go
func calculateGameModeCategory(spinReq *serverProto.SpinReq) serverconst.GameModeCategory {
    if spinReq.ItemID != 0 || spinReq.ItemIndex != 0 {
        return serverconst.Card
    }
    if spinReq.MallBet != 0 {
        return serverconst.BuyFeature
    }
    if spinReq.Extra != 0 {
        return serverconst.Extra
    }
    // 普通模式
    return serverconst.Normal
}
```

普通模式的调控特点:
- **下注成本**: `spinReq.Bet` (原始下注金额)
- **下注模式**: `NormalBet`
- **RTP控制**: 通过调控ID和权重ID决定

### 2.3 FreeSpin 在调控中的角色

#### 2.3.1 特色列表 (Feature List)

特色列表定义了特殊表演的序列,FreeSpin 可以是其中一种表演:

```go
// weight_feature.go
func CalculateWeightFeatureResult(rtpType string, request *entity.AdjustRequest) (*entity.WeightResult, error) {
    // 1. 获取特色列表
    featureList, ok := app.Context.FeatureManager.GetFeatureList(request.GameCode)
    if !ok {
        return calculateWeightResult(rtpType, request.AdjustConfig, 0)
    }
    
    // 2. 计算此次表演指标 (依序表演)
    targetFeatureIndex := calculateTargetFeatureIndex(featureList, 
                            request.UserAdjust.UserAdjustControl)
    
    // 3. 返回对应的权重结果
    return calculateWeightResult(rtpType, request.AdjustConfig, targetFeatureIndex)
}
```

#### 2.3.2 依序表演机制

```go
func calculateTargetFeatureIndex(featureList []int, 
                                 userAdjustControl *model.AdjustUserControlModel) int {
    if featureList == nil {
        return 0
    }
    
    // 使用当前下注次数对特色列表长度取模,实现循环表演
    targetIndex := int(userAdjustControl.CurrentBetCount) % len(featureList)
    
    return featureList[targetIndex]
}
```

**关键点**:
- FreeSpin 触发由 `performanceIndex` 决定
- `performanceIndex` 通过特色列表按序分配
- 用户的 `CurrentBetCount` 决定当前该表演哪个特色
- 这确保了FreeSpin的触发是**可预测和可控**的

#### 2.3.3 调控ID与权重ID

系统通过两个关键参数控制FreeSpin:

1. **AdjustID** (调控ID)
   - `AdjustNormalFix`: 自然机率 (随机)
   - `Adjust1/2/3`: 不同的调控策略

2. **WeightID** (权重ID)
   - `WeightNone`: 无权重
   - `WeightSd1/2/3`: 标准差权重1/2/3
   - `WeightStreamer`: 直播模式
   - `WeightFeature`: **特色模式 (FreeSpin相关)**

```go
// adjustmodule.go
func CalculateRtpType(request *entity.AdjustRequest) string {
    // 处理卡片模式
    if request.GameMode == serverconst.Card {
        return adjustrtp.ChoseRtpTypeWithCard(request)
    }
    
    // 处理自然机率
    if serverconst.AdjustID(request.MerchantAdjust.AdjustID) == serverconst.AdjustNormalFix {
        return adjustrtp.ChoseRtpTypeRandom(request)
    }
    
    // 处理调控模式
    return adjustrtp.ChoseRtpTypeWithAdjustTypeAndMission(request)
}
```

### 2.4 用户调控数据

FreeSpin 触发会影响用户的调控数据:

```go
type AdjustUserControlModel struct {
    CurrentBetCount            int64   // 当前下注次数 ← 用于特色表演
    CurrentFeatureBoardNumber  int64   // 特色触发的局数
    // ... 其他统计数据
}
```

每次spin后更新:
```go
func UpdateUserAdjustControl(userAdjustControl *model.AdjustUserControlModel, 
                             bet, win float64) {
    userAdjustControl.CurrentBetCount++        // 累计下注次数
    userAdjustControl.TotalBetAmount += bet    // 累计下注金额
    userAdjustControl.TotalWinAmount += win    // 累计赢得金额
}
```

---

## 3. FreeSpin 的概率计算

### 3.1 性能指标 (Performance Index)

FreeSpin 的触发概率由 **performanceIndex** 决定,该指标关联到:

1. **权重表 (Weight Performance List)**
   - 二维数组结构: `WeightPerformanceList[][]`
   - 每个元素包含: `{performanceIndex, weight}`

2. **赔率表 (Weight Odds List)**
   - 根据RTP类型 (HIGH/MEDIUM/LOW) 有不同的赔率映射
   - 结构: `RtpType2WeightOddsListMap[rtpType][]`

### 3.2 权重计算流程

```go
// weight_feature.go
func calculateWeightResult(rtpType string, adjustConfig *entity.AdjustConfig, 
                          performanceIndex int) (*entity.WeightResult, error) {
    result := &entity.WeightResult{}
    
    // 1. 获取当前RTP类型对应的赔率列表
    weightOddsList := adjustConfig.PerformanceCtrConfig.RtpType2WeightOddsListMap[rtpType]
    
    // 2. 在权重表中查找performanceIndex
    findFlag := false
    for oddsArrayIndex, performanceWeightPairList := range 
        adjustConfig.PerformanceCtrConfig.WeightPerformanceList {
        
        for performanceArrayIndex, performanceWeightPair := range performanceWeightPairList {
            if performanceWeightPair.PerformanceIndex == performanceIndex {
                result.PerformanceResultArrayIndex = performanceArrayIndex
                result.OddsArrayIndex = oddsArrayIndex
                result.PerformanceIndex = performanceIndex
                findFlag = true
                break
            }
        }
    }
    
    if !findFlag {
        return nil, gserror.NewAppError(gserror.ErrInvalidPerformanceIndex)
    }
    
    // 3. 获取对应的赔率
    result.Odds = weightOddsList[result.OddsArrayIndex].Odds
    
    return result, nil
}
```

### 3.3 RTP 配置示例

以游戏77为例 (`gameresources/normal/game/77/rtpConfig.json`):

```json
{
  "rtpCtrConfig": {
    "currentRtp": 0.96,
    "rtpType2RtpMap": {
      "HIGH": 0.98,
      "MEDIUM": 0.96,
      "LOW": 0.94
    }
  },
  "performanceCtrConfig": {
    "rtpType2WeightOddsListMap": {
      "HIGH": [
        {"odds": 0.0, "weight": 1000},
        {"odds": 2.5, "weight": 50},
        {"odds": 10.0, "weight": 20},
        {"odds": 50.0, "weight": 5},
        {"odds": 100.0, "weight": 1}   // FreeSpin 高倍
      ],
      "MEDIUM": [...],
      "LOW": [...]
    },
    "weightPerformanceList": [
      [
        {"performanceIndex": 0, "weight": 800},     // 不中奖
        {"performanceIndex": 100, "weight": 150},   // 小奖
        {"performanceIndex": 500, "weight": 40},    // 中奖
        {"performanceIndex": 1000, "weight": 9},    // 大奖
        {"performanceIndex": 5000, "weight": 1}     // FreeSpin触发
      ],
      // ... 更多权重组合
    ]
  }
}
```

### 3.4 概率计算公式

对于特定的 `performanceIndex`:

**权重概率** = `weight / Σ(all weights in same group)`

例如,如果 FreeSpin 对应 `performanceIndex = 5000`:
- 权重: 1
- 组内总权重: 800 + 150 + 40 + 9 + 1 = 1000
- **FreeSpin触发概率**: `1/1000 = 0.1%`

但在 **WeightFeature 模式**下,概率变为:
- **按序触发**: `1 / len(featureList)`
- 例如 featureList = [0, 100, 500, 1000, 5000]
- 每5局必定触发一次FreeSpin (第5局)
- **实际概率**: `1/5 = 20%` (确定性触发)

### 3.5 RTP 类型选择

系统根据以下因素选择RTP类型 (影响FreeSpin赔率):

1. **用户RTP状态**
   - 用户当前的输赢比
   - 用户的累计下注和赢得金额

2. **商户水池**
   - 商户的资金池状态
   - 动态调整RTP以平衡商户利润

3. **调控策略**
   - 自然模式: 随机选择
   - 调控模式: 根据任务和池子状态选择

```go
func CalculateRtpType(request *entity.AdjustRequest) string {
    // 自然模式: 随机
    if request.MerchantAdjust.AdjustID == serverconst.AdjustNormalFix {
        return adjustrtp.ChoseRtpTypeRandom(request)
    }
    
    // 调控模式: 根据策略选择
    return adjustrtp.ChoseRtpTypeWithAdjustTypeAndMission(request)
}
```

### 3.6 不同权重模式的FreeSpin概率

| 权重模式 | FreeSpin触发机制 | 概率特性 |
|---------|----------------|---------|
| WeightNone | 原始权重随机 | 纯随机,概率稳定 |
| WeightSd1/2/3 | 标准差调控 | 根据历史波动调整 |
| WeightStreamer | 直播模式 | 强制特色触发 |
| **WeightFeature** | **依序表演** | **确定性触发** |

---

## 4. 总结

### 4.1 普通模式中的 FreeSpin 特点

1. **触发方式**
   - 游戏内部逻辑判断 (如Scatter符号)
   - 特色列表按序触发 (WeightFeature模式)

2. **与调控的关系**
   - 受 `AdjustID` 和 `WeightID` 控制
   - 在 WeightFeature 模式下可实现确定性触发
   - 通过 `performanceIndex` 关联到具体的游戏结果

3. **概率计算**
   - 自然模式: 权重随机,概率 = `weight / Σweights`
   - 特色模式: 按序触发,概率 = `1 / len(featureList)`
   - RTP类型影响FreeSpin的赔率而非触发概率

### 4.2 关键数据流

```
用户下注 (Bet)
  ↓
普通模式判断 (Normal Mode)
  ↓
调控请求 (AdjustRequest)
  ├── CurrentBetCount (决定特色序列位置)
  ├── AdjustID & WeightID (决定调控策略)
  └── AdjustConfig (RTP和权重配置)
  ↓
计算 performanceIndex
  ↓
查找 WeightResult
  ├── Odds (赔率)
  └── PerformanceIndex (对应游戏结果)
  ↓
游戏逻辑执行
  ↓
FreeSpin 触发 (如果 performanceIndex 对应 FreeSpin)
  ├── FreeRemainRound
  ├── FreeTotalRound
  └── FreePlus
```

### 4.3 配置示例位置

- RTP配置: `gameresources/normal/game/{gameCode}/rtpConfig.json`
- 特色列表: `FeatureManager.GetFeatureList(gameCode)`
- Proto定义: `protojili/g0/serverProto/serverProto.proto`
- 游戏特定: `protojili/g{gameCode}/{game}Proto/{game}Proto.proto`

---

## 附录: 相关代码文件

| 文件路径 | 功能说明 |
|---------|---------|
| `service/reqservice/spinservice/gamemodecalculator.go` | 游戏模式判断 |
| `service/reqservice/spinservice/adjustmodule.go` | 调控模块主入口 |
| `service/reqservice/spinservice/weightrtp/weight_flow.go` | 权重流程分发 |
| `service/reqservice/spinservice/weightrtp/weight_feature.go` | **FreeSpin特色计算** |
| `internal/entity/adjust.go` | 调控数据结构 |
| `internal/entity/adjust_config.go` | RTP配置结构 |
| `protojili/g0/serverProto/serverProto.proto` | 通用FreeSpin定义 |
| `protojili/g77/bkProto/bkProto.proto` | G77游戏FreeSpin定义 |

