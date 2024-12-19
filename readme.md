# cmake命令
* https://learn.microsoft.com/zh-cn/vcpkg/get_started/get-started?pivots=shell-powershell，vcpkg的官方说明
* 一定要使用vcpkg add port libxml2 boost，在当前项目中配置依赖项
* cmake --preset=default
* 进入build目录后，cmake -G "Ninja" ..