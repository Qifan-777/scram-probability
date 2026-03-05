# fta-engine 重构计划

**版本**: 1.0  
**日期**: 2026-03-05  

---

## 1. 重构范围与优先级

### 1.1 重构目标

- **项目重命名**: 将SCRAM重命名为fta-engine，明确项目定位
- **CMake现代化**: 采用现代CMake 3.16+特性，提升跨平台构建体验
- **代码模块化**: 从平铺结构重构为分层架构，提升可维护性
- **日志系统升级**: 引入结构化日志，支持异步输出和分级过滤
- **性能优化**: 多线程并行、内存池、缓存优化
- **CLI现代化**: 结构化命令行参数处理，支持子命令模式

### 1.2 优先级矩阵

| 优先级 | 模块 | 影响 | 风险 |
|--------|------|------|------|
| **P0** | CMake现代化 | 高 | 低 |
| **P0** | 目录结构重构 | 高 | 中 |
| **P1** | 日志系统改进 | 中 | 低 |
| **P1** | CLI参数结构化 | 中 | 低 |
| **P2** | 性能优化 | 高 | 中 |
| **P2** | 大文件拆分 | 中 | 中 |
| **P3** | 测试增强 | 高 | 低 |

---

## 2. 项目重命名

### 2.1 重命名范围

将项目标识从 **SCRAM** 更改为 **fta-engine**（Fault Tree Analysis Engine的缩写）。

### 2.2 重命名映射

| 类型 | 原名 | 新名 |
|------|------|------|
| 项目名称 | SCRAM | fta-engine |
| 可执行文件 | scram | fta-engine |
| CLI程序 | scram-cli | fta-engine-cli |
| 静态库 | libscram.a | libftaengine.a |
| 动态库 | libscram.so | libftaengine.so |
| 命名空间 | `namespace scram` | `namespace ftaengine` |
| 宏前缀 | `SCRAM_*` | `FTAE_*` |
| 版本宏 | `SCRAM_VERSION` | `FTAE_VERSION` |
| 异常宏 | `SCRAM_THROW` | `FTAE_THROW` |
| 日志宏 | `LOG` | `FTAE_LOG` |

### 2.3 CMake配置更新

**根CMakeLists.txt:**
```cmake
project(fta-engine 
    VERSION 0.17.0 
    LANGUAGES CXX
    DESCRIPTION "Fault Tree Analysis Engine"
)

# 选项前缀更新
option(FTAE_BUILD_TESTS "Build test suite" OFF)
option(FTAE_BUILD_GUI "Build GUI frontend" OFF)
option(FTAE_ENABLE_OPENMP "Enable OpenMP parallelization" ON)
option(FTAE_ENABLE_TCMALLOC "Use TCMalloc" OFF)
option(FTAE_ENABLE_JEMALLOC "Use JEMalloc" OFF)
option(FTAE_ENABLE_COVERAGE "Enable coverage reporting" OFF)

# 包含模块
include(FtaeCompilerFlags)
include(FtaeDependencies)
```

**src/CMakeLists.txt:**
```cmake
# 库名称更新
add_library(ftaengine STATIC $<TARGET_OBJECTS:ftaengine_core>)
target_link_libraries(ftaengine PUBLIC ftaengine_core)

# 可执行文件
add_executable(ftaengine-cli cli/main.cpp)
target_link_libraries(ftaengine-cli PRIVATE ftaengine)
set_target_properties(ftaengine-cli PROPERTIES OUTPUT_NAME fta-engine)
```

### 2.4 命名空间更新

**所有源文件:**
```cpp
// 之前
namespace scram { ... }
namespace scram::mef { ... }
namespace scram::core { ... }

// 之后
namespace ftaengine { ... }
namespace ftaengine::model { ... }
namespace ftaengine::analysis { ... }
namespace ftaengine::expression { ... }
namespace ftaengine::io { ... }
```

### 2.5 宏定义更新

**error.h:**
```cpp
namespace ftaengine {

#define FTAE_THROW(err) \
    throw err << ::boost::throw_function(BOOST_CURRENT_FUNCTION) \
              << ::boost::throw_file(FILE_REL_PATH) \
              << ::boost::throw_line(__LINE__)

} // namespace ftaengine
```

