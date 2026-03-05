# SCRAM 架构文档

## 概述

SCRAM 是一个故障树分析工具，支持定性分析（最小割集/质蕴含项）和定量分析（概率计算、重要性分析、不确定性分析）。本文档描述从命令行参数到生成报告的完整流程。

## 1. 程序入口 (scram.cc)

### 1.1 主函数流程

```
main()
├── xmlInitParser()                    # 初始化 libxml2
├── ParseArguments(argc, argv, &vm)    # 解析命令行参数
├── 设置日志级别 (verbosity)
└── RunScram(vm)                       # 执行主逻辑
    ├── 捕获各类异常并输出错误信息
    └── 返回状态码 (0=成功, 1=错误)
```

### 1.2 命令行参数解析

**函数**: `ParseArguments()`

支持的参数分类：

| 类别 | 参数 | 说明 |
|------|------|------|
| 分析算法 | `--bdd`, `--zbdd`, `--mocus` | 定性分析算法（互斥） |
| 分析类型 | `--probability`, `--importance`, `--uncertainty`, `--ccf`, `--sil` | 定量分析类型 |
| 计算选项 | `--prime-implicants`, `--rare-event`, `--mcub` | 质蕴含项/近似计算 |
| 阈值参数 | `--limit-order`, `--cut-off` | 割集阶数限制/概率截断 |
| 时间参数 | `--mission-time`, `--time-step` | 任务时间/时间步长 |
| 模拟参数 | `--num-trials`, `--num-quantiles`, `--num-bins`, `--seed` | 蒙特卡洛模拟参数 |
| 输入输出 | `--project`, `--output`, `--no-indent` | 项目文件/输出文件 |
| 验证 | `--validate` | 仅验证输入文件 |

### 1.3 设置构建

**函数**: `ConstructSettings()`

```cpp
scram::core::Settings settings;           // 创建设置对象
settings.algorithm(Algorithm::kBdd);       // 设置算法
settings.prime_implicants(true/false);     // 质蕴含项开关
settings.approximation(Approximation::kRareEvent/kMcub);  // 近似方法
settings.probability_analysis(true/false); // 概率分析开关
settings.importance_analysis(true/false);  // 重要性分析开关
settings.uncertainty_analysis(true/false); // 不确定性分析开关
settings.ccf_analysis(true/false);         // 共因失效分析开关
settings.limit_order(int);                 // 割集阶数限制
settings.cut_off(double);                  // 概率截断值
settings.mission_time(double);             // 任务时间
settings.num_trials(int);                  // 蒙特卡洛试验次数
settings.seed(int);                        // 随机数种子
```

---

## 2. 数据初始化流程

### 2.1 初始化器 (Initializer)

**类**: `scram::mef::Initializer`

```
Initializer(xml_files, settings, allow_extern)
├── CheckFileExistence(xml_files)        # 检查文件存在性
├── CheckDuplicateFiles(xml_files)       # 检查重复文件
└── ProcessInputFiles(xml_files)         # 处理输入文件
    ├── 创建 XML Validator (MEF Schema)
    ├── 遍历每个 XML 文件
    │   ├── ParseDocument()              # 解析 XML 文档
    │   ├── Validate()                   # 验证 Schema
    │   └── ParseModel(xml_element)      # 解析模型元素
    │       ├── ParseInitiatingEvents()  # 解析初始事件
    │       ├── ParseEventTrees()        # 解析事件树
    │       ├── ParseAlignments()        # 解析对齐配置
    │       ├── ParseFaultTrees()        # 解析故障树
    │       ├── ParseBasicEvents()       # 解析基本事件
    │       ├── ParseHouseEvents()       # 解析房屋事件
    │       ├── ParseParameters()        # 解析参数
    │       └── ParseCcfGroups()         # 解析 CCF 组
    └── 解析延迟定义的引用
```

### 2.2 关键数据结构

