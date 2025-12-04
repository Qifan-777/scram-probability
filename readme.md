# 构建工具链说明
## CMake
* 版本:3.31.2
* 下载地址：https://cmake.org/download/

## Ninja
* 版本:1.12.1
* 官网：https://ninja-build.org/
* 下载地址：https://github.com/ninja-build/ninja/releases

## Clang
* 版本：18.1.8
* 官网：https://clang.llvm.org/
* 下载地址：https://github.com/llvm/llvm-project/releases/tag/llvmorg-18.1.8

## VCPkg（依赖管理）
* 官网：https://vcpkg.link/
* 官方使用说明：https://learn.microsoft.com/zh-cn/vcpkg/get_started/get-started?pivots=shell-powershell

# 工程配置说明
* VCPkg配置文件：vcpkg.json vcpkg-configuration.json
* Cmake项目配置文件：CMakePresets.json CMakeUserPresets.json


# 构建说明
* https://learn.microsoft.com/zh-cn/vcpkg/get_started/get-started?pivots=shell-powershell，vcpkg的官方说明
* 一定要使用vcpkg add port libxml2 boost，在当前项目中配置依赖项
* cmake --preset=release,配置release版本的cmake
* cmake --preset=debug, 配置debug版本的cmake
* 进入build目录后，ninja -j8
* windows相关的构建配置依赖需要参考doc目录下的build-instruction-windows

# 调试说明
* 需要依赖的插件为
- CodeLLDB
- C/C++ microsoft

# 故障排查
* 在使用编译出来的exe时，有的时候会出现exe无法启动的情况，而且没有任何报错，这个时候可以尝试使用administrator权限的命令行打开exe文件，这个时候就会出现缺少哪些dll的弹窗了
* share文件夹下，input.rng等文件包含了对xml的schema校验规则，在比较的时候，可以对其中的规则进行删减

# 使用说明
* cut-off 限制最小概率
* limit-order 限制最小割集的阶数
* 可以参考scram文件中的参数解析函数，使用不同的参数进行报告的生成
```
.\bin\scram.exe --bdd --probability --importance --uncertainty --cut-off 1e-20 C:\Users\hu_an\Desktop\1.xml --output C:\Users\hu_an\Desktop\1report.xml
.\bin\scram.exe --bdd --probability --importance --uncertainty --cut-off 1e-15 --limit-order 3 D:\smart\scram\fta_testing_data\available_tree\P08.xml
输入文件路径不能有中文
```