**logger.h:**
```cpp
#define FTAE_LOG(level) \
    if (level <= ftaengine::Logger::report_level()) \
        ftaengine::Logger().Get(level)
```

### 2.6 版本文件更新

**src/version.h.in:**
```cpp
#ifndef FTAE_VERSION_H
#define FTAE_VERSION_H

#define FTAE_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define FTAE_VERSION_MINOR @PROJECT_VERSION_MINOR@
#define FTAE_VERSION_PATCH @PROJECT_VERSION_PATCH@
#define FTAE_VERSION "@PROJECT_VERSION@"
#define FTAE_GIT_REVISION "@FTAE_GIT_REVISION@"

#endif
```

### 2.7 文件重命名

```bash
# 主程序文件
mv src/scram.cc src/ftaengine.cc

# 目录结构保持，内容更新
# src/ -> src/ (目录名不变，内部命名空间和宏更新)
```

### 2.8 全局替换命令

```bash
# 替换命名空间
find src -type f \( -name "*.h" -o -name "*.cc" \) \
    -exec sed -i 's/namespace scram/namespace ftaengine/g' {} +

# 替换宏前缀
find src -type f \( -name "*.h" -o -name "*.cc" \) \
    -exec sed -i 's/SCRAM_/FTAE_/g' {} +

# 替换小写宏
find src -type f \( -name "*.h" -o -name "*.cc" \) \
    -exec sed -i 's/scram_/ftae_/g' {} +

# 替换类名（Logger等）
find src -type f \( -name "*.h" -o -name "*.cc" \) \
    -exec sed -i 's/scram::/ftaengine::/g' {} +
```

### 2.9 文档更新