#### Model (模型)

**文件**: `src/model.h`

```cpp
class Model : public Element, public MultiContainer<...> {
  // 包含的元素容器
  table<InitiatingEvent>()    // 初始事件
  table<EventTree>()          // 事件树
  table<Sequence>()           // 序列
  table<FaultTree>()          // 故障树
  table<BasicEvent>()         // 基本事件
  table<Gate>()               // 逻辑门
  table<HouseEvent>()         // 房屋事件
  table<Parameter>()          // 参数
  table<CcfGroup>()           // CCF 组
  table<Alignment>()          // 对齐配置
  
  MissionTime mission_time_;  // 任务时间
  Context context_;           // 事件树上下文
}
```

#### FaultTree (故障树)

**文件**: `src/fault_tree.h`

```cpp
class FaultTree : public Element, public Container<FaultTree, Gate> {
  std::vector<Gate*> top_events_;     // 顶层事件
  std::vector<Gate*> intermediate_events_; // 中间事件
}
```

#### Gate (逻辑门)

**文件**: `src/event.h`

```cpp
class Gate : public Event {
  Formula::Type type_;                    // 门类型 (AND/OR/NOT/...)
  std::vector<Formula::ArgEvent> args_;   // 输入事件参数
  std::vector<Instruction*> instructions_; // 指令
}
```

#### BasicEvent (基本事件)

```cpp
class BasicEvent : public Event {
  Expression* expression_;  // 概率表达式
}
```

---

## 3. XML 解析与验证

### 3.1 XML 处理流程

**文件**: `src/xml.h`, `src/xml.cc`

```
xml::Document::Parse(file)
├── xmlReadFile()                        # libxml2 解析文件
├── xmlRelaxNGValidateDoc()              # RelaxNG 模式验证
└── 创建 Element 树

xml::Element
├── name()                               # 元素名称
├── attribute(name)                      # 获取属性
├── text()                               # 获取文本内容
├── children()                           # 子元素范围
└── child(name)                          # 查找特定子元素
```

### 3.2 表达式解析

**文件**: `src/expression/`

支持的表达式类型：

| 表达式类型 | 类名 | 说明 |
|-----------|------|------|
| 常数 | `ConstantExpression` | 常量数值 |
| 参数引用 | `Parameter` | 引用模型参数 |
| 指数分布 | `Exponential` | λ * exp(-λt) |
| 测试事件 | `TestEvent` | 事件树测试条件 |
| 外部函数 | `ExternFunction` | 动态库函数调用 |
| 数值运算 | `Add`, `Sub`, `Mul`, `Div` | 四则运算 |
| 随机分布 | `Uniform`, `Normal`, `LogNormal`, `Gamma`, `Beta`, `Histogram` | 概率分布 |

---

## 4. 风险分析流程

### 4.1 RiskAnalysis 类

**文件**: `src/risk_analysis.h`, `src/risk_analysis.cc`

```
RiskAnalysis(model, settings)
└── Analyze()
    ├── 设置随机数种子
    ├── 遍历对齐配置 (Alignment)
    │   └── RunAnalysis(Context)
    │       ├── 应用阶段配置 (Phase)
    │       │   ├── 设置任务时间
    │       │   └── 设置房屋事件状态
    │       ├── 事件树分析 (EventTreeAnalysis)
    │       │   ├── 遍历事件树
    │       │   └── 生成序列门 (Sequence Gate)
    │       └── 故障树分析 (FaultTreeAnalysis)
    │           └── RunAnalysis<Algorithm>(target, result)
    └── 存储结果到 results_
```

### 4.2 定性分析算法

#### 4.2.1 BDD (Binary Decision Diagram)

**文件**: `src/bdd.h`, `src/bdd.cc`

