# cmake命令
* https://learn.microsoft.com/zh-cn/vcpkg/get_started/get-started?pivots=shell-powershell，vcpkg的官方说明
* 一定要使用vcpkg add port libxml2 boost，在当前项目中配置依赖项
* cmake --preset=default
* 进入build目录后，cmake -G "Ninja" ..

# 故障排查
* 在使用编译出来的exe时，有的时候会出现exe无法启动的情况，而且没有任何报错，这个时候可以尝试使用administrator权限的命令行打开exe文件，这个时候就会出现缺少哪些dll的弹窗了