**所有文档中的项目名称更新:**
- README.md
- AGENTS.md
- doc/arch_doc/*.md

**示例更新:**
```markdown
# 之前
SCRAM is a probabilistic risk analysis tool.

# 之后
fta-engine is a fault tree analysis engine.
```

### 2.10 安装路径更新

**CMakePresets.json:**
```json
{
  "cacheVariables": {
    "CMAKE_INSTALL_PREFIX": "${sourceDir}/ftaengine_install"
  }
}
```

---

## 3. 不变范围

以下部分保持稳定：
- XML解析逻辑（`xml.h/xml.cc`）
- 核心BDD/ZBDD算法（保持正确性优先）
- 输入文件格式（Open-PSA MEF标准）
- 公共API接口（仅命名空间变化，接口不变）

---

## 4. CMake现代化

### 2.1 目标架构

```
cmake/
├── modules/
│   ├── FindTcmalloc.cmake
│   ├── FindJeMalloc.cmake
│   └── ScramCompilerFlags.cmake
├── ScramDependencies.cmake
└── ScramInstall.cmake
```

### 2.2 根CMakeLists.txt重构

```cmake
cmake_minimum_required(VERSION 3.16)
project(SCRAM 
    VERSION 0.17.0 
    LANGUAGES CXX
    DESCRIPTION "Probabilistic Risk Analysis Tool"
)

# 现代CMake：选项定义
option(SCRAM_BUILD_TESTS "Build test suite" OFF)
option(SCRAM_BUILD_GUI "Build GUI frontend" OFF)
option(SCRAM_ENABLE_OPENMP "Enable OpenMP parallelization" ON)
option(SCRAM_ENABLE_TCMALLOC "Use TCMalloc" OFF)
option(SCRAM_ENABLE_JEMALLOC "Use JEMalloc" OFF)
option(SCRAM_ENABLE_COVERAGE "Enable coverage reporting" OFF)

# C++标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 包含模块
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")
include(ScramCompilerFlags)
include(ScramDependencies)

# 生成compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 子目录
add_subdirectory(src)
add_subdirectory(share)

if(SCRAM_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if(SCRAM_BUILD_GUI)
    add_subdirectory(gui)
endif()
```

### 2.3 src/CMakeLists.txt重构

```cmake
# 分层源文件收集
set(SCRAM_CORE_SOURCES
    # 基础设施层
    core/logger.cpp
    core/error.cpp
    core/env.cpp
    core/settings.cpp
    
    # 数据模型层
    model/element.cpp
    model/event.cpp
    model/fault_tree.cpp
    model/event_tree.cpp
    model/model.cpp
    model/ccf_group.cpp
    model/alignment.cpp
    model/parameter.cpp
    model/substitution.cpp
    
    # 表达式层
    expression/expression.cpp
    expression/constant.cpp
    expression/numerical.cpp
    expression/exponential.cpp
    expression/conditional.cpp
    expression/random_deviate.cpp
    expression/test_event.cpp
    expression/extern.cpp
    
    # 输入处理层
    io/xml.cpp
    io/initializer.cpp
    io/serialization.cpp
    io/project.cpp
    
    # 分析引擎层
    analysis/pdag.cpp
    analysis/preprocessor.cpp
    analysis/bdd.cpp
    analysis/zbdd.cpp
    analysis/mocus.cpp
    analysis/fault_tree_analysis.cpp
    analysis/probability_analysis.cpp
    analysis/importance_analysis.cpp
    analysis/uncertainty_analysis.cpp
    analysis/event_tree_analysis.cpp
    analysis/risk_analysis.cpp
    analysis/cycle.cpp
    
    # 输出层
    io/reporter.cpp
)

# 创建对象库（加快编译）
add_library(scram_core OBJECT ${SCRAM_CORE_SOURCES})

# 现代CMake：target属性
target_include_directories(scram_core
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    PRIVATE
        ${Boost_INCLUDE_DIRS}
        ${LIBXML2_INCLUDE_DIR}
)

target_compile_features(scram_core PUBLIC cxx_std_17)

# 编译器特定标志
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    target_compile_options(scram_core PRIVATE
        -Wall -Wextra -Wnon-virtual-dtor
        $<$<CONFIG:Debug>:-O0 -g3>
        $<$<CONFIG:Release>:-O3 -DNDEBUG>
    )
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_compile_options(scram_core PRIVATE
        -Wall -Wextra
        $<$<CONFIG:Debug>:-O0 -g>
        $<$<CONFIG:Release>:-O3>
    )
endif()

# OpenMP支持
if(SCRAM_ENABLE_OPENMP)
    find_package(OpenMP REQUIRED)
    target_link_libraries(scram_core PUBLIC OpenMP::OpenMP_CXX)
    target_compile_definitions(scram_core PUBLIC SCRAM_HAS_OPENMP)
endif()

# 链接库
target_link_libraries(scram_core
    PUBLIC
        Boost::boost
        Boost::program_options
        Boost::filesystem
        ${LIBXML2_LIBRARIES}
)

# 主库（静态链接对象库）
add_library(scram STATIC $<TARGET_OBJECTS:scram_core>)
target_link_libraries(scram PUBLIC scram_core)

# CLI可执行文件
add_executable(scram-cli cli/main.cpp)
target_link_libraries(scram-cli PRIVATE scram)
set_target_properties(scram-cli PROPERTIES OUTPUT_NAME scram)

# 安装规则
include(GNUInstallDirs)
install(TARGETS scram scram-cli
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

### 2.4 验证标准

- Windows/MSVC构建通过
- Linux/GCC构建通过
- macOS/Clang构建通过
- vcpkg集成正常
- Ninja生成器支持
- IDE自动补全工作（VS Code/CLion）

---

## 5. 目录结构重构

### 5.1 目标结构

```
src/
├── CMakeLists.txt
│
├── core/                    # 基础设施层
│   ├── logger.hpp
│   ├── logger.cpp
│   ├── error.hpp
│   ├── env.hpp
│   ├── settings.hpp
│   ├── settings.cpp
│   ├── types.hpp
│   └── version.hpp.in
│
├── model/                   # 数据模型层（MEF）
│   ├── element.hpp
│   ├── element.cpp
│   ├── event.hpp
│   ├── event.cpp
│   ├── fault_tree.hpp
│   ├── fault_tree.cpp
│   ├── event_tree.hpp
│   ├── event_tree.cpp
│   ├── model.hpp
│   ├── model.cpp
│   ├── ccf_group.hpp
│   ├── ccf_group.cpp
│   ├── alignment.hpp
│   ├── alignment.cpp
│   ├── parameter.hpp
│   ├── parameter.cpp
│   ├── substitution.hpp
│   ├── substitution.cpp
│   └── cycle.hpp
│
├── expression/              # 表达式层
│   ├── expression.hpp
│   ├── expression.cpp
│   ├── constant.hpp
│   ├── constant.cpp
│   ├── numerical.hpp
│   ├── numerical.cpp
│   ├── exponential.hpp
│   ├── exponential.cpp
│   ├── conditional.hpp
│   ├── conditional.cpp
│   ├── random_deviate.hpp
│   ├── random_deviate.cpp
│   ├── test_event.hpp
│   ├── test_event.cpp
│   ├── extern.hpp
│   ├── extern.cpp
│   └── boolean.hpp
│
├── io/                      # 输入输出层
│   ├── xml.hpp
│   ├── xml.cpp
│   ├── xml_stream.hpp
│   ├── initializer.hpp
│   ├── initializer.cpp
│   ├── project.hpp
│   ├── project.cpp
│   ├── serialization.hpp
│   ├── serialization.cpp
│   ├── reporter.hpp
│   └── reporter.cpp
│
├── analysis/                # 分析引擎层
│   ├── analysis.hpp
│   ├── analysis.cpp
│   ├── pdag.hpp
│   ├── pdag.cpp
│   ├── preprocessor.hpp
│   ├── preprocessor.cpp
│   ├── bdd/
│   │   ├── bdd.hpp
│   │   ├── bdd.cpp
│   │   ├── vertex.hpp
│   │   ├── unique_table.hpp
│   │   └── apply.hpp
│   ├── zbdd/
│   │   ├── zbdd.hpp
│   │   ├── zbdd.cpp
│   │   └── zbdd_cutsets.hpp
│   ├── mocus.hpp
│   ├── mocus.cpp
│   ├── fault_tree_analysis.hpp
│   ├── fault_tree_analysis.cpp
│   ├── probability_analysis.hpp
│   ├── probability_analysis.cpp
│   ├── importance_analysis.hpp
│   ├── importance_analysis.cpp
│   ├── uncertainty_analysis.hpp
│   ├── uncertainty_analysis.cpp
│   ├── event_tree_analysis.hpp
│   ├── event_tree_analysis.cpp
│   ├── risk_analysis.hpp
│   ├── risk_analysis.cpp
│   └── calculator.hpp
│
├── util/                    # 工具库
│   ├── algorithm.hpp
│   ├── scope_guard.hpp
│   ├── linear_map.hpp
│   ├── linear_set.hpp
│   └── float_compare.hpp
│
└── cli/                     # 命令行接口
    ├── main.cpp
    ├── options.hpp
    ├── options.cpp
    ├── commands.hpp
    ├── analyze_cmd.hpp
    ├── validate_cmd.hpp
    └── version_cmd.hpp
```

### 3.2 迁移步骤

**步骤1：创建目录结构**
```bash
mkdir -p src/{core,model,expression,io,analysis/{bdd,zbdd},util,cli}
```

**步骤2：文件移动**
按照上述映射移动文件，每次移动后立即：
1. 更新`#include`路径
2. 更新`CMakeLists.txt`
3. 构建验证
4. Git提交

**步骤3：命名空间调整**
```cpp
// 之前
namespace scram::mef { ... }
namespace scram::core { ... }

// 之后
namespace scram::model { ... }
namespace scram::analysis { ... }
namespace scram::expression { ... }
namespace scram::io { ... }
namespace scram::util { ... }
```

**步骤4：头文件保护更新**
```cpp
#ifndef SCRAM_ANALYSIS_BDD_BDD_HPP
#define SCRAM_ANALYSIS_BDD_BDD_HPP
...
#endif
```

---

## 6. 大文件拆分

### 4.1 拆分目标

| 文件 | 当前行数 | 拆分后 |
|------|---------|--------|
| `initializer.cc` | 1842 | 6个文件 |
| `preprocessor.cc` | 2411 | 5个文件 |
| `pdag.cc` | 1057 | 3个文件 |
| `reporter.cc` | 582 | 2个文件 |

### 4.2 Initializer拆分

```
io/initializer/
├── initializer.hpp
├── initializer.cpp
├── file_checker.hpp
├── file_checker.cpp
├── model_parser.hpp
├── model_parser.cpp
├── event_parser.hpp
├── event_parser.cpp
├── expression_parser.hpp
├── expression_parser.cpp
├── tree_parser.hpp
└── tree_parser.cpp
```

### 4.3 Preprocessor拆分

```
analysis/preprocessor/
├── preprocessor.hpp
├── preprocessor.cpp
├── rules/
│   ├── rule_base.hpp
│   ├── constant_propagation.hpp
│   ├── constant_propagation.cpp
│   ├── duplicate_removal.hpp
│   ├── duplicate_removal.cpp
│   ├── module_detection.hpp
│   ├── module_detection.cpp
│   ├── coalesce_gates.hpp
│   └── coalesce_gates.cpp
└── transformations/
    ├── gate_transform.hpp
    └── graph_transform.hpp
```

---

## 7. 日志系统改进

### 5.1 当前问题

```cpp
// 当前：简单宏
LOG(INFO) << "Processing " << filename;
// 输出: INFO: Processing input.xml
```

### 5.2 目标设计

```cpp
// 改进后：结构化日志
SCRAM_LOG_INFO("file_processed")
    << scram::log::field("filename", filename)
    << scram::log::field("duration_ms", elapsed)
    << scram::log::field("bytes_read", size);

// 输出:
// 2026-03-05 14:32:15.342 [INFO] [io::initializer] file_processed - 
//   filename="input.xml" duration_ms=145 bytes_read=2048
```

### 5.3 实现方案

**文件结构**:
```
core/logger/
├── logger.hpp
├── logger.cpp
├── level.hpp
├── sink.hpp
├── sinks/
│   ├── console_sink.hpp
│   ├── console_sink.cpp
│   ├── file_sink.hpp
│   ├── file_sink.cpp
│   └── json_sink.hpp
├── record.hpp
├── field.hpp
└── async_logger.hpp
```

**核心接口**:
```cpp
// core/logger/level.hpp
enum class Level {
    Trace = 0, Debug, Info, Warn, Error, Fatal
};

// core/logger/record.hpp
struct Record {
    std::chrono::system_clock::time_point timestamp;
    Level level;
    std::string component;
    std::string event;
    std::source_location location;
    std::vector<std::pair<std::string, std::string>> fields;
    std::string message;
};

// core/logger/logger.hpp
class Logger {
public:
    static Logger& instance();
    void add_sink(std::unique_ptr<Sink> sink);
    void set_level(Level min_level);
    void set_async(bool async);
    
    template<typename... Args>
    void log(Level level, std::string_view component, 
             std::string_view event,
             std::source_location loc = std::source_location::current());
private:
    std::vector<std::unique_ptr<Sink>> sinks_;
    Level min_level_ = Level::Info;
    bool async_ = false;
    std::mutex mutex_;
};

// 便捷宏
#define SCRAM_LOG_TRACE(event) \
    scram::core::LogBuilder(scram::core::Level::Trace, #event, SCRAM_CURRENT_COMPONENT)
#define SCRAM_LOG_DEBUG(event) \
    scram::core::LogBuilder(scram::core::Level::Debug, #event, SCRAM_CURRENT_COMPONENT)
```

**使用示例**:
```cpp
#define SCRAM_CURRENT_COMPONENT "analysis::bdd"

SCRAM_LOG_INFO("bdd_construction_started")
    << scram::core::field("vertex_count", pdag.vertex_count())
    << scram::core::field("edge_count", pdag.edge_count());
```

### 5.4 向后兼容

```cpp
// core/logger/compat.hpp
#define LOG(level) \
    scram::core::LegacyLogWrapper(scram::LogLevel::level)

#define TIMER(level, name) \
    scram::core::ScopedTimer timer(name, scram::LogLevel::level)
```

---

## 8. CLI参数结构化

### 6.1 目标设计

```bash
scram --version
scram --help

scram analyze [options] <files...>
  --algorithm {bdd,zbdd,mocus}
  --probability
  --importance
  --uncertainty
  --cut-off 1e-20
  --limit-order 5
  --output report.xml
  
scram validate <files...>
  --strict
  
scram convert <input> <output>
  --from xml
  --to json
  
scram config
  --set key=value
  --get key
  --list
```

### 6.2 实现方案

**文件结构**:
```
cli/
├── main.cpp
├── app.hpp
├── app.cpp
├── options/
│   ├── global_options.hpp
│   ├── analyze_options.hpp
│   ├── validate_options.hpp
│   └── convert_options.hpp
├── commands/
│   ├── command.hpp
│   ├── analyze_cmd.hpp
│   ├── analyze_cmd.cpp
│   ├── validate_cmd.hpp
│   ├── validate_cmd.cpp
│   ├── convert_cmd.hpp
│   └── version_cmd.hpp
└── formatters/
    ├── console_formatter.hpp
    └── json_formatter.hpp
```

**命令接口**:
```cpp
// cli/commands/command.hpp
class Command {
public:
    virtual ~Command() = default;
    virtual std::string name() const = 0;
    virtual std::string description() const = 0;
    virtual int execute(const std::vector<std::string>& args) = 0;
};

// cli/commands/analyze_cmd.hpp
class AnalyzeCommand : public Command {
public:
    std::string name() const override { return "analyze"; }
    std::string description() const override { 
        return "Perform fault tree analysis"; 
    }
    int execute(const std::vector<std::string>& args) override;

private:
    struct Options {
        core::Algorithm algorithm = core::Algorithm::kBdd;
        bool probability = false;
        bool importance = false;
        bool uncertainty = false;
        bool ccf = false;
        bool sil = false;
        std::optional<int> limit_order;
        std::optional<double> cut_off;
        std::optional<double> mission_time;
        std::vector<std::string> input_files;
        std::optional<std::string> output_file;
        std::optional<std::string> project_file;
        bool no_indent = false;
    };
    
    Options parse_args(const std::vector<std::string>& args);
    core::Settings build_settings(const Options& opts);
};
```

**主程序**:
```cpp
// cli/main.cpp
int main(int argc, char* argv[]) {
    try {
        CLIApp app;
        app.register_command(std::make_unique<AnalyzeCommand>());
        app.register_command(std::make_unique<ValidateCommand>());
        app.register_command(std::make_unique<ConvertCommand>());
        app.register_command(std::make_unique<VersionCommand>());
        return app.run(argc, argv);
    } catch (const std::exception& e) {
        std::cerr << "Fatal error: " << e.what() << "\n";
        return 1;
    }
}
```

### 6.3 参数验证

```cpp
// cli/options/validators.hpp
class OptionValidator {
public:
    static void validate_files(const std::vector<std::string>& files);
    static void validate_probability(double p);
    static void validate_positive(int value, const char* name);
    static void validate_algorithm_combination(
        core::Algorithm algo, bool prime_implicants);
};
```

---

## 9. 性能优化

### 7.1 OpenMP并行化

**不确定性分析并行**:
```cpp
// analysis/uncertainty_analysis.cpp
void UncertaintyAnalysis::run_trials_parallel(int num_trials) {
    std::vector<double> results(num_trials);
    
    #pragma omp parallel for schedule(dynamic)
    for (int i = 0; i < num_trials; ++i) {
        auto local_rng = create_thread_rng(seed_, i);
        results[i] = run_single_trial(local_rng);
    }
    
    compute_statistics(results);
}
```

**多故障树并行**:
```cpp
// analysis/risk_analysis.cpp
void RiskAnalysis::analyze_fault_trees_parallel() {
    const auto& fault_trees = model_->fault_trees();
    std::vector<std::future<void>> futures;
    
    for (const auto& ft : fault_trees) {
        for (const auto* gate : ft.top_events()) {
            futures.push_back(
                std::async(std::launch::async, [this, gate]() {
                    analyze_gate(*gate);
                })
            );
        }
    }
    
    for (auto& f : futures) {
        f.wait();
    }
}
```

### 7.2 内存优化

**对象池**（BDD顶点分配）:
```cpp
// analysis/bdd/vertex_pool.hpp
template<typename T>
class ObjectPool {
public:
    ObjectPool(size_t initial_size = 1024);
    
    template<typename... Args>
    T* acquire(Args&&... args);
    
    void release(T* obj);
    
private:
    std::vector<std::unique_ptr<T>> blocks_;
    std::vector<T*> free_list_;
    std::mutex mutex_;
};
```

**内存预分配**:
```cpp
unique_table_.reserve(expected_size);
and_table_.reserve(expected_size / 2);
```

### 7.3 缓存优化

**数据局部性改进**:
```cpp
// 之前
struct Product {
    std::vector<int> event_indices;
};

// 改进
class ProductContainer {
    std::vector<int> data_;
    std::vector<int> offsets_;
};
```

### 7.4 编译优化

**PGO（Profile Guided Optimization）**:
```bash
# 1. 编译插桩版本
cmake -DCMAKE_CXX_FLAGS="-fprofile-generate" ...
ninja

# 2. 运行典型工作负载
./bin/scram analyze --bdd large_model.xml

# 3. 重新编译优化版本
cmake -DCMAKE_CXX_FLAGS="-fprofile-use" ...
ninja
```

**LTO**:
```cmake
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
```

---

## 10. 其他改进

### 8.1 配置管理

**分层配置**:
```
~/.scram/config.toml
./.scram.toml
--config override.toml
```

**配置格式**:
```toml
[analysis]
default_algorithm = "bdd"
default_probability = true
max_order = 10

[logging]
level = "info"
format = "json"
output = "scram.log"

[performance]
threads = 8
memory_limit = "4G"
```

### 8.2 测试基础设施

```
tests/
├── CMakeLists.txt
├── unit/
│   ├── core/
│   ├── model/
│   ├── expression/
│   ├── io/
│   └── analysis/
├── integration/
│   ├── end_to_end/
│   └── regression/
├── fixtures/
│   └── sample_models/
└── benchmarks/
    ├── bdd_construction.cpp
    └── uncertainty_analysis.cpp
```

### 8.3 持续集成

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        compiler: [gcc, clang, msvc]
    
    steps:
      - uses: actions/checkout@v3
      - name: Configure
        run: cmake -B build -S . -DBUILD_TESTING=ON
      - name: Build
        run: cmake --build build --parallel
      - name: Test
        run: ctest --test-dir build --output-on-failure
```

---

## 11. 实施阶段

### 阶段1：基础重构
- 创建新的CMake模块结构
- 重写CMakeLists.txt文件
- 创建新目录结构
- 移动文件到新位置
- 更新所有#include路径
- 更新命名空间
- 构建验证

### 阶段2：核心改进
- 设计新日志API
- 实现核心日志组件
- 添加向后兼容层
- 设计子命令架构
- 实现命令基类
- 迁移analyze命令
- 添加validate命令
- 参数验证改进

### 阶段3：性能优化
- OpenMP不确定性分析
- 多故障树并行
- 性能基准测试
- BDD顶点对象池
- 容器预分配
- 缓存优化

### 阶段4：完善
- Initializer拆分
- Preprocessor拆分
- Reporter拆分
- 测试基础设施
- CI/CD配置
- 文档更新

---

## 12. 验证标准

### 10.1 构建验证
- Windows/MSVC构建通过
- Linux/GCC构建通过
- macOS/Clang构建通过
- vcpkg集成正常
- Ninja生成器支持
- IDE自动补全工作

### 10.2 功能验证
- 单元测试通过
- 集成测试通过
- 示例分析正常运行
- 回归测试通过

### 10.3 性能验证
- 构建时间减少20%
- 代码重复率降低30%
- 圈复杂度平均降低20%
- 不确定性分析加速10x（16核）
- 测试覆盖率>80%

---

## 13. 风险缓解

| 风险 | 可能性 | 影响 | 缓解措施 |
|------|--------|------|----------|
| 重构引入bug | 高 | 高 | 建立回归测试套件，小步提交 |
| 性能退化 | 中 | 高 | 基准测试对比，性能门限检查 |
| 构建失败 | 中 | 中 | 多平台CI，每晚构建 |
| 破坏API | 低 | 中 | 保持公共API稳定，内部重构 |

---

## 14. 迁移指南

### 12.1 开发者迁移

**分支策略**:
```bash
git checkout -b refactor/cmake-modernization
git checkout -b refactor/directory-structure
...

git checkout develop
git merge refactor/cmake-modernization
```

**代码审查重点**:
- include路径是否正确
- 命名空间是否一致
- 宏定义是否更新
- 外部API是否稳定

### 12.2 用户迁移

**向后兼容**:
- CLI参数保持不变
- 配置文件格式提供迁移工具
- 输出文件格式不变

**废弃警告**:
```cpp
[[deprecated("Use --algorithm bdd instead")]]
void parse_bdd_flag() { ... }
```

---

**文档结束**