```cpp
class Bdd : public boost::noncopyable {
  // 核心操作
  IntrusivePtr<Vertex> Apply(IntrusivePtr<Vertex> a, IntrusivePtr<Vertex> b, 
                             BinaryOperator op);
  IntrusivePtr<Vertex> And(IntrusivePtr<Vertex> a, IntrusivePtr<Vertex> b);
  IntrusivePtr<Vertex> Or(IntrusivePtr<Vertex> a, IntrusivePtr<Vertex> b);
  IntrusivePtr<Vertex> Not(IntrusivePtr<Vertex> a);
  
  // 从 PDAG 生成 BDD
  void GenerateBdd(const Pdag& graph);
}
```

**流程**: PDAG → BDD 生成 → 最小割集提取

#### 4.2.2 ZBDD (Zero-suppressed BDD)

**文件**: `src/zbdd.h`, `src/zbdd.cc`

```cpp
class Zbdd : public boost::noncopyable {
  // 从 BDD 转换
  void FromBdd(const Bdd& bdd);
  
  // 割集操作
  void MinimalCutSets();           // 计算最小割集
  void LimitOrder(int order);      // 阶数限制
  void CutOff(double prob);        // 概率截断
}
```

#### 4.2.3 MOCUS (Minimal Cut Sets)

**文件**: `src/mocus.h`, `src/mocus.cc`

自上而下算法，基于逻辑门展开：
- AND 门：合并所有输入事件的割集
- OR 门：取所有输入事件割集的并集

### 4.3 PDAG (Preprocessed Directed Acyclic Graph)

**文件**: `src/pdag.h`, `src/pdag.cc`

预处理阶段：
```
Preprocessor::Run()
├── 合并重复事件
├── 处理常数门
├── 传播房屋事件
├── 处理单输入门
├── 处理模块 (Modules)
└── 生成 PDAG
```

### 4.4 定量分析

#### 4.4.1 概率分析

**文件**: `src/probability_analysis.h`, `src/probability_analysis.cc`

```cpp
class ProbabilityAnalysis {
  // 计算方法
  double CalculateProbability(const Product& product);
  double CalculateTotalProbability();
  
  // 近似方法
  RareEventCalculator;     // 稀有事件近似
  McubCalculator;          // MCUB 近似
}
```

#### 4.4.2 重要性分析

**文件**: `src/importance_analysis.h`, `src/importance_analysis.cc`

计算指标：
- Fussel-Vesely 重要性
- Risk Achievement Worth (RAW)
- Risk Reduction Worth (RRW)
- Birnbaum 重要性

#### 4.4.3 不确定性分析

**文件**: `src/uncertainty_analysis.h`, `src/uncertainty_analysis.cc`

蒙特卡洛模拟流程：
```
UncertaintyAnalysis::Analyze()
├── 对每个基本事件采样 (num_trials 次)
│   └── RandomDeviate::Sample()
├── 计算每次试验的顶事件概率
├── 统计分布 (均值、方差、分位数)
└── 生成直方图
```

---

## 5. 结果输出流程

### 5.1 Reporter 类

**文件**: `src/reporter.h`, `src/reporter.cc`

```
Reporter::Report(risk_analysis, output, indent)
├── ReportInformation()
│   ├── ReportSoftwareInformation()      # 软件版本信息
│   ├── ReportCalculatedQuantity()       # 计算量类型
│   ├── ReportModelFeatures()            # 模型统计信息
│   └── ReportPerformance()              # 性能指标
└── ReportResults()
    ├── 遍历所有分析结果
    │   ├── ReportResults(FTA)           # 故障树分析结果
    │   ├── ReportResults(Probability)   # 概率分析结果
    │   ├── ReportResults(Importance)    # 重要性分析结果
    │   └── ReportResults(Uncertainty)   # 不确定性分析结果
    └── ReportResults(ETA)               # 事件树分析结果
```

### 5.2 输出格式

**文件**: `src/xml_stream.h`

XML 结构：
```xml
<report>
  <information>
    <software>...</software>
    <calculated-quantities>...</calculated-quantities>
    <model-features>...</model-features>
    <performance>...</performance>
  </information>
  <results>
    <gates>
      <gate name="...">
        <cut-sets>...</cut-sets>
        <probability>...</probability>
        <importance>...</importance>
        <uncertainty>...</uncertainty>
      </gate>
    </gates>
    <event-trees>...</event-trees>
  </results>
</report>
```

---

## 6. 关键数据流总结

```
┌─────────────────────────────────────────────────────────────────┐
│                         程序入口                                 │
│  main() → ParseArguments() → RunScram()                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      设置构建                                    │
│  ConstructSettings(vm, &settings)                               │
│  ├── 算法选择 (BDD/ZBDD/MOCUS)                                  │
│  ├── 分析类型 (概率/重要性/不确定性/CCF/SIL)                     │
│  └── 计算参数 (阶数限制/截断/模拟次数等)                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      数据初始化                                  │
│  Initializer(xml_files, settings)                               │
│  ├── XML 解析 (xml::Document)                                   │
│  ├── Schema 验证 (xml::Validator)                               │
│  ├── 构建 Model 对象                                            │
│  │   ├── 故障树、逻辑门、事件                                   │
│  │   ├── 表达式、参数                                           │
│  │   └── 事件树、序列                                           │
│  └── 引用解析与验证                                             │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      风险分析                                    │
│  RiskAnalysis(model, settings)                                  │
│  └── Analyze()                                                  │
│      ├── 事件树分析 (EventTreeAnalysis)                         │
│      └── 故障树分析 (FaultTreeAnalysis)                         │
│          ├── 预处理 (Preprocessor) → PDAG                       │
│          ├── 定性分析                                           │
│          │   ├── BDD → 最小割集                                 │
│          │   ├── ZBDD → 最小割集                                │
│          │   └── MOCUS → 最小割集                               │
│          └── 定量分析                                           │
│              ├── 概率分析 (ProbabilityAnalysis)                 │
│              ├── 重要性分析 (ImportanceAnalysis)                │
│              └── 不确定性分析 (UncertaintyAnalysis)             │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      报告生成                                    │
│  Reporter::Report(risk_analysis, output)                        │
│  ├── 软件/模型/性能信息                                          │
│  ├── 故障树分析结果                                             │
│  ├── 概率/重要性/不确定性结果                                   │
│  └── 事件树分析结果                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 错误处理机制

**文件**: `src/error.h`

异常层级：
```
Error (基类)
├── IOError              # 文件/IO 错误
├── LogicError           # 逻辑错误
├── SettingsError        # 设置错误
├── mef::ValidityError   # 模型有效性错误
│   ├── DuplicateElementError
│   ├── UndefinedElement
│   ├── CycleError
│   └── DomainError
└── xml::Error           # XML 相关错误
    ├── ParseError
    ├── XIncludeError
    └── ValidityError
```

异常抛出宏：
```cpp
SCRAM_THROW(ErrorType) << errinfo_value("...") 
                       << boost::errinfo_file_name("...")
                       << boost::errinfo_at_line(42);
```

---

## 8. 扩展机制

### 8.1 表达式扩展

在 `src/expression/` 目录添加新的表达式类型：
1. 继承 `Expression` 类
2. 实现 `value()` 方法
3. 在 Initializer 注册解析函数

### 8.2 外部函数库

通过 `--allow-extern` 启用动态库加载：
```cpp
// XML 定义
<extern-library name="mylib" path="/path/to/lib.so"/>
<extern-function name="custom_func" symbol="custom_func" library="mylib"/>
```

---

## 9. 性能优化

- **BDD 缓存**: 使用 hash 表缓存已计算的 BDD 节点
- **模块化**: 独立分析故障树模块
- **并行计算**: 多线程 Monte Carlo 模拟
- **内存管理**: intrusive_ptr 减少引用计数开销
